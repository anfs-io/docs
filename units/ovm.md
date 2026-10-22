---
objective: Vaults are a tool's business, not the toolkit's — pvm leaves core as `ovm`, the Obsidian Vault Manager in stack's `obsidian` package, and nothing in anfs knows what a vault is
status: plan 01 written; decisions taken 2026-10-21
---

pvm (Personal Vault Manager) came into core from the user source on 2026-10-16 (core 450241b) and
was made a resource kind on the way: `anfs vault`, `vaults:` in a stack manifest, a place in the
toolkit API's allowlist and two panels in anfs-ui. None of that does anything a vault needs. pvm
never read a source: it finds vaults by their marker file and keeps its own registry, and
`anfs install` of a stack with `vaults:` only says "pvm does not install vaults from sources yet".
No source on this machine has a `vaults/` directory, and no stack names one.

This unit moves the tool to where a tool for one kind of documents belongs, the `stack` source,
next to the Obsidian editor config it serves, and takes every vault-as-resource tie out of core.

Decided with the person, 2026-10-21: the name (`ovm`), the place (`stack/obsidian`), vaults are
not resources, and ovm stays a compliant anfs tool.

Depends on: nothing. Runs beside `linker` (see "linker" below).

## What pvm is today

`core/packages/pvm`: one script (`home/.local/bin/pvm`, 1193 lines), a zsh completion file, an
`install.sh` that registers a wsm listener, and three bats files (52 tests). It depends on
`wsm` and yq.

- **A vault** is any directory with a `pvm.yml` at its root (`id`, `name`, `links`, `profile`).
  The marker is the source of truth; `$XDG_STATE_HOME/pvm/vaults` is a registry that `pvm scan`
  rebuilds (a vault found at a new path is the same vault, moved), and a vault you stand in is
  registered as a side effect.
- **Commands**: `init`, `new`, `list`, `scan`, `remove`, `prune`, `path`, `docs`, `show`,
  `validate`, and the `link` group (`add`, `remove`, `list`, `apply`, `validate`).
- **Links into wsm levels**: `pvm.yml`'s `links:` map a vault directory to a level
  (`source`, `source/ws`, `source/ws/space`). pvm reads `wsm ls --json` for the level's
  directory and puts an absolute symlink there, recorded in `$XDG_STATE_HOME/pvm/links`. wsm
  emits `install`/`remove` per level (`lib/anfs/callbacks.sh`) and pvm's `link apply --event`
  answers; wsm knows nothing about pvm.
- **Documents** have frontmatter with a stable 10-character `id`; `docs`, `show`, `path ID`
  find them, `validate` checks them against the vault's `profile`.
- It is already a `cli.sh` tool: `table.sh` lists (rule 5, not of a source's resources),
  `--json`, `-y`, `-i`, `complete_<kind>`, exit codes as wsm's.

## Where vaults are a resource today (all of it goes)

| Where | What |
| --- | --- |
| core `packages/anfs/home/.local/bin/anfs` | `_anfs_tool vault pvm` (`anfs vault`, alias `anfs pvm`, described as "Secrets"); header comment; `install`'s help ("vaults"); the rpc allowlist, its error and `_rpc_version` add `pvm` to `ANFS_TOOLS` |
| core `packages/anfs/home/.config/sh/anfs.sh` | `vault\|pvm` in the `anfs` shell function |
| core `packages/anfs/home/.local/lib/anfs/stacks.sh` | `vaults` in `STACK_KINDS`, `stack_tool`, the manifest's keys, the "not installed" note in the plan |
| core `containers/ui/lib/anfs/kinds.rb` | the `vaults` and `links` panels |
| core tests | `registry.bats` (`anfs vault`, "vault, pvm" in help), `stacks.bats` (`vaults:` in a manifest, two notes), `bash.bats` (pvm in the entry-point list), `system.bats` (no pvm in `system`), ui's `kinds_spec.rb`, `rpc_spec.rb` |
| core `AGENTS.md` | the stack manifest and its keys, `system`'s "not pmm, qemu or pvm", the vaults bullet, the rpc allowlist, the repository layout, the `anfs <resource>` table and "the six", the callbacks, readers, rule-5, verbs and completion examples |
| core `stacks/system/stack.yml` | the comment "Not pmm, pmm-network, qemu or pvm" |
| anfs-dev `packages/dev/tests/roundtrip`, `roundtrip-macos` | "no pmm, qemu or pvm by default" |
| docs | `units/stacks.md` (`vaults/` pvm, the kinds), `BACKLOG.md` ("pvm is a core tool now"), `ROADMAP-apps.md` (vaults as a primitive; "after pvm leaves core") |

Not resource ties, but readers of pvm that move with it: `user/pjm` depends on `pvm` and runs
`pvm items` (a command pvm does not have), and the live install on this Mac still has `user/pvm`
installed, its `~/.local/bin/pvm` dangling and its wsm listener registered as `user/pvm`.

## Decisions

- **The package is `stack/obsidian`**, which today holds the nvim plugin config and the `ob`
  alias. It gains `ovm`, its completion, the listener and the tests, and `depends: [wsm]`. No
  new package: one package for Obsidian, the editor config and the vault manager.
- **The behaviour is pvm's, renamed.** Commands, options, exit codes, list keys and the
  registry format are unchanged. Plain-markdown vaults still work; "Obsidian" is the name, not a
  restriction. Obsidian-specific commands are later plans.
- **The marker is `ovm.yml`; the state is `$XDG_STATE_HOME/ovm`; variables are `OVM_*`.** No
  fallback to `pvm.yml` (alpha: no backward compatibility), and no migration code: the
  person's one vault is moved by hand after merge (`mv ~/.local/state/pvm ~/.local/state/ovm`, `pvm.yml` renamed
  to `ovm.yml` in apex). Ids are kept, so links and document ids are the same.
- **Vaults are found as today**: the marker, the registry, `ovm scan [dir...]`, the vault you
  stand in. Nothing reads a source. No `vaults/` directory, no `vaults:` key, no source
  resolution.
- **Not an `anfs <resource>`, and not on the toolkit API.** `anfs vault`/`anfs pvm` go; ovm is
  a tool on PATH like `ventoy` or `opm`, with stack's "Tools shipped by packages" contract
  (`lib/anfs/{bash,paths,log,cli,table}.sh`, `zcomp ovm`). anfs-ui loses its two panels, so
  nothing needs ovm through `anfs rpc`; the special case for `pvm` there goes and `run[0]` is
  `anfs` or `ANFS_TOOLS`. No `anfs-ovm` executable through the package seam (decided 2026-10-21).
- **Links stay links into wsm levels**, by `wsm ls --json` and the wsm listener, registered by
  `stack/obsidian`'s `install.sh`. A level is still a wsm thing; ovm is a reader of it.
- **No shims.** `pvm` is not kept as an alias; `user/pjm` is changed to `ovm` in the same job.

## linker

`linker/01-link-library` (job `linker-01`) replaces stow in ppm and wsm with
`lib/anfs/link.sh`. It touches pvm only in prose: its section 3 says "pvm's links into wsm levels
(`pvm link`, `_link_make`) are not in any tracker and are left alone by remove, as today". That
stays true of ovm's links, which are absolute symlinks into a vault outside the sources: wsm's new
plan pass sees one only if a level ships a file of the same name (a `conflict`, as wsm's own walk
says today), and `anfs_link_owner` (plan 03) will call it the user's.

- No code is shared: ovm keeps its own `ln -s` (absolute, because a vault can be on iCloud and
  the level anywhere). Using `anfs_link_one` is a later choice, after linker 01 merges.
- Both jobs edit core's `AGENTS.md` and `packages/anfs/tests/bash.bats` (linker adds `link.sh`
  to the parse list; this one drops `pvm` from the entry-point list). Different lines; whichever
  merges second rebases. Neither plan's text needs the other.
- linker 01's plan keeps saying pvm; its log or the person updates the word after both merge.

## Plans

| # | Plan | What |
| --- | --- | --- |
| 01 | `01-pvm-to-ovm` | `ovm` in `stack/obsidian`, pvm's tests with it; every vault-as-resource tie out of core, anfs-ui and the docs; `user/pjm` on ovm |
| 02 | `02-items` | Stub. `ovm items` and `exclude:`, the source `user/pjm` pulls from; first answer whether pvm once had them |
| 03 | `03-vaults-home` | `~/vaults` (`OVM_VAULTS_HOME`) is the list of vaults; a vault elsewhere is a `links:` entry in the person's repo; registry, `scan`, `remove`, `prune` go |

Later (not written): Obsidian's own things through ovm (open a document by id in the app, the
vault's `.obsidian` settings), a `skills/` entry so an agent finds documents by id.

## Decided with the person (2026-10-21)

1. **No `anfs ovm`.** ovm is a tool on PATH like ventoy; vaults are not anfs's.
2. **Not on the toolkit API.** If a later app needs vaults, `anfs rpc` gets a rule for tools that
   are not core's then; noted against `toolkit-api`, not this unit.
3. **No migration code**: the person's one vault is moved by hand (decided 2026-10-21; this
   reverses the first answer, an automatic migration from `post_install`).
4. **pvm was never a secrets vault.** The "Secrets" description goes with `anfs vault`, and
   `ROADMAP-apps.md`'s two credential lines that wait on pvm (41, 102) are removed; credentials
   are `op`/`fnox`'s.
5. **pjm stays its own tool** in the user source, on ovm by name only.

## Open questions

1. **`pjm`'s `items`.** `user/pjm` runs `pvm items --json --space ...` and its config mentions an
   `exclude` key in `pvm.yml`; core's pvm has neither, so pjm cannot pull today. Plan 01 renames
   the calls and leaves the gap. Is `items` a pvm feature lost in the move to core (restore it in
   ovm), or still to write? Either way it is plan 02 (`02-items`, a stub until this is answered).

## Completion Criteria

- `grep -rn 'pvm\|vault' packages stacks containers/ui install.sh` in core finds only onboard's
  1Password vault (`lib/anfs/onboard.sh`, `lib/ppm/onboard.sh`) and wsm's tests' `opm.yml`
  fixture; `anfs vault` is an unknown command (64), `anfs help` lists five tools, and a
  `stack.yml` with `vaults:` is 64.
- `ovm` from `stack/obsidian` passes pvm's 52 tests renamed, plus `install.bats`'s; its lists,
  `--json`, `-i`, help and completion work as stack's other tools'.
- anfs-ui has no Vaults or Vault links panel and its suite passes.
- On this Mac, after the person installs `stack/obsidian`, removes `user/pvm` and moves apex by
  hand (`mv ~/.local/state/pvm ~/.local/state/ovm`, `pvm.yml` → `ovm.yml`, `ovm scan`): `ovm list`
  shows `apex`, `ovm link list` shows its link into `anfs/` as `ok`, and `anfs callbacks wsm`
  lists `stack/obsidian` and nothing stale.
