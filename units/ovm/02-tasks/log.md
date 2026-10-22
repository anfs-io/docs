---
plan: ovm/02-tasks
job: ovm-02
date: 2026-10-22
---

# Log: ovm/02-tasks

Branch `ovm-02`, cut from `ovm-04b` (stack `e2612d2`, on `ovm-04a` `b4739e7`; neither merged): if
either changes, this rebases on it. The only shared lines are `OVM_VERSION`, the `cli_cmd` block
(one line added after `docs`) and the help epilogue (a paragraph appended). `cmd_docs`, the
header comment, the `source` lines and the `ANFS_LIB` check are untouched.

## What was built (stack, `packages/obsidian`)

- **`ovm tasks [<vault> | <vault>/<space> | <space> | DIR] [--space SPEC]`**, a new section
  `# --- Tasks` after `cmd_show`:
  - `TASKS_AWK`: one awk over every file of a scope (`_walk ... | xargs -0 -r awk`, as
    `_doc_scan`); per file: the frontmatter (its `id` by FM_AWK's rule; a block that never closes
    is replayed as body, `doc` null), CR stripped, blockquote/callout markers (`>`, `>>`) stripped,
    fenced code (```` ``` ````/`~~~`, 3+, closed by the same character at least as many times; a
    backtick fence whose info string has a backtick is not a fence), ATX headings (up to 3 spaces,
    not in a blockquote; closing `#`s stripped; a heading clears the parent stack), and a stack of
    list entries (indent: tab to the next multiple of 4) for `parent`. Control characters in text
    become spaces (so `\x1f` cannot break a row). Prints a sort key (vault, file, zero-padded
    line) and the ten fields.
  - `_tasks_scan <root> <dir> <name> <id>`; `cmd_tasks` resolves the scope as `cmd_docs` does
    (DIR, `<vault>/<space>`, `<vault>`, the vault you are in, every vault), plus a bare name that
    is no vault while you stand in one: that vault's space. `--space`: `_pairs_load`, then every
    pair whose `to` is SPEC; a vault not in `~/vaults` (`gone/x skipped: no vault gone ...`), a bad
    `spaces_path` or a missing folder is warned and skipped (an unreachable vault only by
    `_live_vaults`' own warning); no pair at all: `no vault space is linked into SPEC (...): ovm
    space link <vault>/<space> SPEC`, empty list, 0. An argument with `--space` is 64.
  - Rows go through `anfs_sorted` (vault, file, line) and `anfs_table "SPACE FILE LINE TEXT"
    "space file line text vault_id vault doc section parent tags[]"`: `line` a string, empty
    fields null, `tags` an array.
  - `cli_cmd tasks` (arg `[vault-or-space]`, so completion is `complete_vault_or_space`; `--space
    SPEC` completes from wsm); a help-epilogue paragraph with what is not understood (indented
    code, `%% %%`/HTML comments, setext headings).
- README: the command line and a "Tasks" section with the key table. stack `AGENTS.md`: an
  `ovm tasks` row ("exactly user/pjm's ten; no key added without pjm").
- `tests/pjm-pull.jq`: pjm 0.1.1's `cmd_pull` normalisation (`strip` and the `map({...})`),
  copied with a comment naming its source.
- obsidian **0.6.0** (`OVM_VERSION`, `package.yml`), by hand: a new command (minor). The hook:
  "already 0.6.0 since 0.2.0 was pushed, not bumping".

## Commits (branch ovm-02, not pushed)

stack:
- `d38f5a5` obsidian: ovm tasks lists the open `- [ ]` tasks ... (obsidian 0.6.0)

docs:
- this log

## Tests

`ANFS_TEST_LIB=~/.local/lib/anfs anfs work run ovm-02 stack -- bats packages/obsidian/tests
</dev/null`: **93/93** (was 77).

`tasks.bats` (new, 16): every case of the plan's spec — the five markers, spaces, a tab, a
callout and `>>`; not tasks (`[x]`, `[X]`, `[-]`, `[/]`, `- []`, `-[ ]`, empty and blank text,
the frontmatter, ```` ``` ````, `~~~`, four backticks "closed" by three); an unclosed frontmatter
and CRLF; `section` (null before the first heading, `## Closing ##`, `### C# #` → `C#`, a `#`
line in a fence, `#tag` is not a heading); `parent` (open, done, two levels, a sibling, a plain
bullet stops it, a two-tab child of a 4-space parent, a paragraph ends it); `tags` (several,
`two/sub`, once each, inline code, `#123`, `x#no`); `doc` (id, null, `index.md`/`log.md` null);
`space` (`notes/product` from a deep file, null at the root and directly in `spaces/`, a custom
`spaces_path`); the scopes (in a vault, every vault, `<vault>`, `<vault>/<space>`, a bare space,
DIR, errors 1, an empty space `[]` 0); `--space` (one space in two wsm spaces, two vaults in one,
the wsm space not installed; no entry: 0, `[]`, the warning; missing vault and folder warned;
with an argument 64); `exclude:`, `.obsidian`, `.trash`; `--json` (`map(keys) | unique` is exactly
the ten, `tags` an array, nulls, the sort); the contract through `pjm-pull.jq`; 500 files under 2
s (`EPOCHREALTIME`); help and completion.

Not run: shellcheck (not installed; `bash -n` passes). Core's `packages/anfs/tests/all`: this job
has no core worktree and core is unchanged. No VM, box or round trip: nothing here boots one.
`ANFS_WORK` is never set by these tests. Only macOS's awk (20200816) was run: no gawk or mawk on
this host, so Debian's awk is untested (the program avoids `{n,m}` intervals and gawk-only
functions).

## The pjm contract, against the live pjm (read-only)

A throwaway bats file in the job's tests directory (deleted after), using `helper.bash`'s temp
`HOME` (`/private/tmp/anfs-work-501-5/bats-run-.../home`, XDG dirs inside it, `PJM_*_HOME` unset
so pjm's paths are under that HOME): a vault `notes` with `spaces/product/p.md` (frontmatter id,
`## Work`, `- [ ] ship it #go #x`, a nested `- [ ] sub #go`, `- [ ] plain`); `ovm tasks --json >
src.json`; then the installed script, unmodified:
`~/.local/share/anfs/sources/user/packages/pjm/home/.local/bin/pjm pull --stdin < src.json`.

```
open items        3
tagged            2 (#go)
no heading        0
nested            1
exact duplicates  0 groups covering 0 items
```

and `items.jsonl` (under the temp HOME):

```
{"vault_id":"rogisgk4ox","vault":"notes","file":"spaces/product/p.md","line":5,"doc":"ffffffffff","space":"notes/product","section":"Work","parent":"","tags":["go","x"],"go":true,"text":"ship it #x",...}
{"vault_id":"rogisgk4ox",...,"line":6,...,"parent":"ship it #x","tags":["go"],"go":true,"text":"sub",...}
{"vault_id":"rogisgk4ox",...,"line":7,...,"parent":"","tags":[],"go":false,"text":"plain",...}
```

Every field lands: `line` as a number, `#go` detected and stripped from `text` and `parent`,
`space` the vault space. No real config, state or vault was read or written.

## Decisions made during the work

- **A bare `<space>` while in a vault** (the plan's scope table): a name that resolves as a vault
  is the vault; only a name that is no vault (exit 1 from the resolver) is tried as a space of the
  vault you are in. `docs` does not have this form; it was not added there.
- **The scope code is a copy of `cmd_docs`'**, not a shared function: 04a and 04b are unmerged
  and `cmd_docs` is theirs; refactoring it now would conflict on rebase. Folding both into one
  `_scope` is a small follow-up after merge.
- **An unclosed frontmatter block is body** (Obsidian renders it so; tasks are the point), `doc`
  null. The plan only said "skipped: the frontmatter".
- **One space after the marker** exactly, as the plan says (`-  [ ] x` is not a task).
- **An open box with empty text** (`- [ ]`) is a list entry, not a task: it is not a parent either.
- **Headings inside a blockquote/callout are not sections.**
- **Control characters → spaces** in text, parent, section and doc.
- `grep -rn items packages/obsidian` still finds 04a's `validate` (`items:` is not read, a local
  `items` array); there is no `ovm items` anywhere.

## Not done tonight: user/pjm (the person's repository)

The plan puts `packages/pjm` in this job; the person's repo was off-limits, so here are the exact
edits, for the person to make on a branch of `user` (`~/.local/share/anfs/sources/user` is the
live checkout: make them in a worktree or clone, not there):

1. `packages/pjm/home/.config/pjm/config.yml`, `sources:` and the comment above it:

   ```yaml
   # Where items come from: each entry is a command, one word per element, that prints items as
   # JSON. Start narrow; add more once the first report looks right. One task is pulled once per
   # source that reaches it, so two sources whose links overlap list it twice (dup counts both):
   # one source per vault space, or one --space per wsm space, avoids it.
   sources:
     - [ovm, tasks, --json, --space, anfs-dev]
     # - [ovm, tasks, --json, apex/product]
     # A directory that is not linked to a space is named by its path (the vault-scan spike's
     # other candidates were spaces/cnfs/products, spaces/rws/products and spaces/lgat/product):
     # - [ovm, tasks, --json, "/Users/roberto/Library/Mobile Documents/iCloud~md~obsidian/Documents/apex/spaces/cnfs/products"]
   ```

   The line that changes: `  - [ovm, items, --json, --space, user/dev]` →
   `  - [ovm, tasks, --json, --space, anfs-dev]`.
2. Same file, `report:`'s comment, line 13: `# (the vault's vault.yml excludes _generated).`
3. Same file, `projects:` (lines 33-35):

   ```yaml
   # Dispatch (a proof of concept). An item goes to the Firstmate project its space maps to
   # (firstmate project add); an item whose space is not here is not sent. A space is
   # <vault>/<space>, as ovm tasks prints it.
   projects: {}
   #  apex/product: anfs
   ```
4. `packages/pjm/home/.local/bin/pjm` line 5 and line 211: `` (`ovm items --json`) `` →
   `` (`ovm tasks --json`) ``. No code change.
5. `packages/pjm/package.yml`'s comment: `# ovm (in stack's obsidian package, ovm tasks --json) is
   the one source of items so far. fnox is ...` (drop "it has no `items` yet (units/ovm.md, plan
   02)"). Version `0.1.1` → `0.2.0` (by hand; the hook would make it 0.1.2).
6. `bats packages/pjm/tests </dev/null` (its fixtures already have the ten keys; unchanged), and
   `grep -rn 'ovm items\|pvm' packages/pjm` finds nothing.

## By hand, for the person after merge

1. After 04a's and 04b's by-hand steps: pull stack, `ppm install obsidian`; `ovm --version` is
   0.6.0.
2. Merge the user branch above; in `~/.config/pjm/config.yml` (the user repo's file, linked)
   check the source is `[ovm, tasks, --json, --space, anfs-dev]` (or the scopes you want) and
   `projects:` uses `<vault>/<space>` names.
3. If you use `ovm validate`'s tag check: add `go` to apex's `vault.yml` `tags:` (decided: `#go` is
   validated like any tag).
4. `ovm tasks --space anfs-dev | head`, then `pjm pull`: it prints its counts, no "a source failed".

## Open questions (for the person, before merge)

1. **`--space anfs-dev` depends on 04a's choice for `spaces/anfs/product`** (option A links the
   whole `anfs` space, so its tasks are `apex/anfs`; option B moves `product` up, so `apex/product`).
   `projects:` keys follow from that.
2. **Unclosed frontmatter read as body** and **a bare space name in a vault** (above): both beyond
   the plan's letter; keep?
3. The unit's open question on extra keys (the wsm space a task came through, the checked state)
   is unchanged: none added.
