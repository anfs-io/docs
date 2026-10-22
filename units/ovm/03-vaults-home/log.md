---
plan: ovm/03-vaults-home
job: ovm-03
date: 2026-10-22
---

# Log: ovm/03-vaults-home

## What was built (stack, `packages/obsidian`)

- **`OVM_VAULTS_HOME`**, default `~/vaults`, read after `anfs_tool_paths ovm` and documented in
  `ovm.env` (`#OVM_VAULTS_HOME=~/vaults`; the loader expands the `~/`). A trailing `/` is dropped.
- **The vaults are the entries of `$OVM_VAULTS_HOME`** (`_entries`, `_vaults_load`): every entry but
  hidden ones, in glob order; state `ok` (a directory, or a link to one, with `ovm.yml`),
  `unreachable` (a link whose target is gone, or `ovm.yml` does not answer within
  `OVM_PROBE_SECONDS`: `_reachable` is kept as it was), `not a vault` (no `ovm.yml`, or not a
  directory). `ovm list` prints `NAME ID STATUS PATH`: PATH is the canonical root when `ok`, the
  link's text when unreachable, else the entry.
- **A vault is named by its entry.** `_vault_resolve` takes the entry name, then a vault id, then a
  unique id prefix (4+). `_entry_of <root>` finds the entry that is a directory (`-ef`, each entry
  probed with `_reachable` first, the same-named entry tried first), so a vault stood in by its
  real path (iCloud) is still named by its link. `ovm.yml` no longer carries `name:` and
  `--name` is gone from `init` and `new`.
- **Gone**: `scan`, `remove`/`rm`, `prune`, `list --stale`; `_registry_load`, `_registry_write`,
  `_reg_by_id`, `_reg_by_path`, `_register`, `_register_locked`, `_auto_register` (and the
  dispatch block that ran it), `_is_stale`, `_is_unreachable`, `_cfg_name`, `_now`,
  `OVM_REGISTRY`. No migration code.
- **Kept, simpler**:
  - `new <name>`: `$OVM_VAULTS_HOME/<name>` (made with its parent), then `init`. A name with `/`
    or a leading `.` is 64; an existing entry (a dangling link too) is 73.
  - `init [dir]`: only an entry of `$OVM_VAULTS_HOME` (by the link or by the real path); anything
    else, a directory below an entry included, is 1 with `ovm new NAME, or link it there (ln -s
    '<dir>' ~/vaults/<name>) and ovm init that`. "A vault does not hold another" (73) stays.
  - Every command that works on "the vault you are in" or a directory's vault (`path`, `docs`,
    `validate`, `link add|remove|list|apply`) goes through `_vault_of`: an `ovm.yml` outside
    `~/vaults` fails with 1 and the same hint.
  - `path <vault>` of an entry that is not `ok` prints the entry's path and exits 3 (`<name> is
    unreachable` / `is not a vault`); `cd` (through `_anfs_cd`) goes there as it did for stale.
  - `_live_vaults` (for `docs`, `show`, `path ID`, `link list|validate`, the install event) reads
    the entries; an unreachable one is warned and skipped as before.
  - Completion: the entry names, read with a glob only (nothing inside them is touched).
  - The lock now guards only the links record: `$OVM_STATE_HOME/links.lock` (was `vaults.lock`).
    Nothing is written under `$XDG_STATE_HOME/ovm/vaults`.
- Help epilogue, header comment, `README.md` (`~/vaults`, the `links:` entry, the commands),
  `install.sh`'s messages, stack `AGENTS.md`'s `ovm list` row (`ok`, `unreachable`, `not a
  vault`).
- obsidian **0.3.0** (`OVM_VERSION=0.3.0`), set by hand: commands were removed (minor). The hook
  said "already 0.3.0 since 0.2.0 was pushed, not bumping".

## Commits (branch ovm-03, not pushed)

stack:
- `19d4bd5` (was `c731ebc` before the rebase) obsidian: ovm's vaults are the entries of ~/vaults (OVM_VAULTS_HOME); the registry,
  scan, remove and prune go (obsidian 0.3.0)

docs:
- this log

## Tests

`ANFS_TEST_LIB=~/.local/lib/anfs anfs work run ovm-03 stack -- bats packages/obsidian/tests
</dev/null`: **57/57**. The lib is the live core main (`31cbc05`), which has linker/02 and
config/01.

- `vaults.bats` rewritten (15): init in an entry, no `name:`; init again, an `ovm.yml` without an
  id; init outside `~/vaults` refused with the hint, below an entry refused, a link under another
  name inited by link or real path; a vault does not hold another; `new` (name, exists, dangling,
  path, hidden, none); `list` with a directory, a link, a dangling link, a file, a plain directory,
  `.DS_Store` skipped, an `ovm.yml` outside not listed, and `--json`; no `~/vaults`,
  `OVM_VAULTS_HOME` from the environment and from `ovm.local.env` (`~/` expanded); `scan`,
  `remove`, `rm`, `prune`, `ls --stale` are 64 and nothing under `ovm/vaults`; `path` by name,
  id, prefix, 3 for `not a vault`/`unreachable`; a linked vault named by its entry from its real
  path; an `ovm.yml` outside `~/vaults`; an ambiguous id prefix; help; a tab press.
- **The `links:` fixture** (the plan's last test-spec line): a `package.yml` with the plan's iCloud
  entry (a path with spaces, `platforms: [macos]`), read with `anfs_manifest_read` /
  `anfs_manifest_plan` and placed with `anfs_link_one`, which is what ppm's `stow_package` does
  for `ANFS_MF_SRCS`. `~/vaults/apex` is `../Library/Mobile Documents/.../apex` (relative), and
  `ovm list`/`ovm path apex` see it `ok`. It does not run `ppm install` (ppm's own lib is core's,
  not in `ANFS_TEST_LIB`); ppm's handling of `src:` is covered by core's `ppm/tests/manifest.bats`.
- `docs.bats`, `links.bats`, `cd.bats`: the same tests with vaults under `~/vaults` (paths only);
  `links.bats` gains the lock test (stale `links.lock` cleared, a held one 75) that replaces
  vaults.bats' two registry-lock tests. `install.bats`: the new message.
- Not run: core's `tests/all` (core unchanged); no VM or round trip (none touches ovm).

## Decisions made during the work

- **`ovm.yml`'s `name:` is gone** (not written, not read): the entry is the name, and a second
  name could only disagree. `--name` went with it. 04a's `vault.yml` can start without it.
- **Vault ids still resolve** (`ovm path <id>`, a 4+ prefix), after the entry name: ids are in
  `ovm list` and documents' ids are unchanged, so they stay a way to name a vault. Prefix
  ambiguity is still 2.
- **`init` refusal is exit 1** (not found: no entry of `~/vaults` there), not 64.
- **An `ovm.yml` outside `~/vaults` is refused, not ignored**, wherever a command looks for the
  vault you are in: silently treating it as "not in a vault" would hide why.
- **Exit 3 stays** for a vault that cannot be used (was "stale"; now `unreachable` or `not a
  vault`), so `_anfs_cd`'s handling is unchanged.

## Shared files touched

- stack `AGENTS.md`: one table row (`ovm list`). config-02/03 may touch the "Tools shipped by
  packages" paragraph; different lines.
- `packages/obsidian/home/.local/bin/ovm`: in the header block only the comment, `OVM_VERSION`,
  and the three lines after `anfs_tool_paths ovm` (`OVM_REGISTRY`/`OVM_LOCK` →
  `OVM_VAULTS_HOME`/`OVM_LOCK`); the `source` lines are untouched, so config-02/03's ANFS_WORK
  preamble and `config` group should rebase cleanly. The `cli_cmd` block changed (init, new,
  list; scan, remove, prune gone): a `config` group added there is a neighbouring hunk.
- `ovm.env` gains `OVM_VAULTS_HOME` (config-02/03's `ovm config` reads it).

## By hand, for the person after merge (in this order)

1. In your repository (user), add the obsidian layer (not created by this job):

   ```yaml
   # user/packages/obsidian/package.yml
   version: 0.1.0
   links:
     - src: ~/Library/Mobile Documents/iCloud~md~obsidian/Documents/apex
       dest: vaults/apex
       platforms: [macos]
   ```

2. `ppm install -n obsidian` (expect `link ~/vaults/apex -> ...`), then `ppm install obsidian`.
3. `rm -rf ~/.local/state/ovm/vaults ~/.local/state/ovm/vaults.lock` (the registry; ovm no longer
   reads it).
4. Optional: `yq -i 'del(.name)' ~/vaults/apex/ovm.yml` (unused now; 04a renames the file anyway).
5. `ovm list` (apex `ok`, PATH the iCloud directory), `ovm cd apex`, `ovm link list` (its links
   into spaces as before: the links record is unchanged).

## Open questions

None. Decided by the coordinator before merge (2026-10-22):

1. `ovm list` keeps showing entries that are not vaults (`not a vault`), as the plan says.
2. obsidian 0.3.0 (a minor bump by hand, not the hook's patch) is fine.

## Rebase on main (config/02)

The coordinator rebased stack's ovm-03 onto main (config/02, stack `55b1ceb`: ovm's `config`
group through `lib/anfs/config.sh`). Conflicts, resolved:

- `ovm`: config/02 had added `config` to the auto-register dispatch block's skip list; the block
  is gone here (no registry), so it was dropped. config.sh's sourcing and
  `anfs_config_declare` merged without conflict.
- `README.md`: this plan's text, keeping config/02's example as `ovm config set
  OVM_VAULTS_HOME=~/notes`.

`package.yml` 0.3.0, `OVM_VERSION=0.3.0`. Tests after the rebase, same command (live core main,
which has `config.sh`): **58/58** (57 + config/02's `ovm config` test).
