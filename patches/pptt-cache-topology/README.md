# pptt-cache-topology

Makes the PPTT describe the real cache layout, so the kernel sees private L1/L2 per core and one L3 shared across all cores.

Before (firmware `9.0.3`), every level was reported as shared by all 8 cores, with no sizes:

    L1d / L1i / L2 / L3:  shared_cpu_list 0-7   size (none)

After:

    L1d  shared_cpu_list 0     64K
    L1i  shared_cpu_list 0     64K
    L2   shared_cpu_list 0     512K
    L3   shared_cpu_list 0-7   12M

## Root cause

The generator emitted only the processor-hierarchy nodes (socket, cluster, core) and **zero cache structures**. A PPTT with no cache nodes leaves the kernel with a flat, sizeless view: it cannot tell which caches are private and which are shared.

## What it changes

- `PpttGenerator.c`: emits the cache structures. The geometry (size, associativity, sets, line size) is read at runtime from `CCSIDR_EL1`, so the numbers come from the silicon rather than a hardcoded table. Each core carries its L1I and L1D as private resources; L1 chains to a per-core L2. The shared L3 is a private resource of the socket node.
- `AcpiPpttLibCIX.inf`: adds `ArmPkg/ArmPkg.dec` and the `ArmLib` class, for `ReadCCSIDR`.

## Why the L3 hangs off the socket node

Linux (`drivers/acpi/pptt.c`) attributes a cache to the processor node under which its walk first reaches that cache level, and uses that node as the cache's sharing token. A cache reached by `NextLevelOfCache` chaining from a per-core resource is therefore attributed per core, and shows up as private.

So the per-core L2 chain stops at L2 (`NextLevelOfCache = 0`), and the L3 is listed as a private resource of the socket node. The kernel then finds the L3 while walking up from every core to the socket, and reports it as shared by all of them.

## Applies to

`edk2-platforms` submodule at commit `1a48c6523a3225f3ef01b1c91eb3e3dc0dd1857f`, via `cixtech/bios` branch `cix_p1_community_dev`.

## Apply

    git -C edk2-platforms apply /path/to/0001-pptt-cache-topology.patch

Then build (see `docs/build-and-flash.md`).

## Verified

Built RELEASE with this patch and flashed on real hardware. `/sys/devices/system/cpu/*/cache/` and `lscpu -C` now show private L1/L2 per core with real sizes, and a 12M L3 shared across all 8 cores.

Cross-checked three ways:

- `size == ways x sets x line` holds for every level (64K, 512K, 12M).
- `lscpu` `ALL-SIZE` shows L1 and L2 scaling by 8 (private per core) while L3 does not (one shared instance).
- The geometry matches what `CCSIDR_EL1` reports on the running cores.

## Limitation: homogeneous geometry

This reads cache geometry once (via `CCSIDR_EL1` on the core that builds the table) and applies it to every core node. That is correct on the base this patch targets, where the 8 online cores are all Cortex-A720 with identical geometry.

On a base that enables the little cluster (the 12-core Sky1 config: 4 big + 4 medium Cortex-A720 + 4 Cortex-A520, see #12), the A520 L1/L2 geometry differs, so the generator must read geometry per cluster on the target PE (e.g. via MpServices). Rebasing onto such a base means doing that first.

Fixes #5.
