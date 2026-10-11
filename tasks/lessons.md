# lessons

## 2026-10-11 — README link parity and GPL-2.0-only Unicorn

`scripts/lib/docs-facts.mjs` compares the sorted non-anchor link targets of every `README*.md`. A new markdown link in `README.md` fails that check until the same target is added to every translation. The EMCC usage rule is plain text plus the path `docs/EMCC-NOTES.md`, with no new `](...)` target.

Unicorn 2.1.2, the optional ELF PLT emulator recorded in `third_party/pwntools/README.md`, ships GPL-2.0 `COPYING` with no "or later" grant. Do not link it into a GPL-3.0 program such as DarkCockpit. REA itself is MIT and can sit beside GPL-3.0 darktable as a separate tool; a single combined program would be distributed under GPL-3.0 with the MIT notices kept.

darktable 5.4.1 `v50_order[]` float keys are overwritten by `_ioppr_reset_iop_order`. Pipe order for a built-in v5.0 list is the array order.
