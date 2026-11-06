# GPU acceleration research for Galaxy S8 (dreamlte)

## Current state

The iOS port currently uses the S-Boot framebuffer and CPU rendering. There is no Mali job-submission driver or GLES userspace path in this kernel. Apple's bundled AGX driver targets Apple GPUs, so the plausible route is Panfrost-compatible Mali support plus a GLES path, not enabling AGX with a boot flag.

## Same-device hardware evidence: LinuxS8

[`MakrSas/LinuxS8`](https://github.com/MakrSas/LinuxS8) documents a successful Panfrost bring-up on the same Galaxy S8 / Exynos 8895 / Mali-G71 MP20. Its maintainer reports `GPU_ID=0x60a00001`, `shader_present=0x17771777` (20 shader cores), and Mesa reporting `Mali-G71 MC20 (Panfrost)`. This is strong hardware-specific evidence, but it is a maintainer report and has not been reproduced from this checkout.

The documented Linux setup requires the G3D power domain at PMU `0x16480000 + 0x40e0`, CMU_G3D at `0x13800000`, Mali registers at `0x13900000` (0x5000 bytes), and a sufficiently recent Mesa build (the project reports 26.1.2). Their clock path uses the CMU_TOP G3D switch output; do not copy S9/Exynos9810 voltage or clock programming into this S8 port.

## First XNU checkpoint

`xnu/patches/0125-dreamlte-mali-probe.patch` adds a DREAMLTE-only diagnostic hook. It is disabled unless `mali_probe=1` or `mali_probe=2` is supplied. Level 1 reads and logs the G3D PMU configuration/status and CMU switch-user register. Level 2 also reads GPU identity/features only if the power-domain status already reports fully on. The follow-up `0126-dreamlte-mali-feature-probe.patch` adds opt-in `mali_probe=3` reads for the G71 core, tiler, memory, MMU, job-slot and raw-interrupt feature registers, guarded by the observed G71 product ID. Neither patch enables power, changes a clock, touches GPU registers on the default boot path, installs interrupts, or submits GPU work. These are discovery checkpoints, not acceleration.

## Closest reference: Galaxy S9+ Exynos

[`danstronics/ios-s9plus-exynos`](https://github.com/danstronics/ios-s9plus-exynos) describes a Galaxy S9+ (`star2lte`, Exynos 9810, Mali-G72) build whose default compositor uses Mesa/Panfrost GLES. Its current source snapshot is commit `2d092398040f98685b0c5a07776df5e9f199b9dc`.

Relevant pieces in that repository:

- `core/drivers/gpu/hakios.mali.c`: staged Mali bring-up and job/MMU support.
- `core/drivers/gpu/hakios.mali.cpp`: IOKit `HakiosMali` service exposing the Panfrost DRM request interface.
- `core/tools/mesa/panfrost-darwin.patch` and `core/tools/mesa/hakios-drm/`: Mesa/Panfrost adaptation and Darwin-to-IOKit transport.
- `core/userland/src/eagl.c` and the userland setup: connect the iOS graphics stack to GLES.
- `baseline-support/`: additional mapping/cache changes referenced by that port.

The repository README and installed profile report hardware GLES testing on the S9+. I have not independently reproduced that result. The port's Metal-to-Vulkan work is separate and marked WIP; the installed path is GLES/Panfrost.

## Other requested repositories

I also checked current snapshots of `dipa-studio/ios-sweet` (`45263be`),
`oosexgang/hakios-xnu-kernel` (`63fc2b4`) and standalone
`oosexgang/hakios-metal` (`d36f329`). `ios-sweet` is the closest match for this
port's XNU generation and framebuffer/IOSurface path, but it renders in
software. Hakios XNU's GPU backend targets Qualcomm Adreno (the current
configuration calls out Adreno 642L); Hakios Metal emits Adreno A6xx PM4
commands. Those are not compatible with Mali-G71. Their IOKit user-client and
Metal API designs may still inform higher layers, but S8 needs a new Mali
backend. The standalone `hakios-metal` repository has since moved into
`hakios-xnu-kernel/hakios-metal/`.

For S8 the closest reusable GPU path remains S9+'s `HakiosMali` service and
Mesa/Panfrost request transport. Its G72/Exynos9810 bring-up must be adapted
for G71 and Exynos8895 power, clocks, interrupts, memory and cache behavior.
The current safe experiment is the flashed `mali_probe=1` image. Its kernel
log is needed before attempting writes to PMU or CMU registers. A separate
`mali_probe=2` image is prepared but not flashed (SHA-256
`b17a839c31030b9044b86e6452ceaa437eb6f081c51b0fa4ed80475ac7694cae`); it only
reads G71 registers when the PMU already reports the domain on. After measured
state is available, the next steps are a separate opt-in power/clock stage,
GPU identity and minimal job validation, then the Panfrost user client and iOS
rendering integration.

## What transfers and what does not

| Component | Reuse potential on S8 |
|---|---|
| Panfrost userspace and its DRM request definitions | High: LinuxS8 reports a working G71 path with recent Mesa. The Darwin/IOKit adaptations still need to match this XNU and its memory model. |
| `HakiosMali` IOKit/Panfrost interface | Medium to high: likely a useful base, after checking dependencies and XNU API compatibility. |
| Bifrost job, MMU, buffer and fence code | Medium: shared concepts, but the G71 GPU ID, shader-core layout, MMU features, page-table format and job behavior must be measured/validated on S8. |
| Exynos power, clock, voltage, PMU and ACPM sequence | Low: the S9 implementation is specifically for Exynos 9810. Do not copy its register values or voltage settings. |
| S9 display controller/panel integration | Low: S8 has different display hardware and this port currently scans out through S-Boot's framebuffer. |
| Metal experiments | Low for initial usability: separate work, and not needed to first accelerate GLES/CoreAnimation. |

Linux's Exynos 8895 device tree identifies the S8 Mali block at `0x13900000`; the S9 driver is hard-coded for `0x17500000` and explicitly checks for a G72 (`GPU_ID >> 16 == 0x6221`). The S9 Mali source itself says it was adapted from a MediaTek G52 driver, then changed for Exynos 9810, and warns that the S9 initialization has device-specific power requirements. That makes it a strong architectural reference, not a drop-in driver.

The LinuxS8 report records successful G71 reads on this model: `GPU_ID=0x60a00001`, `L2_FEATURES=0x07130206`, `AS_PRESENT=0x000000ff`, and `shader_present=0x17771777`; Linux reported the G3D domain on as `STATUS=0x0003000f`. These are reference results from Linux experiments, not measurements from the iOS kernel. Compare them against the guarded XNU probe output before designing power or clock writes.

## Safe implementation sequence

1. Keep the current boot path unchanged; add no GPU register access to default boots. The optional XNU checkpoints are in patches `0125` and `0126`.
2. On a controlled boot, collect `mali_probe=1` first; only request `mali_probe=2` if the PMU reports the domain fully on. Use `mali_probe=3` only after identity matches G71, then compare full feature registers with LinuxS8 evidence.
3. Port a minimal Panfrost kernel interface for G71 behind a S8-only opt-in argument. Do not copy the S9 Exynos9810 power/voltage sequence.
4. Build a minimal Darwin/Panfrost transport and userspace GLES stack. Validate memory allocation, cache coherency, MMU mapping, interrupts and job completion before exposing it to applications.
5. Add an off-screen GLES render check, retaining CPU rendering as fallback. Then adapt the existing `SweetSurface`/compositor path and profile the complete frame pipeline.

The checkout includes the kernel archive and ordered patch workflow; the reconstructed XNU source tree is temporary build input. Patch 0125 and its caller built successfully in Actions run `42299498637`. The `mali_probe=1` diagnostic BOOT was flashed only to the verified S8 BOOT partition on 2026-11-06; a readback of the image-sized prefix matches SHA-256 `b77edac37d97e1476f808c6f387250d71e9e34c0a8f9fc09771bc871a31c525f`. On 2026-11-07 the user authorized and the project writer reflashed all 512 SHA-256-verified chunks of the 16-GiB root to USERDATA; the readback volume signature is HFSX (`HX`). The trailing region was zeroed while preserving its last 4 MiB log ring; see `DREAMLTE_PROGRESS.md`. A reboot was requested. ADB disconnect after reboot is not a boot result because iOS does not normally expose ADB; the physical screen result is pending. The probe's PMU/CMU output is also pending. No GPU power-up, job submission, GLES user client, accelerated compositor, or runtime acceleration has been implemented.

## References

- S9+ Exynos port: <https://github.com/danstronics/ios-s9plus-exynos>
- Same-device LinuxS8 graphics report: <https://github.com/MakrSas/LinuxS8/blob/main/docs/graphics.md>
- Same-device LinuxS8 kernel and device-tree changes: <https://github.com/MakrSas/LinuxS8/blob/main/patches/kernel/all-local-changes.patch>
- Its Mali implementation: <https://github.com/danstronics/ios-s9plus-exynos/tree/main/core/drivers/gpu>
- Linux Exynos 8895 device tree: <https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/exynos/exynos8895.dtsi>
- Linux S8 device tree: <https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/exynos/exynos8895-dreamlte.dts>
- Mesa Panfrost driver documentation: <https://docs.mesa3d.org/drivers/panfrost.html>
