---
---

# Plan 02 — `ovm tasks`: the open tasks of a vault, the source `pjm` pulls from

Stub 2026-10-21 (`02-items`); its question answered 2026-10-22 (never written); rewritten
2026-10-22 as `ovm tasks` from the decisions with the person that day (`units/ovm.md`, decision
6). Runs after `04a-vault-spaces` (vault spaces, `links.yml`, `exclude:`, `_walk`) and
`04b-workflow-init`; the order is 03 → 04a → 04b → 02.

## Why

`user/pjm` pulls work items from commands that print JSON. Its one source is `ovm items --json
--space user/dev`, which never existed (checked 2026-10-22: no `items` and no `exclude` in pvm's
or core's history; pjm was written against a command still to be designed), so `pjm pull` fails.
This plan gives pjm its source. An item is an open task line (`- [ ]`): it needs no plugin, every
markdown editor writes it, and it is where the person already writes work down.

## Context — read these files first

The job is `anfs work new <job> stack user docs`. **user is the person's own repository**: only
`packages/pjm` changes there (its shipped `config.yml` and three comments), and the person merges
it separately. Paths are stack's unless they name another repo.

- `units/ovm.md`; plans 04a and 04b and their `log.md`
- `packages/obsidian/home/.local/bin/ovm` — `_walk` (04a), `FM_AWK`, `_doc_scan`, `_space_of`,
  the space addressing and `docs`' argument forms (04a), `_links_read` (or whatever 04a named the
  `links.yml` reader), `cmd_docs`, completion
- `packages/obsidian/tests/{helper.bash,docs.bats}`
- core `packages/anfs/home/.local/lib/anfs/table.sh` — `anfs_table` (`[]` keys for lists, null
  for empty fields, more keys than headers)
- user `packages/pjm/home/.local/bin/pjm` — `cmd_pull` (the field contract: the `map({...})`
  under the header comment at ~211), `_go_tag`, `cmd_dispatch` (`projects[.space]`); `home/.config/pjm/config.yml`;
  `package.yml`'s comment; `tests/helper.bash` (`pull`: the item shape pjm's tests feed it)

## The contract

`ovm tasks --json` prints a JSON array; each element has exactly pjm's ten keys:

| Key | Value |
| --- | --- |
| `vault_id` | the vault's id (`vault.yml`) |
| `vault` | the vault's name: its entry in `~/vaults` |
| `file` | the file's path relative to the vault root (`spaces/product/plan.md`): the same on every machine |
| `line` | the 1-based line number (a JSON string from `anfs_table`; pjm does `tonumber`) |
| `doc` | the file's frontmatter `id`, or null |
| `space` | **the vault space, qualified: `<vault>/<space>`** (`apex/product`), or null for a file outside every vault space |
| `section` | the text of the nearest ATX heading above the line (`#`s and closing `#`s stripped), or null |
| `parent` | the text of the enclosing task (open or not), or null |
| `tags` | the line's `#tags`, without `#`, in order, once each: a list |
| `text` | the line after the checkbox, trimmed; tags left in (pjm strips its `#go`) |

**Why `space` is the vault space, not the wsm space** (decision 6 said decide and say): a vault
space can be linked into several wsm spaces, or none, so a wsm space is not a fact about a task;
and links live in this machine's `links.yml`, so a wsm-valued `space` would change when a link
does. The vault space is one value per task, the same on every machine and whatever the query.
`<vault>/<space>` is the address `ovm space` already uses, and is unique across vaults. pjm keys an
item by vault id, file, section, parent and text, so `space` does not enter its identity;
`projects:` in pjm's config map these names (`apex/product: widget`). The wsm space a task was
found through is what the person asked for on the command line; it is not repeated per task.

No other keys: a key added later is a contract change to agree with pjm first (`units/ovm.md`,
open questions).

## What a task is

- A line whose content, after leading whitespace and any `>` blockquote or callout markers
  (`> `, `>> `), is a list marker (`-`, `*`, `+`, `1.`, `1)`), one space, then `[ ]` (a space
  inside) and a space or the end of the line. Text after it, trimmed, is `text`; an empty text is
  not a task.
- Not open: `[x]`, `[X]`, and any other character in the box (`[-]`, `[/]`, `[>]`). They still
  count as parents.
- Skipped: the frontmatter; lines inside fenced code (```` ``` ```` or `~~~`, three or more,
  closed by the same character at least as many times, per CommonMark); files and directories
  `_walk` prunes (hidden ones, `node_modules`, `exclude:`). Not handled, and said in help:
  indented code blocks (indentation is how tasks nest), `%% %%` and HTML comments, setext headings.
- `index.md` and `log.md` are read (they are left out of the document scan only because OKF
  reserves their frontmatter); their `doc` is null.
- **parent**: indentation is the leading whitespace before the marker, a tab counting to the next
  multiple of 4, blockquote markers stripped first. The parent is the nearest task line above
  with less indentation, where no line between them (other than blank lines and lines indented
  more than the task) is at or below its indentation. A non-task list item in between ends the
  search at it (its own parent is not inherited). The parent's `text` is its text after the box.
- **tags**: `(^|space)#[A-Za-z][A-Za-z0-9_/-]*` in the text, outside inline code spans, as pjm's
  go-tag pattern reads them.
- One awk over every file of the scope (`find ... -exec awk {} +`, as `_doc_scan`), no yq per
  file; the frontmatter id is read by the same awk (FM_AWK's rule).

## The command

```
ovm tasks [<vault> | <vault>/<space> | DIR] [--space SPEC]
```

| Scope | Tasks of |
| --- | --- |
| nothing, in a vault | that vault |
| nothing, outside one | every vault in `~/vaults` (unreachable ones warned and skipped, as `docs`) |
| `<vault>` | that vault |
| `<vault>/<space>` (or `<space>` in a vault) | that vault space's folder |
| DIR (starts with `/`, `.` or `~`) | that directory, in its vault (refused outside one: 1) |
| `--space SPEC` | every vault space `links.yml` links to the wsm space SPEC, across vaults; a link whose vault or folder is not on this machine is warned and skipped; SPEC no `links.yml` entry names: an empty list, 0, with a warning naming `ovm space link` |

The argument forms and their errors are `docs`' (04a): an argument with `--space` is 64; an
unknown vault 1; an ambiguous one 2. A wsm space need not be installed for `--space` (it reads
`links.yml`, not wsm). The table: `SPACE FILE LINE TEXT` (the ten keys behind it with `--json`);
sorted by vault, file, line. Exit 0 with no tasks.

`docs` already skips `exclude:` (04a); nothing to add there.

## user: pjm

- `home/.config/pjm/config.yml`: `sources:` becomes `[ovm, tasks, --json, --space, anfs-dev]`
  (a wsm space with a `links.yml` entry; `user/dev` named a level the old model had); the
  commented examples follow (`[ovm, tasks, --json, apex/product]`, a directory); the `report:`
  comment says "the vault's `vault.yml` excludes `_generated`"; `projects:`'s comment says a
  space is `<vault>/<space>` and its example becomes `apex/product: anfs`.
- A comment above `sources:`: one task is pulled once per source that reaches it, so two sources
  whose links overlap list it twice (pjm's `dup` counts both). One source per vault space, or one
  `--space` per wsm space, avoids it.
- `bin/pjm`'s comments at 5 and 211 and `package.yml`'s comment: `ovm tasks --json`. No code
  change: the contract is what `cmd_pull` reads already. Bump pjm's version.

## Test Spec

`packages/obsidian/tests/tasks.bats` (new), fixtures in the test vaults:

- One open task per form: `-`, `*`, `+`, `1.`, `1)`, indented by spaces and by a tab, inside a
  `> [!todo]` callout; each is found with the right `line` and `text`.
- Not tasks: `[x]`, `[X]`, `[-]`, `- []`, `-[ ]`, `- [ ]` with nothing after, a task in the
  frontmatter, in a ```` ``` ```` fence, in a `~~~` fence, in a fence opened by four backticks and
  "closed" by three (still open).
- `section`: the nearest heading, closing `#`s stripped; null before the first heading; a `#` line
  inside a fence is not a heading.
- `parent`: a nested open task under an open one, under a done one; two levels; a sibling is not a
  parent; a non-task bullet in between stops it; a tab-indented child of a 4-space parent.
- `tags`: several, a nested `a/b`, a `#` inside inline code ignored, `#123` not a tag, a repeated
  tag once.
- `doc`: the frontmatter id; null without frontmatter; `index.md`'s tasks are listed with null.
- `space`: `apex/product` for a file under `spaces/product/`; null at the vault root; a custom
  `spaces_path`.
- Scopes: in a vault; outside every vault (two vaults); `<vault>`; `<vault>/<space>`; a bare
  space inside a vault; DIR; `--space` through `links.yml` with one vault space in two wsm spaces
  and two vaults in one; `--space` naming a space with no entry (0, empty, warning); `--space` with
  an argument (64).
- `exclude:`: a pruned directory's tasks are not listed; `.obsidian/` and `.trash/` never are.
- `--json`: exactly the ten keys on every element (`jq 'map(keys) | unique'`); `tags` an array;
  empty fields null; the output piped into pjm's `cmd_pull` normalisation reads back the same
  values (the test runs the jq `map({...})` from pjm's script, copied as a fixture with a comment
  naming its source, so a contract break fails here).
- Speed: a fixture of 500 files is listed in under 2 seconds (one awk, not one per file).

user: `bats packages/pjm/tests` passes unchanged (its fixtures already have the ten keys).

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing is
installed on this host; no real vault, `links.yml` or pjm state is read or written.

- [ ] While working: `bats packages/obsidian/tests/tasks.bats` (`-f <name>` for one)
- [ ] stack, at the end: `bats packages/obsidian/tests`
- [ ] user: `bats packages/pjm/tests`
- [ ] `grep -rn 'items' packages/obsidian` in stack and `grep -rn 'ovm items\|pvm' packages/pjm`
      in user find nothing
- Not run, for the person after merge (in `log.md`): merge the user branch; in their live
  `~/.config/pjm/config.yml` (the user repo's file, linked), check the source line is
  `[ovm, tasks, --json, --space, anfs-dev]` (or the scopes they want) and `projects:` uses
  `<vault>/<space>` names; add `go` to apex's `vault.yml` `tags:` if they use `ovm validate`'s tag
  check; `ovm tasks --space anfs-dev | head`; `pjm pull` prints its counts (no "a source failed").
