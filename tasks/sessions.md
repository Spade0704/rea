# sessions

## 2026-10-11 — EMCC setup and darktable 5.4.1 module trace

Workflow: Coordinate→Plan→Verify→Build→Test→Debug→Optimize→Validate.

Coordinate: Read `LICENSE`, `third_party/*/LICENSE*`, `third_party/README.md`, the pwntools/pwndbg/evmole/web-network notes, and `package-lock.json` license fields. Confirmed no open `rea/2026-10-11` branch. Read DarkCockpit `README.md`, `CLAUDE.md`, `docs/architecture-scoping.md`, and `sidecar/serialize/{exposure,temperature,fields}.py`.

Plan: Record MIT plus the locked-tree and caller-supplied licenses, state GPL-3.0 compatibility, put the hard usage rule on the README and in `docs/EMCC-NOTES.md`, build with the pinned Node, and trace darktable modules from source because the target is GPL-3.0-or-later.

Verify: Git blob SHA-1 of DarkCockpit `darktable-5.4.1` module sources matched `darktable-org/darktable` tag `release-5.4.1` (`c3f96cadf0ab739965a55f692a061070e19604fd`) for the files cited in `docs/traces/darktable-5.4.1-modules.md`, including `iop_order.c`.

Build: `nvm install 24.18.0` (`.nvmrc`), `npm ci`, `npm run build:cached`. The first `node` on `PATH` was v22.14.0 and is below `engines`.

Test: `node scripts/rea.mjs --version` → 6.4.0. `doctor` → Node and Ubuntu healthy; Hopper, Ghidra, and IDA missing; process exit 0 with `healthy: false`. `npm run test:local -- src/domain/artifactPathSyntax.test.ts` → 4 passed. `analyze-javascript-application` on `website/public/examples/notes-electron` returned evidence `ev_f1297b4253e8b5c5f30d9fccf49f120badf027da4ad1c125e3b80946459aa074`.

Debug: Unicorn 2.1.2 `COPYING` is GPL-2.0 without an "or later" grant, so it stays out of any GPL-3.0 combination. v5.0 pipe order is array order after `_ioppr_reset_iop_order`, not the legacy float keys. Tone-equalizer C member names do not match the UI labels.

Optimize: Did not compile darktable or install Hopper/Ghidra. Source already holds the param structs. REA native call graphs would not add the `$DEFAULT` tables.

Validate: Notes and the trace distinguish source observation from the unrun native providers. No Adobe target was opened.
