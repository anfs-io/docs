---
---

# Plan 05 — The round trips as "a stack onto a target"

Needs plans 01, 02 and 04.

## Context — read these files first

- `units/stacks.md`; plans 01, 02, 04 and their logs
- anfs-dev `packages/dev/tests/{harness,roundtrip,roundtrip-macos,roundtrip-sources}`,
  `fixtures/acme/`; stack `packages/firstmate/tests/roundtrip`

## Implementation

- The round trip is: make a box (plan 04), `anfs install <stack> -t <box>`, the checks,
  `anfs remove <stack> -t <box>`, then the diff of the box's HOME against before. The box setup
  and the source mounting the scripts do by hand go to plan 04's code.
- The acme fixture gets `stacks/acme/stack.yml` naming every resource kind it holds, so the
  round trip installs it as a stack (`anfs src install acme` stays covered by one check).
- `roundtrip-macos` the same on a tart box; firstmate's round trip on whichever box it needs.
- The harness keeps only what is not a box or an install: the checks and the diff.

## Test Spec

The scripts are run by a person (they need boxes). `bats` covers what can be stubbed: the
argument handling, and that the scripts call `anfs install|remove -t`.

## Verification

- [ ] Deferred to the person: `roundtrip debian`, `roundtrip fedora`, `roundtrip-macos`,
      firstmate's, each clean
