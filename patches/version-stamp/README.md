# version-stamp

Stamps the firmware version so it reads as ours, not stock, where users look.

- **Splash** ("Tianocore/EDK2 firmware version ..."): `orion-o6/firmware 1.0 (cix 9.0.3)`
- **`dmidecode -s bios-version`** (SMBIOS BIOS Information): `orion-o6/firmware 1.0 (cix 9.0.3)`

## Applies to

`edk2-platforms` submodule at commit `1a48c6523a3225f3ef01b1c91eb3e3dc0dd1857f`, via `cixtech/bios` branch `cix_p1_community_dev`.

## Apply

    git -C edk2-platforms apply /path/to/0001-version-stamp.patch

Then build (see `docs/build-and-flash.md`).

## What it changes

Two files, three lines. It hardcodes the version string as a plain C literal in the SMBIOS BiosVersion (`SmbiosType0.c`) and as a wide string in `PcdFirmwareVersionString` (`RadxaCommon.dsc.inc`). It does not go through the `DEB_VERSION` build macro, because a value with spaces and parentheses breaks the C define that macro also feeds. Bumping the version for a new release means editing the string in this patch.

The FlashUpdate utility's "New Version" line is left untouched: it reads `UEFI_FW_VERSION`, which stays the build-provenance string (`9.0.3-<commit>-debug:<build-date>`). That is intentional; it tells you exactly which build you are flashing.

## Verified

Built (this patch on top of a DEBUG build) and flashed on real hardware. The board booted, and both the splash and `dmidecode -s bios-version` showed `orion-o6/firmware 1.0 (cix 9.0.3)`.

Fixes #3.
