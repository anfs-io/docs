---
---

# Plan 04: install round trip (no LLM)

## Context: read these files first

- `sources/anfs/packages/anfs/tests/roundtrip`: the template (`box()`, `check`, `$HOME` before/after diff, `IGNORE_RE`).
- `sources/anfs/packages/anfs/tests/harness`: the box lifecycle.
- Plans 01–03 of this unit.

## Overview

`sources/ai/packages/firstmate/tests/roundtrip [debian|fedora]` proves the package installs,
starts, stops and uninstalls cleanly in a fresh box. No model calls are made, so it is cheap and
deterministic.

## Implementation

Copy the structure of `anfs/tests/roundtrip` and trim it.

1. Start from `bootstrap`, or from the `herdr` snapshot if plan 01 made one. Mount sources as roundtrip does, then run `bash /src/anfs/install.sh`.
2. Snapshot `$HOME`.
3. `ppm install ai/firstmate`.
4. Checks:
   - `check "repo at pinned ref"`
   - `check "doctor clean" 'firstmate doctor --no-auth'`. Add a `--no-auth` flag to `doctor` in plan 03 if needed.
   - `check "up" 'firstmate up --headless'`
   - `check "agent listed" 'herdr --session firstmate agent list | jq -e ...'`. The agent can't authenticate; the check is only that the claude process is detected in the pane.
   - `check "down" 'firstmate down'`
   - `check "no processes"`
5. `ppm remove ai/firstmate`, then diff `$HOME` against the snapshot, extending `IGNORE_RE` for brew/mise caches as roundtrip does.

## Test Spec

- `tests/roundtrip debian` passes from a cold box.
- Deliberately break the `ref` and confirm the run fails at the install check.

## Verification

- Record the run time and any `IGNORE_RE` additions, with reasons, in `log.md`.
