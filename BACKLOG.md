# Backlog

Ideas captured on 2026-10-13, not yet units. Each has the note that came with it and what it
touches. `ROADMAP.md` holds the master checklist (what is next, in order); an item moves there when it
gets a unit.

## Toolkit

- **Rename the `stack` source** (anfs-io/stack): with `stacks/` a resource, `stack/ruby` and
  `stacks/` read alike. Later; live with it until then (2026-10-19).

- **CLAUDE.md becomes AGENTS.md** in every repo. Do it when the `machine-images` branch is
  merged (agents are editing CLAUDE.md on it now). Check first that every agent in use reads
  AGENTS.md on its own; if one does not, leave a CLAUDE.md that only imports it.

- **The pre-commit hook and a pathspec commit** (`git commit -- <path>`): the version bump lands
  in the commit, but the index keeps the old `package.yml`, so the file shows as staged and
  modified. Seen 2026-10-19 in core. The hook should `git add` what it changed only when the
  commit is not pathspec-limited, or re-stage after.

- **anfs-dev's git hooks out of `~/.config`** (from `units/config.md`, plan 05's audit,
  2026-10-21): hook code is shipped under `~/.config/git/ppm-hooks` (code, not config), and
  `ppm install anfs-dev/dev` sets `core.hooksPath` in every source's `.git/config` as a side
  effect. Move the hooks to `~/.local/lib/anfs-dev/hooks`; set `hooksPath` only from `anfs hooks
  install` (and `anfs work new`). An anfs-dev job.

## Now units

- pmm out of the default install (pcm stays, for the dashboard): `machine-images` plan 05.
- Our own linker instead of GNU stow; per-platform files under `stow:`; `anfs mv`: `units/linker.md`.
- Worktrees for parallel jobs (`~/work`, `anfs work`): `dev-source` plans 02 and 03.
- The tools' command lines alike (psm groups, `wsm repo`, one list shape): `units/cli.md`.
- Profiles for more than one machine (hosts, roles, software): `units/profiles.md`.

## Sources

- pvm left core as stack's `ovm` (`units/ovm.md`).

- `anfs-admin` (`anfs-io/admin-anfs`, private): anfs.io's own infrastructure. The website, the
  demo cloud instances, anything not public. Nothing of the toolkit lives there.
