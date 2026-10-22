---
plan: ovm/04a-vault-spaces
job: ovm-04a
date: 2026-10-22
---

# Log: ovm/04a-vault-spaces

## What was built (stack, `packages/obsidian`)

- **`vault.yml`** (`OVM_CONFIG=vault.yml`): no fallback to `ovm.yml`, no migration code. An
  entry of `~/vaults` with only an `ovm.yml` is `not a vault`. `init`'s template: `id`,
  `spaces_path: spaces`, `exclude: [_templates]`, `required: [type, id]`, the rest (`kinds`,
  `fields`, `tags`, `spaces`) commented out. An existing `vault.yml` only gets a missing id.
- **`_vault_cfg <root>`**: `spaces_path` (default `spaces`; a leading `./` and trailing `/`
  dropped; absolute, hidden or `..` makes it unusable, and `validate` says why) and `exclude:`
  (a list or one string), read with one yq per vault per run (cached).
- **`_walk <root> <dir>`**: the one walk. `find` from the vault's root with `-path "./<pattern>"
  -prune` per `exclude:` pattern (find's `-path` matches `*` across `/`, as `[[ $rel == $pat ]]`
  does), hidden names and `node_modules` always pruned. `_doc_files` (drops `index.md`/`log.md`),
  `_doc_scan` and `_doc_find` go through it. 02-tasks should too.
- **links.yml** (`$OVM_CONFIG_HOME/links.yml`, then `links.local.yml`): `_pairs_load` reads both
  into one pair per (vault, space, wsm space), the first wins; `K_RO` marks a pair declared by a
  file that is a symlink (shipped). A file that is not a list of maps with `vault`, `space` and a
  non-empty `to` (string or list), an optional `as`, and nothing else, stops every `space`
  command and `docs --space` with 64, naming the file and the entry number. One yq reads a file
  (plus one for its root tag). Writes: `yq -i` (comments kept; a new file gets a commented
  header with the example); `links.local.yml` when `links.yml` is a link, said once.
- **`ovm space`** (the `link` group, `_link_dir`, `_links_load`, `_default_as`, `_link_rows`
  gone; `ovm link ...` is 64): `list|ls [vault] [--problems]`, `add`, `remove|rm` (asks, `-y`),
  `link <space> <spec> [--as]`, `unlink <space> [spec]`, `apply [--event install|remove]
  [spec...]`. A space is `<vault>/<space>` or a bare name in the vault you are in (else 64 "name
  the vault"). The link machinery (`_wsm_load`, `_rec_*`, `_link_state`, `_points_at`,
  `_in_git`, `_link_make`, `_link_drop`, `anfs_link_one`, the record and its lock) is kept; its
  input is now the vault root and `spaces_path/space`.
  - `list`'s columns `SPACE TO AS STATUS LINK VAULT`, `--json` adds `path` (the folder).
    States: `ok`, `pending`, `unlinked`, `missing`, `conflict`, `unknown`, `no folder`,
    `dangling`, `no vault` (only when listing every vault), `orphan` (scoped to the vault when
    one is named). `--problems` prints only the others and exits 1 when there is one ("no
    problems" and 0 otherwise).
  - `link` refuses (before any write): no wsm space 1, no folder 1 (with `ovm space add`), a
    conflict 73, a git repository 69, `--as` with a `/` 64. Again with another `--as` renames
    (the old link goes). `--as` equal to the space's name is stored as no `as`.
- **`docs`**: SPACE (and `show --json`'s `space`) is the vault space (`_space_of <root> <rel>`:
  the first part under `spaces_path`, a file directly in `spaces/` has none). Argument: `DIR`
  (starts with `/`, `.` or `~`), `<vault>`, or `<vault>/<space>`; with `--space` it is 64.
  `--space SPEC` is the documents of every vault space links.yml links into SPEC, across vaults.
- **`validate`**: one yq dump of `vault.yml` (every node: path, tag, value), one yq dump per
  document (as before, one yq per document), one awk over the scope for inline tags (outside
  frontmatter, fenced code and inline code; `#name` at the start of a line or after a space or a
  tab). Findings as the plan's section 5; `vault.yml`'s own are listed first as `vault.yml: ...`.
- `install.sh`: the listener is `ovm space apply --event` (`anfs_callback_register` replaces by
  package); messages say `ovm space link`. README rewritten (`vault.yml`, the vocabulary,
  links.yml, `ovm space`, the states). stack `AGENTS.md`: `ovm space list`'s keys, `ovm docs`'
  `space`. Help epilogue and the script's header comment. `ovm.env`'s comments say `vault.yml`.
- obsidian **0.4.0** (`OVM_VERSION`, `package.yml`), by hand: a command group replaced (minor).
  The hook: "already 0.4.0 since 0.2.0 was pushed, not bumping".

## Commits (branch ovm-04a, not pushed)

stack:
- `b4739e7` obsidian: vault.yml ...; ovm space ... replaces ovm link; ... (obsidian 0.4.0)

docs:
- this log

## Tests

`ANFS_TEST_LIB=~/.local/lib/anfs anfs work run ovm-04a stack -- bats packages/obsidian/tests
</dev/null`: **65/65** (was 58).

- `spaces.bats` (new, 20; `links.bats` deleted, its cases re-expressed): every case of the plan's
  test spec, plus the lock test that was in `links.bats`, adoption of a hand-made link, `apply
  SPEC` filtering, an unknown `--event` (64).
- `docs.bats` (23): the vault-space SPACE column and `docs <vault>/<space>`, a custom
  `spaces_path`, `docs --space` across two vaults, `exclude:` (a directory, `*/drafts`, a file
  pattern, `.trash` with no `exclude`; `path` and `validate` through the same walk); validate:
  `required` merged with a space's (and the default, and `[]`), `kinds` and a kind's `required`
  (and a space's kinds), `fields` (each type, a list of values, a list value's items, an unknown
  type said once), `tags` (frontmatter list and string, inline, fenced and inline code ignored,
  `project/x` allowed by `project`, a space's tags only in that space), `vault.yml`'s findings
  with the three hints. `DIR` arguments are now `./...` or absolute.
- `vaults.bats` (16): the template's keys; an `ovm.yml` alone is `not a vault`; help shows
  `space <list|add|remove|link|unlink|apply>` and `~/.config/ovm/links.yml`; the tab press
  offers `<vault>/<space>` (a custom `spaces_path` too) and the bare names inside a vault.
- `install.bats`: the handler is `ovm space apply --event`.
- The grep of the plan's Verification finds only the hint texts and the two tests of what is
  gone (`vaults.bats`' old marker, `spaces.bats`' `ovm link list` is 64), which the test spec
  asks for.
- **Not run**: shellcheck (not installed on this host; nothing installed by this job). No VM,
  round trip or core suite (core unchanged). ANFS_WORK is never set by these tests.

## Decisions made during the work

- **`apply` stays** (not one of decision 4's five): wsm's listener needs a command, and a fresh
  machine needs "make what links.yml says". Without `--event` it is machine-wide (links.yml is
  the person's, not a vault's), and takes optional wsm spaces to narrow it; pairs whose vault or
  folder is not here are skipped (`space list` shows them).
- **`apply --event install` with an invalid links.yml warns and exits 0** ("no links read"): the
  plan says every `space` command is 64, and also that an event never fails its emitter; the
  emitter wins. `apply` without `--event` is 64.
- **`dangling` vs `no folder`**: the folder is gone and ovm's link is still at its path (pointing
  at nothing) is `dangling`; gone with no link there is `no folder`. Without wsm (or the wsm space
  not installed) it can only be `no folder`.
- **Without wsm, `space list`** warns once and shows declared pairs as `unknown` (folders still
  listed, `unlinked` ones too). `unlink`/`remove` work without wsm (entries and recorded links).
- **`space remove` of a space linked nowhere is 1** (like `unlink`), saying the folder is the
  person's. `unlink` does not ask; `remove` does.
- **`unlink` of a vault no longer in `~/vaults`** works by name (removes the entries; its links
  then show as `orphan`). `link` and `add` need the vault usable (1 none, 3 not usable).
- **A shipped pair**: `unlink` leaves it and its link, names the file, exits 1 (after removing any
  other matching pairs that are not shipped). `link` of a shipped pair under a different `--as`
  is 1 naming the file (a local entry cannot override it).
- **Field types**: `string` is any single value (a YAML `2024` or `true` in a title is not a
  finding); `date` is `YYYY-MM-DD` (YAML date or string); `datetime` needs a time (`T` or a
  space, optional seconds, fraction and zone); `number` is int or float.
- **`name:`** stays an allowed key of `vault.yml` (the plan lists it) but is not read or written:
  03 made the entry the name.
- **`kinds.<name>`** may only hold `required`; another key there is a finding.
- **Completion** reads `spaces_path` from `vault.yml` with a bash regex (no yq), and probes each
  entry with `_reachable` first so a hung iCloud cannot hang a tab press.
- The links.yml writes are not under the lock (only the record is, as before).

## Shared files touched

- `packages/obsidian/home/.local/bin/ovm`: the header comment, `OVM_VERSION`, and the lines after
  `anfs_tool_paths ovm` (`OVM_LINKS_YML`, `OVM_LINKS_LOCAL`, `OVM_CONFIG`). The `source` lines
  and the `ANFS_LIB` check are untouched, so config-03's ANFS_WORK preamble should rebase
  cleanly. The `cli_cmd` block changed (docs, the `space` group, validate); the
  `anfs_config_declare` line is unchanged.
- stack `AGENTS.md`: two table rows (`ovm space list`, `ovm docs`).
- `packages/obsidian/tests/helper.bash`: `vault()` reads `vault.yml`, a `links` helper. If
  config-03 adds `unset ANFS_WORK` to `ovm_setup`, it is a neighbouring hunk.
- 04b shares `cmd_init`'s template and `README.md`.

## By hand, for the person after merge (in this order)

apex's `ovm.yml` today (read 2026-10-22): `id: fl24qptnnh`, `name: "apex"`, `profile.required:
[type, id]`, `items.exclude: [_generated, _templates, _archive, "0 - Me", "1 - GTD"]`, and
`links: [{source: spaces/anfs/product, target: anfs-dev, as: apex}]`.

1. Pull stack, then `ppm install obsidian` (re-registers the listener as `ovm space apply
   --event`). `anfs callbacks wsm` shows it under `stack/obsidian`.
2. Replace apex's marker (no `name:`, no `profile:`, no `items:`, no `links:`):

   ```bash
   cd ~/vaults/apex
   cat > vault.yml <<'EOF'
   # An ovm vault (ovm help). The id is this vault's identity: do not change it.
   id: fl24qptnnh
   spaces_path: spaces
   # Whole paths from the vault's root, not substrings as items.exclude was: a folder below the
   # root needs */ before its name (add "*/_archive" etc. where they are nested)
   exclude: [_generated, _templates, _archive, "0 - Me", "1 - GTD"]
   required: [type, id]
   EOF
   rm ovm.yml
   ```

3. Decide what `spaces/anfs/product` becomes: a vault space is a folder directly under
   `spaces/`, so `anfs/product` cannot be one. Either:

   - **A. link the whole `anfs` space** into anfs-dev, under the same name as today:

     ```yaml
     # ~/.config/ovm/links.yml
     - vault: apex
       space: anfs
       to: [anfs-dev]
       as: apex
     ```

     (anfs-dev's `apex` then shows `anfs/` — `inbox`, `marketing`, `product`, `projects`, ... —
     not only `product`.)

   - **B. move `product` up a level** (`mv ~/vaults/apex/spaces/anfs/product
     ~/vaults/apex/spaces/product`; Obsidian's wikilinks resolve by name, path links would need
     fixing) and link it:

     ```yaml
     # ~/.config/ovm/links.yml
     - vault: apex
       space: product
       to: [anfs-dev]
       as: apex
     ```

4. Remove the old link first (it points at `spaces/anfs/product`, so either option would see a
   `conflict` at the same path):

   ```bash
   old="$(wsm ls --json | yq -p json -r '.[] | select(.space == "anfs-dev") | .dir')/apex"
   [ -L "$old" ] && rm "$old"
   ```

5. Write `~/.config/ovm/links.yml`: `ovm space link apex/anfs anfs-dev --as apex` (A) or `ovm
   space link apex/product anfs-dev --as apex` (B), which writes it and makes the link; or by
   hand as above; or ship it in your repository's layer
   (`user/packages/obsidian/home/.config/ovm/links.yml`, then `ppm install obsidian`; ovm then
   writes this machine's additions to `links.local.yml`).
6. `ovm space apply`, then `ovm space list --problems` (exit 0, "no problems"); `ovm space list
   apex` shows each folder of `spaces/` and where it is linked.
7. `ovm validate` in apex: fix the documents, or extend the vocabulary (`kinds:`, `fields:`,
   `tags:` — list `go` if `#go` is used).

## Open questions

None blocking. For the person:

1. apex's `spaces/anfs/product`: option A or B above.
2. `exclude:` patterns are whole paths from the root (the plan's choice): apex's old substring
   list may have matched nested folders that now need `*/` entries.
