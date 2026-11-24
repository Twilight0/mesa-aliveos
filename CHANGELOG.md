# Changelog

All notable changes to **mesa-aliveos** will be documented in this file.

---

## [26.3.1-3] - 2026-11-24

### Nouveau Codegen `gl_FragCoord` Linear Interpolation Fix (`0007`)
- **Force Linear Interpolation for `TGSI_SEMANTIC_POSITION` (`nv50_ir_from_nir.cpp`)**:
  - Updated `Converter::setInterpolate()` to always set `var->linear = 1` for `TGSI_SEMANTIC_POSITION` (`VARYING_SLOT_POS` / `gl_FragCoord`), even when NIR marks it `INTERP_MODE_SMOOTH` in linked fragment shaders.
  - Prevents `OP_PINTERP` (`pinterp mul`) from multiplying screen-space `SV_POSITION.z` (`a[0x78]`) and `SV_POSITION.w` (`a[0x7c]`) by `w_clip`, fixing `gl_FragCoord.z` corruption in Dolphin (`FastDepthCalc`, e.g., *Metal Gear Solid: The Twin Snakes* cargo dock perspective fog).

---

## [26.3.1-2] - 2026-11-22

### Chromium VA-API & Direct Progressive `NV12` Output (`0006`)
- **Direct Progressive `NV12` from VP4 `MSPPP` (`nouveau_vp3_video.c`, `nvc0_video_ppp.c`)**:
  - Enabled `PIPE_VIDEO_CAP_SUPPORTS_PROGRESSIVE` on `nvc0` (`chipset >= 0xc0`) and allocated progressive `PIPE_TEXTURE_2D` (`array_size = 1`, `PIPE_BIND_SHARED`) `NV12` surfaces when `!templat->interlaced`.
  - Cleared `MSPPP` field-split bit `0x10` in `low700` (`nvc0_decoder_setup_ppp()`) for progressive targets so `MSPPP` writes full progressive `NV12` planes directly with zero VPP copy, zero extra VRAM, and zero extra `PGRAPH` draws.
  - Verified byte-for-byte identical `YUV420p` output against software decode at `64x64`, `640x360`, `854x480`, `1280x720`, `1920x1080`, and `720x480` interlaced H.264, plus `52.69 dB` PSNR on `1280x720` MPEG-2 and 15x rapid teardown stress testing with zero kernel errors.
- **GBM / DMABUF `NV12` Modifier Support (`nvc0_miptree.c`)**:
  - Returned `tile_flags = 0xfe` for `util_format_is_yuv(format)` in `nvc0_choose_tiled_storage_type()` so `nvc0_is_dmabuf_modifier_supported()` and `gbm_bo_import()` accept block-linear `DRM_FORMAT_NV12` DMABUFs.
  - Enables smooth hardware-accelerated `VaapiVideoDecoder` playback in Chromium / Google Chrome (YouTube H.264) and direct `vaapi[nv12]` EGL DMABUF interop in `mpv`.

---

## [26.3.1-1] - 2026-11-22

### Nouveau VA-API / VP4 Hardware Decode & Teardown Stability
- **Channel & Subchannel Object Teardown Ordering**:
  - Set `obj->parent = &drm->client` in `nouveau_object_subchan_new()` (`winsys/nouveau/drm/nouveau.c`) so engine objects outlive their parent channel cleanly.
  - Waited on active decoder BOs (`ref_bo`, `inter_bo`, `bsp_bo`) and freed `dec->pushbuf` and `dec->channel` (`DRM_NOUVEAU_CHANNEL_FREE`) before deleting VP4 engine objects (`bsp`, `vp`, `ppp`) in `nouveau_vp3_decoder_destroy()` and `firmware_present()`, eliminating PFIFO `CTXSW_TIMEOUT` (`SCHED_ERROR 0d`) hangs on Fermi (`GF106M` / `NVC3`).
  - Applied the same idle-and-channel-first teardown sequence to `nvc0_screen_destroy()` before releasing `eng3d..nvsw`.
- **Decoder Synchronization & Pushbuf Separation**:
  - Implemented `nouveau_vp3_decoder_fence_wait()` and `nouveau_vp3_decoder_destroy_fence()`, and exported `*picture->out_fence` in `nvc0_decoder_end_frame()` so `vaSyncSurface()` waits properly for video decode completion.
  - Passed `NULL` as the kick-notify context in `nvc0_create_decoder()` so decoder channel pushbuf flushes do not corrupt the 3D screen fence sequence.
  - Guarded `_nouveau_fence_update()` in `nouveau_pushbuf_cb()` to run only when `push->channel == p->screen->channel`, and promoted `NOUVEAU_FENCE_STATE_EMITTED` fences to `FLUSHED` even when `sequence_ack == sequence`.
- **Transfer Bounds & VA Frontend Interlaced / H.264 Handling**:
  - Added defensive bounds clamping in `nvc0_m2mf_transfer_rect()` to prevent `GF100_M2MF` `XY_OUT_OF_BOUNDS` hardware faults.
  - Synced `surface->templat.interlaced = surface->buffer->interlaced` in `vlVaHandleSurfaceAllocate()`, checked `surf->buffer->interlaced` in `vlVaVideoSurfaceSize()` and `vlVaGetImage()`, and clamped `vlVaUploadImage()` transfers to texture bounds (`tex->width0` / `tex->height0`).
  - Updated `vlVaHandlePictureParameterBufferH264()` to recreate `context->decoder` only when `context->templat.max_references < MIN2(num_ref_frames, 16)`.

### Video Layer (VL) Compositor YUV420p Chroma Fix
- **Plane Ordering & V-Plane Graphics Shader Sampling (`vl_compositor.c`)**:
  - Indexed destination surfaces through `vl_video_buffer_plane_order(dst->buffer_format)` in `vl_compositor_yuv_deint_full()` and `vl_compositor_convert_rgb_to_yuv()`.
  - Bound `sampler_views[1] = sampler_views[2]` in `set_yuv_layer()` when rendering `VL_COMPOSITOR_PLANE_V` on the graphics shader path (`fs_yuv.bob.uv` / `fs_yuv.weave.uv`), fixing duplicated U chroma in `yuv420p` VPP outputs (verified in `mpv` and `ffmpeg`).

### Nouveau NVC0 Compiler & OpenGL Capabilities
- **Register Allocator Spill Retries (`nv50_ir_ra.cpp`)**:
  - Increased `RegAlloc::execFunc()` spill retry limit from `3` to `8` for high-register-pressure compute and fragment shaders.
- **GLSL Feature Level (`nvc0_screen.c`)**:
  - Raised `caps->glsl_feature_level` and `caps->glsl_feature_level_compatibility` from `430` to `460` on `nvc0` (verified with OpenGL 4.6 / GLSL 4.50 and 4.60 feature tests).
- **Verified Pre-Existing Code Audit**:
  - Audited pre-existing `separateFragData` modifications against OpenGL MRT/broadcast compliance and reverted them to upstream semantics (`gl_FragColor` broadcast and explicit `layout(location = N)` MRT outputs verified).

### Packaging & Distribution
- **Arch Linux Base Patches & CachyOS-Style Mesa 26.3.1 Packaging (`mesa-aliveos`)**:
  - Carried Arch Linux `extra/mesa 1:26.3.1-2`'s upstream patches (`0001-panfrost-Fix-types-of-panfrost_vdrm-functions.patch` and `0002-Revert-mesa-don-t-advertise-APIs-that-are-not-built.patch` for Mesa #16934) as the base patch set (`0001`–`0002`) before applying the AliveOS patches (`0003`–`0005`).
  - Packaged upstream Mesa `26.3.1` with clean `git format-patch` patches, `options=(!lto !debug)`, full x86_64 Gallium drivers (`r300,r600,radeonsi,nouveau,virgl,svga,llvmpipe,softpipe,iris,crocus,i915,zink`), VA-API enabled, and `-Dvideo-codecs=all`.
  - Built as a drop-in replacement for Arch `mesa` that coexists cleanly with Arch's official `vulkan-*` and `opencl-mesa` packages.
  - Set `VERSION` to `26.3.1-aliveos.1` for automatic Chromium GPU shader cache invalidation on upgrades.
  - Added GitHub Actions build and release workflow (`.github/workflows/build.yml`).
