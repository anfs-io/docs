---
---

# Plan 01 — Spaces by `space.yml`, path ids, no tracker, one root

Written 2026-10-21 against main, which holds `linker-01` and `ovm-01` (both merged that day)

## Context — read these files first

The job is `anfs work new <job> core anfs-dev stack`: core for wsm, `link.sh`, `resolve.sh` and
the readers; anfs-dev for `anfs work`, its tests and the acme fixture; stack for ovm's words.
Paths below are core's unless they name another repo.

- `units/wsm-spaces.md` — decisions 1–8 and those of 2026-10-21 (review), "Decided here",
  "Performance", "linker", "ovm"
- `packages/wsm/home/.local/bin/wsm` — all of it. What goes: `_is_level`, `_each_level`,
  `_unit_files`' level argument, `_unit_of`, `_in_repository`'s three levels, `_want_workspace`,
  `_expand_spec`'s `SPEC_LEVEL`/`SPEC_WS`, `_install_unit`'s ws/space arguments, the trackers
  (`WSM_INSTALLED`, `_installed_load`, `_tracker_*`, `INST_*`, `ISRC_*`, `IWS_*`, `OLD_FILES`),
  `_is_named`, `_has_spaces`, `_unlink_orphans`, `_src_target`, the DIR argument,
  `_match_installed`, `_resolve_query`'s three passes, `cmd_status`, `EX_STALE`, the epilogue
- `packages/wsm/{README.md,install.sh}`, `home/.config/sh/wsm.sh`, `tests/{helper.bash,
  install.bats,manage.bats,events.bats}`
- `packages/anfs/home/.local/lib/anfs/link.sh` — `anfs_link_plan`'s classes, `_anfs_link_classify`
  (the `keep` fast path: one `readlink`, a string compare), `_anfs_link_resolve`, `anfs_relpath`;
  `tests/link.bats`
- `packages/anfs/home/.local/lib/anfs/resolve.sh` — `_anfs_split`, `ANFS_NAME_PARTS`,
  `anfs_find`, `anfs_resolve`, `ANFS_RESOLVE_FIRST`; `tests/resolve.bats`
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `_backup_file` (the model for `-f`)
- `packages/anfs/home/.local/lib/anfs/stacks.sh` — `stack_have_load` (spaces), `stack_actual`,
  `_stack_install_kind` ("wsm installs one level per run"); `tests/stacks.bats` (the wsm stub)
- `packages/anfs/home/.local/lib/anfs/callbacks.sh` — the header ("wsm emits ... the levels")
- `packages/anfs/home/.local/bin/anfs` — header comments (`wsm install lgat/main/tech`),
  `_anfs_tool space wsm "Workspaces of spaces"`, `cmd_src_install`
- `packages/anfs/home/.config/anfs/anfs.conf` (`WSM_SPACES_HOME`);
  `packages/anfs/home/.config/sh/anfs.sh` (`_anfs_cd`, exit 3)
- `containers/ui/lib/anfs/kinds.rb` (`spaces`, `repos`, `TONES`), `spec/anfs/kinds_spec.rb`
- anfs-dev `packages/dev/home/.local/bin/anfs-work` (`_space_repo`); `tests/work.bats`
  (`docs_space`, which passes DIR); `tests/fixtures/acme/spaces/main/{org,tech}/space.yml`;
  `tests/roundtrip`, `tests/roundtrip-macos` (the wsm checks); `README.md` (the `wsm ls` line)
- stack `packages/obsidian/home/.local/bin/ovm` (`_wsm_load`, the help and comments that say
  "level"), `README.md`, `tests/helper.bash` (the `wsm` stub)
- `AGENTS.md` (core's) — the table at the top, "Stacks", "wsm: workspaces", "Callbacks Between
  Tools", "Lists and --json", "The Command Line" (completion timing), "Key Files", "Linking",
  "Claiming and Protecting Files", "Lib Structure", "Testing"; `README.md`

## Overview

wsm stops knowing about levels and keeps no record. A space is a directory with a `space.yml`;
its id is `<source>[/<path>]`; its files are those whose nearest enclosing space it is. A
source's `spaces/` says what exists; the links under `<root>/<source>/` say what is installed.
Specs are ids or prefixes, `depends:` names ids, a repository is `<space id>/<path>`. `wsm list`
reports each space `ok`, `partial` (with counts) or `not in the source`, and per file for one
space. The target is always `WSM_SPACES_HOME`, so `wsm install` takes several specs. `wsm status`
goes. The old state in `~/.local/state/wsm` is ignored, not migrated (the person deletes it).

## Implementation

### 1. Finding spaces

- **The model, once per run per source.** One `find <srcdir>/spaces/ \( -type f -o -type l \)
  -print` (the trailing `/` follows a local-path source's link). From it, in bash: the spaces
  (every directory holding a `space.yml`; the root `space.yml` is the space `<source>`), the
  source's directories (for the leftover walk), and for every other file its owner: the longest
  space path that is a prefix of the file's directory. `.DS_Store` is skipped. Files with no owner
  are warned about once per run (`not in any space, not linked: <source>/<path> (and N more)`).
- Held in associative arrays (the bash floor is 5.2): `SPACE_DIR[id]` (the directory in the
  source), `SPACE_FILES[id]` (newline-separated, relative to the space), `SPACE_SRC[id]`,
  `SRC_DIRS[source]`. `_spaces_load [source...]` fills them; every command that reads the sources
  calls it once.
- Hidden directories are content like any other; a `space.yml` in one makes it a space
  (decision 1: no constraint).
- `_space_of <source> <path>`: the nearest enclosing space of a path below `spaces/`. It replaces
  `_unit_of`.

### 2. Ids and specs

- An **id** is `<source>` or `<source>/<path>`. A **spec** is an id, or a prefix ending in `/`
  (`lgat/`, `lgat/tech/`): every space whose id equals the prefix without its `/` or starts with
  it. `_specs_expand <spec>...` turns specs into ids, in the sources' order then path order, and
  fails (1, before anything is touched) on a spec that matches nothing.
- **Qualification** (unit, "Decided here"): the first segment is the source when it is the alias
  of a source with `spaces/`; otherwise the spec is a path, resolved in the highest-priority source
  that has it (a prefix: that has a space at or under it).
- **resolve.sh**: `ANFS_NAME_PARTS` goes. `_anfs_split` takes a mode for kinds whose names nest
  to any depth, `ANFS_NAME_NEST=true`: the first segment is the source when `anfs_is_source` says
  so, and the rest (possibly empty) is the name. `anfs_find` takes `ANFS_RESOLVE_MARKER` (a file
  the resource directory must hold; wsm: `space.yml`), and an empty name with a source is `<dir>/`
  itself. ppm, pcm, psm and pmm set neither and are unchanged.
- **`depends:`** entries are specs without a trailing `/`: an id (another source by its alias,
  accepted 2026-10-21), or a bare path, resolved from the declaring source down (`anfs_resolve`'s
  `min_index`, unchanged). `_space_deps` no longer prefixes a workspace. Install orders with
  `anfs_resolve spaces _space_deps <ids>` and `ANFS_RESOLVE_FIRST=true`, as today.
- Installing a space installs its dependencies, not the spaces above it (decision 3).

### 3. `anfs_link_check` in `link.sh`

```
anfs_link_check <from> <to> <files-file>   print "<state>\t<file>" for each file; read-only
```

- States: `ok` (a link whose text is what `anfs_link_apply` writes, or that resolves to the
  file), `missing` (nothing there), `wrong` (a link to anything else), `in the way` (a file, a
  directory, or a directory on the way that is not real).
- No fork per file: `-L`/`-e`/`-d` tests in bash, the expected text built by string operations
  (`../` per directory of the file, then `anfs_relpath <to> <from>` once, then the file), and the
  text of every link read with **one** `readlink -- <link>...` (BSD and GNU take many operands;
  a name with a newline or a tab is refused as in the plan). Only a link whose text differs is
  resolved (`_anfs_link_resolve`).
- Bash 3.2 like the rest of the file.

### 4. The state, computed

- **The root** is `_spaces_home` (`WSM_SPACES_HOME`, `~` and `$HOME` expanded), always; a space's
  directory is `<root>/<source>/<path>`. The DIR argument, `_src_target` and "a source lives in
  one target" go. `_target_check` stays for the root: refused when it is inside a source checkout.
- **Installed**: `anfs_link_check` of the space's `space.yml` says `ok`. The other files of an
  installed space are checked with one `anfs_link_check` call per space (or per source, the files
  prefixed).
- **Leftovers** (`_leftovers <source>`): for every directory `D` of the source's `spaces/` whose
  mirror `<root>/<source>/D` exists, the mirror's entries are listed with bash globs (`* .[!.]*
  ..?*`, no `find`). Its links go into one `readlink` batch; a **leftover** is a dangling link
  whose target is under `<srcdir>/spaces/` and is not a current file of the source. For an entry
  that is a real directory and not itself a mirror, only `<entry>/space.yml` is looked at: the one
  level that notices a vanished space. Nothing else is walked, so a clone inside a space costs one
  directory entry, and a space that vanished more than one level below the nearest mirrored
  directory is not seen (the limit is kept on purpose, 2026-10-21; AGENTS.md says so).
- A leftover counts against the nearest installed space enclosing it; a dangling `space.yml`
  leftover makes a `not in the source` row with id `<source>/<path>`.
- Nothing is stored. `$WSM_STATE_HOME` holds only `backup/` (section 6). wsm does not read,
  delete or migrate an old `$WSM_STATE_HOME/installed/` (decided 2026-10-21: no migration code).
- `EX_STALE` (3) goes: a space whose directory is gone is not installed. `_anfs_cd` keeps
  accepting 3 for other tools; `wsm.sh`'s comment drops it.

### 5. Install and remove

- **`wsm install [SPEC...] [-n] [-f] [-y]`**: specs expanded, ordered with dependencies, then per
  space: its files linked from `SPACE_DIR[id]` into its directory with `anfs_link_plan`/
  `anfs_link_apply` (`_link_unit`, without the level argument and without `OLD_FILES`: a link into
  the space's source directory, moved, renamed or dangling, is already the plan's `replace`, and
  `other` and `conflict` are the user's or another tool's, decision 7), its leftovers removed,
  then its repositories cloned (section 8). Every space installed clones its own repositories.
  With no SPEC: the space at `.`, which must hold a `space.yml` (a source's `spaces/` or the tree).
- `wsm install a/ b/` installs both; `anfs src install a b` (which passes every spec to one run)
  works.
- **`wsm remove SPEC...`** matches specs against installed spaces and unlinks, for each: its files
  that still point at their source file, its leftovers, then `space.yml` last (an interrupted
  remove leaves the space `partial`, not invisible), then the directories that leaves empty, up to
  `<root>/<source>`. A child space's files are not the parent's and stay. Clones and the user's
  files keep a directory alive.
- **`wsm implode`** does that for every installed space of every source, then deletes
  `$WSM_STATE_HOME` (backups included; the confirmation lists it).
- Events: `install`/`remove` with the ids of spaces that became installed or were removed,
  `<source>` for the root space (no trailing `/`). `callbacks.sh`'s header says spaces.

### 6. `-f`: move aside, as ppm

`wsm install -f`: what the plan reports in the way (a file, a directory, a link elsewhere) is
moved to `$WSM_STATE_HOME/backup/<source>/<path>/<file>` (`.1`, `.2` when taken, as
`_backup_file`), said in one line, then linked. `wsm remove` does not restore backups: that needs
a record, and there is none (decided 2026-10-21). Without `-f` the space fails before anything
is linked, every file named (today's behaviour).

### 7. The other commands

- **`wsm list [SPEC]`**: one row per space the sources define under SPEC, then each `not in the
  source` row the walk found. Headers `SPACE STATUS DIR NOTE`; keys `space status dir missing
  wrong in_the_way leftover` (`level` goes; the counts are `--json` only; NOTE is the table's
  summary, `2 missing, 1 in the way`). STATUS: null (not installed, `dir` null), `ok`, `partial`
  (any count above 0), `not in the source`. **`wsm list <id> --json`** (an id, not a prefix) adds
  `files: [{file, status}]`: `anfs_link_check`'s states plus `leftover`, relative to the space.
  One check per source; `log.md` records `wsm ls`'s time.
- **`wsm path` / `wsm cd [-s] [QUERY]`**: an installed id; failing that, the installed ids that
  end in `/<QUERY>` (`web`, `tech/web`); failing that, `<source>/` or `<source>` names
  `<root>/<source>`. Several: the candidates and exit 2. No QUERY: the deepest installed space
  directory containing `.`. `-s`: `<srcdir>/spaces/<path>` for a qualified query, nothing
  installed needed. Not installed: 1. firstmate's `wsm path <name>` keeps working.
- `path`, `cd`, `new` and `add` read "installed" from the `space.yml` links only (one `-L` and one
  `readlink` batch over all spaces), never the full check.
- **`wsm new SPEC`**: a qualified id works anywhere. A bare path is relative to where you are:
  below a source's `spaces/` or inside the tree it is created under that position (`wsm new web`
  in `~/spaces/lgat/tech` is `lgat/tech/web`); in a checkout outside `spaces/`, under `spaces/`. It
  writes `space.yml` (the template: `depends:` and `repositories:`, path-id wording) and installs
  the new space when its nearest enclosing space is installed. A space made inside a space takes
  the files below it; their links are already right.
- **`wsm add FILE...`**: the file's space is `_space_of` its path below `<root>/<source>/`, and
  must be installed. It moves to `<srcdir>/spaces/<same path>` and is linked back; nothing is
  recorded. Refusals as today (clones, repository paths, `.gitignore`d, `.DS_Store`), plus a path
  under no space.
- **`wsm status`**, its help, README section and tests are removed.
- **Completion**: `complete_spec` lists every id and every prefix with spaces below it from one
  `find "$ANFS_SOURCES_HOME"/*/spaces/ -name space.yml` (no yq, no git); `complete_installed`
  keeps the ids whose `<root>/<source>/<path>/space.yml` is a link (`-L`, no `readlink`);
  `complete_query` adds each installed id's last segment.
- The help epilogue and the header comment describe spaces, ids, prefixes and the one root.

### 8. Repositories

- `_repo_entries` is unchanged (url, path, ref). A **repository id** is `<space id>/<path>`.
- **Validation** at install, `repo install` and `repo list`: a path is refused (warned, counted
  as failed, row status `invalid`) when it lies inside a child space of its space or contains
  one, besides today's refusals (empty, absolute, `..`).
- **An id splits** at the longest prefix that is a space id, the rest being the path
  (`_repo_split`). `wsm repo install <id>` uses it (`_repo_find_level` and `REPO_WANT` go).
- **`wsm repo list [SPEC]`** keeps `repository status url ahead` and adds `space`, `path` and
  `dir` (the clone's absolute path; null when its space is not installed), so readers never split
  an id. `wsm repo install [SPEC]` clones for installed spaces only, as today.
- `_in_repository` checks the file's own space's repositories only.

### 9. Readers

- **stacks.sh**: a stack's `spaces:` entry is a spec. `stack_have_load` keeps every row of
  `wsm list --json`: `ok` is `installed`, `partial` is `partial`, null is absent. `stack_actual`
  for spaces: an id matches its row; a prefix (`lgat/`) is `installed` when it has rows and all
  are installed, `partial` when some are, absent when none. `_stack_install_kind` passes every
  space to one `wsm install -y`, and its comment goes. `stacks.bats`' stub emits path ids and
  `status`.
- **`anfs`**: `cmd_src_install` unchanged (it now works for several sources); the header's
  examples (`wsm install lgat/tech`); `_anfs_tool space wsm "Spaces"`.
- **anfs-ui** `kinds.rb`: `spaces` columns `Space`, `Directory`, `Note` (no `Level`); `unset` is
  `not installed`; `ok`, `partial`, `not in the source` are toned already; `action` stays; the
  source is the id's first segment even with no `/` (the root space `lgat`): a `sourced: :first`
  option used by `spaces` and `repos`. `kinds_spec.rb` follows.
- **anfs-dev `anfs-work`**: `_space_repo` reads `wsm repo list --json` alone: rows whose `path`'s
  last segment is the name and whose `dir` has a `.git`. No `wsm list`, no longest-prefix guess.
  Its comment and the dev README's line follow. `work.bats`' `docs_space`:
  `WSM_SPACES_HOME="$T/spaces" wsm install -y anfs-dev` (no DIR).
- **anfs-dev fixtures and round trips**: `acme/spaces/main/tech/space.yml` says `depends:
  [main/org]` (a bare name is no longer a sibling). The round trips' checks keep their commands
  (`wsm path acme/main/org`, `wsm path tech`, `wsm path lgat/`); their comments say spaces.
- **stack's ovm**: no code change (`_wsm_load` trims a trailing `/` and keys by id, and reads
  `.dir`). The help, `--to`'s description, the `ovm.yml` comment and the README say "a wsm space
  (wsm ls)" instead of "level: source, source/ws or source/ws/space".

### 10. Documentation

Core `AGENTS.md`: the table at the top (`spaces/`: "spaces (`spaces/**/space.yml`)"); "Stacks"
(a spaces entry is a spec; prefixes; one `wsm install`); "wsm: workspaces" becomes "wsm: spaces",
rewritten (spaces by file, nearest-space ownership, ids and prefixes, `depends:`, repository ids
and their validation, no tracker: the check, the walk and its one-level limit, the one root, `-f`
without restore, `list`'s STATUS and detail, remove and implode); "Callbacks Between Tools" (space
ids); "Lists and --json" (`wsm list`, `wsm repo list`: keys above); "The Command Line" (completion
reads `find` once); "Key Files" (no `$WSM_STATE_HOME/installed`); "Linking" (`anfs_link_check`);
"Claiming and Protecting Files" (no `spaces/<ws>/<space>/`); "Lib Structure" (`resolve.sh`:
`ANFS_NAME_NEST`, `ANFS_RESOLVE_MARKER`); "Testing" (the wsm line, no status). `README.md`
(core): the `wsm` row and example. `packages/wsm/README.md`: rewritten (commands, SPEC, QUERY,
`space.yml`, how it is kept), no workspace, no status, no tracker. `anfs.conf`'s comment on
`WSM_SPACES_HOME`: the only target. `packages/wsm/install.sh`'s messages (no trackers).

## Test Spec

wsm's suites are rewritten around spaces; `helper.bash` gets `space_def <source> <path>` and
`space_file <source> <path>` for any depth, sets `WSM_SPACES_HOME` in the temp dir, and loses
`TRACKERS`/`installed`.

- Discovery: `spaces/space.yml`, `spaces/a/b/c/space.yml`, `spaces/a/x.md`: `wsm ls` lists `acme`
  and `acme/a/b/c` (not `acme/a`, not `acme/a/b`); installing `acme` links `a/x.md`, not
  `a/b/c/*`. Without the root `space.yml`, `a/x.md` is not linked and the warning names it once.
- Ownership: files in `c/` and `c/d/` (no `space.yml` in `d`) belong to `acme/a/b/c`; adding
  `c/d/space.yml` moves `d/`'s files to `acme/a/b/c/d`.
- Specs: `acme/` installs every space; `acme/a/` installs `acme/a/b/c` and not `acme`; an id
  installs that space only; a bare path comes from the highest-priority source; an alias equal to
  another source's top-level directory names the source; two specs in one run; a spec matching
  nothing fails before anything is created.
- `depends:` with an id of another source, a bare path in the same source, a lower-priority
  source; a cycle and an unknown id fail before anything is created.
- State: nothing under `$WSM_STATE_HOME` after an install; `ls` STATUS `ok`; a deleted source file
  gives `partial`, NOTE `1 leftover`, and a re-install `ok`; an added one `1 missing`; a user file
  in a link's place `1 in the way`; a link elsewhere `1 wrong`; `ls <id> --json` has `files`; a
  space whose `space.yml` left the source, one level below a mirror, is `not in the source` and
  `remove` takes it; one two levels below is not seen (the documented limit).
- Decision 7: a user file, a user link and an ovm-style absolute link in a space are never
  listed, changed or removed by `ls`, `install`, `remove` or `implode`.
- Speed: a space holding a clone of 5000 files: `ls` does not descend into it (a `find` wrapper on
  PATH logs its arguments; nothing under the clone appears).
- The root: `WSM_SPACES_HOME` with `~`; DIR is a usage error (64); a root inside a source is
  refused.
- `-f` moves a file, a directory and a foreign link aside and links; `remove` leaves the backups
  where they are; without `-f` every file in the way is named and nothing is linked; a link into
  the space's source directory at a renamed path is replaced.
- Repositories: every installed space clones its own; a path inside a child space and one
  containing a child space are `invalid`; `repo list --json` has `space`, `path`, `dir`; `repo
  install acme/a/b/c/web` splits at the longest space prefix even when `acme/a` is a space.
- remove: a prefix removes every installed space under it; removing a child leaves its parent's
  links; `space.yml` goes last; empty directories go up to `<root>/<source>`, a clone keeps its
  directory.
- path/cd: an id, a last segment, two last segments, ambiguous (2), none or not installed (1), no
  exit 3, `-s` with nothing installed, no query inside a nested space gives the deepest.
- new: qualified anywhere; bare inside the tree and inside a checkout's `spaces/`; inside an
  installed space it installs; an existing one is 73. add: into the nearest space; refused under
  no space.
- events: ids without a trailing `/` (`acme` for the root space); nothing on `-n`.
- `wsm status` is an unknown command (64). An old `$WSM_STATE_HOME/installed/` changes nothing.
- `link.bats`: `anfs_link_check`'s states; what `anfs_link_apply` wrote is `ok`; an absolute link
  to the file is `ok`; names with spaces and `+`; a constant number of forks for 50 files (a
  `readlink` wrapper on PATH counts them).
- `resolve.bats`: `ANFS_NAME_NEST`, `ANFS_RESOLVE_MARKER`; ppm-style names unchanged.
- `stacks.bats`: a spaces prefix installed, partial, absent; one `wsm install` for several.
- The anfs suite: `anfs src install a b` passes both to one `wsm install` (stub).
- anfs-ui: `kinds_spec.rb` (no Level column; Note; source of `anfs-dev`).
- anfs-dev `work.bats`: a space repository found through `repo list`'s `dir`, in a nested space,
  installed with `WSM_SPACES_HOME`.
- stack: `bats packages/obsidian/tests` unchanged and passing.

pmm's and pcm's suites are off (ROADMAP, 2026-10-19); this plan does not touch them.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that
boots a VM or a box runs, and nothing is installed on this host.

- [ ] While working, the file you touch: `bats packages/anfs/tests/link.bats`,
      `bats packages/anfs/tests/{resolve,stacks}.bats`, `bats packages/wsm/tests/<file>`
- [ ] `bats packages/wsm/tests`, `bats packages/anfs/tests`
- [ ] anfs-ui: `anfs work run <job> core -- sh -c 'cd containers/ui && bundle exec rspec && bundle
      exec rubocop'`
- [ ] anfs-dev: `anfs work run <job> anfs-dev -- bats packages/dev/tests/work.bats`
- [ ] stack: `anfs work run <job> stack -- env ANFS_TEST_LIB=<job's core>/packages/anfs/home/.local/lib/anfs
      bats packages/obsidian/tests` (`anfs work path <job> core` gives the path)
- [ ] `grep -rn '_is_level\|_each_level\|_is_named\|_unlink_orphans\|ANFS_NAME_PARTS\|WSM_INSTALLED\|_tracker_\|EX_STALE\|cmd_status' packages`
      in core: nothing
- [ ] With `HOME` and the XDG dirs in a temp dir, `ANFS_SOURCES_HOME` at copies of this Mac's 7
      sources and `WSM_SPACES_HOME` at a copy of the links under `~/spaces` (not the clones):
      `wsm ls` lists `lgat/next`, `lgat/tech`, `lgat/orig`, `cnfs/partitia`, `cnfs/orig`,
      `user/evals` and the four root spaces (`anfs-admin`, `anfs-dev`, `roteoh`, `rws`); record
      its time in `log.md` against the 56 ms check of the unit's "Performance"
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`), by hand: `ppm install core/anfs core/wsm`;
  `rm -rf ~/.local/state/wsm` (the old trackers and stow directory, ignored by the new wsm);
  remove the stale directories the old levels left, after checking each holds only wsm's dangling
  `space.yml` link and clones you no longer want: `~/spaces/lgat/spikes/{tech,orig}`,
  `~/spaces/cnfs/dev`, `~/spaces/user/dev` (`wsm ls` does not see them: they are deeper than the
  walk's one level). Then `wsm ls`: every installed space `ok`.
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian`, `roundtrip-macos`
  (the wsm checks, the acme fixture's `depends: [main/org]`).
