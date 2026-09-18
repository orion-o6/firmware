# Patches

One directory per fix. Each holds the diff against a pinned CIX commit and a short note stating:

- the issue it addresses,
- the exact submodule and commit it applies to,
- how it was verified (with evidence).

Apply a patch against a matching checkout of the CIX community source (see `../docs/build-and-flash.md`), then build.

Nothing here changes bootloader1. Every patch targets BL31, BL32, or BL33.
