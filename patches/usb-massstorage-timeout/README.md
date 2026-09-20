# usb-massstorage-timeout

Bounds how long the firmware waits on a USB mass-storage device that stops answering: one attempt per boot instead of minutes. Fixes the boot hang of issue #2.

## Root cause

The hang is not a console-connect race, as #2 first guessed. It is a USB mass-storage device (on our bench the JetKVM's virtual media) that enumerates, gets configured, and then never completes a bulk-in transfer. The firmware's SCSI `INQUIRY` is accepted on the bulk-out endpoint and never answered, and `UsbMassStorageDxe` waits on it far longer than it should:

- One attempt is the data phase timeout (5 s) plus three CSW reads of 3 s each: about 14 s.
- `UsbBootExecCmdWithRetry` repeats the attempt until a 60 s timer expires or `USB_BOOT_COMMAND_RETRY` (5) is exhausted: 71 s per driver start.
- The driver is started again on every connect pass of the boot (DXE dispatch, `ConnectAll`, the console connect, boot option enumeration), so the 71 s repeats. Six starts were counted in one boot.

Measured over the UART2 serial console (see `docs/build-and-flash.md`, "Measuring a boot fix") on a DEBUG build, JetKVM virtual media mounted, warm reset: `E550` (BmAfterConsole) reached after 523 s instead of about 5 s. The trigger is a warm reset (ATX reset or OS reboot), not a mains cold boot: 0 of 8 cold boots stalled, 1 of 1 warm reset did.

Why the device goes silent is a JetKVM bug, reported upstream (jetkvm/kvm#684, fix proposed in jetkvm/kvm#1641): its USB recovery loop rebinds the gadget every 5 s while the host is not enumerated, and a rebind that lands on the host's XHCI reset leaves the gadget configured but bulk-dead. That side has to be fixed there. The firmware side is to stop waiting minutes for a device that does not answer, whatever the device.

## What it changes

Two changes in `MdeModulePkg/Bus/Usb/UsbMassStorageDxe`, neither specific to any device. The timeouts and retry counts themselves are left as upstream has them.

Silence is terminal. In `UsbBootExecCmdWithRetry`, an `EFI_TIMEOUT` from the transport ends the loop instead of being retried. A device that did not answer within the transport timeout is not going to answer a repeat of the same command. Media that answers `NOT READY` while it spins up is still retried for up to 60 s, and fast errors still get their `USB_BOOT_COMMAND_RETRY` repeats, as before. A stall costs one 14 s attempt per driver start instead of 71 s.

One start per boot. `UsbMassImpl.c` remembers a device whose start failed, at the transport init, the single-LUN init, or the multi-LUN init (any status but out of resources), and `DriverBindingSupported` declines it for the rest of the boot. Without this the boot still pays the 14 s on every connect pass, about 60 s at the splash. The key is the device path plus the vendor and product ids: it survives the port resets that give the device a new handle, and a different device on the same port is not declined. Bounded list of eight, oldest entry overwritten. A remembered device stays hidden until the next reset, even if it recovers meanwhile; on a RELEASE build there is no trace of it, on a DEBUG build the driver logs the device path and the status.

The change is not specific to this board and is worth proposing to upstream EDK2; that has not been done.

## Applies to

`edk2` submodule at commit `a58bf7145015be5afb13e1a00987fea6a7368976`, via `cixtech/bios` branch `cix_p1_community_dev`.

## Apply

    git -C edk2 apply /path/to/0001-usb-massstorage-timeout.patch

Then build (see `docs/build-and-flash.md`).

## Verified

RELEASE builds on the pinned base, flashed in-band, JetKVM 0.5.9 plugged in, virtual media mounted, ATX resets and mains cold boots in a loop. The oracle is the `E1FF` to `E550` gap on the serial console. The only USB mass-storage device measured is the JetKVM virtual media; no USB stick or spinning drive was on the bench.

| Build | Boots | Gap when the device stalls | Gap when it does not |
| --- | --- | --- | --- |
| stock driver, DEBUG build | 1 warm reset | 430 s (`E550` at 523 s, `E1FF` itself already delayed to 93 s by the DEBUG output) | 4 to 5 s |
| stock driver, RELEASE build (the three merged patches, same instrument as the row below) | 5 warm resets, 2 cold | 284 to 286 s (4 of 5 resets), and one cold boot at 44 s | 4 s |
| this patch | 7 warm resets, 5 cold | 16 to 18 s (6 of 7 resets), and two cold boots at 44 s | 4 s |

Earlier revisions of this patch, measured the same way: shortening the timeouts alone gave 58 to 60 s per stall (the attempt repeated on every connect pass); a memo keyed only on `EFI_TIMEOUT` from the single-LUN path gave 14 to 18 s with one 44 s outlier in 12 boots. The current memo covers every failed start.

The 44 s cold-boot case is not this driver. It shows on the stock driver with the same 44 s (one of two cold boots) and on this patch (two of five), while the mass-storage stall on stock costs 285 s. It is a different failure of the same gadget on a mains cold boot, somewhere else in the USB stack, and this patch neither causes nor bounds it. Not investigated; it needs a DEBUG build.

Booting from the virtual media, JetKVM app 0.5.9, warm reset with `BootNext` set to the JetKVM entry, Debian installer ISO mounted: it works when the LUN does not stall on that boot (GRUB reached in about 11 s, once, on the previous revision) and fails when it does (two of two attempts on this patch stalled, the device was remembered as failed and not offered, and the boot fell through to the next entry). With the stock firmware a stalled LUN costs five minutes and then is not bootable either. So on a warm reset the virtual media is bootable only when the JetKVM gadget survives the reset, which is what jetkvm/kvm#1641 fixes. The dead gadget also reaches the OS: Linux reset the JetKVM device twice while probing it during one of those boots.

Unmounting the virtual media does not avoid the stall on a warm reset (2 of 3 resets at 18 s): the JetKVM keeps the LUN with no medium. With the mass storage function disabled in the JetKVM USB device settings, 4 of 4 resets at 1.5 to 4 s. That is the workaround without this patch, and the way to boot from virtual media on stock firmware: enable mass storage only for that boot.

Fixes #2. Closes #23.
