# todo

## Open

- Operator review of the EMCC usage rule in `README.md` and `docs/EMCC-NOTES.md` before any analysis run.
- Do not run REA against Adobe or any proprietary binary whose EULA bans reverse engineering. Out-of-bounds examples are listed in `docs/EMCC-NOTES.md`.
- Optional later: `rea setup` only after a human approves the plan. Hopper, Ghidra, and IDA were absent on the 2026-10-11 smoke.
- Optional later: full `npm run test:local` or `npm run check:fast` on a machine with the pinned Node already first on `PATH`. The 2026-10-11 smoke was one source file (4 tests) plus CLI `--version`, `doctor`, and one JavaScript analysis.
- DarkCockpit sidecar: next serializers after exposure v7 and temperature v4 are `toneequal` v2 (rename the UI labels), one tone mapper (`filmicrgb` v6, `sigmoid` v3, or `agx` v7), and `colorbalancergb` v5. Assert `sizeof` before trusting them. See `docs/traces/darktable-5.4.1-sidecar-summary.md`.
- `denoiseprofile` v12: clone an in-app history step. Do not author `a[3]`/`b[3]` from zeros.

## Done this pack

- License, third-party obligations, and GPL-3.0 compatibility recorded in `docs/EMCC-NOTES.md`.
- Hard usage rule at the top of `README.md` and in `docs/EMCC-NOTES.md`.
- `npm ci`, `npm run build:cached`, CLI smoke, and a source test recorded with the commands that ran.
- darktable 5.4.1 module trace written from source blobs that match DarkCockpit.
