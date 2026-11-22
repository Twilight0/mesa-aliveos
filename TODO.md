# mesa-aliveos TODO

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

## 3. Chromium / VA-API DMABUF Export Experiments (Completed — Branch B Shipped in `26.3.1-2`)

Both approaches were implemented and verified on dedicated git branches in `src-dev/`:

### Branch A: VPP Copy (`vaapi-export-vpp-nv12`) — Verified & Kept for Reference (`f8f2d2d6`)
- [x] Implemented lazy progressive `PIPE_FORMAT_NV12` `export_buffer` + `vlVaUpdateExportBuffer()` (`vlVaPostProc` weave) + `nvc0` YUV DMABUF modifier support.
- [x] Verified in `mpv`, `test_va_export`, `test_gbm_nv12`, and YouTube H.264 in `google-chrome-stable` with zero kernel errors.

### Branch B: Direct Progressive `NV12` from `MSPPP` (`vaapi-msppp-progressive-nv12`) — Shipped in `26.3.1-2` (`a47e2a6b`)
- [x] Disassembled `nvc3_fuc086` (`MSPPP`) and confirmed bit `0x10` of `low700` controls field-separated output (`0x718`/`0x71c`, `0x720`/`0x724`).
- [x] Cleared `0x10` when `!target->base.interlaced`, enabled `PIPE_VIDEO_CAP_SUPPORTS_PROGRESSIVE` on `nvc0`, allocated progressive `PIPE_TEXTURE_2D` `NV12` surfaces, and added `util_format_is_yuv()` `tile_flags = 0xfe` in `nvc0_choose_tiled_storage_type()`.
- [x] Verified byte-for-byte identical `YUV420p` output at `64x64`, `640x360`, `854x480`, `1280x720`, `1920x1080`, and `720x480` interlaced H.264, `52.69 dB` PSNR on `1280x720` MPEG-2, smooth YouTube H.264 playback in Chrome, and 15x rapid teardown stress testing with zero kernel errors.

## 4. Additional Verification & Packaging

- [ ] Test MPEG-1/2 and VC-1 hardware decoding clips through VA-API to confirm VP4 teardown and fence sync beyond H.264.
- [ ] Check `src/gallium/frontends/va/decode.c` (`handleVASliceDataBufferType`) for packed multi-slice H.264 buffers without start codes between slices.
- [x] **Package & distribute `mesa-aliveos` 26.3.1-1** — published to GitHub (`Twilight0/mesa-aliveos`), AUR (`mesa-aliveos`), and `aliveos-repo` (`v26.3.1-1`).

## 5. Compiler & Gallium `nvc0` Performance (`nv50_ir` / Z-Cull / Tile Compression)

- [ ] **Dual-Issue Warp Scheduling & Superscalar Execution in `nv50_ir`**
  - Files: `src/nouveau/codegen/nv50_ir_*.cpp` / `src/gallium/drivers/nouveau/codegen/`
  - Fermi SM20 (`GF100`/`GF104`/`GF106`) features two warp schedulers and two instruction dispatch units per SM capable of co-issuing independent instruction pairs (e.g. ALU + TEX/MEM or INT + FP32 FMA). Enhance `nv50_ir` scheduling to bundle dual-issue pairs.
- [ ] **Register Spilling Mitigation & Graph-Coloring Constant Rematerialization**
  - File: `src/gallium/drivers/nouveau/codegen/nv50_ir_ra.cpp`
  - Rematerialize uniform expressions, constants, and address calculations during register allocation instead of spilling to slow local VRAM.
- [ ] **Lossless Z-Cull & Color Tile Compression Integration**
  - Files: `src/gallium/drivers/nouveau/nv50/nv50_surface.c`, `nvc0/`
  - Bind kernel `TAGRAM` buffer allocations (paired with `nouveau-dkms`) to depth/stencil and color targets to activate hardware Z-Cull and tile compression, reducing rasterization memory bus pressure.

## 6. Compute, OpenGL 4.x Stability & Desktop Compatibility (`nvc0` / Wayland EGL)

- [ ] **Fermi SSBO 64-Bit Address Bounds Clamping (`TargetNVC0`)**
  - File: `src/gallium/drivers/nouveau/codegen/nv50_ir_target_nvc0.cpp` (`emitSSBOOp`)
  - Clamp effective SSBO offsets via `MIN(offset, buffer_size - access_size)` before 64-bit pointer construction so out-of-range compute/fragment reads/writes do not trigger `GPC/TPC/MP trap: warp [OOR_ADDR]` and `PT_NOT_PRESENT` faults (observed in Blender 4.x).
- [ ] **Color Target 256-Byte (`0x100`) Base Alignment Enforcement**
  - File: `src/gallium/drivers/nouveau/nvc0/nvc0_surface.c`
  - Enforce `ALIGN(offset, 256)` on render target base addresses sent to `NV9097_SET_COLOR_TARGET_B` (`0x0804`), complementing `nouveau-dkms`'s BO/GEM alignment to prevent `PGRAPH DATA_ERROR [INVALID_VALUE]` on suballocated buffers.
- [ ] **Wayland EGL Visual Format Matching & Graceful Fallback**
  - File: `src/egl/drivers/dri2/platform_wayland.c`
  - Filter out unsupported 10-bit (`RGB10_A2`) and float surface formats on pre-Kepler/Fermi hardware so Wayland clients (e.g. Blender) match 8-bit visuals cleanly instead of hitting `EGL_BAD_MATCH`.
- [ ] **Nouveau Fermi `drirc` (`mesa_glthread=false`)**
  - File: `/usr/share/drirc.d/10-nouveau-fermi.conf` (or `00-mesa-defaults.conf`)
  - Disable `mesa_glthread` by default on Nouveau Fermi to prevent asynchronous pushbuf reordering races.
- [ ] **Missing Native OpenGL 4.5 / 4.6 Extensions on `nvc0`**
  - `GL_ARB_indirect_parameters` (`NV9097_SET_MME_SHADOW` + indirect draw counter reads on `0x9097`)
  - `GL_ARB_pipeline_statistics_query` & `GL_ARB_transform_feedback_overflow_query` (`0x9097` method `0x1b00` `QUERY_COUNTER`)
  - `GL_ARB_gl_spirv` & `GL_ARB_spirv_extensions` (wire `nir_spirv` into `nvc0_program.c` -> `TargetNVC0`)
- [ ] **Mesa Rusticl: Raise `CL_DEVICE_MAX_MEM_ALLOC_SIZE`**
  - Location: `src/gallium/drivers/nouveau/nvc0/`
  - Raise single-buffer allocation cap from 25% of VRAM (`768 MiB`) to ~90% (`~2.75 GiB`) so `llama.cpp` and `CLBlast` can load quantized SLM weights into contiguous buffers.

## 7. Hardware Video Encoding & Vulkan (`VA-API Encode` / `nvk-fermi`)

- [ ] **Hybrid Compute-Shader VA-API H.264 Encoder (`VAEntrypointEncSlice`)**
  - Files: `src/gallium/frontends/va/`, `src/gallium/drivers/nouveau/nvc0/`
  - Implement a 4-stage compute-backed `pipe_video_codec` (`RGB -> NV12`, parallel SM `SAD/SATD` motion estimation, integer DCT/quantization, host CAVLC/CABAC packing) for OBS Studio (`obs-vaapi`), FFmpeg (`h264_vaapi`), PipeWire, and GStreamer.
- [ ] **Kepler (`GK104+`) & Maxwell (`GM107+`) Fixed-Function `MSENC` (`0x1c8000`) VA-API Encode**
  - Files: `src/gallium/drivers/nouveau/nve4/`, `nv50/`
  - Expose hardware `NVENC`/`MSENC` H.264 encoding via Gallium VA-API.
- [ ] **Fermi Vulkan 1.0 / 1.1 (`nvk-fermi` Bound-Descriptor Lowering) & Zink**
  - Location: `src/nouveau/vulkan/`
  - Prototype `hasvk`-style bound-descriptor lowering (`128 TIC` / `32 TSC` slots per stage) pairing `NVK`'s command buffer/sync runtime with `NIR -> nv50_ir` (`TargetNVC0`), enabling Zink OpenGL 4.6 on top of `nvk-fermi`.
