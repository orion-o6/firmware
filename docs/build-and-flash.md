# Building and flashing

## Where it builds

The CIX packaging and signing tools (`cert_uefi_create_rsa`, `fiptool`, `cix_package_tool`, and others in `cix_package-tool/`) are prebuilt **x86_64** Linux binaries. So the full signed `cix_flash_all.bin` cannot be produced natively on an aarch64 host; you need an x86_64 Linux environment, or x86_64 emulation. On an Apple Silicon Mac, an x86_64 Linux container through Rosetta works well.

The compile toolchain is the Arm GNU-A 10.2-2020.11 `aarch64-none-elf` cross-compiler; the x86_64-hosted build is the one `cix-build.sh` defaults to.

## Build

On an x86_64 Debian/Ubuntu environment:

    sudo apt-get install -y build-essential uuid-dev acpica-tools git nasm \
      python3 python3-setuptools python3-pip python-is-python3 \
      python3-cryptography python3-pyelftools bison flex wget xz-utils \
      device-tree-compiler ca-certificates

    git clone --depth 1 --shallow-submodules -b cix_p1_community_dev \
      https://github.com/cixtech/bios.git o6bios --recursive
    cd o6bios
    ln -sf cix_package-tool/cix-build.sh cix-build.sh
    mkdir -p tools/gcc tools/gcc && cd tools
    git clone --depth 1 https://github.com/acpica/acpica.git --branch R2024_12_12
    cd gcc
    # download the x86_64-hosted aarch64-none-elf toolchain from the Arm GNU-A
    # 10.2-2020.11 release, extract it here, then:
    cd ../.. && ./cix-build.sh

The output is `output/cix_flash_all.bin`. To apply a patch from this repo first, check out the matching submodule commit and apply the diff before running `cix-build.sh`.

Note: the community package tool emits an image sized to its last payload (around 6 MB), not the full 8 MiB of the chip. For a full-chip SPI write, pad it to 8388608 bytes with 0xFF (matching the vendor image, whose tail is 0xFF). For in-band flashing, the tool handles the layout.

## Flash

Two paths. Keep a known-good vendor image and an SPI programmer either way.

### In-band, via the board's UEFI Shell (no programmer)

Place `FlashUpdate.efi`, `startup.nsh`, and your `cix_flash_all.bin` at the root of the board's EFI System Partition (mounted at `/boot/efi` under Linux). Set the next boot to the built-in UEFI Shell (`efibootmgr --bootnext <shell entry>`), reboot, and the shell auto-runs `startup.nsh`, which flashes the image. After it completes, do a full power removal before powering on again, so the new firmware loads.

This runs from the currently booting firmware, so it cannot recover an image that does not boot. Use it once you trust the image.

### Off-board, with an SPI programmer

The BIOS chip is a socketed 1.8 V SPI part. With a CH341A-class programmer and a 1.8 V adapter, read a backup first, then write the full 8 MiB image. This is also the recovery path when an in-band flash leaves the board dead.

## Measuring a boot fix

For boot-reliability work, the serial console (UART2, 115200 8N1) is the oracle. The firmware prints progress post-codes with a timestamp; the useful pair is `E1FF` (DxeMainEnd) and `E550` (BmAfterConsole). On a good boot the gap between them is about 4 s. What looks like a hang at the splash is usually a long gap: the boot manager is waiting on a device and reaches `E550` minutes later. Record the gap for every boot in a loop of cold boots and warm resets, before and after a patch, and report the counts. Issue #2 is the worked example.
