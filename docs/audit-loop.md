# North Star Audit Loop

North star: the system is secure, correct, efficient, reliable, clean and operable, proven by evidence, not opinion. The loop ends when proven defects run out, not when auditors run out of ideas.

Applies to every audit of JP's apps, repos and production websites, whether run by Grok bots, Claude, Cursor or dev agents (house rule, JP 2026-11-14).

## 0. Setup (once per project)
1. Write `tasks/qa-rubric.md` with the six areas: Security, Correctness, Optimization, Reliability, Cleanup, Operability. For each area, write fixed, objective definitions of scores 3, 4 and 5 (5 means no proven HIGH or MED, and all of that area's measured gates pass).
2. Define the measured gates, tuned to the stack:
   - Mutation testing (e.g. Stryker) with a kill rate of at least 70%.
   - Coverage on core modules of at least 80% branch and 90% line.
   - Property-based tests (e.g. fast-check) for parsers, math and input handling.
   - A worst-case load/cron simulation with at least 50% headroom against every platform limit (CPU, requests, DB ops, memory, quotas).
   - A bundle/size budget, dead-code scan (e.g. knip), and a full-tree dependency audit at moderate level with a target of 0 moderate, high or critical.
   - Production websites: add Lighthouse/Core Web Vitals budgets, accessibility (axe) with 0 serious, security headers/TLS checks, broken-link and SEO checks.
3. Create `tasks/audit-rounds.md` (scorecards), `tasks/redteam.md` (attack log), `tasks/decisions-log.md`, and `tasks/backlog-v2.md` for out-of-scope ideas.
4. Set the stop conditions: a cap of 8 rounds; 80% usage stop (or the project's usage rule); escalate to the owner on a 2-round plateau, any spend, secrets, scope change or veto.

## 1. Each round
1. Freeze the audited commit (record the SHA).
2. Run all measured gates; record the raw numbers.
3. Two **blind** auditors on different models score every area independently against the rubric. If any area differs by 2 or more, a third auditor breaks the tie. The **lower** score counts.
4. A red-team agent runs real attacks against a local build only (never production without the owner's GO) and logs each attack and its result in `tasks/redteam.md`.
5. Write the scorecard: one line per area plus measured numbers plus the red-team tally (tried, blocked, passed through).

## 2. Evidence rules (anti-hallucination)
- **No proof, no bug.** A HIGH or MED counts only with a failing test, a reproduced attack, or a tool-measured number. Otherwise it is logged as *unconfirmed* and does not block.
- **Scope freeze.** Every finding must map to a rubric line. Anything else goes to `tasks/backlog-v2.md`.
- **Fixes ship with tests.** Every fixed HIGH or MED adds a regression test that failed before the fix.
- Anything that needs live keys or production access is marked *unverified pending secrets*, not passed.

## 3. Fix round
The builder fixes all proven HIGH, MED and LOW items on the branch, keeping all tests green. Dependency or tooling changes go to the council (or the reviewer) first. Then go to the next round.

## 4. Exit bar (all required)
- Every area 5/5 (the lower of the blind scores).
- Every measured gate passes.
- Every red-team attack fails.
- Zero proven HIGH or MED.
- Two clean rounds in a row (a round with only LOW or style items counts as clean).
- Council/reviewer approval (e.g. 3/3, with a Security veto).

## 5. Diminishing returns
If the count of proven HIGH or MED items doesn't shrink for 2 rounds, or the round cap is hit, stop and escalate to the owner with the scorecard history and a recommendation (ship with known LOWs, narrow scope, or invest more).

## 6. Release
Passing the loop never merges or deploys by itself. Merge, apply and deploy each still need the owner's separate named GO, batched into one release card listing results, unverified items, and the secrets or decisions needed.
