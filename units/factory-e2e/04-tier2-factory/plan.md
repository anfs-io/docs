---
---

# Plan 04: Tier 2, the factory claim

## Context: read these files first

- Plan 03 (its assertion helpers are reused).
- `~/evals/firstmate/AGENTS.md` §7: dispatch policy. Independent work runs in parallel with no concurrency cap; only real dependencies are serialized.
- `fixtures/prd-greet.md` and the `json → usage` dependency.

## Overview

`tests/e2e --tier 2` hands Firstmate a PRD and asserts that it breaks the PRD into tasks, runs
independent work in parallel, respects the dependency, and finishes with everything merged and
green.

## Implementation

1. Start from a fresh `greet` project and send `firstmate note /fixtures/prd-greet.md`.
2. The harness approves merges automatically: on every `needs-decision` status, or when the primary's pane asks for a merge, send an approval prompt naming that task.
3. Assertions:
   - At least 4 `task.dispatched` records, all `kind: ship`.
   - **Parallelism:** at least 2 tasks had overlapping `working` → `done` intervals in ledger timestamps.
   - **Dependency:** the task that implements json was dispatched after the usage task's `task.merged`, or the two were done as one task. Record which happened and fail on neither.
   - **Outcome:** `bats test/` passes on `main` in full, and no `fm/*` branches remain unmerged.
4. Timeout is 45 min. When it fires, dump everything (plan 05).

## Test Spec

- Run it 3 times; pass means at least 2 of 3.
- Append a row per run to `tests/results/tier2.tsv`: date, pass/fail, wall time, tasks dispatched, maximum concurrency, Firstmate ref, herdr version.

## Verification

- `log.md` summarises the 3 runs and any failure patterns, such as over-splitting, a missed dependency, or a merge stall.
