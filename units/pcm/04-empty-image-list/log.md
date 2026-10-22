---
unit: pcm/04-empty-image-list
job: pcm-remove-img
date: 2026-10-22
report: r-20261022-3
---

What: `pcm remove` passes `--rmi local` to compose down only when the definition has a `build:` service and its project has containers.
Why: podman-compose 1.6 down --rmi collects images from the project's containers and runs `podman rmi --ignore --force` even with none, so a stock-image definition printed "Error: image name or ID must be specified".
How: `builds_images` in remove.sh (shared with remove's summary); pcm's own podman list calls (one-offs, implode, snapshot images, networks, volumes) were already guarded or looped, so unchanged.

# Log: pcm/04-empty-image-list

## Tests

- tests/run.bats: a compose runner that, like podman-compose, prints podman's error for `--rmi`
  with no built image. A stock-image remove has no `--rmi` and no `Error:`; a `build:` definition
  with containers gets `--rmi local`, without containers it does not. Both fail without the fix.
- pcm's bats, each with `ANFS_TESTS_PMM_PCM=1`: all pass but the known manifest.bats "machine
  resize" and toolkit.bats "macOS: compose runs with the grant's token" failures.
- `packages/anfs/tests/all`: 662 passed, 0 failed (pmm and pcm off).

## By hand, after merge

Reinstall core/pcm (0.3.1), then with a local stock-image definition (e.g. busybox:1.37, no
`build:`):

    pcm up <name> && pcm remove -y <name>

No `Error: image name or ID must be specified` line. Close r-20261022-3 in BACKLOG.md.
