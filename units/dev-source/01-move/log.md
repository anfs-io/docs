# Log — Plan 01, move the dev and test assets to `anfs-dev` (the rest, job `dev-move`)

2026-10-19. The package half, the docs space (4a) and section 6's repair were done by hand earlier
(core 15d1aaa, anfs-dev e570f58). This is the rest.

## What moved

From core (as of 1066363) to anfs-dev, deleted in core and added in anfs-dev:

- `containers/anfs-test-{debian,fedora}` → anfs-dev `containers/`
- `machines/anfs-test-macos`, `machines/images/{anfs-test-macos,anfs-dev}` → anfs-dev `machines/`
- `packages/anfs/tests/{harness,roundtrip,roundtrip-macos,roundtrip-sources,fixtures}` → anfs-dev
  `packages/dev/tests/`

Core now has no `containers/`; `machines/` holds only `images/{debian-13,fedora-44}`.

## Commits (branch `dev-move`, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| core | 4306020 | the deletions; `sources.bats` asserts core's tree provides no `containers`; CLAUDE.md, README, anfs README, pmm-images skill; pmm `image-validate.bats` |
| anfs-dev | 99f8019 | the additions (message names core 1066363); paths; `anfs hooks`; `hooks.bats`; new `CLAUDE.md` |
| stack | 2693ce4 | firstmate's round trip: `$SRC/anfs-dev/packages/dev/tests/harness` |
| docs | (this log) | |

## Changes besides the move

- **harness**: unchanged code. It already found `containers/` at `../../../containers` from its own
  location, which in anfs-dev is anfs-dev's. Comment updated.
- **roundtrip / roundtrip-macos**: the fixture is `anfs-dev/packages/dev/tests/fixtures/acme`
  (`$copy/anfs-dev/...` and `/src/anfs-dev/...`). anfs-dev is one of the sources the box is given,
  because `roundtrip-sources` lists every checkout under the root, and the root defaults to the
  directory that holds the anfs-dev checkout. `roundtrip-sources` also falls back to the installed
  `anfs-dev` (as it already did for core and stack) when the root has no anfs-dev.
- **container.sh / vm.sh**: they already resolved the boxes by bare name through `pcm path` /
  `pmm path` and globbed every source, so no core path was in them. Only the comments and headers
  changed (`core/dev` → `anfs-dev/dev`, also in move.sh and user.sh). Not refactored (plan 03).
- **anfs hooks** (`packages/dev/home/.local/bin/anfs-hooks`): with no names it now acts on
  `system.list` plus the source the dev package was installed from. That source is found by
  following `~/.config/git/ppm-hooks/pre-commit`'s links (stow-folded directory too), falling back
  to the script's own path, and matching the result against `$ANFS_SOURCES_HOME/*`. If the result
  is not one of the sources (a job's worktree, for example), nothing is added. `--all` is
  unchanged. install.sh's comment and message say so.
- **pmm `image-validate.bats`**: "the shipped images validate" now validates `core/debian-13` and
  `core/fedora-44` only (2 images). The `anfs-dev` image's assertions (preseed inherited, the
  `.anfs` step, test.sh inherited) are dropped, because that image is no longer core's.
  **Not run** (pmm suite off).
- **Docs**: core CLAUDE.md loses "Git Hooks", "Moving a Package Between Repos", the round-trip
  blocks, the harness and fixture bullets, the box contract and the four macOS differences. All of
  these are in the new `anfs-dev/CLAUDE.md`, together with a layout section. Core's "Testing" says
  in two lines where they are now. "Repository Layout" no longer lists `containers/` or the macOS
  box. The remaining `core/dev` mentions now say `anfs-dev/dev`. Core README drops the `dev`
  package row. The pmm-images skill says which images are anfs-dev's. ROADMAP.md is not touched.

## Deviations

- anfs-dev has no root README, so none was edited. Its `CLAUDE.md` is new.
- The `anfs-dev` image's assertions in pmm's image-validate test were removed, not moved. anfs-dev
  has no pmm validate test of its own yet.

## Tests

- core `packages/anfs/tests/all`: **299 passed, 0 failed**, 21 files (pmm and pcm off).
  `sources.bats` has one new test (14/14).
- anfs-dev `bats packages/dev/tests`: **29/29**, including 3 new tests in `hooks.bats`:
  - the dev package's source is wired without `--all`, and another `user.list` repo is not
  - the source is found when stow folded the hook directory into a link
  - hooks stowed from outside the sources wire only `system.list`
- stack: no bats for firstmate (only the round trip).
- shellcheck: not installed here, so not run.

## Deferred (no VMs, boxes, podman or tart in this job)

- `packages/dev/tests/roundtrip debian` from anfs-dev
- `ppm container up|install debian`
- stack's firstmate round trip finding the harness. Only the path was checked, by reading it.
- `roundtrip-macos`
- A default install showing no test resources in `pcm list`, `pmm list` and `pmm image list`
- Test Spec `PMM_INSTALL=validate anfs install acme` from the fixture's new place. It needs a full
  toolkit install in an isolated HOME, which is not quick.
- pmm's and pcm's suites (off), including the edited `image-validate.bats`

## Not verified

- That `anfs hooks install` run from `post_install` (from the package directory) finds the source
  on a real install. The fallback through the script's path covers that case, but the bats tests
  only cover the stowed-link path.
- The grep for `anfs-test|core/dev|tests/harness|tests/roundtrip|tests/fixtures` in core and stack
  now finds only box names (`anfs-test-<distro>` as a name, e.g. in the `anfs work` variables table
  and firstmate's `podman rm`) and the notes that say the images are anfs-dev's.

## After merge (the person)

1. Merge `dev-move` in core, stack and anfs-dev; pull all three.
2. `ppm install anfs-dev/dev` again, so the new `anfs-hooks` is stowed. Then run `anfs hooks status`:
   it should show `anfs-dev` as installed (or run `anfs hooks install`).
3. `pcm list`, `pmm list` and `pmm image list` should show the `anfs-test-*` boxes and the
   `anfs-dev` / `anfs-test-macos` images under `anfs-dev`, with their state intact.
4. Run the deferred items above, starting with `packages/dev/tests/roundtrip debian` from anfs-dev,
   and pmm's suite with `ANFS_TESTS_PMM_PCM=1` (it covers the edited `image-validate.bats`).
5. ROADMAP.md (orchestrator): the gate for a step that changes an install path is anfs-dev's
   round trip.
