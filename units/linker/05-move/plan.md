---
---

# Plan 05 — `move`: a resource to another source, with its history

Written 2026-10-22 against main (linker 01 and 02, wsm-spaces 01–03 and ovm 01 merged), after the
decisions taken with the person that day (`units/linker.md`, "Decided with the person, 2026-10-22
(move)"). It replaces section 3a of plan 03 ("A whole package — to be discussed"), and it does
not depend on plan 03.

## Context — read these files first

The job is `anfs work new <job> core anfs-dev` (docs are not in the job: this plan and its log are
the only docs it touches, and the log is written beside the plan as usual). core holds the work;
anfs-dev only loses `ppm move`.

Paths are core's unless they name another source.

- `units/linker.md` (the Decisions, all of them); plan 02's `log.md`; plan 03's notes at its top
  (the `file` group and `anfs owner|mv|status`; this plan is what its 3a becomes)
- anfs-dev `packages/dev/home/.local/lib/ppm/move.sh` — today's `ppm move`, broken since plan 01
  (it calls `stow` and loops over `${PPM_GROUP_ID:-}`). Its comments are the semantics this plan
  keeps: trackers moved, claims renamed, hooks not re-run, dirty repos refused, the dependents
  rule (`_move_broken_dependents`, `_move_resolvable`), the base packages refused, pathspec-limited
  commits and the re-add after a hook rewrote a file (`_move_commit`)
- anfs-dev `AGENTS.md` ("Layout", "Moving a Package Between Repos"), `packages/dev/README.md`
  (line 15)
- `packages/anfs/home/.local/lib/anfs/resolve.sh` — `anfs_sources` (only the sources holding
  `<dir>/`, so their *filtered* index is what `min_index` compares), `_anfs_split`
  (`ANFS_NAME_NEST`), `anfs_find` (`min_index`, `ANFS_RESOLVE_MARKER`), `_anfs_resolve_one` (a
  dependency resolves from the depending layer's index down, qualified or not)
- `packages/anfs/home/.local/lib/anfs/repo.sh` — `anfs_repo_dirty`, `anfs_repo_status`
- `packages/anfs/home/.local/lib/anfs/sources.sh` — `gitsrc_collect`, `GITSRC_NAMES`,
  `gitsrc_dir_reply`
- `packages/anfs/home/.local/lib/anfs/cli.sh` — `cli_cmd`; `iso.sh`'s `anfs_iso_cli` is the
  precedent for a library that declares commands for the tool that calls it
- `packages/anfs/home/.local/lib/anfs/link.sh` — `anfs_link_plan`, `anfs_link_apply` (a dangling
  link into `$ANFS_SOURCES_HOME` is `replace`), `anfs_unlink`
- `packages/anfs/home/.local/lib/anfs/callbacks.sh` — registrations in
  `$ANFS_STATE_HOME/callbacks/<tool>.yml`, keyed `repo/pkg`
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `unstow_package` (it also unlinks the
  `links: {src}` links, which a move must not), `stow_package`, `_backup_file`/`_backup_restore`
  (the backup manifest is keyed `repo/pkg`), `ppm_callback_invoke` (finds the registering
  package's `install.sh` from the registration's `repo/pkg`)
- `packages/ppm/home/.local/lib/ppm/packages.sh` — the trackers (`_tracker_path`:
  `installed/<repo>/<pkg>.yml`; nothing inside one names its repo), `meta_depends`
- `packages/ppm/home/.local/lib/ppm/file.sh` — `claims.yml` (`claimant`, `owner`: `repo/pkg`)
- `packages/ppm/home/.local/bin/ppm` — the declarations, `cli_before_command` (`force`,
  `dry_run`), the `$PPM_LIB_DIR/*.sh` loop
- `packages/wsm/home/.local/bin/wsm` — the header (ids, the tree, "nothing is recorded"),
  `_spaces_load`, `_state_load`, `_space_home`, `_install_space`, `_space_repositories`,
  `_space_links`, `_space_skills`, `_space_deps`, `_rmdir_empty_up`, `_wsm_emit`, the help epilogue
- anfs-dev `packages/dev/home/.local/bin/anfs-work` (~226: `git worktree add` from a space's
  clone, so a clone can have worktrees in `~/work/`)
- `core/stacks/system/stack.yml` — a stack names resources `source/name`
- `AGENTS.md` (core): "wsm: spaces" ("wsm never commits"), the `ppm move` mentions (~735, ~1194),
  "Lib Structure", "Testing"

### What the history replay does on real history (tried 2026-10-22)

On scratch clones (`git clone --no-local`) of all eight sources on this Mac, every resource
directory was replayed into an empty repository with
`git format-patch --root -o <tmp> -- <dir>/<name>` and `git am --committer-date-is-author-date`,
then compared (tree hash of the directory at HEAD in both, and every commit's author name, email,
date and full message). The real repositories were not touched; the scratch directory is deleted.

- **93 resource directories** (core 14, stack 49, user 10, anfs-dev 7, rws 4, lgat 4, cnfs 4,
  roteoh 1: packages, containers, machines, images, spaces), **782 commits**, 780 patches (two
  merge commits, which format-patch skips).
- **92 replayed exactly**: the same tree, and every commit's author, author date and whole message.
  zsh's 61 commits take about 1 s. Commits that also touched other paths are cut down to the
  directory's part (no other path appears in any patch).
- **Renames**: 8 directories have renames inside them (op's `linux/` and `macos/` into `home/`,
  nvim's README); they apply as renames. A rename *across* directories (zsh's `.inputrc` to
  `stack/cli`, `ruby-tools` into `ruby`) is a delete on one side and an add on the other, and
  applies.
- **The one failure**: user `packages/claude`. Two branches both deleted
  `home/.claude/commands/rjr/cd.md` and were merged (`f68b9aa`); linearised, the second deletion
  finds nothing (`does not exist in index`). `git am --abort` left the repository as it was.
- **Committer**: the person running it, and the committer date becomes the author date (3 of op's
  28 commits had a later committer date, from an amend or a rebase). So "dates kept" means
  author dates.
- **What the defaults get wrong** (on a fixture, since none of it is in the real repositories):
  - a subject starting with `[...]` loses it (`[wip] x` becomes `x`) unless format-patch and am
    both take `-k`;
  - CRLF line endings are stripped (the tree differs) unless am takes `--keep-cr`;
  - am prints trailing-whitespace warnings, and `apply.whitespace=fix` in someone's config would
    change the content: `--whitespace=nowarn` pins it;
  - a commit message holding a line `diff --git ...` makes am fail (`bad git-diff`);
  - a commit message holding a line `---` is **cut there, silently**, and am succeeds. None of the
    real repositories has such a line (checked core, stack, user), but it is why the replay is
    verified rather than trusted.
  - Binary files, a symlink and the executable bit replay exactly (with format-patch's default
    `--binary`). No real resource has a binary patch.
- **A dirty destination**: an unstaged change or an untracked file elsewhere in it does not stop
  am. A staged change does (`Dirty index: cannot apply patches`), **and leaves
  `.git/rebase-apply` behind**: every failure path runs `git am --abort`.
- **Hooks**: `git am` runs `applypatch-msg`, `pre-applypatch` and `post-applypatch`, never
  `pre-commit`. anfs-dev's hooks directory has only `pre-commit` (the version bump), so a replay
  bumps nothing; a plain commit (the source's removal, the fallback copy) runs it.

## Overview

Today a package moves between sources with anfs-dev's `ppm move`, which no longer works, which
lives in a source that only toolkit developers install, and which loses the package's history
(the directory is `mv`'d and committed as new in the destination). After this plan:

```
ppm move stack/rails user          # a package
wsm move lgat/tech/web rws         # a space (and the spaces under it)
<tool> move <source/name> <to-source> [-n] [-f]
```

- **The tool orchestrates; the library supplies building blocks.** `lib/anfs/move.sh` has three
  functions and no hooks: `anfs_move_check`, `anfs_move_tree`, `anfs_move_cli`. Each tool that
  has a move writes its own `cmd_move`, which reads top to bottom with that tool's own functions
  (ppm: check, unlink, tree, tracker, relink, renames; wsm: check, tree, the installed directory,
  relink). A tool without a `cmd_move` has no move.
- **History moves with the directory**, as commits: replayed into the destination
  (format-patch/am), or, when the replay fails, copied in one commit that names where it came
  from. Then one commit in the origin removes it. Both repositories are left clean, never pushed.
- `-n` prints the checks, the commits that would be replayed (and whether the replay works), and
  the tool's own steps; it changes nothing.
- ppm and wsm in this plan. pcm, pmm and psm are a follow-on plan (see "Not in this plan" for
  why, and what was found).

## Implementation

### 1. `lib/anfs/move.sh` (new)

Functions only, the bash floor (5.2; not in the bootstrap set: it needs git and is used by tools
that run after the bootstrap). Requires `sources.sh`, `resolve.sh` and `repo.sh`, sourced by the
tool first. Messages through `log.sh`, with the tool's name. The header says what a move is (the
directory with its history, never pushed, both repositories left clean), and that this is the one
place the toolkit commits on the user's behalf (decision 3e).

Names: `<dir>` is the tool's top-level directory (`packages`, `spaces`, ...); `<id>` is
`<source>/<name>`, always qualified (a bare name is refused: it is a layer in every source);
`<name>` may be a path for a nesting kind (`ANFS_NAME_NEST=true`, `ANFS_RESOLVE_MARKER`, as the
tool sets them for `resolve.sh`). The path in both repositories is `<dir>/<name>`.

#### `anfs_move_check [-f] <dir> <id> <to> [deps-fn]`

Prints one line per check, `<state><TAB><check><TAB><detail>`, `ok`, `refused` or `forced` (a
refusal `-f` went past), plus `note` lines; the same lines are `-n`'s first part. Returns 0 (go
on), 1 (refused; `-f` would go on) or 2 (refused, whatever the flags).

Positions are compared in the whole source order (`gitsrc_collect`, `GITSRC_NAMES`), not in
`anfs_sources <dir>`'s filtered one: the destination may not hold `<dir>/` yet, and the two orders
agree on which source is above which.

Refused always (2):

- `<id>` is not qualified, or `<from>/<dir>/<name>` is missing (with a marker: has no marker
  file), or `<from>` is `<to>`.
- `<to>` is not a configured source (`anfs src list`), or its clone is missing; either clone is not
  a git repository (a `local` directory has no history to carry: say so).
- The destination has the path: `<to>/<dir>/<name>` exists, is tracked (`git ls-files`), or a
  directory on its way is a file.
- Either repository is in the middle of something (`.git/rebase-apply`, `rebase-merge`,
  `MERGE_HEAD`, `CHERRY_PICK_HEAD`), or on no branch (a detached HEAD: the commits would be lost).
- **The resource's own path has uncommitted or untracked changes** (`git status --porcelain --
  <dir>/<name>`, ignored files excepted). New compared with today's `ppm move`, which carried
  them along under `-f`: the history that moves is what is committed; commit or stash first.

Refused unless `-f` (1):

- **Dirty repositories**: anything else uncommitted in either one (`anfs_repo_dirty`). Under `-f`
  the commits are still limited to `<dir>/<name>`. A *staged* change in the destination stops
  `git am` (see Context), so under `-f` with a dirty index the replay is skipped and the fallback
  copy used, said as the reason.
- **Dependents** (given `deps-fn`, the function the tool passes to `anfs_resolve`, which prints a
  resource directory's dependencies): every resource of `<dir>` in every source whose
  dependencies name `<name>` (or, with a marker, a name inside the moved subtree):
  - named bare: it still resolves after the move only if a source at or below the dependent's
    position holds it, `<to>` counting and `<from>` not (move.sh's `_move_resolvable`). Moving
    *up* in priority hides the resource from the dependents below its new place;
  - named qualified, `<from>/<name>`: it never resolves after the move.
  Each is listed as `<source>/<name> depends on <dep>` with the file that says so.
- **Its own dependencies**: each dependency of the moved resource (and of every resource in the
  moved subtree) must resolve from `<to>`'s position down (`anfs_find <dir> <dep> <index>`; for a
  dependency on a resource inside the moved subtree, it moves too). Moving *down* in priority can
  lose them. Today's `ppm move` does not check this.
- **Qualified references in stacks**: `stacks/*/stack.yml` in every source, the `<dir>` list's
  entries (a string, or a map's key) equal to `<from>/<name>` (core's `system` stack names
  `core/pcm`, `core/wsm`, ...). Listed with the file; not rewritten.

Notes (never refusals):

- **Layers and shadowing**: other sources holding `<dir>/<name>`. For packages it is the layer
  order before and after (`git: user, stack -> user, rws, stack`); the files each layer links
  are recomputed only by the next `ppm install`. For a kind that resolves first-match
  (`ANFS_RESOLVE_FIRST`, spaces), a bare spec then names another source's resource: said.
- The branch each repository is on, and that nothing is pushed.

#### `anfs_move_tree [-n] <dir> <id> <to>`

Moves `<dir>/<name>` from `<from>` to `<to>` with its history, in four steps; `<from>`'s and `<to>`'s
HEADs are recorded first. Sets `ANFS_MOVE_METHOD` (`replay` or `copy`), `ANFS_MOVE_COMMITS` (how
many were replayed) and `ANFS_MOVE_WHY` (why it copied). Prints one line at the end:
`moved packages/rails: stack -> user, 10 commits replayed` or `... copied in one commit (the
replay failed: <why>); its history stays in stack`. Returns 0, or 1 with both repositories as they
were.

1. **Patches**: `git -C <from> format-patch -q -k --root -o <tmp> -- <dir>/<name>` (`<tmp>` from
   `mktemp -d`, removed on every exit). No patches (nothing committed there; only possible for a
   path the check let through, so never in practice) is a `copy`.
2. **A trial**, so the destination is touched only by a replay known to work: a scratch repository
   (`git init` in `<tmp>`, one empty commit) takes
   `git am -q -k --keep-cr --whitespace=nowarn --no-3way --committer-date-is-author-date
   --no-verify <tmp>/*.patch`. Valid because the path is free in `<to>`: the patches touch nothing
   else, so whether they apply does not depend on the rest of its tree. Then the **verification**:
   the tree of `<dir>/<name>` at the scratch HEAD equals `<from>`'s at HEAD, and the
   author/date/message list of the replayed commits equals `git -C <from> log --no-merges
   --format=... -- <dir>/<name>`. A failure (am's message, the first line of it) or a mismatch
   ("the replay changed <what>: <commit>") makes this a `copy`.
3. **Into the destination**:
   - `replay`: the same `git am` in `<to>`, verified the same way. If it still fails: `git am
     --abort` when `.git/rebase-apply` exists, `git -C <to> reset -q --keep <to's HEAD>` (keeps
     changes elsewhere under `-f`), then `copy`.
   - `copy`: `git -C <from> archive HEAD -- <dir>/<name> | tar -x -C <to>` (exactly what is
     committed: modes and links kept), `git -C <to> add -- <dir>/<name>`, and one commit limited to
     that path:
     ```
     <dir>/<name>: moved from <from>

     Copied from <from> (<its origin URL>) at <full SHA>, where its history is:
     git log <SHA> -- <dir>/<name>. The history was not replayed: <why>.
     ```
     This commit runs the repository's hooks (the version bump may touch the manifest); the path is
     re-added afterwards, as `_move_commit` does, so a hook's rewrite is not left staged.
4. **Out of the origin**: `git -C <from> rm -r -q -- <dir>/<name>` and one commit limited to the
   path, hooks run, message `<dir>/<name>: moved to <to> (<method>, <to's new short SHA>)`. If this
   commit fails (a hook refuses): `git -C <from> reset -q -- <dir>/<name>` and `git -C <from>
   checkout -- <dir>/<name>` put it back, `<to>` is reset to its recorded HEAD, and it returns 1.
   Then **ignored files** left in the origin's directory (`git ls-files -o -i --exclude-standard --
   <dir>/<name>`: a `.DS_Store`, a build output) are moved to the same place in `<to>` and the
   emptied directories removed, because a directory left at `<from>/<dir>/<name>` would still
   resolve as a resource (`anfs_find` tests `-d`). One that `<to>` does not ignore makes `<to>`
   dirty: said.

Never pushed. Never `--force`d. After step 4 both repositories are clean if they were before
(checked; a difference is said). `-n` runs steps 1 and 2 only, and prints the commits that would be
replayed (`<short> <author date> <author> <subject>`, the count first), and `replay ok` or `would
copy: <why>`.

#### `anfs_move_cli <noun> <arg-kind>`

Declares the command for the tool that calls it, beside its own declarations (after `cli.sh`), as
`anfs_iso_cli` does:

```
cli_cmd move "Move a <noun> to another source, with its history (commits in both; nothing is
pushed)" \
  --opt "-f --force: Despite uncommitted changes elsewhere in either repository, a dependent
that will not resolve, or a stack that names it" \
  --opt "-n --dry-run: Print the checks, the commits to replay and the steps; change nothing" \
  --arg "<arg-kind>" --arg "source"
```

The completion kinds are the tool's (`complete_<arg-kind>`, `complete_source`). The help's
paragraph about what a move does and does not do is the same in every tool (a `cli_help_command`
addition is not needed: the summary says it).

### 2. ppm: `cmd_move` (`packages/ppm/home/.local/lib/ppm/move.sh`, new)

In ppm's lib directory, so the `$PPM_LIB_DIR/*.sh` loop sources it; `bin/ppm` sources
`lib/anfs/move.sh` and `repo.sh` with the others and calls `anfs_move_cli package package` after
its declarations. `cmd_move <id> <to>`, top to bottom (today's semantics, on the linker):

1. `collect_repos`. Refuse `core/anfs` and `core/ppm` unless `-f` (install.sh links them from
   `$PPM_REPO_DIR`), as today.
2. `anfs_move_check ${force:+-f} packages <id> <to> meta_depends`; stop on non-zero.
3. Whether it is installed (`meta_is_installed <from> <pkg>`). If so, the **relink pre-plan**:
   `anfs_link_plan "<from-dir>/home" "$HOME" <tracker files>` must say `keep` for every one (they
   are this package's links). Anything else (an editor replaced a link with a file, another package
   took the path) refuses the move before anything changes: `~/<path> is not stack/rails's link
   any more (<class>): ppm install -f stack/rails, or ppm file protect it, then move`. So the
   relink after the tree move cannot meet a conflict.
4. `-n`: the check's lines, `anfs_move_tree -n`'s, and ppm's steps, then return:
   ```
   unlink   14 links into stack/packages/rails (no remove hooks)
   relink   14 links from user/packages/rails (no install hooks)
   tracker  installed/stack/rails.yml -> installed/user/rails.yml
   claims   2 entries (claimant or owner stack/rails)
   callback ppm: stack/rails -> user/rails
   backup   1 entry
   ```
   Not installed: `not installed: ppm install user/rails links it` instead of the first five.
5. Installed: `anfs_unlink "<from-dir>/home" "$HOME" <tracker files>`. Only the tracker's files:
   not `unstow_package`, which also unlinks the `links: {src}` links (they point outside the
   package and are left as they are), and never the hooks.
6. `anfs_move_tree packages <id> <to>`. On failure: relink from `<from>` (`anfs_link_apply` with
   the tracker's list) and return 1; the repositories are as they were.
7. Installed: the tracker moves (`installed/<from>/<pkg>.yml` to `installed/<to>/<pkg>.yml`,
   unchanged: version, files, links, `installed_deps`, resources; the emptied `<from>` directory
   removed), then the **relink**: `anfs_link_apply "<to-dir>/home" "$HOME" <tracker files>` (the
   paths are free after step 5; a directory link is one entry). Exactly the files the layer had,
   which also leaves protected files and a higher layer's alone. If it fails (only I/O can make it
   fail after step 3): `ppm_fail "<to>/<pkg> moved, but its links are not all made: ppm install -f
   <to>/<pkg>"`; the tree is not moved back (that would be another move).
8. The **renames** (`<from>/<pkg>` to `<to>/<pkg>`), each a no-op when nothing names it:
   - `claims.yml`: `claimant` and `owner` (move.sh's `_move_claims_rename`);
   - the callback registration (`$ANFS_STATE_HOME/callbacks/ppm.yml`), else `ppm_callback_invoke`
     looks for `install.sh` in the old directory and skips the package's callback for good. Today's
     `ppm move` misses this;
   - the backup manifest's owner column (`$PPM_STATE_HOME/backup/manifest`), else `ppm remove`
     never puts the user's original file back. Today's `ppm move` misses this too.
9. `flush_user_messages`.

No mise run, no declared resources, no `PPM_SKIP_FILE` rebuild: the content did not change.

### 3. wsm: `cmd_move` (in `packages/wsm/home/.local/bin/wsm`)

wsm sources `lib/anfs/move.sh` and `repo.sh`, declares `anfs_move_cli space spec`, and gains
`complete_source` (ppm's, over `gitsrc_collect`). A space moves with **every space under it**: its
directory in the source holds them, and so does its directory in the tree.

`cmd_move <id> <to>`:

1. `_spaces_load`, `_state_load` for `<from>` and `<to>`. Refuse a bare path, and a source's root
   space (`<from>` alone: its directory is the source's whole `spaces/`; move the spaces under it
   instead). The moved set is `<id>` and the spaces under it, as `<from>/<path>...` and
   `<to>/<path>...`.
2. `anfs_move_check ${force:+-f} spaces <id> <to> _space_deps` (wsm already sets
   `ANFS_NAME_NEST`, `ANFS_RESOLVE_MARKER=space.yml`, `ANFS_RESOLVE_FIRST`). A dependency that
   names a space by id is the common case in spaces (`depends: [lgat/tech]`), so most dependents on
   a moved space are refusals of the qualified kind; the person edits them (`-f` moves anyway, and
   they then fail with `no space ...`). Ids are not rewritten anywhere: not in other spaces, not in
   the moved ones' own `depends:` (a dependency between two moved spaces written `<from>/...` is
   listed like any other).
3. wsm's own refusals, before anything changes:
   - the target in the tree is free: `$ROOT/<to>/<path>` does not exist, and no directory on its way
     is a file or a link (no `-f`: the user's files could be there);
   - every moved space that is installed reads `ok` or `partial` with only `missing` files (no
     `wrong`, `in the way` or leftovers: `wsm install -f <id>` first), so the relink meets nothing;
   - a `links: {src}` or a `skills: {repo}` written as a relative path that leaves the space's
     directory (`../shared`) would point into another source after the move: refused unless `-f`,
     each listed.
4. `-n`: the lines of both and wsm's steps, then return:
   ```
   move     ~/spaces/lgat/tech/web -> ~/spaces/rws/tech/web (3 clones inside, untouched)
   relink   lgat/tech/web: 12 links; lgat/tech/web/api: 4 links
   worktree repair  repos/app (2 worktrees)
   events   wsm remove lgat/tech/web lgat/tech/web/api; install rws/tech/web rws/tech/web/api
   ```
   Not installed: `not installed: wsm install rws/tech/web links it`.
5. `anfs_move_tree spaces <id> <to>`; on failure return 1 (nothing of wsm's has changed yet).
6. Installed: **the installed directory moves**: `mkdir -p "$ROOT/<to>/<parent>"`, `mv
   "$ROOT/<from>/<path>" "$ROOT/<to>/<path>"` (one rename under one root: the clones, the user's
   files and the copies go with it untouched), then `_rmdir_empty_up` from `$ROOT/<from>/<parent>`.
   The links inside keep their text; at the same depth they now point at
   `sources/<from>/spaces/<path>/...`, which the tree move emptied, so they dangle into the sources.
7. **Relink**: `_spaces_load` and `_state_load` again, then for each moved space, parents first,
   `_install_space <to>/<path> false false` (a dangling link into the sources is `replace`: the
   link is rewritten to the new source; `files:` copies are `present` and left), `_space_links`
   (a `{src}` link at the same depth still resolves: `ok`), and `_space_skills` (a local-path skill
   whose lock names the old source's path is installed again from the new one; to verify while
   building, see open question 3). No `_space_repositories`: the clones are there. If the `mv` or a
   relink fails: `fail` with `the source moved; wsm install <to>/<path>` (not moved back).
8. **Worktrees**: for each clone under the moved directory (`_space_repositories`' list) that has
   linked worktrees (`git -C <clone> worktree list --porcelain`, more than one), `git -C <clone>
   worktree repair`. `anfs work` makes worktrees of a space's clones in `~/work/<job>/`; their
   `.git` files name the clone's old path.
9. **Events**: `WSM_EV_REMOVED+=(<from>/<path>...)`, `WSM_EV_INSTALLED+=(<to>/<path>...)`, so ovm
   (`ovm link apply --event`) sees the move as what it is for it, one space gone and one new. A vault
   link that names the old id becomes `unknown` to ovm until the person edits it: said in the
   output.

"wsm never commits" becomes "wsm commits only in `wsm move`" (AGENTS.md). `WSM_VERSION` minor bump,
by hand.

### 4. anfs-dev

- Delete `packages/dev/home/.local/lib/ppm/move.sh` (its `cli_cmd move` declaration is in it; with
  it go the only `stow` calls and the only `PPM_GROUP_ID` in anfs-dev's code).
- `AGENTS.md`: `ppm move` out of "Layout"; the section "Moving a Package Between Repos" goes (its
  content is now core's, section 5).
- `packages/dev/README.md` line 15: `ppm move` out of the list.
- No test, fixture or round trip mentions `ppm move` (checked: `grep -rn 'ppm move\|move\.sh'`
  finds only the three places above). The other `stow` words in anfs-dev (the hooks' comments,
  README's "the first stow", AGENTS.md's hooks section) are prose about linking, not this plan's;
  left (see "Not in this plan").

### 5. Documentation (core)

- `AGENTS.md`:
  - a section "Moving a resource between sources", from anfs-dev's, rewritten: `<tool> move`, the
    three library functions, what each check refuses, the replay and its fallback (with the
    flags and why each is there), what ppm and wsm do around it, that it commits and never pushes;
  - "wsm: spaces": "wsm never commits" as above;
  - "Lib Structure": `move.sh`;
  - the `ppm move` mentions (~735, ~1194): `ppm move` is core's now.
- ppm's and wsm's help come from the declarations. wsm's epilogue gets one paragraph: a move takes
  the spaces under it, the installed directory moves with its clones, ids are not rewritten.

### 6. Commits, in order

1. core: `lib/anfs/move.sh`; `packages/anfs/tests/move.bats`
2. core: ppm's `move.sh`, `bin/ppm`; `packages/ppm/tests/move.bats`
3. core: wsm's `cmd_move`, `complete_source`; `packages/wsm/tests/move.bats`; `WSM_VERSION`
4. core: `AGENTS.md`
5. anfs-dev: `move.sh` deleted; `AGENTS.md`, `packages/dev/README.md`

The pre-commit hooks bump the package versions.

## Test Spec

All with fixture repositories made in `$BATS_TEST_TMPDIR` (`git init`, commits with set authors
and dates through `GIT_AUTHOR_*`/`GIT_COMMITTER_*`, `GIT_CONFIG_GLOBAL=/dev/null`, as `repo.bats`
does), `HOME` and the sources inside it, the source lists written by the test. No network, no real
repository.

`packages/anfs/tests/move.bats` (new):

- **Replay**: a resource with 5 commits (two authors, dates a year apart, one commit that also
  touches another directory, one rename inside it, a binary file, a symlink, an executable, a CRLF
  file, a subject `[wip] ...`) moves with `ANFS_MOVE_METHOD=replay`: 5 new commits in the
  destination with the same authors, author dates and messages, committer date = author date; the
  tree hash of the path equal to the origin's before the move; the other directory's change not in
  the destination; one commit in the origin naming the destination; both repositories clean;
  nothing pushed (the bare remotes' refs unchanged).
- **Fallback**: two fixtures that cannot replay, a history with two branches deleting the same
  file then merged (user/claude's case), and a commit message containing `diff --git`: `copy`, one
  commit in the destination whose message names the origin, its URL and the SHA, the same tree;
  the destination's history has no partial replay; `.git/rebase-apply` absent in both. A message
  with a line `---`: the trial's verification catches it, `copy`, and the reason names the commit.
- **The real am failing after a good trial** (a `git` wrapper first on PATH that fails the
  second `am` only): the destination is back at its HEAD with no `.git/rebase-apply`, then
  `copy`. A destination `pre-applypatch` hook that exits 1 does not stop a replay
  (`--no-verify`).
- **Dirty**: an untracked file and a modified file elsewhere in either repository: 1, both named;
  with `-f`: 0, the move's commits hold only the path, the other changes still uncommitted after.
  A staged change in the destination under `-f`: `copy`, the reason says the index. A change
  inside the resource's own path: 2 with or without `-f`.
- **Target exists**: the path in the destination (a directory; a tracked file at it; a file on the
  way): 2 with or without `-f`, nothing changed.
- **Not a source / not git / same source / missing / bare name / detached HEAD / rebase in
  progress**: 2 each, with its message.
- **Dependents**: sources A > B > C. A resource in C that B's `x` depends on (bare), moved to A:
  refused, `B/x` listed; moved to B: allowed. `x` depending on `C/y` qualified: refused whatever
  the destination. The moved resource depending on something only in B, moved to C: refused (own
  deps). `-f`: 0, `forced` lines. A destination with no `<dir>/` yet: positions still right.
- **Nested** (`ANFS_NAME_NEST`, marker `space.yml`): a subtree with two spaces moves as one; a
  dependent on the inner one is found.
- **Stacks**: a `stack.yml` naming `<from>/<name>` (string and map form): refused, the file named.
- **Ignored files**: a `.DS_Store` (ignored in both) in the moved directory ends in the destination;
  nothing is left at the origin's path.
- **A hook that refuses the origin's commit**: both repositories back as they were, 1.
- **`-n`**: the check lines, the commit list and `replay ok` / `would copy: ...`; both repositories'
  HEADs, indexes and working trees byte-for-byte unchanged (`git status --porcelain`, `rev-parse`,
  a listing of the trees).

`packages/ppm/tests/move.bats` (new; ppm's libs sourced as `backup.bats` does, the sources as git
fixtures):

- An installed package with a per-file link, a directory link and a `links: {src}` entry, a claim
  naming it as claimant and another as owner, a callback registration, a backup manifest line:
  after `cmd_move`, every link resolves into `<to>`'s package and is relative; the `src` link is
  the same inode; the tracker is at `installed/<to>/<pkg>.yml` with the same content and the old
  one and its emptied directory gone; claims, callback and backup name `<to>/<pkg>`; `ppm install
  -n <to>/<pkg>` afterwards plans only `keep`.
- **Hooks not run**: an `install.sh` whose `pre_remove`, `post_remove`, `pre_install`,
  `post_install` and `install_$(os)` each write a marker: none written.
- **A layered package**: `user/git` above `stack/git`, sharing a file; moving `stack/git` to `rws`
  relinks only the files stack's layer had; the shared file still points into user.
- **The pre-plan**: a link replaced by a real file: refused before anything changes (both repos'
  HEADs, the links and the tracker unchanged).
- **Tree move fails** (the origin's `pre-commit` hook refuses the removal commit): the links are
  back into `<from>`, the tracker where it was, both repositories as before.
- Not installed: the tree moves, claims renamed, no tracker made, the message says how to install.
- `core/ppm` refused without `-f`.
- `-n`: ppm's steps printed; nothing changed (links, tracker, claims, repositories).

`packages/wsm/tests/move.bats` (new; `helper.bash`'s fixtures, with git sources):

- An installed space with a nested installed space, a clone inside (a real repository with a
  commit and an untracked file), a `files:` copy, a `{src}` link and a user's own file: after `wsm
  move`, `$ROOT/<from>/<path>` is gone (and its emptied parents up to `$ROOT/<from>`),
  `$ROOT/<to>/<path>` has everything; the clone's HEAD, status and inode of its directory are the
  same; the copy and the user's file untouched; every link of both spaces resolves into `<to>` and
  `wsm ls` reads `ok` for both new ids and has no row for the old ones.
- A clone with a linked worktree elsewhere in the temp dir: after the move, `git -C <worktree>
  status` works (repair ran).
- **Events**: a registered listener receives `remove` with the old ids and `install` with the new
  ones (as `events.bats` checks).
- **Ids**: another space with `depends: [<from>/<path>]`: refused, the file named; `-f`: moved, and
  `wsm install` of that dependent then fails with `no space`. A relative `links: {src: ../x}`:
  refused, then `-f`.
- **Refusals**: the root space; a bare path; `$ROOT/<to>/<path>` existing; a moved space with a
  `wrong` link (until `wsm install -f`).
- Not installed: only the source moves; nothing appears under `$ROOT`.
- `-n`: wsm's steps; the tree, the links and the repositories unchanged.
- The existing wsm tests pass.

## Verification

Every command runs in the job: `anfs work run <job> core -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs; nothing is installed on this host and no real source is moved.

- [ ] While working, only the file touched: `bats packages/anfs/tests/move.bats`,
      `packages/ppm/tests/move.bats`, `packages/wsm/tests/move.bats`
- [ ] Once, at the end: `packages/anfs/tests/all` (pmm's and pcm's suites stay off; neither
      changes in this plan)
- [ ] anfs-dev: `anfs work run <job> anfs-dev -- bats packages/dev/tests </dev/null`;
      `grep -rn 'ppm move\|move\.sh\|PPM_GROUP_ID' packages AGENTS.md` in anfs-dev finds nothing
- [ ] **On copies of real sources** (the experiment of the Context, through the job's code): clone
      `stack`, `user`, `rws` and `lgat` with `--no-local` into the job's `TMPDIR`, and point an
      isolated `HOME`'s source lists at them (built as plan 02's verification describes; `env -i
      HOME=<copy> PATH=...`, no `*_HOME` reaching a real directory). Then:
  - `ppm move -n stack/rails user`, then `ppm move stack/rails user` (not installed in the copy:
    tree only): `replay`, as many commits as `git log --oneline -- packages/rails` had, the tree
    hash equal, both copies clean
  - `ppm move -n user/claude rws`: `would copy`, naming the failing commit
  - `wsm move -n` of a space in the `lgat` copy
  - the outputs recorded in the log
- Deferred to the end of the roadmap (in the log, not run): nothing; this plan touches no VM path.

## After merge, on this Mac (by hand, for the person)

core and anfs-dev merge together: until anfs-dev's `move.sh` is gone, `~/.local/lib/ppm/move.sh`
is a live link into `anfs-dev/dev`, and installing `core/ppm` would find another package's link
at the path its new `move.sh` takes.

1. `anfs src update core anfs-dev` (or pull both), then `ppm install core/ppm` (the old link now
   dangles into the sources: `replace`), then `ppm install anfs-dev/dev` (its tracker still lists
   `move.sh`; the cleanup leaves the link, which points into core/ppm now).
2. `ppm move -n` on any package, to see it run.

## Not in this plan

- **pcm, pmm and psm: a follow-on plan** (`linker/06-move-containers-machines-skills`, to write
  after pmm's and pcm's suites are back on). Why not here: their suites are off until further
  notice (ROADMAP, 2026-10-19), so a `cmd_move` there could not be tested in this job; they find
  resources through `catalog.sh`, not `resolve.sh`, with no priority rule for dependencies, so
  `anfs_move_check`'s dependents part would take a catalogue variant (shadowing, qualified names).
  What was found (2026-10-22, from the code), for that plan:
  - **pcm**: everything at runtime is keyed by the bare name (compose project, the volumes
    `$PCM_VOLUMES_HOME/<name>/` in DATA, `$PCM_ENV_HOME/<name>.env`, snapshots, the generated
    override in CACHE); only the labels `io.pcm.id` and `io.pcm.source` name the source, and the
    next `up` rewrites them. Stopping first is required: `compose -f <definition dir>/compose.yml`
    needs the old path, relative bind mounts were resolved to it when the containers were made, and
    `check_not_running_elsewhere` refuses `up` of `<to>/<name>` while containers labelled
    `<from>/<name>` run. So: check, `compose_command down` (with `--force`, the dependents it
    stops recorded), `anfs_move_tree containers`, `load_sources`, `compose_command up` of it and
    of those dependents; if the tree move fails, `up` from the old place. A dependent naming
    `<from>/<name>` breaks; a bare name may resolve elsewhere after the move. No `-n` exists in
    pcm today.
  - **pmm machines**: nothing names the source (disks, `state.yml`, the run directory, sockets,
    tart's `pmm-<name>` are all by name), and a running machine never reads its definition again,
    so a move needs **no stop and start**: check (shadowing: a higher source with the same name
    would hide the moved one) and `anfs_move_tree machines`. **Images** need more: their `from:`
    parent must stay at or below, every child naming them must still find them, and the build
    record's `.id` follows only on the next `pmm image build` (`build_adopt`). Machines in the
    follow-on, images left out or refused when anything refers to them.
  - **psm**: a move makes sense (a skill is `skills/<name>/` at a source's top level), but the
    skills CLI installs a *copy* and its lock records the source path, so after the tree move the
    installed skill is removed and installed again from the new source (`psm update --local`
    re-copies from the declared row; whether the lock follows is to check).
- `anfs mv <source/name> --to <source>` (plan 03) only runs the owning tool's `move`. Not built
  here; this plan does not depend on 03, and 03 depends on this one for that row.
- Rewriting references to a moved id (other spaces' `depends:`, stacks, ovm's vault links): listed
  by the check, edited by the person. No migration code (decision 6).
- A move back replays the history again: the origin then has the original commits and the
  returning ones. Accepted.
- Git LFS pointers and submodules inside a resource: none in any source; the replay's verification
  catches a difference, and the copy path carries what is committed.
- anfs-dev's other `stow` wording (hook comments, README, AGENTS.md's hooks section): prose, a
  rewording for a docs pass.
- ROADMAP's checklist line for this plan: the person's.

## Open questions (the plan proceeds on the default given)

1. **`anfs_move_check`'s `deps-fn` argument.** The decision's signature is `<kind> <name>
   <to-source>`; the dependents rule needs the tool's dependency reader. Default: an optional fourth
   argument, the same function the tool already passes to `anfs_resolve` (not a hook: it only reads
   a manifest). Without it the dependents and own-dependencies checks are skipped and said.
2. **Hooks on the move's commits.** Default: the replay runs `git am --no-verify` (the history was
   judged where it was made), the two new commits (the origin's removal, the fallback copy) run the
   repository's hooks like any commit. Alternative: `--no-verify` on all three.
3. **wsm's skills after a move.** Default: `_space_skills` with local paths re-copied for the moved
   spaces. If the skills CLI's lock keeps the old path anyway, the step is a remove then install of
   each local-path skill; the implementer checks which with a fixture and records it in the log.
4. **wsm events.** Default: `remove` of the old ids and `install` of the new ones. Alternative: none
   (a move is not an install), at the cost of ovm not noticing until its next `link apply`.
5. **A change inside the resource's own path** is refused even with `-f` (new). Default: refused;
   today's `ppm move -f` carried it along uncommitted. Say if `-f` should commit it into the
   destination as a last commit instead.
