---
---

# Plan 03 — `~/vaults` is where vaults are; no registry, no scan

Written 2026-10-21. Runs after `linker/02-links-and-files` (`src:` entries in `package.yml`) and
`config/01-tool-env` (`ovm.env`).

## Why

Vaults can be anywhere today, so ovm keeps a registry in `$XDG_STATE_HOME/ovm/vaults`, rebuilt by
`ovm scan` and added to as a side effect of standing in a vault. That is discovered state with no
one place to look. The toolkit already has one directory per kind of thing: `~/work` for agents'
worktrees, `~/spaces` for spaces. Vaults get `~/vaults`.

## Decisions (with the person, 2026-10-21)

- **`OVM_VAULTS_HOME`**, default `~/vaults`, a setting in `ovm.env` (config/01). A vault is an
  entry of that directory, a directory or a link to one, with an `ovm.yml`. That list is the
  only list: `ls ~/vaults` is the truth, as for `~/spaces`.
- **A vault stored elsewhere (iCloud) is a link in `~/vaults`, declared in the person's repo** with
  linker 02's vocabulary, in a layer of `stack/obsidian`:

  ```yaml
  # user/packages/obsidian/package.yml
  links:
    - src: ~/Library/Mobile Documents/iCloud~md~obsidian/Documents/apex
      dest: vaults/apex
      platforms: [macos]
  ```

  No `.conf` file, no second format: the link that makes `~/vaults` useful is the declaration.
  On a machine without that path the entry's state is `missing src`.
- **Gone**: `ovm scan`, `ovm remove`, `ovm prune`; the registry (`_registry_load`,
  `_registry_write`, `_register`, `_register_locked`, `_auto_register`, `_live_vaults` as it reads
  the registry) and `$XDG_STATE_HOME/ovm/vaults`. Removing a vault is removing its entry in
  `~/vaults` (or its `links:` line). No migration code.
- **Kept, simpler**: `list` (the entries of `~/vaults`: name, id, state `ok`/`unreachable`/`not a
  vault`, path), `new <name>` (creates `~/vaults/<name>` and inits it), `init [dir]` (only inside
  `~/vaults`; refused elsewhere with the hint to link it there), `path`, `cd`, `docs`, `show`,
  `validate`, the `link` group. `_reachable`'s timeout stays so an asleep iCloud cannot hang a
  list. A vault is named by its entry name; document ids are unchanged.
- ovm's links from vault directories into spaces stay as they are (its `link` group); whether a
  space's own `links:` replaces them is a later question.

## Context

- `units/ovm.md`; plans 01 (and `log.md`), 02 (stub)
- stack `packages/obsidian/` — `home/.local/bin/ovm` (the functions above, `cmd_init`,
  `cmd_new`, `cmd_list`, `_vault_resolve`, `_doc_scan`, `_find_root`), `home/.config/sh/ovm.sh`,
  completion, `tests/{vaults,links,docs,cd,install}.bats`, `README.md`
- `units/linker/02-links-and-files/plan.md` (`src:` in `package.yml`), `units/config.md`
  (`ovm.env`)

## Test Spec

- `list` reads only `$OVM_VAULTS_HOME`: a directory vault, a link to one, a link whose target is
  gone (`unreachable`), an entry without `ovm.yml` (`not a vault`); nothing outside it is listed
  even with an `ovm.yml`.
- `new` creates and inits under `$OVM_VAULTS_HOME`; `init` outside it is refused.
- `scan`, `remove`, `prune` are unknown commands (64); no file is written under
  `$XDG_STATE_HOME/ovm/vaults`.
- `path`, `cd`, `docs`, `show`, the `link` group work by entry name as before.
- A `user/obsidian`-style layer with the `links:` entry above (fixture source) installs
  `~/vaults/apex` as a relative link (needs linker 02).

## Verification

Through `anfs work run <job> stack -- bats packages/obsidian/tests </dev/null`; nothing installed
on the host. For the person after merge: add `user/packages/obsidian/package.yml` with the
`links:` entry, `ppm install obsidian`, `rm -rf ~/.local/state/ovm/vaults`, then `ovm list`
(apex `ok`), `ovm cd apex`.
