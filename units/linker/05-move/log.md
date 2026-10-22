---
plan: linker/05-move
job: linker-05
date: 2026-10-22
---

# Log: linker/05-move

## The plan against current main

The plan was written before config/01–05 merged. Checked against the job's base (core `2789609`,
anfs-dev `c6fa934`):

- Libraries load through `$ANFS_LIB_DIR` (config/03): `bin/ppm` and `wsm` source
  `$ANFS_LIB_DIR/repo.sh` and `$ANFS_LIB_DIR/move.sh`. `lib/anfs` is one directory link, so
  `move.sh` arrives with the pull. `lib/ppm/move.sh` is a per-file link (core/ppm's), sourced by the
  `anfs_lib_files ppm` loop.
- The XDG rule (config/05): a move writes nothing under any config directory. Its temporary
  patches and scratch repository are in `$TMPDIR` (`mktemp -d`), removed on every exit; what it
  renames is ppm's STATE (tracker, claims, backup manifest) and anfs's (callbacks).
- `PPM_GROUP_ID` was already gone from core; anfs-dev's `move.sh` held the last of it and the last
  `stow` calls in anfs-dev's code.
- Nothing else in the plan was out of date.

## What was built

core:

- **`lib/anfs/move.sh`** (new, bash floor, not in the bootstrap set). The header says what a move
  is, that it is the one place the toolkit commits on the person's behalf, and why each am flag is
  there.
  - `anfs_move_check [-f] <dir> <id> <to> [deps-fn]`: `<state>\t<check>\t<detail>` lines (`ok`,
    `refused`, `forced`, `note`); 0, 1 (refused, `-f` would go on), 2 (never). Never: a bare name,
    an unknown or same source, no clone, not a git repository, a missing resource or marker, the
    path in the destination (there, tracked, or a file on the way), either repository mid-am,
    rebase, merge, cherry-pick or revert, on no branch or with no commit, uncommitted or untracked
    changes inside the resource's own path. Unless `-f`: other uncommitted changes (each repository
    named with up to three `status` lines), dependents, own dependencies, stacks. Notes: layers
    before/after (or which source a bare name finds, for first-match kinds), the branches, nothing
    pushed. Positions in the whole source order.
  - `anfs_move_tree [-n] <dir> <id> <to>`: format-patch, the trial in a scratch repository, the
    verification (tree of the path; every commit's author, author date, whole message), the am in
    the destination verified again, else the copy; then the origin's commit; ignored files follow;
    both checked clean against how they were. `ANFS_MOVE_METHOD`, `_COMMITS`, `_WHY`; the one line.
    `-n` prints `commits<TAB>N to replay from <from>`, the commits, and `replay<TAB>ok` or
    `replay<TAB>would copy: <why>`.
  - `anfs_move_cli <noun> <arg-kind>`.
- **ppm**: `lib/ppm/move.sh` (`cmd_move`, the `-n` steps, the renames), `bin/ppm` sources `repo.sh`
  and `move.sh` and calls `anfs_move_cli package package` after its declarations.
- **wsm** (`WSM_VERSION` 0.9.0): `cmd_move`, `_move_escapes`, `_move_clones`, `_move_worktrees`,
  `complete_source`, `anfs_move_cli space spec`, the help epilogue's paragraph.
- **Docs**: AGENTS.md "Moving a resource between sources (lib/anfs/move.sh)" (after "Claiming and
  Protecting Files"), "wsm commits only in `wsm move`", Lib Structure (both `move.sh`), the two
  `ppm move` mentions (Repository Layout, The Command Line), Testing; ppm README "Moving a Package
  to Another Source"; wsm README's command table.

anfs-dev: `packages/dev/home/.local/lib/ppm/move.sh` deleted; AGENTS.md (Layout; the section
"Moving a Package Between Repos" gone); `packages/dev/README.md`. `grep -rn 'ppm move\|move\.sh\|PPM_GROUP_ID' packages AGENTS.md`
in anfs-dev finds nothing.

## Commits (branch linker-05, not pushed)

core:
- `caa660e` lib/anfs/move.sh; `packages/anfs/tests/move.bats`
- `7b8eec3` ppm move (`lib/ppm/move.sh`, `bin/ppm`); `packages/ppm/tests/move.bats`; move.sh's `-f`
  help on one line
- `5d0d98f` wsm move, `complete_source`, epilogue, `WSM_VERSION` 0.9.0; `packages/wsm/tests/move.bats`
- `b459d97` docs: AGENTS.md, ppm and wsm READMEs
- `78aac47` tests: settings.bats allow-lists `ANFS_MOVE_MANIFEST`

anfs-dev:
- `77ff15d` dev: `ppm move` goes

Versions: the pre-commit hook bumped nothing: core's anfs (0.2.15), ppm (0.2.8), wsm (0.1.9) and
anfs-dev's dev (0.1.15) already carry their bump for the unpushed series. `WSM_VERSION` 0.8.0 →
0.9.0 by hand (a new command).

## Decisions made during the work

- **The copy is `read-tree` + `checkout-index` into a temporary index**, not `git archive | tar`:
  archive applies `export-ignore` and `export-subst`, which would drop or rewrite files. The
  origin's index and working tree are not touched; modes and links are kept. `git add -f`, so a
  file the origin tracks and the destination ignores is still copied.
- **Pinned against the person's git config**: format-patch also takes `--no-signoff --no-attach`
  (`format.signOff`, `format.attach`), am also `--no-message-id` (`am.messageid`). The scratch
  repository runs with its own identity, `core.hooksPath=/dev/null` and no signing; the real am
  uses the person's config. Anything else that changes a commit is caught by the verification.
- **Dependents and own dependencies: only what resolves now and would not after** is listed. A
  dependency that is already broken is not the move's.
- **`ANFS_MOVE_MANIFEST`** (new, a per-caller knob like `ANFS_RESOLVE_MARKER`): the file a
  dependency is declared in, for "the file that says so" (ppm sets `package.yml`; the marker wins,
  so wsm's lines name `space.yml`). Allow-listed in settings.bats as not a setting.
- **REVERT_HEAD and a repository with no commit** are refused too (2), beside the plan's list.
- **The check's lines are printed as they are** (tab-separated) in a real run and under `-n`;
  ppm's and wsm's own `-n` steps are aligned columns, as the plan's examples.
- **A refused move exits with the check's code** (1 or 2), not 64. wsm's own refusals are 1; the
  root space and a bare path are 64 (usage).
- **ppm's pre-plan also accepts `replace` for a live link** (an absolute link into the package,
  stow's or a person's): it is still the package's, and the relink makes it relative.
- **ppm's tracker list naming a file the package no longer has** (`anfs_link_plan` returns 2):
  refused with `ppm install <id>, then move`.
- **wsm**: the directory under the root moves when it exists, also when only an inner space is
  installed; emptied parents are removed up to the root (so `$ROOT/<from>` goes when empty, as the
  plan's test says). A relative `links:` src or `skills:` repo out of the moved directory is
  printed in the check's form (`refused` or `forced`, check `paths`); one that stays inside the
  moved directory (a sibling that moves too) is fine. After a real move wsm prints that ids are not
  rewritten (what named the old id, an ovm vault link or a `depends:`, now names nothing).
- `-f`'s help is one line (`cli.sh`'s option help does not span lines): "Despite other uncommitted
  changes, dependents it breaks, stacks naming it".
- The ppm README got a short section, the wsm README a table row (beyond the plan's "help comes
  from the declarations").
- anfs-dev's AGENTS.md and README point nowhere for `ppm move`, so the plan's closing grep is empty.

## Open question 3 (wsm's skills after a move): what was checked

`_space_skills <new id>` runs after the relink, so a local-path skill (`skills: [{repo: ./x}]`) is
resolved from the new source's `space.yml` directory and installed again with `force_local`
(`anfs_skill_sync_rows ... true`): the skills CLI is asked to `add` the new path. Whether the CLI
then rewrites its `skills-lock.json` entry's `source` (or keeps the old path) is the real CLI's
behaviour; wsm's suites stub npx, and the real CLI needs the network, so it was **not verified**.
If it keeps the old path, the step becomes a remove then install of each local-path skill (the
plan's alternative).

## Tests (all through `anfs work run linker-05 …`, `</dev/null`)

- While working: `packages/anfs/tests/move.bats` **17/17** (replay with two authors, a year
  apart, a commit touching another package, a rename, a binary, a symlink, an executable, a CRLF
  file, a `[wip]` subject: authors, dates, messages, committer date = author date, tree, other
  package not carried, one origin commit, clean, remotes' refs unchanged; merged parallel
  deletions → copy naming URL and SHA, no partial replay, no rebase-apply; `diff --git` in a
  message → copy; `---` in a message → caught by the verification, `-n` says so; the real am
  failing after a good trial (a `git` wrapper) → back at HEAD, no rebase-apply, copy; a refusing
  `pre-applypatch` does not stop a replay; dirty elsewhere 1 / `-f` 0 with commits limited to the
  path; a staged destination → copy; a change inside the path 2 with `-f`; the path taken three
  ways; not a source, not git, same source, missing, bare, detached, rebase; dependents A > B > C,
  qualified, own dependencies, `-f`, a destination with no `packages/`; layers; nested spaces;
  stacks as string and map; ignored files follow; a refusing origin hook puts both back; `-n`
  changes nothing; `anfs_move_cli`), `packages/ppm/tests/move.bats` **7/7** (an installed package
  with a per-file link, a directory link and a `{src}` link: relinked relative, `{src}` same inode,
  tracker identical under the new source, claims, callback and backup renamed, `plan_package`
  afterwards all `keep`, no hook ran; a layered package; a replaced link refused with nothing
  changed; the tree move failing puts links and tracker back; not installed; `core/ppm`; `-n`),
  `packages/wsm/tests/move.bats` **8/8** (directory moved whole with the clone's HEAD, status and
  inode, the user's file and a changed copy kept, links into `b`, `wsm ls` ok for the new ids and no
  old rows; a linked worktree repaired; events; a qualified dependent refused then `-f` and its
  install fails; a relative `links:` src refused then `-f`; root, bare path, place taken, a wrong
  link; not installed; `-n`), the whole wsm suite **111/111**, `settings.bats` 8/8.
- `ppm help move` and `ppm move x` (usage, 64) from the job's tree under a temp `HOME` (`env -i`).
- **Once, at the end**: `packages/anfs/tests/all` first **621 passed, 1 failed** (settings.bats:
  `ANFS_MOVE_MANIFEST` not allow-listed), fixed in `78aac47`, then **622 passed, 0 failed**, 46
  files (pmm and pcm off), 78 s.
- anfs-dev: `bats packages/dev/tests` **38/38**.

### On copies of real sources

`lgat`, `rws`, `user`, `stack`, `core` cloned with `--no-local` from `~/.local/share/anfs/sources`
into a temp `HOME` in the job's `TMPDIR` (in that priority order), the job's `lib/anfs` and
`lib/ppm` linked into it, the tools run from the job's tree with `env -i HOME=<copy>`; the temp
directory deleted after. The real sources were only read by `git clone`.

- `ppm move -n stack/rails user`: every check `ok`, **10 commits to replay** (`git log
  --no-merges -- packages/rails` in stack: 10), `replay ok`, `not installed: ppm install
  user/rails links it`.
- `ppm move stack/rails user`: `moved packages/rails: stack -> user, 10 commits replayed`; the tree
  of `packages/rails` equal in both; both copies clean; stack's last commit `packages/rails: moved to
  user (replay, 74c065a)`.
- `ppm move -n user/claude rws`: checks `ok`, `note layers claude: user, stack -> rws, stack`,
  14 commits, `would copy: git am failed: packages/claude/home/.claude/commands/rjr/cd.md: does
  not exist in index` (the plan's experiment's one failure, found by the trial).
- `wsm move -n lgat/tech rws` and `wsm move -n lgat/next rws`: checks `ok`, 1 commit each, `replay
  ok`, `not installed: wsm install rws/<path> links it`.

## Not run

- pmm's and pcm's suites (off since 2026-10-19; neither changed here). stack's suites: stack is not
  in the job and nothing of it changed.
- Nothing boots a VM or a box in this plan; nothing deferred.
- shellcheck (not installed on this host).
- The skills CLI for real (open question 3 above).
- Nothing was installed or moved on this host; `ANFS_WORK` was never set.

## Deviations from the plan

- The copy's mechanism (`read-tree`/`checkout-index`, not `archive | tar`) and the extra pinning
  flags: above.
- `ANFS_MOVE_MANIFEST`, REVERT_HEAD and the no-commit refusal: small additions, above.
- Dependents are listed only when the move breaks them (above).
- ppm and wsm README text beyond the plan.

## Open questions (for the person)

1. **The skills lock after a wsm move** (plan's open question 3) is not verified against the real
   skills CLI: check once on this Mac after merge with a space that has a local-path skill, or
   accept the default.
2. **Before the merge, `ANFS_WORK=linker-05 ppm move` runs anfs-dev's old `move.sh`**, not this
   one: the installed `~/.local/lib/ppm/move.sh` is anfs-dev's live link, and config/03's rule
   keeps an installed file whose job copy was deleted (anfs-dev's here) and skips a job file of the
   same name (core's). It then fails as it does today (stow). Try ppm's move only after the merge,
   or with an isolated `HOME` as above. `ANFS_WORK=linker-05 wsm move -n ...` is fine.
3. The check's lines are tab-separated on a terminal too (unaligned). Keep, or align them in the
   tools?
4. ROADMAP's checklist line for this plan is the person's.

## By hand, for the person after merge (in this order)

core and anfs-dev merge **together**: until anfs-dev's `move.sh` is gone, `~/.local/lib/ppm/move.sh`
is a live link into `anfs-dev/dev` (checked 2026-10-22: `-> …/sources/anfs-dev/packages/dev/home/.local/lib/ppm/move.sh`,
listed in `installed/anfs-dev/dev.yml`), and installing `core/ppm` would find another package's
link at the path its new `move.sh` takes.

1. Merge `linker-05` in core and anfs-dev at the same time, and have both clones under
   `~/.local/share/anfs/sources/` on the merged main before ppm runs (`anfs src update core
   anfs-dev` if the merge happened elsewhere); then `anfs work remove linker-05`. `lib/anfs/move.sh`
   is live at once (directory link); the old `move.sh` link now dangles, and ppm's library loop
   skips it.
2. `ppm install core/ppm` (the dangling link into the sources is the plan's `replace`), then check:

   ```bash
   readlink ~/.local/lib/ppm/move.sh     # ../../../.local/share/anfs/sources/core/packages/ppm/home/.local/lib/ppm/move.sh
   ```

3. `ppm install anfs-dev/dev` (its tracker still lists `move.sh`; the cleanup leaves the link, which
   points into core/ppm now, and the tracker drops it).
4. See it run, nothing changed:

   ```bash
   ppm move -n stack/rails user
   wsm move -n lgat/tech rws
   ppm help move; wsm help move
   ```

Nothing to install for wsm (its script is linked from the core clone) or for `lib/anfs`.
