# GPU acceleration research for Galaxy S8 (dreamlte)

## Current state

The iOS port currently uses the S-Boot framebuffer and CPU rendering. The phone has passed a one-shot G71 `WRITE_VALUE` job through four-level GPU page tables. Patch `0132` now adds an opt-in kernel Panfrost user-client/BO transport candidate, but it has not yet passed kernel CI or device validation. There is still no GLES userspace path or accelerated compositor, so iOS remains unaccelerated. Apple's bundled AGX driver targets Apple GPUs; the route is a Panfrost-compatible Mali backend plus a GLES path.

## Same-device hardware evidence: LinuxS8

[`MakrSas/LinuxS8`](https://github.com/MakrSas/LinuxS8) documents a hardware-tested Panfrost bring-up on the same Galaxy S8 / Exynos 8895 / Mali-G71 MP20. Its current `docs/graphics.md` reports `GPU_ID=0x60a00001`, `shader_present=0x17771777` (20 shader cores), and `GL_RENDERER = Mali-G71 MC20 (Panfrost)` using Mesa 26.1.2. The same report says its GNOME/Mutter desktop remains on software rendering because `simpledrm` provides no KMS device. This is useful same-device evidence, but does not prove the iOS port's GLES/EAGL path.

The documented Linux setup requires the G3D power domain at PMU `0x16480000 + 0x40e0`, CMU_G3D at `0x13800000`, Mali registers at `0x13900000` (0x5000 bytes), and a sufficiently recent Mesa build (the project reports 26.1.2). Their clock path uses the CMU_TOP G3D switch output; do not copy S9/Exynos9810 voltage or clock programming into this S8 port.

## First XNU checkpoint

`xnu/patches/0125-dreamlte-mali-probe.patch` adds a DREAMLTE-only diagnostic hook. It is disabled unless a `mali_probe` boot argument is supplied. Level 1 reads and logs the G3D PMU configuration/status and CMU switch-user register. Level 2 also reads GPU identity/features only if the power-domain status already reports fully on. Patch `0126-dreamlte-mali-feature-probe.patch` adds opt-in `mali_probe=3` reads for G71 feature registers, guarded by the G71 product ID. Patch `0127-dreamlte-mali-clock-state-probe.patch` adds read-only CMU_TOP and CMU_G3D clock state. Patch `0128-dreamlte-mali-clock-200mhz.patch` adds one explicit write-enabled boot argument, `mali_clock=200`: after the same PMU/G71 guards, it selects the Exynos8895 CMU_TOP switch path, requests the known 200-MHz setting, enables the clock gates, and verifies register readbacks. With the argument absent, 0128 makes no writes. This is a controlled clock bring-up stage; it does not reset the GPU, submit a job or provide acceleration yet.

Patch `0131-dreamlte-mali-mmu-walk-test.patch` adds an optional follow-up controlled by `mali_mmu_job=1` alongside `mali_probe=3 mali_clock=200 mali_job=1`. It allocates five kernel pages, builds a four-level AArch64 GPU page table mapping one page at VA `0x100000`, and runs a `WRITE_VALUE` job through that mapping. It reports the MMU fault status/address and retains pages if GPU completion or AS invalidation is uncertain. This remains a one-shot kernel diagnostic; it does not expose user buffers or render GLES. Kernel CI run `42361163053` passed. The 512-MiB image was run on the phone, and its USERDATA log confirms `GPU MMU WALK PASS` at VA `0x100000`, value `0x51a8c0de`, with no MMU fault. The UI still runs without acceleration.

Patch `0132-dreamlte-panfrost-userclient.patch` adds the Panfrost client and persistent GPU address-space support, activated only with `mali_service=1` alongside the preceding guarded bring-up flags. It exposes GET_PARAM, BO allocation/mapping, synchronous job submit/wait, and close requests. This is transport groundwork only: no EAGL integration, IOSurface binding, render verification, or UI acceleration exists yet. Kernel CI and phone validation are pending; do not flash this candidate until the build passes and a separate image is reviewed.

## Closest reference: Galaxy S9+ Exynos

[`danstronics/ios-s9plus-exynos`](https://github.com/danstronics/ios-s9plus-exynos) describes a Galaxy S9+ (`star2lte`, Exynos 9810, Mali-G72) build whose default compositor uses Mesa/Panfrost GLES. Its current source snapshot is commit `2d092398040f98685b0c5a07776df5e9f199b9dc`.

Relevant pieces in that repository:

- `core/drivers/gpu/hakios.mali.c`: staged Mali bring-up and job/MMU support.
- `core/drivers/gpu/hakios.mali.cpp`: IOKit `HakiosMali` service exposing the Panfrost DRM request interface.
- `core/tools/mesa/panfrost-darwin.patch` and `core/tools/mesa/hakios-drm/`: Mesa/Panfrost adaptation and Darwin-to-IOKit transport.
- `core/userland/src/eagl.c` and the userland setup: connect the iOS graphics stack to GLES.
- `baseline-support/`: additional mapping/cache changes referenced by that port.

The repository README and installed profile report hardware GLES testing on the S9+. I have not independently reproduced that result. Its Mesa build pins 25.0.7 and uses OSMesa behind a private EAGL bridge; the port's Metal-to-Vulkan work is separate and marked WIP.

## Mesa version for the S8 EAGL path

The S9 userland uses Mesa 25.0.7's OSMesa target behind its EAGL bridge. The S8 reports `GPU_ID=0x60a00001`, so the Panfrost `GPU_PROD_ID` value returned by the S9-style uAPI is `0x60a0`. Mesa 25.0.7's `panfrost_get_model()` compares that value exactly against the G71 table entry `0x6000`, so the stock table does not recognize this S8 stepping. The local patch `tools/dreamlte/mesa/0001-panprops-match-s8-g71-r0a.patch` changes the entry to `0x60a0`, matching the measured S8 ID only.

I applied the complete S9 Mesa patch series in its documented order to the pinned Mesa 25.0.7 source, then applied this S8 model-ID patch; all patches applied without rejects. Mesa 25.2.0 and 26.1.2 no longer contain the OSMesa target/source paths used by the S9 `mk-mesa.sh`, although their Panfrost model lookup masks the revision bits. Therefore the immediate S8 EAGL port can retain Mesa 25.0.7 and its OSMesa bridge with the narrow model-ID change, rather than switching Mesa APIs and rebuilding the EAGL backend. This is source-level integration evidence only; no S8 GLES library has been built or run yet.

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
Mesa/Panfrost request transport. Its G72/Exynos9810 initialization must be
adapted for the measured G71 and Exynos8895 power, clocks, interrupts, memory
and cache behavior. The iOS kernel now verifies the 200-MHz route and a single
G71 `WRITE_VALUE` job; the persistent buffer/MMU service, Mesa library, EAGL
bridge and compositor integration remain to be built.

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

The LinuxS8 report records successful G71 reads on this model: `GPU_ID=0x60a00001`, `L2_FEATURES=0x07130206`, `AS_PRESENT=0x000000ff`, and `shader_present=0x17771777`; Linux reported the G3D domain on as `STATUS=0x0003000f`. The iOS kernel later read the same GPU ID and features, including `MMU_FEATURES=0x2830`: 48-bit GPU virtual addresses and 40-bit physical addresses. Its S8 `WRITE_VALUE` job also succeeded with cache maintenance around a kernel page; full user-buffer coherency is not established. Linux first ran through the CMU_TOP switch path at 200 MHz, then raised that divider to 400 MHz; the source describes the G3D PLL path as not locking. Linux results are not substitutes for XNU measurements. The upstream Exynos8895 clock driver places the CMU_TOP G3D divider/gate at offsets `0x1880`/`0x2088` and describes the divider as three bits wide; patch 0128 starts with the 200-MHz route and logs the shared2 PLL state plus clock readbacks. ([Linux Exynos8895 clock source](https://github.com/torvalds/linux/blob/master/drivers/clk/samsung/clk-exynos8895.c))

## S9 driver port audit (2026-11-07)

The S9 `hakios.mali.c` is not a safe file-level transplant. Its stage code directly maps Exynos9810's PMU at `0x14060000`, CMU_G3D at `0x17400000`, and Mali at `0x17500000`; it also contains S9 ACPM BUCK6 changes and Samsung DTZPC SMC handling. The S8's observed Linux map is PMU `0x16480000`, CMU_G3D `0x13800000`, and Mali `0x13900000`; its already-tested XNU path uses the separate CMU_TOP switch at 200 MHz. Keep the S8's guarded reset/job sequence as the hardware baseline and port only the reusable Bifrost mechanisms after comparing each register and memory assumption.

The S9 `hakios.mali.cpp` is a Panfrost request dispatcher, BO allocator, and IOKit user client. Its surface bridge depends on `MiniIOSurface` and `MiniIOMobileFramebufferShow`; this fork instead exposes `SweetSurface` and copies to the S-Boot framebuffer. The required S8 adaptation therefore has three concrete layers: (1) a G71-safe persistent kernel MMU/job service based on the successful S8 registers and page-size/address-width probes, (2) the Panfrost user-client request ABI with BO mappings and fences, and (3) an EAGL/OSMesa integration that routes IOSurface rendering through that client and waits for completion before scanout. Until all three are implemented and exercised, SpringBoard remains on CPU rendering.

## Safe implementation sequence

1. Keep GPU access opt-in; patches `0125`–`0129` leave the default boot without Mali register access or writes.
2. Discovery and the Exynos8895 200-MHz clock route are verified on this phone: PMU `0x3000f`, G71 ID `0x60a00001`, and matching CMU readbacks.
3. The guarded job smoke test is verified at the 512-MiB setting. The recovered USERDATA ring records `S8-MALI: GPU WRITE_VALUE PASS`, IRQ `0x2`, and sentinel `0xc0ffee01` on the physical G71. This proves a single job can access kernel memory; it does not render the UI.
4. Implement the S8 kernel service first. Port the S9 Panfrost request ABI and BO bookkeeping, retaining S8-only MMIO addresses and bring-up. The XNU feature log confirms 48-bit GPU VA, 40-bit PA, and the phone has passed a four-level page-table job. User submissions still need full-range page management, cache synchronization, fences, and error recovery.
5. Build the Mesa 25.0.7 OSMesa/EAGL bridge and an off-screen GLES render check, retaining CPU rendering as fallback. Confirm Panfrost is selected and render a known pixel before exposing it to the system UI.
6. Connect `SweetSurface` and the existing framebuffer only after BO allocation, GPU VA mapping, job fences, and the off-screen test work; then profile the complete frame pipeline. RAM remains separate: two earlier 2-GiB boots panicked in `kmem_range_init`; a third diagnostic 2-GiB boot also panicked, but its claim details were not legible in the photo and the persistent logs did not capture that attempt. The user reports the known 512-MiB BOOT p7 image has since restored iOS startup.

The checkout includes the kernel archive and ordered patch workflow; the reconstructed XNU source tree is temporary build input. Patches 0125–0128 built in Actions runs `42299498637`, `42311503465`, `42313291998`, and `42315170764`; patch 0129's opt-in G71 `WRITE_VALUE` job built in run `42317239859`. The preceding 512-MiB `mali_probe=3 mali_clock=200 mali_job=1` boot confirmed SpringBoard, four initialized CPUs, PMU `0x3000f`, G71 ID `0x60a00001`, and matching clock route readbacks. The recovered USERDATA ring now confirms `S8-MALI: GPU WRITE_VALUE PASS`, IRQ `0x2`, and sentinel `0xc0ffee01` at PA `0x883ade000`. This verifies one identity-mode job and one GPU MMU/page-table job on the phone, not GLES or accelerated composition; the user still reports a very sluggish UI with no visible acceleration. Two prior 2-GiB boots stopped in `kmem_range_init`; a third 2-GiB diagnostic image with patch 0130 was written only to BOOT p7 and the user supplied a photo of an XNU panic. The diagnostic claim is still unknown because logs have not yet been retrieved. Patch 0130 adds claim-name/size and map/constraint fields; CI run `42357416273` succeeded (kernel SHA-256 `6770d6ad86ef81b1eabb0d34088e61821afde52b0541be5eb165479df9f9e08a`). The diagnostic 2-GiB image's exact 17,700,880-byte BOOT p7 readback matched SHA-256 `bc0ad839f5f4729cab1793c6fc29d4a53f95ae7196692a943e8a90320adac8ec`. TWRP reports three RAM ranges and about 3.4 GiB usable; the high 2-GiB range is the only one documented free of firmware carve-outs. The XNU handoff still describes one contiguous physical range, so using RAM beyond that bank requires discontiguous-memory support. The USERDATA HFSX reflash and zeroing were verified on 2026-11-07.

## References

- S9+ Exynos port: <https://github.com/danstronics/ios-s9plus-exynos>
- Same-device LinuxS8 graphics report: <https://github.com/MakrSas/LinuxS8/blob/main/docs/graphics.md>
- Same-device LinuxS8 kernel and device-tree changes: <https://github.com/MakrSas/LinuxS8/blob/main/patches/kernel/all-local-changes.patch>
- Its Mali implementation: <https://github.com/danstronics/ios-s9plus-exynos/tree/main/core/drivers/gpu>
- Linux Exynos 8895 device tree: <https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/exynos/exynos8895.dtsi>
- Linux S8 device tree: <https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/exynos/exynos8895-dreamlte.dts>
- Mesa Panfrost driver documentation: <https://docs.mesa3d.org/drivers/panfrost.html>
