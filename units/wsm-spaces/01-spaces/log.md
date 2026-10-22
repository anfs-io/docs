---
plan: wsm-spaces/01-spaces
job: wsm-01
date: 2026-10-21
---

# Log: wsm-spaces/01-spaces

## What was built

- **wsm, rewritten** (`core/packages/wsm/home/.local/bin/wsm`, `WSM_VERSION` 0.5.0). Levels,
  trackers, targets and `wsm status` are gone: none of `_is_level`, `_each_level`, `_unit_files`,
  `_unit_of`, `_want_workspace`, `_expand_spec`, `_installed_load`, `_tracker_*`, `INST_*`,
  `ISRC_*`, `IWS_*`, `OLD_FILES`, `_is_named`, `_has_spaces`, `_unlink_orphans`, `_src_target`,
  `_match_installed`'s three forms, `_resolve_query`'s three passes, `cmd_status`, `EX_STALE`.
  - **Spaces by `space.yml` at any depth** (`_spaces_load`): one `find` over every source's
    `spaces/` and one `sort` per run. Ids are `<source>[/<path>]`; `spaces/space.yml` is the root
    space `<source>`. Hidden directories are content; a `space.yml` in one makes it a space.
  - **Nearest-space ownership** (`_files_load`, `_space_of`); a file under no space is not linked
    and warned about once per run (`not in any space, not linked: <src>/<path> (and N more)`).
  - **Specs** (`_spec_parse`, `_spec_match`, `_specs_expand`): an id, or a prefix ending in `/`;
    the first segment is the source when it is the alias of a source with `spaces/`, else a path
    from the highest-priority source that has it. A spec naming nothing fails (1) before anything
    is touched, with a hint when it is a source alias that has no root space.
  - **`depends:`** by id or path, through `anfs_resolve` with `ANFS_NAME_NEST` and
    `ANFS_RESOLVE_MARKER=space.yml`; a bare path is the declaring source's or a lower one's.
  - **No tracker**: `_state_load` (one `anfs_link_check` for every source at once) gives
    `INSTALLED` (the `space.yml` link is `ok`) and every file's state (`FSTATE`). Light for path,
    cd, new, add and repo (only the `space.yml` links); full for list, install, remove, implode.
  - **The leftover walk** (`_leftovers_scan`): only directories of `<root>/<source>/` that mirror a
    directory of the source's `spaces/`, bash globs, one `readlink` for the dangling candidates,
    plus `<entry>/space.yml` for a real directory that is no mirror. A leftover counts against the
    nearest installed space above it; a dangling `space.yml` where the source has no space is a
    `not in the source` row.
  - **`wsm list`**: `SPACE STATUS DIR NOTE`; keys `space status dir note missing wrong in_the_way
    leftover`; `wsm list <id> --json` adds `files: [{file, status}]`.
  - **One root** (`WSM_SPACES_HOME`), no DIR (a path-looking spec is 64 with "is a directory, not a
    space"); several specs per install; a root inside a source is refused.
  - **`-f`** moves what is in the way aside to `$WSM_STATE_HOME/backup/<source>/<path>` (`.1`, `.2`),
    as ppm's `_backup_file`; obstacles on the way are never moved; nothing restores.
  - **remove**: files whose link is `ok`, the space's leftovers, `space.yml` last, then empty
    directories up to `<root>/<source>`; a `not in the source` space by walking its directory for
    dangling links into the source (not into clones or live spaces). **implode** does it for all,
    then deletes `$WSM_STATE_HOME`.
  - **Repositories**: ids `<space id>/<path>`; a path inside a child space or holding one is
    `invalid`; `repo list` adds `space`, `path`, `dir`; `repo install` splits at the longest prefix
    that is a space (`_repo_split`).
  - **path/cd**: installed id, then installed ids ending in `/QUERY`, then a source; no query, the
    deepest installed space; no exit 3.
  - **new** (`[SOURCE/]PATH`, relative to where you are; installs when the space around it is) and
    **add** (into the nearest space, which must be installed; refused under no space).
  - **Completion**: one `find … -name space.yml` (ids and prefixes), `-L` per space for the
    installed ones.
  - Events: ids without a trailing `/`.
- **`lib/anfs/link.sh`**: `anfs_link_check <from> <to> <files-file>...` (states `ok`, `missing`,
  `wrong`, `in the way`; several triplets in one call, one `readlink` for all; states also in
  `_ANFS_ST`, and `_ANFS_ABS` marks an `ok` that is an absolute link). Bash 3.2.
- **`lib/anfs/resolve.sh`**: `ANFS_NAME_PARTS` gone; `ANFS_NAME_NEST`, `ANFS_RESOLVE_MARKER`; the
  qualified name of a source alone is its alias (no trailing `/`). `anfs_sources` uses the new
  `gitsrc_dir_reply` (sources.sh), no subshell per source.
- **Readers**: `stacks.sh` (a spaces entry is a spec; a prefix `installed` when all its spaces are,
  `partial` when some; one `wsm install -y` for every space); `anfs` header and `_anfs_tool space
  wsm "Spaces"`; `callbacks.sh`, `anfs.conf`, `cli.sh` comments; anfs-ui `kinds.rb` (no Level,
  Note, `sourced: :first`, `invalid` is bad); anfs-dev `anfs-work` `_space_repo` reads `wsm repo
  list --json` alone (`path`'s last segment, `dir` with a `.git`); `work.bats`; the acme fixture's
  `depends: [main/org]`; stack's ovm help, comments, README.
- **Docs**: core AGENTS.md ("wsm: spaces" rewritten; the table, Stacks, Callbacks, Lists and
  --json, The Command Line, Key Files, Linking, Claiming, Lib Structure, Testing), core README,
  `packages/wsm/README.md` rewritten, wsm's `install.sh` messages and `wsm.sh` comment.

## Commits (branch wsm-01, not pushed)

core:
- `571e238` lib/anfs: anfs_link_check; resolve.sh names that nest (ANFS_NAME_NEST, ANFS_RESOLVE_MARKER)
- `e07cec5` wsm: a space is any directory with a space.yml, its id a path; no tracker, no levels, one root
- `619f807` anfs: a stack's spaces entry is a wsm spec; one wsm install for every space
- `a014902` anfs-ui: spaces without Level, with Note; sourced: :first; invalid is bad
- `3e0a12b` wsm: ls --json has note too, and files for any one id
- `be5bb9b` docs: AGENTS.md and README
- `6239908` lib/anfs, wsm: no fork per file (the performance pass, below)

anfs-dev: `32fc349` dev: anfs work finds a space's repository through wsm repo list's dir; work.bats; acme fixture

stack: `311976d` obsidian: ovm says spaces, not levels

The pre-commit hook bumped nothing ("already 0.1.8 since 0.1.7 was pushed" and the like: each
package already carries an unpushed bump). `WSM_VERSION` was set to 0.5.0 by hand.

## Decisions made during the work

- **`wsm list` SPEC narrows like install**: an id is that space only (one row), a prefix the spaces
  under it. `wsm list lgat/tech --json` therefore has exactly one row, which carries `files`.
- **`note` is in `--json` too**, not only the table: anfs-ui renders columns from JSON keys and
  needed it. Keys are added, none renamed (allowed by "Lists and --json").
- **A partial spaces entry of a stack is installed again** by `anfs install` (it is not counted
  "already installed"), so a re-run picks up a space new under `lgat/`. `stack_actual` still
  answers true for partial, so remove and `anfs list` treat it as there.
- **`wsm add FILE` of a copy where the source has the file** puts the copy in the source (git shows
  the diff) when named, and only warns under a directory. Without a tracker wsm cannot tell "my
  link was replaced by a copy" from "my file collides with a new source file"; both are `in the
  way` in `ls`.
- **A space that is not installed has no counts**: its status is null even if something of the
  user's is where its `space.yml` would go (install then reports it).
- **Leftovers under a directory the source no longer has are not seen** (only a `space.yml` one
  level below a mirror is). This follows the plan's walk literally; such links stay until the
  space is removed or the directory is cleaned by hand. Documented in AGENTS.md and the README.
- **Performance pass in `link.sh`** (shared with ppm and install.sh, behaviour unchanged, bash 3.2):
  `_anfs_link_canon` resolves with `cd -P` in the shell and back (`_anfs_link_physical`, a
  subshell only when the current directory cannot be returned to); `_anfs_relpath` sets a variable;
  `anfs_link_apply` makes a directory's links with one `mkdir`, one `rm`, one `ln -s` (a 2,000-file
  first install went from 21 s to 3.5 s). wsm skips the plan for a space whose links all check `ok`
  and relative.
- **ovm's unknown-space message** changed ("no wsm space 'x' (wsm ls)"), so `links.bats` changed one
  expected string. The plan said "no code change" for ovm; the old text named the levels.
- The plan's Verification grep over all of core `packages` also matches ppm's own `_tracker_*`
  and pcm's legacy `cmd_status`; over `packages/wsm` and `packages/anfs` it finds nothing.

## Tests (all through `anfs work run wsm-01 …`, `</dev/null`)

- While working: `link.bats` 27 (new: the check's states, other ways of writing a link, names,
  constant forks for 50 files, several triplets with one readlink, batched apply, bash 3.2),
  `resolve.bats` 13 (nest, marker, nested resolve, unchanged non-nest), `stacks.bats` 26 (a prefix
  absent/partial/installed, one wsm install, `anfs src install a b` to one wsm install), wsm's
  `install.bats` 29, `manage.bats` 31, `events.bats` 9 (all per the plan's Test Spec, including the
  5,000-file clone never walked, with a `find` wrapper).
- **Once, at the end**: `packages/anfs/tests/all`: **399 passed, 0 failed**, 27 files (pmm and pcm
  off), 41 s.
- anfs-ui: `bundle exec rspec` 64 examples, 0 failures; `rubocop` 29 files, no offenses.
- anfs-dev: `bats packages/dev/tests/work.bats` 22/22 (new: a repository of a nested space).
- stack: `ANFS_TEST_LIB=<job core lib> bats packages/obsidian/tests` 53/53.

### This Mac, copied (temp HOME, read-only on the real side)

A temp `HOME` under the job's `TMPDIR` with copies of the 7 sources' `spaces/` and of the 35 links
under `~/spaces` (texts kept, no clones), every `*_HOME` and `WSM_SPACES_HOME` inside it:

```
anfs-dev       ok      lgat/orig      -       cnfs/orig      ok      user/evals   ok
anfs-admin     ok      lgat/tech      -       cnfs/partitia  -       cnfs/dev     not in the source
lgat/next      ok      roteoh         ok      rws            ok
```

All ten expected ids are listed. `cnfs/dev` shows as `not in the source` (its `space.yml` link is
one level below the `cnfs` mirror); `lgat/spikes/{tech,orig}` and `user/dev` are not seen (deeper).

### Performance (best of 3, this Mac)

| Case | Time |
| --- | --- |
| `wsm ls`, this Mac's copy (10 spaces, 35 links) | 141 ms (the old wsm on the same copy: 410-480 ms; the unit's figure was 240 ms) |
| `wsm path anfs-dev` (light check) | 106 ms |
| `wsm __complete install ""` | 66 ms |
| the same with 60,000 files added under the root (a clone in `lgat/next`, an unrelated `junk/` tree) | `ls` 147 ms, `path` 109 ms: not walked |
| a source of 20 spaces × 100 files: first `install -y many/` / re-install | 3.6 s / 2.5 s |
| … `wsm ls` / `wsm ls many/team/s7 --json` / `wsm path s7` | 0.90 s / 0.84 s / 0.36 s |

What is in the root costs nothing; the cost is the source's files, about 0.2 ms each for a light
command and 0.45 ms for `ls`, in bash (the forks per run are constant: one `find`, a few `sort`s,
one `readlink`).

## Not run

- Deferred to the end of the roadmap (boot a box or a VM): `roundtrip debian`, `roundtrip fedora`,
  `roundtrip-macos` (the wsm checks; the acme fixture's `depends: [main/org]`), the harness, pmm's
  e2e.
- pmm's and pcm's suites are off (ROADMAP, 2026-10-19); this plan does not touch them.
- Nothing was installed on this host; the live `~/spaces` and `~/.local/state/wsm` were only read
  (to copy them into the temp HOME).

## Open questions

- **Leftovers below a vanished directory** (decision "Leftovers … not seen" above): fine as the
  documented limit, or should a real directory that is no mirror get its own entries listed (one
  more level, at the cost of listing a clone's top directory)?
- **`wsm add` of a colliding file** goes into the source when named (decision above): acceptable?
- **`link.sh` changed for speed** in ways ppm and install.sh also run (`cd` in the shell, one `ln`
  per directory). The ppm suites pass; worth a look before merge since linker plan 01 is the
  owner of that file.

## By hand, for the person after merge

1. `ppm install core/anfs core/wsm` (links the new `link.sh` functions, `wsm` 0.5.0).
2. `rm -rf ~/.local/state/wsm` (the old trackers and stow directory; the new wsm ignores them).
3. Remove the stale directories the old levels left, after checking each holds only wsm's dangling
   `space.yml` link and clones you no longer want: `~/spaces/lgat/spikes/{tech,orig}`,
   `~/spaces/cnfs/dev`, `~/spaces/user/dev`. (`wsm ls` shows `cnfs/dev` as `not in the source`, so
   `wsm remove -y cnfs/dev` can take its links; the other three are deeper than the walk.)
4. `wsm ls`: every installed space `ok`.
