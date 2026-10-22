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
- **Tests failing on main (2026-10-22):** stack `op/tests/ssh-hook.bats` 25, 31, 34, 35 and
  `tailscale/tests/tsm.bats` 61; user `pjm/tests/pjm.bats` 12 ("dispatch sends only what is
  ready"). Found by linker/02 and ovm/01; not caused by them.
- **anfs-dev after linker/02:** `roundtrip-macos`'s bootstrap parse check lists the new `lib/anfs`
  files (`manifest.sh`, `files.sh`, `platform.sh`); `move.sh` still uses stow and `PPM_GROUP_ID`
  (goes with `linker/05-move`).
- **Port clash (2026-10-22):** `cnfs/gemstash` and `core/ui` both publish host port 9292
  (`pcm validate` warns). Pick another port for one of them.
- **pcm's own suite** has not run since config/04 changed `packages/pcm/tests/helper.bash` and
  `edges.bats`: `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm` (for the person; see "Last").
- **`anfs config list --all`** treats any `~/.config/<d>/<d>.env` as an installed tool; an unrelated
  file of that shape would be listed. Accepted for now (2026-10-22); could read installed packages
  instead.


- pcm suite (`ANFS_TESTS_PMM_PCM=1`), two old failures found in pcm/01-run: `manifest.bats` 55
  expects exit 1 but a confirm without a terminal returns 77 since setup/01; `toolkit.bats` 83
  uses a shell function as the compose runner, which pcm can't `exec`. One-line fixes each.
- `units/container-images.md` plan 01 still proposes `pcm run <image>`; decided against
  2026-10-22 (`units/pcm.md`): edit that draft when the unit is refreshed.
