# Contributing

## How a fix moves

Each fix follows the same loop, in the open.

1. **Issue.** A board problem is filed with enough detail to reproduce or recognise it: firmware version, symptom, serial log where possible.
2. **Triage.** We label it by layer and feasibility (below).
3. **Patch.** A small diff against a pinned CIX commit, in `patches/`.
4. **Build and flash.** Built from source, flashed on real hardware.
5. **Measure.** The before and after is recorded with evidence, serial logs and counts, not an impression.
6. **Publish.** The result goes in the issue and the patch lands.

## What is in scope

The board's firmware is split across stages. What we can and cannot change:

- **Patchable** (rebuilt and re-signed with the OEM key): BL31 (TF-A runtime, PSCI and power management), BL32 (OP-TEE), BL33 (UEFI). Boot hangs, power-management and PSCI issues, CPU errata, and UEFI-level bugs live here.
- **Off-limits:** bootloader1 (BL1/BL2). It is verified by the Security Enclave against a fused CIX key and cannot be self-signed. See `docs/bootloader-model.md`.

Many board complaints are not firmware fixes at all: kernel drivers, userland stacks such as the NPU tooling, and hardware are out of this repo's scope. We triage honestly and say so.

## Issue labels

- `firmware-fixable`: in BL31/BL32/BL33, patchable here.
- `kernel`: belongs in the Linux kernel, not firmware.
- `hardware`: a hardware limitation or defect.
- `bootloader1-locked`: would need the SE-verified stage; not possible.
- `needs-info`: not enough detail to act.
- `needs-triage`: not yet sorted.

## Rules

- Never commit closed binaries or a built or signed image. Patches and tooling only.
- Patches keep the SPDX header and license of the file they modify.
- Claims carry evidence. "It hangs less" is not a result. "3 of 4 cold boots to 0 of 20" is.
