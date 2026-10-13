---
---

# Plan 01 — Move the dev and test assets to the `anfs-dev` source

## Context — read these files first

- `units/dev-source.md` — what moves, what stays, and why
- core `packages/dev/` — `install.sh` (`post_install` wires the hooks), `home/.local/lib/ppm/
  {hooks,container,vm,move,user}.sh`, `home/.config/git/ppm-hooks/`
- core `packages/anfs/tests/harness`, `roundtrip`, `roundtrip-macos`, `fixtures/acme/`
- core `containers/anfs-test-*/`, `machines/anfs-test-macos/`, `machines/images/{anfs-test-macos,anfs-dev}/`
- stack `packages/firstmate/tests/roundtrip` — it runs core's harness by path
- core `CLAUDE.md`: "Git Hooks", "Moving a Package Between Repos", "Testing", "Repository Layout"

## Overview

Files move from core to `anfs-dev` with no change in behaviour. The work is in what named the
old places: the harness finds the boxes' `Containerfile`s by path, the round trips find the
fixture and each other by path, stack's firstmate round trip finds core's harness by path, and
`ppm hooks` assumes the hook's own source is in `system.list`.

Three repos change, each on a branch `dev-source`: core, stack, and `anfs-dev` (whose first
commit this is).

## Implementation

### 1. The layout in `anfs-dev`

```
anfs-dev/
  CLAUDE.md, README.md
  packages/dev/                  as it is in core, plus:
    tests/harness
    tests/roundtrip
    tests/roundtrip-macos
    tests/fixtures/acme/
  containers/anfs-test-debian/   compose.yml, Containerfile
  containers/anfs-test-fedora/
  machines/anfs-test-macos/      machine.yml
  machines/images/anfs-test-macos/
  machines/images/anfs-dev/
```

- The round trips and the harness go under `packages/dev/tests/`: a package's tests are beside
  it, as everywhere else, and they are that package's (`ppm container` and the harness build the
  same boxes).
- `machines/images/anfs-dev` has `from: debian-13`. That resolves to core's, a lower-priority
  source, which is the direction `from:` allows.
- The first commit's message names the core commit the files were copied from.

### 2. Paths that named core

| Where | Today | After |
| --- | --- | --- |
| `harness` | `$HARNESS_CONTAINERS` is core's `containers/` | the dev source's `containers/`, found from the script's own location |
| `roundtrip`, `roundtrip-macos` | `core/packages/anfs/tests/fixtures/acme` | `anfs-dev/packages/dev/tests/fixtures/acme` |
| `roundtrip`, `roundtrip-macos` | copy or mount the sources under `$ROUNDTRIP_SRC_ROOT` | the same, and `anfs-dev` is one of them, registered in the box's `user.list` |
| stack `firstmate/tests/roundtrip` | `$SRC/core/packages/anfs/tests/harness` | `$SRC/anfs-dev/packages/dev/tests/harness` |
| `container.sh`, `vm.sh` | the boxes are `core/anfs-test-*` | resolved by bare name through pcm and pmm; no source is named |
| headers and messages | `core/dev` | `anfs-dev/dev` |

Check `ppm container` and `ppm vm` for any place that builds a path into core instead of asking
pcm or pmm (`pcm path`, `pmm image path`).

### 3. `ppm hooks`

`ppm hooks install` with no names acts on the `system.list` repos, and `core/dev`'s
`post_install` runs it. The hook's own source is now in `user.list`.

- With no names, `ppm hooks install` acts on the `system.list` repos **and the source the `dev`
  package was installed from**. `--all` still adds the rest of `user.list`.
- The `pre-commit` hook's rule that every directory under `packages/` has a `package.yml` is
  unchanged; it applies to whatever repo it is wired into.

### 4. What core loses

- `packages/dev`, the two containers, `machines/anfs-test-macos`, the two images, the three
  scripts and the fixture are deleted.
- `machines/` in core then holds only `images/` (and `profiles.yml` if core has one). Check that
  a source whose `machines/` has no machine definitions installs and lists cleanly.
- `containers/` in core is empty until anfs-ui. Remove the directory; `anfs src list` must not
  claim core provides containers.
- Core's README and any `ppm container` mention in a tool's help text.

### 5. Documentation

- Core `CLAUDE.md`: "Git Hooks" and "Moving a Package Between Repos" move out whole. "Testing"
  keeps the bats suites and the tools' e2e, and says in two lines where the round trips and the
  boxes are. "Repository Layout" loses `packages/dev`, `containers/` and `machines/`' test box.
  The mentions of `core/dev` elsewhere (declared resources, lib structure, callbacks) become
  `anfs-dev/dev`.
- `anfs-dev/CLAUDE.md`: those two sections, the box contract (two users, sources mounted
  read-only at `/src/<alias>`), the four macOS differences, the round trips, the fixture.
- `ROADMAP.md`'s agent section: the gate for a step that changes an install path is the round
  trip in `anfs-dev`.

### 6. This Mac

Not for an agent; listed in `log.md` for the person who merges:

1. `ppm remove core/dev` before pulling core (its files are about to leave the source).
2. Pull core, stack and `anfs-dev`; `ppm install anfs-dev/dev`.
3. The running `anfs-test-macos` machine and the `anfs-test-*` boxes keep their state: a machine
   and a service are named by name, and only the definition's source changes. Check `pmm list`
   and `pcm list` show them under `anfs-dev` with their state intact.

## Test Spec

- Core, with no `anfs-dev` anywhere: `bats packages/{anfs,ppm,pcm,pmm,psm,wsm}/tests </dev/null`
  all pass. Nothing in a suite reads the fixture, the harness or a test box.
- `anfs-dev/packages/dev/tests/`: a bats file for `ppm hooks install` with the dev package's
  source in `user.list` (it is wired without `--all`; another `user.list` repo is not).
- The acme fixture still has every resource kind, and `PMM_INSTALL=validate anfs install acme`
  passes against it from its new place (isolated HOME).

## Verification

- [ ] `packages/dev/tests/roundtrip debian` from `anfs-dev` passes against the three working trees
- [ ] `ppm container up debian` and `ppm container install debian` from an install that has `anfs-dev`
- [ ] stack's `packages/firstmate/tests/roundtrip` finds the harness
- [ ] A default install (no `anfs-dev`): `pcm list`, `pmm list`, `pmm image list` show no test resources
- [ ] `grep -rn 'anfs-test\|core/dev' .` in core and stack: history notes only
- [ ] `roundtrip-macos` (needs the tart box; may be left for a person)

## Open Questions

1. **`~/anfs-dev` is two things.** It is the default `ROUNDTRIP_SRC_ROOT` (a directory of
   checkouts named by alias) and now also a source's alias, so the dev source's checkout there
   is `~/anfs-dev/anfs-dev`. Recommended: leave the default, since it is only a default.
2. **`anfs-admin`.** Nothing here moves to it. If it is for the org's own operations (releases,
   the GitHub org), say so in the unit so the next agent does not guess.
3. **Does `ppm move` do the package's part?** It moves a package between sources with its
   tracker, on the live install, and commits both repos on whatever branch they are on.
   Recommended: no. The plan works in branch worktrees; step 6 is two commands.
