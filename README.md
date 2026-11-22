# mesa-aliveos

AliveOS patched Mesa package built on top of official Mesa releases using a clean patch queue (`CachyOS`-style PKGBUILD on upstream tarballs).

Provides universal x86_64 Gallium OpenGL and VA-API hardware video acceleration drivers while coexisting cleanly with Arch Linux's standalone `vulkan-radeon`, `vulkan-intel`, `vulkan-nouveau`, `vulkan-swrast`, and `opencl-mesa` packages.

## Key Fixes & Improvements (26.3.1-2)

- **Arch Linux `extra/mesa 1:26.3.1-2` Base Patches (`0001`–`0002`)**:
  - Includes `0001-panfrost-Fix-types-of-panfrost_vdrm-functions.patch` and `0002-Revert-mesa-don-t-advertise-APIs-that-are-not-built.patch` (Mesa #16934 fix for `-Dgles1=disabled` builds) as the Arch Linux baseline beneath the AliveOS patches (`0003`–`0006`).
- **Chromium `VaapiVideoDecoder` & Direct Progressive `NV12` (`0006`)**:
  - Decodes progressive `NV12` directly from VP4 `MSPPP` (`low700 &= ~0x10`) with zero extra VRAM or VPP copy, and enables `nvc0` block-linear `DRM_FORMAT_NV12` DMABUF modifier support for `gbm_bo_import()` in Chromium and `mpv`.
- **Nouveau Fermi / VP4 (`NVC3` / `GF106M`) VA-API Stability**:
  - Fixed `nouveau_vp3_decoder_destroy()`, `firmware_present()`, and `nvc0_screen_destroy()` channel/subchannel teardown ordering to eliminate PFIFO `CTXSW_TIMEOUT` hangs.
  - Implemented `fence_wait` / `destroy_fence` and `out_fence` export on `nouveau_vp3` decoder channels and isolated decoder pushbuf callbacks from the 3D screen fence sequence.
  - Fixed VA-API interlaced surface sizing, image upload clamping (`GF100_M2MF` `XY_OUT_OF_BOUNDS`), and H.264 `max_references` decoder recreation.
- **VL Compositor `yuv420p` Chroma Output**:
  - Fixed plane-order indexing and `VL_COMPOSITOR_PLANE_V` sampler binding on the graphics shader deinterlace path so VA-API VPP `yuv420p` outputs accurate U/V planes.
- **Nouveau `nvc0` Compiler & OpenGL 4.6**:
  - Increased register allocator spill retry limit from `3` to `8` for high-register-pressure shaders.
  - Enabled GLSL `460` (`OpenGL 4.6`) feature level on `nvc0`.
- **Full Codec & x86_64 Gallium Support**:
  - Built with `-Dvideo-codecs=all`, `-Dgallium-va=enabled`, `-Dgallium-extra-hud=true`, and drivers `r300,r600,radeonsi,nouveau,virgl,svga,llvmpipe,softpipe,iris,crocus,i915,zink`.

## Installation

### From `aliveos-repo` (Prebuilt Binary)

With `[aliveos-repo]` configured in `/etc/pacman.conf`:

```bash
sudo pacman -Syu mesa-aliveos
```

### From GitHub Releases

Download the `.pkg.tar.zst` archive from [Releases](https://github.com/Twilight0/mesa-aliveos/releases/latest) and install:

```bash
sudo pacman -U mesa-aliveos-26.3.1-1-x86_64.pkg.tar.zst
```

### From AUR

```bash
yay -S mesa-aliveos
```

### Manual Build

```bash
git clone https://github.com/Twilight0/mesa-aliveos.git
cd mesa-aliveos
makepkg -si
```

## Documentation

- [CHANGELOG.md](CHANGELOG.md) — release history and included patches
- [NOTES-nvc0-m2mf-xy-out-of-bounds.md](NOTES-nvc0-m2mf-xy-out-of-bounds.md) — root-cause analysis of Fermi VP4 and M2MF bugs
- [TODO.md](TODO.md) — ongoing investigations (Dolphin PR #14993 `gl_FragCoord.z`, Chromium progressive `NV12` DMABUF export)
