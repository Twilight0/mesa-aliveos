# Fermi (`nvc0`) `GF100_M2MF` `XY_OUT_OF_BOUNDS` & VP4 Teardown Hang on `mpv --hwdec=auto`

Date: 2026-11-22
Target GPU: NVIDIA GeForce GT 555M (`GF106M`, `NVC3`, `0000:01:00.0`, Fermi / `nvc0`)
Snapshots: `/var/log/nouveau-faults/fault-20261120-211751.txt`, `/var/log/nouveau-faults/fault-20261122-115435.txt`

## Summary

Playing H.264 video in `mpv` (`VO: [gpu-next]`) with `--hwdec=auto` (`vaapi`) triggered two separate bugs on Fermi (`nvc0`):

1. **`GF100_M2MF` (`class 9039`) `DATA_ERROR 00000023 [XY_OUT_OF_BOUNDS]`** during `mpv`'s `hwdec_vaapi` startup probe (`try_format_upload`), where a 16x16 `NV12` surface was allocated with `surf->templat.interlaced = false` (`memory_attribute` / `prime_desc` path in `vlVaCreateSurfaces2`), backed by an interlaced 2-layer `nouveau_vp3_video_buffer` (`surf->buffer->interlaced = true`, luma slice `16x8`, chroma slice `8x4`), and uploaded with `vlVaPutImage` -> `vlVaUploadImage` using `surf->templat.interlaced` (`false`) instead of `surf->buffer->interlaced` (`true`), transferring 16 rows into an 8-row texture slice.
2. **PFIFO Runlist 0 `CTXSW_TIMEOUT` (`SCHED_ERROR 0d`) & `failed to idle channel 9`** during `nouveau_vp3_decoder_destroy()`:
   - `nouveau_vp3_decoder_destroy()` called `nouveau_object_del(&dec->bsp)`, `nouveau_object_del(&dec->vp)`, and `nouveau_object_del(&dec->ppp)` (`NVIF_IOCTL_V0_DEL`) **before** calling `nouveau_object_del(dec->channel)` (`DRM_NOUVEAU_CHANNEL_FREE`).
   - In the kernel (`nvkm_uchan_object_fini_1`), `NVIF_IOCTL_V0_DEL` does not idle the channel; instead it preempts the channel, zeroes out the engine context pointer (`0x0270` / `0x0250` / `0x0260`) in `chan->inst`, re-enables the channel on the runlist, and powers off the Falcon engine (`nvkm_falcon_fini()` writes `0` to `base + 0x048`, disabling `FIFO | CHSW`).
   - Immediately afterwards, `DRM_NOUVEAU_CHANNEL_FREE` (`nouveau_abi16_chan_fini`) calls `nouveau_channel_idle(chan)`, which unconditionally pushes a new `nvc0_fence_emit32` (`NV906F_SEMAPHORED` with `RELEASE_WFI = EN`) onto `chan` and rings `chan`'s doorbell (`FIRE_RING`).
   - Because `chan`'s RAMFC still records `MSPPP` (`0x086000`) as its active engine, PFIFO initiates a hardware context switch (`CHSW`) onto `MSPPP` whose `0x086048` (`FIFO | CHSW`) was already set to `0` and whose context pointer in `chan->inst[0x0260]` was already cleared to `0`, hanging PFIFO in hardware context switch (`SCHED_ERROR 0d`).
3. **Additional VA-API / `nvc0_video` synchronization & pushbuf bugs**:
   - `nvc0_create_decoder()` (`src/gallium/drivers/nouveau/nvc0/nvc0_video.c:162`) passed `&nvc0->base` (the 3D context on `screen->channel`) as the `context` argument to `nouveau_pushbuf_create()` for the video decoder channel's pushbuf (`dec->pushbuf[i]`), causing every `PUSH_KICK` on the video decoder channel to invoke `nvc0_default_kick_notify(&nvc0->base)` and emit un-flushed 3D fences onto the 3D channel while marking the 3D fence list as flushed.
   - `nouveau_vp3_decoder` did not implement `fence_wait` / `destroy_fence` and `nvc0_decoder_end_frame()` did not populate `*picture->out_fence`, causing `vaSyncSurface()` (`_vlVaSyncSurface`) on decoded surfaces to return `VA_STATUS_SUCCESS` immediately without waiting for the video decoder channel, and preventing `vlVaEndPicture()` from retaining a reference to `context->decoder` in `surf->codec`.
   - `vlVaHandlePictureParameterBufferH264()` (`src/gallium/frontends/va/picture_h264.c:123`) destroyed `context->decoder` whenever `context->templat.max_references != context->desc.h264.num_ref_frames` instead of only when `max_references < MIN2(num_ref_frames, 16)`.

## Root-Cause Fixes Applied

1. **`src/gallium/frontends/va/image.c` & `src/gallium/frontends/va/surface.c`**:
   - Sync `surface->templat.interlaced = surface->buffer->interlaced;` in `vlVaHandleSurfaceAllocate()`.
   - Check `surf->buffer ? surf->buffer->interlaced : surf->templat.interlaced` in `vlVaVideoSurfaceSize()` and `vlVaGetImage()`.
   - Clamp `width` and `height` to `tex->width0` and `tex->height0` in `vlVaUploadImage()`.
2. **`src/gallium/drivers/nouveau/nvc0/nvc0_transfer.c`**:
   - Add a defensive bounds clamp in `nvc0_m2mf_transfer_rect()` so `GF100_M2MF` (`class 9039`) can never be programmed with out-of-bounds `nblocksx` / `nblocksy` coordinates.
3. **`src/gallium/winsys/nouveau/drm/nouveau.c`, `src/gallium/drivers/nouveau/nouveau_vp3_video.c`, & `src/gallium/drivers/nouveau/nvc0/nvc0_screen.c`**:
   - Point subchannel `obj->parent` at `&drm->client` in `nouveau_object_subchan_new()` so subchannel engine objects can be deleted after their parent channel has been freed.
   - In `nouveau_vp3_decoder_destroy()`, wait for all in-flight decoder BOs (`ref_bo`, `inter_bo[0..1]`, `bsp_bo[0..QDEPTH-1]`) with `nouveau_bo_wait()`, then destroy `dec->pushbuf` and delete `dec->channel` (`DRM_NOUVEAU_CHANNEL_FREE`) **before** deleting `dec->bsp`, `dec->vp`, and `dec->ppp` (`NVIF_IOCTL_V0_DEL`).
   - Do the same in `firmware_present()` (`nouveau_object_del(&channel)` before `nouveau_object_del(&bsp)`) and `nvc0_screen_destroy()` (wait on `screen->fence.bo` and free `screen->base.channel` before deleting `eng3d..nvsw`).
   - Implement `nouveau_vp3_decoder_fence_wait()` and `nouveau_vp3_decoder_destroy_fence()` on `nouveau_vp3_decoder` and populate `*picture->out_fence` in `nvc0_decoder_end_frame()`.
4. **`src/gallium/drivers/nouveau/nvc0/nvc0_video.c`, `src/gallium/drivers/nouveau/nouveau_screen.c`, & `src/gallium/drivers/nouveau/nouveau_fence.c`**:
   - Pass `NULL` for `context` when creating `dec->pushbuf[i]` in `nvc0_create_decoder()`.
   - In `nouveau_pushbuf_cb()`, only call `_nouveau_fence_update(p->screen, true)` (with `p->screen->fence.lock` held) when `push->channel == p->screen->channel`.
   - In `_nouveau_fence_update()`, promote `NOUVEAU_FENCE_STATE_EMITTED` fences to `NOUVEAU_FENCE_STATE_FLUSHED` when `flushed == true` even if `fence_list->sequence_ack == sequence`.
   - Remove the experimental deferred `PUSH_KICK` in `nvc0_vbo.c` and fence BO recycling cache in `nouveau_fence.c`.
5. **`src/gallium/frontends/va/picture_h264.c`**:
   - Only destroy and recreate `context->decoder` if `context->templat.max_references < MIN2(context->desc.h264.num_ref_frames, 16)`.
