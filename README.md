# Orion O6 firmware

Community firmware patches, build tooling, and documentation for the Radxa Orion O6 (CIX P1 "Sky1" SoC).

This repository does not contain a firmware. It holds patches on top of the CIX community source, the tooling to build and flash them, and the knowledge behind the work. The board's actual problems live as GitHub issues, not as files here. The docs stay agnostic: how the board boots, how to build and flash, how the pieces fit together.

## What is established

- The CIX community BIOS builds from source, and a self-built image signed with the OEM key flashes and boots on real hardware.
- BL31 (the TF-A runtime), BL32 (OP-TEE), and BL33 (UEFI) can be rebuilt, re-signed with the OEM key, and accepted by the board. Firmware, secure-OS, and boot-level bugs are patchable.
- bootloader1 (BL1/BL2 plus the PM firmware, the SCP) is verified by the Security Enclave against a fused CIX key and cannot be self-signed. It stays stock. That is the one stage we cannot touch. CPU DVFS lives in the SCP, so the frequency ceilings are not ours to change either.

See `docs/bootloader-model.md` for the full trust model.

## Ground rules

- We never redistribute closed binaries. The `edk2-non-osi` blobs and CIX's signed bootloader1 carry no license granting redistribution, so they are not ours to share. You pull the CIX community source yourself and build your own image from it plus these patches.
- Flashing experimental firmware can leave the board unbootable. The BIOS chip is socketed, so recovery is off-board with an SPI programmer (CH341A class). Do not flash without a way to recover.

## How the repo is organised

- `docs/` holds agnostic knowledge and the build and flash method. It does not track bugs.
- Issues track the board's real problems, with a template for filing them.
- `patches/` holds the fixes, one directory per patch, as diffs against pinned CIX commits.

## Base

Built against the CIX community source at https://github.com/cixtech/bios, branch `cix_p1_community_dev`. That branch is a superproject pulling edk2, edk2-platforms, edk2-non-osi, tf-a, tee, and the packaging tool. It is the branch CIX's own README tells you to clone; there is no `main`. Each patch records the exact commit it was built against.

## Build and flash

See `docs/build-and-flash.md`. In short, build in an x86_64 Linux environment (the signing tools are x86_64) to produce a `cix_flash_all.bin`, then flash it through the board's own EFI System Partition and built-in UEFI Shell, or with an SPI programmer.

## Contributing

Hit a problem with your board? Open an issue with your firmware version and a serial log if you have one. `CONTRIBUTING.md` explains how a fix goes from issue to verified patch, and what is in scope.

## Status

Early. The build, flash, and boot method is proven. Fixes land one verified patch at a time.
