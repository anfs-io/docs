---
---

# Plan 02 — `system`: the toolkit is a stack

Needs plan 01.

## Context — read these files first

- `units/stacks.md`, `units/stacks/01-stacks/plan.md` and its `log.md`
- core `install.sh` — `main`: `stow_base`, `anfs src update`, `install_packages`
- core `packages/anfs/package.yml` — `depends:` (every tool and the software they run on), and
  its comment
- core `packages/anfs/home/.local/bin/anfs` — `cmd_implode`, `_record_load`, `_bootstrap_summary`,
  `_implode_bootstrap`, `_login_shell_guard`
- core `packages/anfs/home/.local/lib/anfs/paths.sh` — `ANFS_TOOLS`
- the anfs-dev round trips (`packages/dev/tests/roundtrip*`): they check a default install

## Implementation

1. **`core/stacks/system/stack.yml`**: `packages: [core/ppm, core/pcm, core/psm, core/wsm,
   core/podman, core/node, core/varlock]`. Not pmm, pmm-network, qemu or pvm: they are
   `ppm install core/pmm` (or a stack that names them). Check each package's own `depends:` so
   the list is the toolkit and nothing else.
2. **`core/anfs` depends on what the `anfs` command itself runs** and nothing else (ppm, for
   `anfs src install` and stacks). The comment says why.
3. **`install.sh` ends with `anfs install system -y`** instead of installing `core/anfs`'s
   dependencies; its other steps are unchanged. `install.sh --packages` still installs extra
   packages after it.
4. **`anfs implode` is `anfs remove system`** followed by what implode does after the tools
   (the sources, anfs's dirs, the links; `--all` the bootstrap). Every other installed stack is
   removed first, in reverse install order, so nothing is left holding a toolkit resource. The
   confirm text lists the stacks.
   - A tool that is not installed is skipped, never called (pmm on a default install).
   - A machine installed before stacks has no record: implode falls back to today's loop over
     `ANFS_TOOLS` for each tool still installed after `remove system`.
5. **`anfs src install`** and the tools still work on a machine without the `system` record.

## Test Spec

- `packages/anfs/tests`: `install.sh`'s last step calls `anfs install system` (a stub);
  `anfs implode` removes other stacks first, then `system`, then the rest, and skips a missing
  tool; implode on a machine with no records falls back.
- The `system` manifest validates (`anfs show core/system` in an isolated HOME).

## Verification

- [ ] `packages/anfs/tests/all`
- [ ] Deferred (needs a box): `roundtrip debian`: a default install has no `pmm`, `qemu-system-*`
      or `pvm`; `anfs list` shows `core/system installed`; the implode diff is clean
