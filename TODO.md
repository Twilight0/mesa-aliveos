# mesa-nouveau TODO

Target hardware: NVIDIA GeForce GT 555M (`GF106M`, `NVC3`, Fermi / `nvc0`, VP4).

## 1. Pre-Existing Functional Changes Since Upstream (Tested)

- [x] **MRT / `separateFragData` handling** — **Reverted to upstream (bug found during testing)**
  - Files: `src/gallium/drivers/nouveau/codegen/nv50_ir_from_nir.cpp`, `src/gallium/drivers/nouveau/nvc0/nvc0_program.c`
  - Finding: The pre-existing patch inverted the semantics of `separateFragData` and `fp->hdr[0] |= 0x4000` (`SPHV3_T1_MRT_ENABLE`), breaking legacy `gl_FragColor` broadcast to multiple render targets (`RT1` got `(0,0,0,255)` in Test `1d`) and allowing single-output `layout(location = 0) out` shaders to broadcast. Upstream `separateFragData` is a misnomer for `color0_writes_all_cbufs` (`MRT_ENABLE = 0` broadcasts, `MRT_ENABLE = 1` enables separate MRTs). Reverted both files to upstream; all 6 MRT/broadcast tests (`1a`–`1f`) now pass.
- [x] **Register allocator spill retry increase (`3 -> 8`)**
  - File: `src/gallium/drivers/nouveau/codegen/nv50_ir_ra.cpp`
  - Verified: 72-live-scalar compute shader (`max_err = 3.8e-6`) and 96-live-scalar fragment shader both compiled and matched CPU reference outputs with zero GPU faults.
- [x] **GLSL feature level bump (`430 -> 460`)**
  - File: `src/gallium/drivers/nouveau/nvc0/nvc0_screen.c`
  - Verified: `glxinfo` and EGL Core 4.6 context report OpenGL `4.6` / GLSL `4.60` on `NVC3` (all GL 4.4–4.6 required extensions are exposed by `nvc0`). Tested `#version 450 core` (`ARB_enhanced_layouts` `component` + `offset`, `dFdxFine`/`dFdyFine`, `gl_CullDistance`) and `#version 460 core` (`gl_BaseVertex`/`gl_BaseInstance`/`gl_DrawID`, `allInvocations`/`allInvocationsEqual`/`anyInvocation`, `atomicCounterAdd`) — all passed with exact readbacks and zero GPU faults.

## 2. Investigate & Fix `gl_FragCoord.z` Corruption in Complex Shaders (Dolphin PR #14993)

Reference: https://github.com/dolphin-emu/dolphin/pull/14993 (closed because root cause is in Mesa/Nouveau, not Dolphin).

- **Symptom**: With `FastDepthCalc` enabled (deriving fog/ztexture depth `zCoord` from hardware `gl_FragCoord.z` / `rawpos.z`), full game shaders on Fermi (`NVC3`) read a wrong `gl_FragCoord.z`, saturating perspective fog across scenes:
  - *Metal Gear Solid: The Twin Snakes* — cargo dock renders as a solid dark-green wash (HUD/minimap unaffected)
  - *Super Smash Bros. Melee* — menus wash out, some characters/stages wrong
  - *Super Mario Sunshine* — Mario turns monochrome in goop
- **Key clue from PR #14993**: Minimal isolated GL repros (fog math, `gl_FragCoord.z` under `ARB_clip_control`/reversed depth, `early_fragment_tests`, dynamic sampler indexing) passed, whereas full game shaders failed with a cold shader cache.
- **Mesa Investigation Plan**:
  - [ ] Capture the failing fragment shader NIR + `nv50_ir` / emitted Fermi SASS (e.g. via `MESA_SHADER_CAPTURE_PATH` or `NV50_PROG_DEBUG` on a debug build, or apitrace/RenderDoc capture of one failing draw call) and compare `gl_FragCoord.z` (`SV_POSITION.z` / `IPA` / `INTERP`) lowering against a working shader.
  - [ ] Audit `nv50_ir_from_nir.cpp`, `nv50_ir_lowering_nvc0.cpp`, `nv50_ir_peephole.cpp`, and `nv50_ir_ra.cpp` for:
    - `FRAG_COORD` / `SV_POSITION` interpolation (`IPA` / `0x78` slot, `fragCoord` register allocation or spilling across heavy register pressure / multiple RA retries!).
    - `nvc0_program.c` (`nvc0_fp_gen_header`: `fp->hdr[5]` `imap.sysval` bit for `SV_POSITION.z` / `0x78` when `gl_FragCoord.z` is read, `early_fragment_tests`, `depth_layout`, or `centroid`/sample shading interactions).
    - `nvc0_state_validate.c` / `nvc0_shader_state.c` (`NVC0_3D_DEPTH_RANGE` / `CLIP_CONTROL` / `VIEWPORT_TRANSFORM` / `FRAG_COORD` state dirtying across draws).
  - [ ] Build a minimal standalone Mesa reproducer once the failing NIR/SASS pattern is identified, implement the fix in `nvc0` / `nv50_ir`, and verify with `FastDepthCalc = True` on cold shader cache.

## 3. Chromium / VA-API DMABUF Export Experiments (Test on Separate Git Branches)

Currently, `mpv` and `ffmpeg` VA-API hardware decoding work cleanly with accurate colours and zero `nouveau` faults, while Chromium (`VaapiVideoDecoder`) falls back to CPU decode because `vlVaExportSurfaceHandle()` rejects interlaced `nouveau_vp3` `NV12` surfaces (`VA_STATUS_ERROR_INVALID_SURFACE`).

Both approaches below will be evaluated on dedicated git branches:

### Branch A: VPP Copy (`vaapi-export-vpp-nv12`) — Recommended Baseline
- [ ] Add a lazily allocated, cached progressive `PIPE_FORMAT_NV12` `vl_video_buffer` (`export_buffer`) to `vlVaSurface` (`src/gallium/frontends/va/`).
- [ ] In `vlVaExportSurfaceHandle()`, when `surf->buffer->interlaced` is true:
  - Allocate `surf->export_buffer` via `vl_video_buffer_create()` with `interlaced = false` if not already allocated.
  - Run `vlVaPostProc()` (`vl_compositor_yuv_deint_full` with `VL_COMPOSITOR_WEAVE`, 2 GPU draws: `Y` + `UV`) from `surf->buffer` to `surf->export_buffer` when dirty.
  - Export `surf->export_buffer`'s progressive `DRM_FORMAT_NV12` DMABUF.
- [ ] In `vlVaEndPicture()`, if `surf->export_buffer` already exists (for callers like Chromium that cache an exported DMABUF handle across frames for the same `VASurfaceID`), trigger the 2-draw weave update immediately; otherwise mark `surf->export_dirty = true`.
- [ ] Free `surf->export_buffer` in `vlVaDestroySurface()`.
- [ ] Test with `google-chrome-stable` (`chrome://media-internals` -> `VaapiVideoDecoder` on H.264 / `intro_transition.mp4` and H.264 YouTube) and `mpv --hwdec=auto` (verify direct `vaapi[nv12]` without `scale_vaapi=format=yuv420p`), plus `journalctl -k`.
- Trade-offs: ~15–50 MB extra VRAM for exported decode surfaces, ~1–3% DDR3 bandwidth (~185–370 MB/s at 1080p 30–60 fps), ~0.5–1.5 ms/frame on `PGRAPH`; zero Falcon microcode risk.

### Branch B: Direct Progressive `NV12` from `MSPPP` (`vaapi-msppp-progressive-nv12`) — Experimental
- [ ] Investigate whether VP4 `MSPPP` (`0x90b3`) microcode (`src/gallium/drivers/nouveau/nvc0/nvc0_video_ppp.c`, methods `0x700` / `0x738`) supports writing a non-field-separated progressive `NV12` surface directly instead of splitting top/bottom fields across `mt->base.address` and `mt->base.address + mt->total_size/2`.
- [ ] Verify whether `MSPPP`'s tiled output (`tile_mode = 0x10`, `memtype = 0xfe`, macroblock stride) can be imported directly by EGL / `libplacebo` / Chromium via `DRM_FORMAT_NV12` DMABUF modifiers without scanout/modifier mismatches.
- [ ] Compare stability, VRAM usage, GPU/CPU load, and colour/field correctness against Branch A (note high risk of `MSPPP` Falcon/PFIFO hangs if probing undocumented registers).

## 4. Additional Verification & Packaging

- [ ] Test MPEG-1/2 and VC-1 hardware decoding clips through VA-API to confirm VP4 teardown and fence sync beyond H.264.
- [ ] Check `src/gallium/frontends/va/decode.c` (`handleVASliceDataBufferType`) for packed multi-slice H.264 buffers without start codes between slices.
- [ ] Once branches are tested and finalized, package/install the verified `mesa-nouveau` build system-wide so browsers and desktop apps use it without `LD_LIBRARY_PATH` / `--no-sandbox`.
