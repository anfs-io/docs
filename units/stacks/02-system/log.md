---
plan: stacks/02-system
job: stacks-02
date: 2026-10-19
---

# Log: stacks/02-system

## What was built

- `core/stacks/system/stack.yml`: `core/anfs, ppm, pcm, psm, wsm, podman, node, varlock`. No
  pmm, pmm-network, qemu or pvm (pvm depends on wsm, pmm on qemu and tart: none come in).
- `core/anfs` `depends: [ppm]`; its comment says why and points at the stack.
- `install.sh`: `install_packages` runs `anfs install system -y`, then `ppm install` for each extra
  package (they used to come before the toolkit). The other steps are unchanged.
- `anfs implode`, in order: every other installed stack (`anfs remove`, last installed first by the
  records' `installed_at`, each in a subshell so one failure doesn't stop the rest); each installed
  tool's implode except ppm's (wsm psm pmm pcm; a tool that is not installed is never called);
  `anfs remove system` with `core/ppm` and `core/anfs` kept (new `STACK_REMOVE_KEEP` in
  `stack_remove`); `ppm implode`; then the sources, anfs's dirs and the links as before (`--all`
  the bootstrap). The confirm text lists the stacks. No system record: `remove system` is
  skipped and `ppm implode` removes the toolkit, the same as before stacks.
- `anfs src install` skips a tool that is not installed (a line on stderr: `ppm install
  core/<tool>`) instead of dying. Before, a source with `machines/` killed the run on a default
  install. `tool()`'s hint names `core/<tool>` instead of `core/anfs`.
- Docs: core AGENTS.md (overview, the repo layout, a `system` paragraph under Stacks), README.md,
  packages/anfs/README.md, packages/ppm/README.md, and the comment in ppm's `implode.sh`.
- anfs-dev round trips: `anfs install acme` is now `anfs src install acme` (broken since plan 01).
  New checks: `anfs list` shows `core/system installed`, and there is no pmm, pvm or `qemu-system-*`
  (Linux also checks ppm's list). acme's pmm checks are gone. `roundtrip-macos`'s host-side "needs
  pmm" hint says `ppm install core/pmm`.

## Commits

core (stacks-02):
- `fd7e50e` system: the toolkit is the stack core/system; install.sh ends with anfs install system
- `121f93d` anfs implode: other stacks, the tools' implodes, remove system, ppm implode (the hook
  bumped ppm to 0.2.5 here)
- `35ad7b4` docs: the system stack, install.sh and implode

anfs-dev (stacks-02):
- `a800a1e` roundtrips: a default install is core/system (no pmm, qemu or pvm); anfs src install

## Deviations and decisions

- **`core/anfs` is in `system`.** The plan's list left it out. Without it nothing installs
  core/anfs through ppm: it would be stowed but not tracked, and its `depends:` would mean nothing.
- **The tool implodes still run, before `remove system`.** A literal "remove system, then the
  loop for the tools still installed" would leave pcm's, psm's and wsm's state behind (ppm
  removing a package does not run the tool's implode), and pcm's implode needs podman, which
  `remove system` takes. So the tools implode first, then system's packages go.
- **ppm and anfs are kept by `remove system` during implode.** Removing core/ppm unstows ppm, and
  then `ppm implode` could not run: ppm's dirs and the packages installed outside a stack (the
  shell package) would stay. `ppm implode` removes both packages last.
- The fallback is that last `ppm implode`: with no record it removes the whole toolkit, which is
  the old behaviour.
- `src install` skipping a missing tool is new. Plan item 5 needed it, because a default install
  has no pmm.

## Tests

- `packages/anfs/tests/system.bats` (new, 8): the manifest via `anfs show core/system` (the real
  stack.yml copied into an isolated HOME), core/anfs's depends, install.sh's `install_packages`
  (extracted and run against stub `anfs`/`ppm`, and the last step of `main`), implode's order
  (other stacks in reverse, the tools, system without ppm/anfs, ppm implode, pmm never called), the
  confirm with no answer, the no-record fallback, `src install` without a record, `src install`
  skipping pmm.
- `packages/anfs/tests/all`: 333 passed, 0 failed, 23 files (pmm and pcm off).
- anfs-dev `bats packages/dev/tests`: 36 passed.

## Deferred / not verified

- No round trip was run (no boxes or VMs): the new checks are written but not executed.
  `roundtrip debian` is still open: a default install has no pmm, qemu-system-* or pvm, `anfs
  list` shows `core/system installed`, and the implode diff is clean.
- Real `install.sh` and a real implode were not run. Everything was checked against stubs.
- The round trips' `anfs list` grep assumes the row starts `core/system  installed` (aligned or
  tsv).

## After merge, on this Mac

- The existing install has no record. `anfs implode` would still work (the fallback).
- Run `anfs install system` once to write the record. Everything is already installed, so no tool
  runs, and each package is recorded `before: true`. That means a later `anfs remove system` keeps
  them, while `anfs implode` still removes them through `ppm implode`.
- pmm, qemu and pvm stay installed (ppm doesn't drop dependencies that core/anfs no longer
  declares). That is what the round trips need on this host.
- `ppm install core/anfs` picks up the new package.yml. It is safe and not required.
