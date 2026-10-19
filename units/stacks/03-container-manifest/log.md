# Log — 03 container-manifest

2026-10-19. Branch `stacks-03` in core, stack, anfs-dev, docs. Not pushed.

## Built

- **pcm `container.yml`** (`lib/pcm/manifest.sh`): `version description upstream license url
  memory`; unknown key, missing version, unparsable memory, non-map: validate errors; no file:
  warning. `pcm list` gains `VERSION` (`--json`: all six keys), `pcm show` prints the fields.
- **memory: as the limit**: the generated override sets `mem_limit` on every compose service;
  a different compose `mem_limit` is a validate error.
- **Room check in `pcm up`** (after validate): Linux, what starts vs `free`'s available; macOS,
  what runs + what starts vs `podman machine inspect` memory. Too little on macOS with nothing
  running: stop/set/start to sum + `PCM_MACHINE_HEADROOM` (1024 MiB); with something running:
  exit 1 with numbers, the containers a resize stops, `pcm machine resize <size>`.
  `pcm machine resize [size]` (macOS) confirms when containers run (`-y`). No shrinking.
- **pre-commit hook**: `META_KINDS` table in `lib/package-meta.sh` (one line per kind); bump and
  "every resource has its manifest" for packages, containers, machines, images, networks,
  stacks. A manifest with no `version:` gets `0.1.0` (after its opening comment).
- **Manifests**: `container.yml` for stack's 11 containers and anfs-dev's 2 test boxes
  (description, upstream, url where compose/Containerfile says them; no license or memory
  anywhere, so left out). `version: 0.1.0` added to core's 2 `image.yml`, anfs-dev's 2
  `image.yml` and 1 `machine.yml`; pmm's `PMM_KEYS` accepts `version`.
- Docs: core `AGENTS.md` new "pcm: containers" section, `skills/pcm-containers` Files table,
  pcm help epilogue; anfs-dev `AGENTS.md` "Git Hooks".

## Commits

- core: `58eb083` pcm container.yml + memory; `602f350` pmm version key, image.yml versions
- stack: `c9fe24e` container.yml for every service
- anfs-dev: `3d6c7a9` pre-commit for every resource; `641d5d3` test boxes' container.yml,
  version on images/machine

## Tests

- core `packages/anfs/tests/all`: 304 passed, 0 failed (pmm and pcm off).
- anfs-dev `bats packages/dev/tests`: 36 ok (7 new pre-commit tests in `hooks.bats`).
- **Written, not run under bats**: `core/packages/pcm/tests/manifest.bats` (16 tests: validate,
  mem_mib, list/show/--json, override mem_limit, the macOS/Linux memory check, machine resize);
  `sources.bats` list expectations updated for `VERSION`. Their bodies were exercised once
  through a throwaway plain-bash shim (isolated temp dirs, podman/free stubbed): manifest 16/16,
  sources 13/13, validate 9/9, machine 7/7, ingress, privileges, remove all pass; edges has 5
  shim-only failures that are identical without this change.
- By hand, isolated HOME (HOME, XDG_*, *_HOME unset, podman stub on PATH): `pcm list` (tsv,
  aligned, --json), `pcm show`, `pcm validate` on fixtures and on stack's and anfs-dev's real
  containers (no container.yml errors; the remaining errors are pre-existing env/privilege ones
  under a stubbed podman).
- Verification: a commit in a throwaway clone of stack (`stacks-03`, new hook) touching
  `containers/postgres/` bumped its `container.yml` to 0.1.1.

## Deviations and decisions

- **Linux check counts only what starts** against available memory (running containers already
  take theirs out of MemAvailable; adding them again would double-count). macOS counts both.
- **Multi-service definitions**: `memory:` is set as each compose service's `mem_limit` (each
  capped at the definition's total), not split between them.
- **`--json` keys are on `pcm list`** (show has no --json).
- `pcm machine resize` with no size: what runs (by container.yml memory) + headroom.
- Hook messages now name the directory (`meta: packages/pcm — ...`) since names repeat across kinds.
- A file directly under a kind's parent (`containers/README.md`, `machines/profiles.yml`) is in
  no resource.
- `upstream` is a string; telemetry and n8n list their images comma-separated.
- Not changed: core `AGENTS.md` "Lists and --json" table row for `pcm list` (stacks-01's
  section): it should gain `version`, `description`, `upstream`, `license`, `url`, `memory`.
- Test fixtures under `anfs-dev/packages/dev/tests/fixtures/acme/containers/` got no
  container.yml (not resources; pcm only warns).

## Not verified — the person checks by hand (needs podman)

- macOS: `pcm up` of a container with a large `memory:` with nothing running resizes the podman
  machine (stop, set --memory, start) and starts it; with something running it refuses with the
  numbers and the `pcm machine resize` command; `pcm machine resize` itself.
- That podman compose honours the override's `mem_limit` (`podman inspect` .HostConfig.Memory).
- `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests` (and pmm's, for the `version` key).
