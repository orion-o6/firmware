# The boot trust model, and what we can patch

The Orion O6 boots through a chain of signed stages. What matters for patching is which stage is verified against a key we cannot produce, and which stages are signed with a key we have.

## The split

CIX splits the firmware into stages carried in separate images.

- **bootloader1.img** holds BL1/BL2 and the PM firmware (the SCP), the first code the Security Enclave runs. The SE verifies it against a key fused into the chip. You cannot self-sign it: a byte-perfect image re-signed with your own key is rejected, and a modified image is rejected. This stage is off-limits.
- **bootloader2.img** holds BL31 (the TF-A runtime: PSCI, CPU power up and down, suspend and resume, the secure monitor) and BL32 (OP-TEE). It is signed with the OEM key, which the packaging tool holds.
- **bootloader3 / BL33** is the UEFI firmware, also OEM-signed.

## What this means

The OEM key re-signs the FIP (BL31, BL32, BL33). The board accepts an image whose BL31, BL32, and BL33 we rebuilt and re-signed, as long as bootloader1 stays the genuine CIX-signed blob. This is confirmed on real hardware: a self-built community BIOS, signed with the OEM key, flashes and boots.

So most of the firmware is within reach:

- BL31 (TF-A): power management and PSCI, which is where wake-from-deep-sleep lives, and where CPU errata workarounds belong.
- BL32 (OP-TEE): the secure OS.
- BL33 (UEFI): boot-level and firmware bugs, for example an intermittent boot hang.

The only thing beyond reach is bootloader1, the root of the chain. Everything the runtime, the secure OS, and the UEFI do is ours to change.

## Frequency lives in the SCP, so it is off-limits

CPU DVFS is not in the FIP. The operating points (the per-domain frequency and voltage tables) live in the PM firmware, the SCP, which is inside `bootloader1.img`. The SCP hands them to the OS over SCMI; BL33 only transcribes what SCMI reports into the ACPI `_CPC` objects that `cppc_cpufreq` reads. So BL33 can mask or clamp what Linux sees, but it does not set the ceilings. The authority is the closed SCP.

That has a direct consequence: raising a frequency ceiling is not a patch we can build and re-sign. Current CIX SCP builds compile the tables into the binary and ignore the once-editable `pm_config` SPI partition. The only routes to change them are a genuine CIX-signed older bootloader1 whose SCP still reads `pm_config`, or CIX reopening the config upstream. Anything advertised through `_CPC` (per-core ceilings, the spread across identical cores) traces back here, not to a table we own.

## Recovery

Flashing an image the board rejects, or one that hangs before it can be reflashed in-band, leaves the board unbootable. The BIOS SPI chip is socketed, so recovery is off-board with an SPI programmer (CH341A class). Keep a known-good vendor image. Do not flash without this.
