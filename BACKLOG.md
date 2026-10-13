# Backlog

Ideas captured on 2026-10-13, not yet units. Each has the note that came with it and what it
touches. `ROADMAP.md` is the ordered work; an item moves there when it gets a unit.

## Toolkit

- **CLAUDE.md becomes AGENTS.md** in every repo. Do it when the `machine-images` branch is
  merged (agents are editing CLAUDE.md on it now). Check first that every agent in use reads
  AGENTS.md on its own; if one does not, leave a CLAUDE.md that only imports it.
- **Worktrees for parallel jobs** (`~/work/<job>/<alias>/`). See the sketch in `units/dev-source.md`.

## Now units

- pmm out of the default install (pcm stays, for the dashboard): `machine-images` plan 05.
- Our own linker instead of GNU stow; per-platform files under `stow:`; `anfs mv`: `units/linker.md`.
- Profiles for more than one machine (hosts, roles, software): `units/profiles.md`.

## Sources

- `anfs-admin` (`anfs-io/admin-anfs`, private): anfs.io's own infrastructure. The website, the
  demo cloud instances, anything not public. Nothing of the toolkit lives there.
