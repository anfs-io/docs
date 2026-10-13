# Backlog

Ideas captured on 2026-10-13, not yet units. Each has the note that came with it and what it
touches. `ROADMAP.md` is the ordered work; an item moves there when it gets a unit.

## Toolkit

- **pcm and pmm are not in the default install.** Too many dependencies (podman, varlock, node,
  qemu). `core/anfs` depends on every tool today; the base becomes anfs, ppm, psm, wsm, and
  `anfs install`/`implode` skip a tool that is not installed.
  Conflicts with `toolkit-api`'s completion criterion (the dashboard is a container in a default
  install, which needs pcm and podman): decide which gives way before that unit's plan 05.
- **CLAUDE.md becomes AGENTS.md** in every repo. Do it when the `machine-images` branch is
  merged (agents are editing CLAUDE.md on it now). Check first that every agent in use reads
  AGENTS.md on its own; if one does not, leave a CLAUDE.md that only imports it.
- **Evaluate GNU stow; maybe write our own linker** with only what the tools use and what the
  standards need. What stow costs today, from CLAUDE.md: its built-in ignore list drops
  `README.*` and `.gitignore` (wsm works around it with a fake HOME); an absolute link is
  foreign; it aborts on the first conflict; `ppm move` and layering both need a complement
  ignore list; nothing is per platform. ppm and wsm both use it. Same piece of work as
  "per-platform files" below.
- **Worktrees for parallel jobs** (`~/work/<job>/<alias>/`). See the sketch in `units/dev-source.md`.

## Customization (the user repo)

- **Profiles for more than one machine**, as Ansible has hosts and roles: `hosts/<host>.yml` and
  `roles/<role>.yml` in the user repo, each declaring what to install per resource kind.
- **A brew declaration per profile**, in effect a Brewfile, which can differ per machine.
- `anfs profiles list`; `anfs profile apply host/<name>` or `role/<name>` reads the file and
  installs.
- **Per-platform files without the `macos/` and `linux/` directories** that stand in for `home/`
  in a package: a declaration in `package.yml`, perhaps under `stow:`, with `macos`, `linux` or a
  distro.

## Sources

- `anfs-admin` (`anfs-io/admin-anfs`, private): anfs.io's own infrastructure. The website, the
  demo cloud instances, anything not public. Nothing of the toolkit lives there.
