---
plan: cli/02-ppm-status
job: ppm-status
date: 2026-10-19
---

# Log — cli/02-ppm-status

## What changed

| | Before | After |
| --- | --- | --- |
| `ppm list` | `PACKAGE INSTALLED` (`package installed`) | `PACKAGE STATUS VERSION` (`package status version`) |

- STATUS: `installed` (tracker's version equals, or is not older than, the source's `package.yml`),
  `outdated` (the source's is newer, by `sort -V`), `-` / JSON null (not installed).
- VERSION: the installed version, else the source's.
- `--installed` filters as before; a tracker whose source package is gone reads `installed`.

## Readers

- core `lib/anfs/stacks.sh` `stack_have_load`: `select(.status != null)`; its stub in `tests/stacks.bats`.
- core `AGENTS.md`: "Lists and --json" key row, and the stacks note on what `ppm list --installed` answers.
- anfs-dev round trips select by `.package` only: unchanged. stack: only `ppm list @...` in READMEs, no readers.

## Commits

- core `80e0c83` ppm list: PACKAGE STATUS VERSION (ppm bumped to 0.2.5 by the hook)

## Tests

- New `packages/ppm/tests/list.bats` (5): header, the three statuses (outdated by a bumped source
  version), an older source still `installed`, `--installed`, `--json` keys.
- core `packages/anfs/tests/all`: 330 passed, 0 failed (pmm and pcm off).
- anfs-dev `bats packages/dev/tests`: 36/36.
