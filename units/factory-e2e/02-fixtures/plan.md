---
---

# Plan 02: fixtures

## Context: read these files first

- `notes/orchestrators-eval.md (this repo)` §5: the `greet.sh` demo used in the spike.
- `sources/core/packages/node/home/.config/mise/conf.d/node.toml`: bats comes from `npm:bats`.

## Overview

Create `sources/stack/packages/firstmate/tests/fixtures/`. It holds a small project with failing
acceptance tests, plus the two inputs.

## Implementation

1. **`fixtures/greet/`**, copied into the box and `git init`-ed at run time so each run starts fresh:
   - `greet.sh`: `echo "hello $1"`.
   - `README.md`.
   - `test/shout.bats`: `greet.sh --shout bob` gives `HELLO BOB`, and `greet.sh bob` is unchanged.
   - `test/usage.bats`: no name or an empty name gives a usage message on stderr and exit 2.
   - `test/count.bats`: `greet.sh --count 3 bob` prints the greeting 3 times.
   - `test/from-file.bats`: `greet.sh --file names.txt` greets each line.
   - `test/json.bats`: `greet.sh --json bob` gives `{"greeting":"hello bob"}`. It **depends on** usage handling: `--json` with no name gives the JSON error `{"error":"usage"}` and exit 2.
   - Every feature test fails against the initial `greet.sh`.
   - Add a `test/baseline.bats` that passes from the start, to catch broken setups.
2. **`fixtures/spec-shout.md`:** one task (`--shout`), with its acceptance criteria and a pointer to `test/shout.bats`.
3. **`fixtures/prd-greet.md`:** a short PRD.
   - Goal, the four features (usage, count, from-file, json) and their acceptance tests.
   - It states plainly that json depends on usage, and that the rest are independent.
   - No implementation hints.

## Test Spec

- In a box with bats: the baseline passes and each feature `.bats` file fails on the initial repo.
- A hand-written reference solution, kept outside the fixtures in `tests/reference/`, makes every test pass. This proves the tests can be satisfied.

## Verification

- The tests are Bash 3.2–compatible, because macOS runs them too.
