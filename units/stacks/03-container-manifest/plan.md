---
---

# Plan 03 — `container.yml`, the hook for every resource, pcm's memory

Independent of plans 01 and 02.

## Context — read these files first

- `units/stacks.md`
- core `packages/pcm/home/.local/lib/pcm/{sources,validate,compose,runtime,inspect}.sh`, the
  `pcm` command; `x-pcm.allow_mounts` in `validate.sh` (pcm's one compose extension today)
- the containers that exist: `grep -rl . */containers/*/compose.yml` across the sources (stack's,
  anfs-dev's test boxes)
- anfs-dev `packages/dev/home/.config/git/ppm-hooks/pre-commit` and `lib/package-meta.sh` — the
  version bump and the "every `packages/*` has a `package.yml`" rule
- pmm's `image.yml` and `machine.yml` readers, for how a resource manifest is read elsewhere

## Implementation

### 1. One manifest per resource, `<kind>.yml`

Every resource directory has a manifest named for its kind: `package.yml`, `container.yml`,
`machine.yml`, `image.yml`, `network.yml`, `stack.yml`. Each has at least `version:`.

### 2. `container.yml`

```yaml
version: 0.1.0
description: The anfs dashboard
upstream: docker.io/library/node:22        # what it is built from (attribution)
license: MIT
url: https://github.com/anfs-io/anfs-ui
memory: 512m                               # what it needs; also its limit
```

- Optional keys beside `version`; unknown keys are a validate error. `compose.yml` stays plain
  compose: nothing anfs needs moves into it. `x-pcm.allow_mounts` stays where it is (plan 04
  revisits it).
- `pcm list` gains `VERSION`; `pcm show` prints the manifest's fields; `--json` keys `version`,
  `description`, `upstream`, `license`, `url`, `memory`.
- A container with no `container.yml` is still valid (version `-`), with a validate warning.
  Every container in core, stack and anfs-dev gets one in this plan.

### 3. The hook bumps any resource's manifest

- A change under `packages/<p>/`, `containers/<c>/`, `machines/<m>/`, `machines/images/<i>/`,
  `machines/networks/<n>/` or `stacks/<s>/` bumps that directory's `<kind>.yml` `version`
  (patch), as `package.yml` is bumped today, with the same "already ahead of what was pushed"
  rule.
- The rule "every directory has its manifest" applies to each of those resource directories.
- One table in `package-meta.sh` maps directory pattern to manifest name, so a new kind is one
  line.

### 4. Memory

- `memory:` becomes the container's limit: pcm passes it to podman (`--memory`) whatever compose
  says, and validate errors when compose sets a different `mem_limit`.
- **Before `pcm up`**: add up `memory:` of the running containers plus the new one. On Linux
  compare with the host's available memory; on macOS with the podman machine's. Too little:
  - nothing running (macOS): resize the podman machine (`podman machine stop`, `set --memory`,
    `start`) to the sum plus headroom, say so, then start;
  - something running: refuse (exit 1) with the numbers, the containers a resize would stop,
    and the command to do it (`pcm machine resize`).
- `pcm machine resize [size]` (macOS): stop, set, start the podman machine; with no size, the
  sum of what is defined to run plus headroom. It confirms when containers are running.
- No automatic shrinking.

## Test Spec

pmm's and pcm's bats suites are off (2026-10-19): write pcm's tests (validate, list/show, the
memory check with a stubbed `podman machine inspect` and `free`/`vm_stat`), list them in the log
as written and not run, and check `pcm list`/`show`/`validate` by hand in an isolated HOME with
podman stubbed.

- anfs-dev `packages/dev/tests/hooks.bats`: a change in `containers/x/`, `stacks/y/` and
  `machines/images/z/` bumps that manifest; a missing manifest fails the commit.

## Verification

- [ ] `packages/anfs/tests/all`; anfs-dev `bats packages/dev/tests`
- [ ] A commit in a worktree of stack touching a container bumps its `container.yml`
- [ ] Deferred to the person (needs podman): `pcm up` refusing and resizing on macOS
