# EMCC notes for REA

This file is the EMCC record for JP's fork of [morluto/rea](https://github.com/morluto/rea) (`rea-agents` 6.4.0 in this tree). It is an operator note. It does not change REA's MIT grant.

## Hard usage rule

**REA is used ONLY on software JP owns or on open-source projects whose licenses permit analysis. NEVER on Adobe or any proprietary binary whose EULA bans reverse engineering.**

That includes Photoshop, Lightroom, Camera Raw, and any other Adobe application or plugin. It also includes any other proprietary binary whose license or EULA forbids reverse engineering, decompilation, or circumvention. A local copy, a "research" label, or a showcase write-up does not make that target allowed.

Allowed examples for this fork:

- Software JP owns, including the DarkCockpit sources JP wrote (`sidecar/`, `frontend/`, `lua/`).
- Open-source projects whose licenses permit analysis. darktable 5.4.1 is GPL-3.0-or-later in the module sources checked below, so reading that source is allowed. The stock darktable binary may be run as the program it is; this note does not authorize patching it to defeat a license check.

When the target is open source, read the source first. Use REA when it adds evidence the source does not already give (call graphs, runtime traces, artifact identity). Do not point REA at a proprietary binary to recover behavior that an open-source implementation already documents.

### Out-of-bounds defaults already in this tree

REA does not ship an Adobe path as a default target. These published examples still point at proprietary software and are out of bounds for EMCC use:

| Where                            | What it points at                                                   | Why it is out of bounds                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `README.md` quick start          | "the Notes app" (Apple Notes)                                       | Proprietary Apple application. The prompt tells an agent to investigate it.                                                   |
| `README.md` showcases            | DX-Ball, Notion, TH04 (Touhou Project, PC-98)                       | Proprietary games and a proprietary Electron app. The reconstruction repos do not make the original binaries allowed targets. |
| `docs/cli.md`                    | A 334 MB Obsidian application Evidence file, used as a size example | Obsidian is proprietary. Do not recreate that capture.                                                                        |
| `src/application/Doctor.test.ts` | The string `/Applications/Notes.app` as a doctor-check fixture      | A unit-test stub, not a target the CLI launches. Still a proprietary path. Do not supply a real Notes bundle.                 |

Allowed fixture, not a proprietary app: `website/public/examples/notes-electron/` is a source-owned six-file teaching app (`website/evidence/guide-examples.md`). EMCC may analyze that directory. It is not Apple Notes.

Contract examples under `/Applications/Example.app` and `/Applications/Electron.app` are placeholders. They are not Adobe, and they are not permission to analyze whatever happens to be installed at those paths.

## License

| Item                                                                                           | License                                                                                                                                                                                                                                                                                                                        | Where                               |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------- |
| REA itself (`package.json` `"license": "MIT"`, file `LICENSE`)                                 | MIT. Copyright (c) 2026 morluto                                                                                                                                                                                                                                                                                                | `LICENSE`                           |
| Locked npm tree (`package-lock.json` `license` fields, 562 package entries including the root) | MIT 482, Apache-2.0 31, ISC 17, MPL-2.0 12 (`lightningcss` and its platform packages), BSD-3-Clause 9, BlueOak-1.0.0 5, BSD-2-Clause 2, CC0-1.0 1 (`@iconify-json/simple-icons`), 0BSD 1 (`tslib`), MIT AND Zlib 1 (`pako`), MIT AND BSD-3-Clause 1 (`sha.js`). No GPL, LGPL, AGPL, SSPL, or unlicensed entry in the lockfile. | `package-lock.json`                 |
| Vendored EVMole license text                                                                   | MIT. Copyright (c) 2023 Maxim Andreev. Packaged because the npm tarball omits the license file. Engine is the locked dependency `evmole@0.9.3`.                                                                                                                                                                                | `third_party/evmole/LICENSE`        |
| pwndbg license text                                                                            | MIT. Copyright (c) 2026 Dominik 'Disconnect3d' Czarnota. The engine is not vendored.                                                                                                                                                                                                                                           | `third_party/pwndbg/LICENSE.md`     |
| NativeAOT parser note                                                                          | MIT. Copyright 2023-2025 Washi. Adapts Washi1337/ghidra-nativeaot (MIT) and cross-checks .NET runtime MethodTable layout (MIT).                                                                                                                                                                                                | `licenses/nativeaot-parser-MIT.txt` |

### Caller-supplied engines (not in the npm lockfile)

Recorded in `third_party/README.md`, `third_party/pwntools/README.md`, `third_party/pwndbg/README.md`, and `third_party/web-network-captures/README.md`. REA does not vendor these implementations.

| Engine                                            | Recorded profile                                                                                       | License as recorded here                                                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| jadx-headless-mcp                                 | v0.7.1, commit `5844800a486d2248fa949b74c9896285f5b54de2` (submodule not initialized in this checkout) | Apache-2.0                                                                                                   |
| Binwalk                                           | v3.1.0                                                                                                 | MIT                                                                                                          |
| Unblob                                            | 26.6.4                                                                                                 | MIT                                                                                                          |
| pwntools                                          | 4.15.0                                                                                                 | MIT                                                                                                          |
| pyelftools                                        | 0.33                                                                                                   | Public domain                                                                                                |
| Unicorn                                           | 2.1.2                                                                                                  | GPL-2.0. Upstream `COPYING` at tag 2.1.2 is the GNU GPL version 2 text (June 1991), not an "or later" grant. |
| pwndbg                                            | 2026.09.15                                                                                             | MIT                                                                                                          |
| mitmproxy                                         | v12.2.3                                                                                                | MIT                                                                                                          |
| har-schema, jsonc-parser, ajv, ajv-formats        | locked npm versions                                                                                    | ISC or MIT, as in the lockfile counts above                                                                  |
| Appium ApiDemos fixture (downloaded, not tracked) | v6.0.18                                                                                                | Apache-2.0                                                                                                   |

Hopper, Ghidra, and IDA are bring-your-own. This checkout does not include their source or licenses. Hopper is proprietary. `rea setup` must show its plan and wait for approval before installing Hopper.

## Obligations if this fork is copied or combined

MIT (REA, EVMole, pwndbg, NativeAOT note, and the MIT npm packages): keep the copyright notice and the permission notice in every copy or substantial portion.

Apache-2.0 (MCP SDK, TypeScript, playwright-core, canonicalize, and the other Apache packages in the lockfile; jadx if you ship it): keep the copyright, license, and any NOTICE text. State changes to Apache-licensed files you modify.

BSD-2-Clause and BSD-3-Clause: keep the copyright notice and the disclaimer. BSD-3-Clause also forbids using the authors' names to endorse a derived product without permission.

ISC: keep the copyright notice and the permission notice.

MPL-2.0 (`lightningcss*`): those files stay under MPL. If you distribute a binary that includes them, you must offer the corresponding MPL source for those files. MPL-2.0 section 3.3 allows the larger work to be under GPL-3.0.

0BSD, CC0-1.0, BlueOak-1.0.0, and pyelftools' public-domain statement: no copyleft. BlueOak still expects the license text to travel with the code when you redistribute that code.

Unicorn GPL-2.0: if you distribute Unicorn or a work based on it, you must license that work under GPL-2.0 and provide corresponding source. GPL-2.0-only is not compatible with GPL-3.0. Do not link Unicorn into DarkCockpit or into any other GPL-3.0 program. Leaving Unicorn as a separate optional process that REA does not ship is mere aggregation, which this note relies on. EMCC analysis of darktable does not need Unicorn.

## Compatibility with GPL-3.0 use

REA's own license is MIT. The Free Software Foundation treats the MIT/Expat license as compatible with GPL-3.0: MIT code may be included in a GPL-3.0 work, the combined work is distributed under GPL-3.0, and the MIT notices stay on the MIT files.

Using this MIT tool to study GPL-3.0 darktable source does not relicense REA and does not relicense darktable. darktable's module sources checked for the trace say "GPL version 3 or any later version". `darktable-5.4.1/LICENSE` in DarkCockpit is the GPL-3.0 text. Some darktable headers in that tree (`src/common/iop_order.h`, `src/iop/iop_api.h`) are LGPL-3.0-or-later; LGPL-3.0 code may be combined into a GPL-3.0 program.

If a later change distributes REA modifications and DarkCockpit as one program, ship that combination under GPL-3.0 and keep the MIT notices. Do not add Unicorn (GPL-2.0-only) to that combination.

Apache-2.0, ISC, BSD, MPL-2.0, 0BSD, CC0, and BlueOak-1.0.0, as used in the locked tree above, are compatible with distributing a larger GPL-3.0 work. MPL files remain MPL inside that work.

This is an operator record of the licenses in the tree, not legal advice.

## Setup that was actually run

Host: Ubuntu 24.04. The shell's first `node` was `/exec-daemon/node` v22.14.0, which is below `package.json` `engines` (`^22.19.0 || ^24.11.0 || >=26`). `.nvmrc` pins 24.18.0. `packageManager` is `npm@11.16.0`. Node 24.18.0 ships that npm.

```bash
export NVM_DIR="$HOME/.nvm"
. "$NVM_DIR/nvm.sh"
nvm install 24.18.0
nvm use 24.18.0
export PATH="$HOME/.nvm/versions/node/v24.18.0/bin:$PATH"
hash -r
node -v   # v24.18.0
npm -v    # 11.16.0

cd /path/to/rea
npm ci
npm run build:cached
node scripts/rea.mjs --version
node scripts/rea.mjs doctor
npm run test:local -- src/domain/artifactPathSyntax.test.ts
node scripts/rea.mjs analyze-javascript-application \
  website/public/examples/notes-electron --json
```

Results on 2026-10-11:

- `npm ci`: 377 packages added, 0 vulnerabilities. `esbuild@0.25.12` postinstall was not in the allow-scripts list. The build still compiled. npm printed an upgrade notice for npm 12; it was not installed.
- `npm run build:cached`: turbo `artifacts:skills` succeeded (metadata check, `tsc -p tsconfig.build.json`, skill metadata).
- `node scripts/rea.mjs --version`: `6.4.0`.
- `node scripts/rea.mjs doctor`: Node 24.18.0 and Ubuntu 24.04 healthy. Hopper, Ghidra (`GHIDRA_INSTALL_DIR` unset), and IDA MCP were missing. The installed skill and Cursor MCP registration were absent. The process exited 0 with `healthy: false`. No `rea setup` was run, because setup writes agent config and can install Hopper, and both require an approval step this session did not take.
- `npm run test:local -- src/domain/artifactPathSyntax.test.ts`: vitest 5.0.3, 1 file, 4 tests passed. This is a source-level smoke, not the full suite and not a real Hopper/Ghidra/browser provider run.
- `analyze-javascript-application` on `website/public/examples/notes-electron`: completed. Evidence id `ev_f1297b4253e8b5c5f30d9fccf49f120badf027da4ad1c125e3b80946459aa074`, predicate `rea.javascript-application-analysis`, confidence `derived`.

Native call graphs and runtime traces were not run. Hopper, Ghidra, and IDA are not installed here, and darktable was not compiled or executed. The darktable module trace is source observation. See `docs/traces/darktable-5.4.1-modules.md`.
