---
---

# Plan 05: run hygiene

## Context: read these files first

- `tests/e2e` as of plans 01–04.
- `sources/core/packages/anfs/tests/roundtrip`: cleanup and the `--keep`-style options.

## Overview

Make failures cheap to diagnose and runs safe to repeat.

## Implementation

- **Usage:** `tests/e2e [debian|fedora] [--tier 1|2|all] [--keep]`. The default is `debian --tier 1`.
- **Timeouts:** each step has its own (dispatch 5 min, done 10 min, Tier 2 total 45 min) and its own exit code.
- **Failure dump.** Write to `tests/results/<ts>/`:
  - the ledger;
  - `state/*.status` and `state/*.meta`;
  - `data/backlog.md`;
  - `herdr agent read <pane> --source recent --lines 400` for every pane;
  - `herdr status --json`;
  - `git log --all --oneline` for the project.
  - The token value is filtered out: `sed` it out by value before writing.
- **`--keep`:** leave the box and the herdr session running, and print `harness shell debian` plus `firstmate attach` instructions.
- **Always tear down at the end unless `--keep`:** `firstmate down`, then the box is removed.
- `tests/results/` is git-ignored except `tier2.tsv`.

## Test Spec

- Kill the box mid-run. The next run starts clean.
- A forced failure produces a complete dump that contains no token.

## Verification

- The README documents usage, cost expectations per tier, and how to read a dump.
