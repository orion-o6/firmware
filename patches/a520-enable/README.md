# a520-enable

Brings up the four Cortex-A520 little cores, so the board runs all 12 cores instead of 8.

Before, `/sys/devices/system/cpu/possible` is `0-7`: only the eight Cortex-A720 come up. After, it is `0-11`, with cpu2-5 reporting Cortex-A520 (MIDR `0x410fd801`).

## Root cause

The little cores are not a hardware or bootloader1 limitation. Radxa's platform DSC turns them off. `Silicon/CIX/Sky1/CixPkg.dec` defaults every `PcdCpuCore<N>En` to `TRUE`, but `Platform/Radxa/Orion/O6/O6.dsc` overrides cores 2 to 5 to `FALSE`, under the comment `# Change for SystemReady`. Those PCDs feed `PlatformSetupVar.CpuCoreEnable[]`, then the topology (`CM_CIX_CPU_TOPO_INFO`), and the MADT clears `GIC_ENABLED` for a disabled core. So the four A520 ship declared-but-disabled (MADT `Flags = 0x0`).

## What it changes

Four lines in `O6.dsc`: `PcdCpuCore2En` through `PcdCpuCore5En` from `FALSE` to `TRUE`. Nothing in bootloader1 or the SCP. The SCP already powers the little cluster; `PSCI CPU_ON` (in BL31, part of the FIP) brings the cores online.

## Applies to

`edk2-platforms` submodule at commit `1a48c6523a3225f3ef01b1c91eb3e3dc0dd1857f`, via `cixtech/bios` branch `cix_p1_community_dev`.

## Apply

    git -C edk2-platforms apply /path/to/0001-enable-a520-little-cores.patch

Then build (see `docs/build-and-flash.md`).

## Verified

Built RELEASE with this patch and flashed on real hardware. `/sys/.../cpu/possible` and `online` are both `0-11`, `nproc` is 12, and `lscpu` reports 4 Cortex-A520 plus 8 Cortex-A720. No `PSCI`/bringup failures in `dmesg`.

## Note

This only re-enables the cores. The cache topology (`patches/pptt-cache-topology`) reads geometry once on the boot core and assumes identical cores, so on a 12-core build the A520 would advertise A720 L1/L2 sizes. A 12-core release image needs that patch extended to read geometry per cluster. Stability under sustained load on the little cluster should also be burn-in checked before calling this production.

Fixes #15.
