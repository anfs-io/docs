---
---

> **Superseded 2026-10-19 by the `stacks` unit (`units/stacks.md`).** Kept for the record.


# Plan 05 — pmm is not part of the default install

## Context — read these files first

- `packages/anfs/package.yml` — `depends:` (every tool and what they run on)
- `packages/anfs/home/.local/lib/anfs/paths.sh` — `ANFS_TOOLS`
- `packages/anfs/home/.local/bin/anfs` — `cmd_install`, `cmd_implode`: the loops over `ANFS_TOOLS`
- `packages/pmm/package.yml`, `packages/qemu/package.yml`
- `packages/anfs/tests/roundtrip`, `roundtrip-macos` — they check pmm after a default install
- `install.sh` — what the base install ends with

## Overview

`ppm install core/anfs` installs every tool. pmm brings qemu (and on Linux, with
`core/pmm-network`, libvirt), which most machines never use. pcm stays in the default install:
the dashboard ships as a container (`toolkit-api`).

After this plan the default install is anfs, ppm, pcm, psm and wsm. pmm is `ppm install
core/pmm`.

## Implementation

1. `core/anfs` no longer depends on `pmm` or `qemu`. `core/pmm` depends on `qemu` (check it
   already does).
2. A tool in `ANFS_TOOLS` that is not installed is skipped by `anfs implode`, and by
   `anfs install <source>` for a source that has nothing of its kind. When the source does have
   that kind (`machines/`), `anfs install` says what it skipped and how to get it:
   `acme has machines; pmm is not installed (ppm install core/pmm)`. It does not install the
   tool itself: a source must not be able to pull qemu onto a machine. `--with pmm` does.
3. `anfs src list` keeps showing `machines` in `provides` whether or not pmm is installed.
4. Shell completion and any place that runs `pmm` unguarded (`ppm vm` in the `dev` package
   already checks). `grep -rn '\bpmm\b'` outside `packages/pmm`.
5. The round trips: a default install has no `pmm` on PATH; a second pass installs `core/pmm`,
   then runs today's pmm checks (`PMM_INSTALL=validate`). The implode diff covers both.
6. CLAUDE.md: "What This Is" (which tools are the base), the `core/anfs` line in "Repository
   Layout", and pmm's section opening with how to install it.

## Test Spec

- `packages/anfs/tests`: `anfs install <fixture>` with a stub `PATH` that has no pmm skips
  `machines/` with the message and exit 0; `--with pmm` is passed to ppm; `anfs implode` does
  not call a missing tool.
- `bash.bats` and anything that lists the tools' entry points still pass.

## Verification

- [ ] `bats packages/anfs/tests </dev/null`
- [ ] `roundtrip debian`: no `pmm`, no `qemu-system-*` after the default install; both after
      `ppm install core/pmm`; clean diff after implode
