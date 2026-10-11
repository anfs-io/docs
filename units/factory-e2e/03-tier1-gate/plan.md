---
---

# Plan 03: Tier 1, the gate

## Context: read these files first

- `~/evals/firstmate/docs/fleet-ledger.md`: event types (`task.dispatched`, `task.status`, `task.merged`, `task.cleaned_up`) and reader rules (ignore unknown events, tolerate duplicates).
- `~/evals/firstmate/bin/fm-crew-state.sh` and the `state/<id>.meta` keys (`backend`, `herdr_session`, `herdr_pane_id`, `branch`, `kind`).
- The `firstmate` launcher (`firstmate-foundation` plan 03): `project add`, `note`, `prompt`, `status`.

## Overview

`tests/e2e --tier 1` takes one spec through the whole chain and checks every stage from outside
the model.

## Implementation

Steps and assertions, with each wait a polling loop with a timeout:

1. `firstmate up --headless`, then `firstmate project add /fixtures/greet` (local-only, yolo off).
2. `firstmate note /fixtures/spec-shout.md`.
3. **Dispatched:** within 5 min, the ledger has a `task.dispatched` record with `kind: ship` and `project: greet`. Record the task id.
4. **In herdr:** `state/<id>.meta` has `backend=herdr`, and its `herdr_pane_id` is in `herdr --session firstmate agent list`.
5. **Done:** within 10 min, the ledger shows `task.status` `done`, and `state/<id>.status` ends with `ready in branch <b>`.
6. **Correct:** `bats test/shout.bats` passes on `<b>`. On `main` it still fails, and `main` has not moved.
7. **Authority gate:** read the primary's pane with `herdr agent read firstmate`. It must be asking for a merge decision, and nothing has merged yet.
8. **Approve:** `firstmate prompt "Approved — merge <id> locally."`
9. **Merged:** the ledger has `task.merged` (`via: local`) and then `task.cleaned_up`. `bats test/` passes on `main` except for Tier 2's tests, which are still pending.
10. `firstmate down`. No processes remain.

## Test Spec

- Two cold runs in a row pass.
- Break `spec-shout.md`, for example by pointing it at a nonexistent test, and confirm the run fails at a named step.

## Verification

- `log.md` records wall time per step and token usage (from `claude` usage output, if available).
