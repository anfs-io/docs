---
---

# Plan 04a — `vault.yml`, vault spaces, `links.yml`: `ovm space` replaces `ovm link`

Written 2026-10-22 from the decisions with the person that day (`units/ovm.md`, "Decided with the
person (2026-10-22)"). Runs after `ovm/03-vaults-home`: it assumes `~/vaults` is the list of
vaults and that a vault is named by its entry there. Before `04b-workflow-init` and
`02-tasks`, which build on the `vault.yml` and the scopes made here.

**Why 04 is two plans.** 04a replaces how ovm reads a vault (the marker, the links machinery,
`docs`' space, `validate`) and rewrites `links.bats`; 04b adds a downloader, a pinned plugin
list, shipped vault files and Obsidian's JSON. They share only `cmd_init`'s template and the
package's `README.md`, and each is a full session on its own. 04b's tests need 04a's `vault.yml`,
so 04a goes first and 04b rebases on it. One agent per plan.

## Context — read these files first

The job is `anfs work new <job> stack docs`. Paths are stack's unless they name another repo.

- `units/ovm.md` — the decisions of 2026-10-21 and 2026-10-22, all of them; plan 03 and its
  `log.md` (what 03 left: `_vault_resolve` on `~/vaults`, `cmd_list`, `cmd_init` inside
  `~/vaults` only)
- `packages/obsidian/home/.local/bin/ovm` — all of it, especially: `OVM_CONFIG`, `_cfg_id`,
  `_cfg_name`, `_find_root`, `cmd_init`'s template, `_links_load`, `_space_of`, `_doc_files`,
  `_doc_scan`, `_doc_find`, `cmd_docs`, `cmd_show`, the Links section (`_wsm_load`, `_rec_load`,
  `_rec_set`, `_link_state`, `_points_at`, `_default_as`, `_in_git`, `_link_make`, `_link_drop`,
  `_link_dir`, `cmd_link_*`, `_link_rows`), `cmd_validate`, the completion functions, the
  dispatch
- `packages/obsidian/{install.sh,package.yml,README.md}`, `home/.config/zsh/ovm.zsh`,
  `tests/{helper.bash,links.bats,docs.bats,vaults.bats,install.bats}`
- core `packages/anfs/home/.local/lib/anfs/link.sh` — `anfs_link_one`, `anfs_link_src_state`
  (read only; the job does not change core)
- `units/config.md` decision 1 (`<tool>.env` / `<tool>.local.env`: the local pattern `links.yml`
  follows), decision 5 (the XDG rule)
- stack `AGENTS.md`, "Tools shipped by packages" (the keys table lists `ovm link list`)

## Overview

The marker becomes `vault.yml`, named for the kind as `space.yml` is. It holds only the vault's
own concerns: where its spaces are, what to skip, and the vocabulary its documents are checked
against. A **vault space** is any folder under `spaces_path`, named by its path there
(`anfs/product`; revised 2026-10-22, see `log.md`, "Revision: nested vault spaces"). Which vault spaces show up in
which wsm spaces is not the vault's business, and not wsm's: it is this person's, declared in
`~/.config/ovm/links.yml`, and ovm is the only bridge. `ovm space` replaces `ovm link`.

What a user sees: `ovm space list` in a vault (its folders and where each is linked),
`ovm space link apex/product anfs-dev`, `ovm space unlink ...`, `ovm space add journal`;
`ovm link ...` is an unknown command (64); `ovm docs`' SPACE column is the vault space;
`ovm validate` checks tags, kinds and field types from `vault.yml`.

## Implementation

### 1. `vault.yml`

`OVM_CONFIG=vault.yml`. No fallback to `ovm.yml` and no migration code (decision 1): the person
renames apex's by hand. Every message, comment, help line and test says `vault.yml`.

```yaml
# An ovm vault (ovm help). The id is this vault's identity: do not change it.
id: fl24qptnnh
name: apex
spaces_path: spaces              # the folder whose subfolders are the vault spaces (default spaces)
exclude: [_templates, _generated] # what docs, tasks and validate skip (.obsidian/, .trash/ always)
# The vocabulary `ovm validate` checks documents against; each key may also appear per space
required: [type, id]             # keys every document's frontmatter must have (default [type, id])
kinds:                           # allowed values of `type`; each may require more keys
  note: {}
  decision: {required: [status]}
fields:                          # the value type of a frontmatter key
  status: [proposed, accepted, superseded]   # a list: one of these
  timestamp: datetime            # string, number, bool, date, datetime, list
tags: [idea, decision, go]       # allowed tags, frontmatter and inline; a/b is allowed by a
spaces:                          # per vault space: more vocabulary (no links: see links.yml)
  product: {tags: [linker, wsm]}
  journal: {}
```

- **Keys** (top level): `id`, `name`, `spaces_path`, `exclude`, `required`, `kinds`, `fields`,
  `tags`, `spaces`. Under `spaces.<name>`: `required`, `kinds`, `fields`, `tags`. Anything else is
  a `validate` finding naming the key; for the three keys of the old marker it says what replaced
  them (`links:` → `ovm space link`, in `~/.config/ovm/links.yml`; `profile:` → `required:` and
  `kinds:`; `items:` → `exclude:`). Nothing reads them.
- **`spaces_path`**: relative to the vault root, no `..`, not hidden; default `spaces`; `.` is
  the vault's root. A vault space is any folder under it, none of its parts hidden, named by its
  path relative to `spaces_path` (`product`, `anfs/product`). The vault's other folders are not
  spaces. (Revised 2026-10-22: it was "a direct subfolder".) `spaces:` entries are optional; one naming a folder that
  is not there is a `validate` finding.
- **`exclude`**: a list of bash patterns, each matched (`[[ $rel == $pat ]]`, so `*` crosses `/`)
  against the path, relative to the vault root, of every directory and file the walk meets; a
  matching directory is pruned. A trailing `/` is dropped. Hidden names (`.obsidian`, `.trash`,
  `.git`) and `node_modules` are always skipped, as now. One function, `_walk <root> <dir>`,
  prints the `.md` files under `<dir>` with the vault's `exclude` applied, and `_doc_files`,
  `_doc_scan`, `_doc_find` and `02-tasks` all go through it (today three copies of the same
  `find`). `index.md` and `log.md` stay out of the document scan (OKF), not out of `_walk`.
- **The vocabulary, merged** for a document under vault space S: `required` and `tags` are the
  union of the vault's and S's; `kinds` and `fields` are the vault's with S's entries replacing
  the same names. A document outside every space gets the vault's.
- `init`'s template (no file there): the block above with `id`, `name`, `spaces_path: spaces`,
  `exclude: [_templates]` (04b ships Templater templates there, which do not parse as
  frontmatter), `required: [type, id]`, and the rest commented out. An existing `vault.yml` is
  never rewritten beyond what `init` does today (an id when there is none, `--name`).

### 2. `links.yml`: where vault spaces are linked

`$OVM_CONFIG_HOME/links.yml` (`~/.config/ovm/links.yml`) is the person's declaration. The
person's own repo can ship it (a layer of `stack/obsidian` in `user`, linked or as a `files:`
copy). A machine's own additions go in `links.local.yml` beside it (`config.md` decision 1's
pattern):

```yaml
- vault: apex                 # an entry of ~/vaults
  space: product              # a folder under that vault's spaces_path
  to: [anfs-dev, lgat/tech]   # wsm space ids (wsm ls)
  as: apex                    # optional: the link's name in each wsm space (default: the space's)
```

- **Read**: both files, entries concatenated (`links.yml` first); a pair (vault, space, to-item)
  given twice is one link. A local file adds; it cannot take away (an open question in
  `units/ovm.md`). A missing file is no links. A file that is not a list of maps with `vault`,
  `space` and a non-empty `to` (a string is a one-item list), or has another key, makes every
  `ovm space` command and `docs --space` exit 64 naming the file and the entry; nothing is linked
  or unlinked from it.
- **Written** by `ovm space link`, `unlink` and `remove` only (explicit commands, the XDG rule):
  `links.yml` when it is a regular file or missing; when it is a link (shipped by a layer of the
  person's repo), they write `links.local.yml` instead and say so, and an entry that exists only in
  the shipped file is not removed: `unlink` names the file to edit and exits 1. Edits keep the
  file's comments and order (`yq -i`); an entry whose `to` becomes empty goes.
- The record of what ovm made stays `$OVM_STATE_HOME/links` (path, vault id, what it points at,
  the wsm space), unchanged.
- No `links:` in `vault.yml`, and wsm knows nothing about vaults (decided 2026-10-22). wsm still
  emits `install`/`remove` per space, and ovm listens.

### 3. `ovm space`

The `link` group, its five commands, `_link_dir`, `_links_load`, `_space_of` (as it is) and
`_link_rows` go. A vault space is addressed as `<vault>/<space>`, or as `<space>` in the vault you
are in (outside one, a bare name is 64 "name the vault: <vault>/<space>"). The link machinery
underneath (`_wsm_load`, `_link_state`, `_points_at`, `_in_git`, `_link_make`, `_link_drop`,
`_rec_*`, `anfs_link_one`, relative links) is kept as it is; its inputs change from "source dir,
target, as" to "vault root, space folder, wsm space, as".

| Command | Does |
| --- | --- |
| `space list [vault]` (`ls`) | One row per vault space and wsm space it is linked to: `SPACE TO AS STATUS LINK VAULT`; a space with no links one row, TO empty, STATUS `unlinked`. Default: the vault you are in, else every vault in `~/vaults`. Also the `links.yml` entries whose vault is not in `~/vaults` (`no vault`) or whose folder is not there (`no folder`), and ovm's symlinks no entry declares any more (`orphan`), as `link validate` found them. `--problems`: only the rows that are not `ok`, `pending` or `unlinked`, and exit 1 when there is one (what `link validate` was) |
| `space add <[vault/]space>` | `mkdir` the folder under `spaces_path` when it is not there (an existing one is fine: 0, "already a space"); never writes `vault.yml` |
| `space remove <[vault/]space>` (`rm`) | Unlink it from every wsm space (as `unlink` with no wsm space) and leave the folder: its notes are the person's. Confirms (`anfs_confirm`, `-y`) |
| `space link <[vault/]space> <wsm-space> [--as NAME]` | Refuses an unknown wsm space (`wsm ls`, 1), a missing folder (1), a conflict at the link's path (73), a wsm space inside a git repository (69), each before `links.yml` changes; then records the pair and makes the link, or says `pending` when the wsm space is not installed. Again with another `--as` renames it (today's `link add` rule) |
| `space unlink <[vault/]space> [<wsm-space>]` | Removes the link(s) ovm made (never what replaced one) and the pair(s) from `links.yml`; none matching is 1 |
| `space apply [--event install\|remove] [wsm-space...]` | Without `--event`: every `links.yml` link whose vault and folder are here, made when missing (what `link apply` did, but machine-wide, since the declaration is). With `--event`: wsm's listener, exactly as `link apply --event` today (install: the entries whose `to` names a space just installed; remove: from the record). Never fails its emitter |

- `apply` is not one of the five commands of decision 4; it stays because wsm's listener needs a
  command, and because a fresh machine needs "make what links.yml says". Say so in `log.md`.
- `install.sh`: the listener is `ovm space apply --event`; the messages say `ovm space link`.
  `anfs_callback_register` overwrites by package, so reinstalling the package replaces the old
  handler.
- Completion: `complete_space` lists `<vault>/<space>` for every vault in `~/vaults` (directory
  listing only, no yq), and the bare names of the vault you are in; `complete_spec` (wsm spaces)
  stays.
- `--json` keys: `space to as status link vault` (and `path`, the folder, not shown).

### 4. `docs`, `show`, `path`: the space is the vault space

- `_space_of <root> <rel>`: the vault space a vault-relative path is in (the first component under
  `spaces_path`), or empty.
- `docs`' SPACE column and `show --json`'s `space` are that name. `docs --space SPEC` keeps its
  name and takes a wsm space: the documents of the vault spaces `links.yml` links to it, across
  vaults. A new optional argument form: `docs [<vault> | <vault>/<space> | DIR]` (a DIR starts with
  `/`, `.` or `~`; anything else is a vault name or a space address). An argument and `--space`
  together are 64.
- `docs` and validate apply `exclude:` through `_walk`.

### 5. `validate`: the vocabulary

Kept: no frontmatter, unclosed frontmatter, unparseable YAML, missing required keys, duplicate
ids, `--fix`, `--strict`, the output and `--json` keys (`path finding`). Changed and added, each a
finding with the document's vault-relative path:

- `required`: the merged list (default `[type, id]`).
- `kinds`: when set (merged non-empty), `type` must be one of its names ("type 'x' is not one of:
  ..."), and the kind's own `required` keys must be there.
- `fields`: a key present with a value of another type: `string`, `number` (int or float),
  `bool`, `date` (`YYYY-MM-DD`), `datetime` (ISO 8601, as YAML or a string), `list`; an allowed
  list means the value, or each item of a list value, is one of them. An unknown type name in
  `vault.yml` is one finding against `vault.yml`, reported once.
- `tags`: when the merged list is non-empty, every tag in the frontmatter's `tags:` (a list, or a
  string of names) and every inline `#tag` in the body outside fenced code and inline code must be
  listed, or have a listed ancestor (`project` allows `project/x`). The tag pattern is pjm's
  (`#[A-Za-z][A-Za-z0-9_/-]*`, preceded by start of line or a space), so `#go` is a tag here too:
  the person lists it.
- `vault.yml` itself: unknown keys (with the hints of section 1), a `spaces:` name with no folder,
  a `spaces_path` that is absolute, hidden or has `..`. Reported as `vault.yml: ...`.
- One yq per document at most, as now; the inline-tag scan is one awk over the scope.

### 6. README, AGENTS.md, help

- `README.md`: what a vault is (`vault.yml`, the keys), vault spaces, `links.yml` (both files, who
  writes them), `ovm space`, what `validate` checks. The `ovm link` section goes.
- `cli_cmd` lines, `cli_help_epilogue` (vault and space addressing; `links.yml`'s path), the
  header comment of the script (the three things ovm does: links are now "vault spaces linked into
  wsm spaces through links.yml").
- stack `AGENTS.md`'s keys table: `ovm link list` → `ovm space list` (`space to as status link
  vault`); `ovm docs`' `space` is the vault space.
- `OVM_VERSION=0.4.0`, `package.yml` the same.

## Test Spec

`packages/obsidian/tests/` (helper: `vault <name>` makes it under `$OVM_VAULTS_HOME`, as 03 left
it; a `links <yaml>` helper writes `$XDG_CONFIG_HOME/ovm/links.yml`). No test names `ovm.yml` or
`ovm link`.

`vaults.bats`:
- `init` writes `vault.yml` (the template's keys; `exclude: [_templates]`); an `ovm.yml` alone is
  not a vault (`not a vault` in `list`).

`spaces.bats` (new; replaces `links.bats`, whose cases move here re-expressed):
- `space list` in a vault: three folders under `spaces/`, a hidden one and a file are not spaces;
  a custom `spaces_path`; unlinked spaces show `unlinked`.
- `space add` makes the folder, and is 0 on an existing one; `vault.yml` unchanged (byte-compare).
- `space link apex/product anfs-dev` writes the entry to `links.yml` and makes a relative link in
  the wsm space; a second `space link apex/product lgat/tech` adds to `to:`; `--as` names the link,
  and again with another `--as` renames it.
- many-to-many: two vaults' `product` linked into one wsm space need `--as` (the second is a
  conflict, 73); one space linked into two wsm spaces.
- `space link` refuses: an unknown wsm space, a missing folder, a conflict, a wsm space inside a
  git repository; `links.yml` unchanged after each.
- bare `<space>` outside a vault is 64; `<vault>/<space>` works anywhere.
- `space unlink` with and without a wsm space; removes only what ovm made; an entry with an empty
  `to` goes.
- `links.yml` a symlink (to a fixture "repo" file): `space link` writes `links.local.yml` and says
  so; `unlink` of an entry only in the shipped file exits 1 naming it and leaves it; both files'
  entries are listed, a pair in both is one row.
- an invalid `links.yml` (an entry without `to`, an unknown key): every `space` command is 64 and
  nothing is linked.
- `space remove` unlinks from every wsm space, leaves the folder and its notes, needs `-y` when
  not interactive.
- `space list`: `no vault`, `no folder`, `orphan`, `pending`, `conflict`, `dangling`; `--problems`
  exits 1 with one of them, 0 with only `ok`/`pending`/`unlinked`.
- `space apply` makes every missing link machine-wide; the install event links only the wsm spaces
  named, the remove event unlinks from the record, a later install puts them back, an event never
  fails (today's four event cases).
- without wsm: `space link`/`apply` are 69; `space list` still lists folders.
- `ovm link list` is 64 (unknown command).

`docs.bats`:
- SPACE is the vault space; `--space anfs-dev` lists the documents of every vault space linked to
  it, across two vaults; `docs apex/product`; a DIR argument; an argument with `--space` is 64.
- `exclude:`: a pruned directory, a pattern with `*` across `/`, a file pattern; `.obsidian` and
  `.trash` are skipped with no `exclude`.
- `validate`: each finding of section 5, one test each for `required` (merged with a space's),
  `kinds` (and a kind's own `required`), `fields` (each type, an allowed list, a list value's
  items, an unknown type name reported once), `tags` (frontmatter list and string, inline, inside
  a code fence ignored, `a/b` allowed by `a`, a space's tags added to the vault's), `vault.yml`'s
  unknown keys with the hints for `links:`, `profile:` and `items:`.

`install.bats`: the handler registered is `ovm space apply --event`.

Completion (in `vaults.bats`' tab-press case): `ovm space link <TAB>` offers `<vault>/<space>`.

## Verification

Every command runs in the job: `anfs work run <job> stack -- <cmd> </dev/null`. Nothing is
installed on this host, and nothing touches `~/vaults`, `~/.config/ovm` or apex.

- [ ] While working, the file you touch: `bats packages/obsidian/tests/<file>` (`-f <name>` for
      one test)
- [ ] At the end: `bats packages/obsidian/tests`
- [ ] `grep -rn 'ovm\.yml\|ovm link\|link apply\|profile:' packages/obsidian` finds nothing but
      the hint text of section 1 and its tests
- [ ] shellcheck on `ovm` and `install.sh`, as stack's other tools
- Not run, for the person after merge (in `log.md`): in apex, rename `ovm.yml` to `vault.yml`;
  move `items.exclude` to `exclude:` as patterns (`[_generated, _templates, _archive, "0 - Me",
  "1 - GTD"]` matched whole names, not substrings as the old key said); `profile.required` to
  `required:`; delete `links:`; decide what `spaces/anfs/product` (today linked into `anfs-dev` as
  `apex`) becomes, since a vault space is a direct subfolder of `spaces/` (either link
  `apex/anfs` with `as: apex`, or move `product` up a level); write `~/.config/ovm/links.yml`
  (or ship it in `user/packages/obsidian`); `ppm install obsidian` (re-registers the listener);
  remove the old symlink if its name changes; `ovm space apply`; `ovm space list --problems`
  (exit 0); `anfs callbacks wsm` shows `ovm space apply --event`; `ovm validate` in apex and fix
  or extend the vocabulary.
