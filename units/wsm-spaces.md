---
objective: A space is any directory of a source's spaces/ that has a space.yml, at any depth, named by its path; what is installed is read off the tree, not a tracker; a space declares its repositories, skills and links
status: plans 01–03 written 2026-10-21; decisions taken with the person 2026-10-21, revised after review the same day
---

wsm cuts a source's `spaces/` into levels by depth (`spaces/`, `spaces/<ws>/`,
`spaces/<ws>/<space>/`), records each installed level in a tracker, and allows two levels and no
more. The levels are an artificial structure the user has to fit into, and the trackers are a
second record of what the links already say. This unit removes both: a space is a directory with
a `space.yml`, an id is a path, and what is installed is computed from the sources and the links.
It then gives a space the two things it lacks: its own agent skills and links out of the tree.

Written against main plus two branches that were unmerged when this started and were merged the
same day (core `b45c368`): job `linker-01` (wsm and ppm link with `lib/anfs/link.sh`, no stow;
`units/linker/01-link-library/`) and job `ovm-01` (pvm left core as stack's `ovm`; vaults are no
resource; `units/ovm.md`). Every plan names the code as it is with both in.

Depends on: `linker/01-link-library` and `ovm/01-pvm-to-ovm`, both merged. See "linker" below
for plans 03 and 04 of that unit.

## What wsm is today (main + linker-01)

`core/packages/wsm`: one script (`home/.local/bin/wsm`, 1677 lines), `wsm.sh`/`wsm.zsh`, an
`install.sh` with messages only, and three bats files (`install`, `manage`, `events`: 57 tests).

- **Levels by depth.** `_is_level` (any non-hidden directory at that depth), `_each_level`,
  `_unit_files <dir> <level>` (a workspace's or source's files are what is not inside a directory
  below it), `_unit_of` (the level of a path, by depth), `ANFS_NAME_PARTS=2` in `resolve.sh`
  (`ws/space` is a name, `src/ws/space` qualified).
- **Trackers are the state.** `$WSM_STATE_HOME/installed/<src>.yml`, `<src>/<ws>.yml`,
  `<src>/<ws>/<space>.yml` (source, workspace, space, target, dir, files, `named`).
  `_installed_load` reads them into ten parallel arrays; `_unlink_orphans` removes a workspace or
  source level with its last space unless `named: true`.
- **Targets.** `wsm install SPEC [DIR]`; a source lives in one target, remembered in its tracker.
  `anfs src install a b` runs `wsm install -y a/ b/`, and `b/` becomes DIR (a bug).
- **`wsm status`** walks the whole installed tree of every source (`_walk_local`), listing
  `local` (user files), `changed`, `missing`, `broken`, `unlinked`.
- **Repositories.** `space.yml`'s `repositories:` at any level; `_repositories` clones with its own
  `git clone` and `_repo_name` (URL to directory), while `sources.sh` has another clone
  (`gitsrc_update`) and another name parser (`basename "$url" .git`). Ids are
  `<level>/<path>`; anfs-dev's `anfs work` guesses the level as the longest installed id the
  repository id starts with.
- **Readers of its lists** (`--json`, never columns): `stacks.sh` (`wsm list`: `select(.dir !=
  null) | .space`; `_stack_install_kind` runs one `wsm install` per spec because of DIR), anfs-ui
  `kinds.rb` (`spaces`: `space`, `level`, `dir`; `repos`: `repository`, `status`, `url`), anfs-dev
  `anfs-work` (`_space_repo`: `wsm list` and `wsm repo list`), stack's `ovm` (`_wsm_load`:
  `.space` with `/` trimmed, `.dir`), stack's `firstmate` (`wsm path <name>`).

On this Mac (2026-10-21): 7 sources with `spaces/`, 10 files in them, 16 trackers, four of them
for levels the sources no longer have (`lgat/spikes/{tech,orig}`, `cnfs/dev`, `user/dev`).
`~/spaces/lgat/spikes/tech/space.yml` is a dangling link into `lgat/spaces/spikes/tech/`.
`~/spaces/anfs-dev/sources` is an absolute link made by hand, exactly what `links:` will declare.

## Decisions (taken with the person, 2026-10-21)

1. **A space is any directory under a source's `spaces/` that has a `space.yml`, at any depth.**
   `spaces/a/b/c` with only `c/space.yml` is fine; `a` and `b` are directories. No levels:
   LEVEL, source/ws/space, `_is_level`, `_each_level`, `named`, `_unlink_orphans` go. Ids are
   paths: `lgat`, `lgat/tech/web`. A source-root `spaces/space.yml` is the space `<source>`. No
   artificial constraints on how a user organizes `spaces/`.
2. **A file in `spaces/` belongs to the nearest enclosing space**: the closest ancestor directory
   with a `space.yml`. A file under no space is not linked, and is warned about once per run.
3. **Specs are path prefixes.** `wsm install lgat/` is every space at or under `lgat`
   (`lgat/tech/` the same below `lgat/tech`). Installing a space does not install the spaces above
   it. `depends:` works for any space, with path ids.
4. **No tracker.** The sources' `spaces/` trees say what exists; what is installed is computed. A
   space is installed when `<root>/<source>/<path>/space.yml` is a link to that space's
   `space.yml`. Per-file state comes from comparing each source file with its link. `~/spaces` is
   never walked broadly. Leftover links (of files or spaces deleted from the source) are found by
   walking `<root>/<source>/` only into directories that mirror a directory of the source's
   `spaces/`, plus one level to notice a vanished space, and only links into the source count.
   It must stay fast with huge clones in `~/spaces`. The trackers go; the links stay, already
   where the model expects them. No backward compatibility (and no migration code: decision 12).
5. **One root**: `WSM_SPACES_HOME` (`anfs.conf`, default `~/spaces`); a source installs at
   `<root>/<source>/`. The DIR argument goes (and with it the `anfs src install a b` bug).
6. **`wsm status` is removed.** Its states fold into `wsm list`'s STATUS: `ok`, `partial` (with
   counts of missing and wrong links, files in the way, leftover links), `not in the source`;
   `wsm list <space> --json` gives per-file detail. Drift across the machine is `anfs status`
   (linker plan 03).
7. **wsm acts only on what it derives from the source**: the space's files (links), repository
   paths (cloned when missing; never pulled, never deleted), `links:` dests, and the skill
   directories the skills CLI writes. Anything else in `~/spaces` (the user's files, ovm's links,
   other tools' output) is never listed, changed or deleted. A user file where wsm would link is
   a conflict: reported, never overwritten; `-f` backs it up, as ppm does.
8. **Repository ids are `<space id>/<path>`.** A repository path may not lie inside a child space
   (validated); then the longest prefix of an id that is a known space is the space and the rest
   the path, unambiguously. `anfs work` uses this instead of guessing.
9. **`lib/anfs/repo.sh`**: the primitives for one clone at one path, shared by `anfs src`
   (`sources.sh`) and `wsm repo`: clone url dir [ref], status, ahead, update (pull),
   name-from-url, the ssh rewrite. Policy stays with the caller: sources pull on a TTL, space
   repositories are never pulled. The duplication goes (two `git clone`s, two URL parsers).
   `wsm repo` verbs align with `anfs src` where cheap; `-n` everywhere is linker plan 04's sweep,
   not duplicated here.
10. **`lib/anfs/skill.sh`**: the core of psm moves into a library used by `psm` (unchanged
    behaviour: global `-g`, the sources' top-level `skills/`, `~/.config/psm` configs; psm and
    `skills/` stay) and by a new `wsm skill install|list|remove`, which installs a space's skills
    into the space's directory (the `skills` CLI without `-g`, run with the space directory as
    cwd). Shared: agent detection (ppm trackers' `meta.agent`), building rows, calling the CLI
    with a scope (global or a directory), reading what is installed. A space's skills apply only
    to that space, not to the spaces below it. Installing an agent package re-syncs the spaces'
    skills, as psm's ppm callback does for the global ones.
11. **`space.yml`'s keys**, in full:

    ```yaml
    depends: [lgat/tech]
    repositories:
      - url: git@github.com:lgat/web.git
        path: web        # optional
        ref: main        # optional
    skills:
      - repo: vercel-labs/agent-skills   # owner/repo, URL or path, as the skills CLI takes
        skills: [deploy-to-vercel]       # optional, all when left out
      - repo: ./skills                   # relative = from this space.yml in the source
    links:
      - src: ~/.local/share/anfs/sources # ~ and $VAR expanded; relative = from the space dir
        dest: sources                    # optional: basename of src, as `ln -s src`; must stay inside the space dir
    ```

    `links:` are absolute symlinks, with the states `ok`, `missing src` (not an error), `wrong`,
    `in the way`; remove deletes a link only while it still points at its src. They overlap with
    ovm's vault links into spaces; both stay for now.

### After review (2026-10-21)

12. **No migration code.** The new wsm ignores the old state in `~/.local/state/wsm` (it neither
    reads nor deletes it). After the merge the person (or the orchestrator, by hand) deletes
    `~/.local/state/wsm` and the stale directories the old levels left (`~/spaces/lgat/spikes/
    {tech,orig}`, `~/spaces/cnfs/dev`, `~/spaces/user/dev`): plan 01's Verification lists the
    steps. The leftover walk's one-level limit stays, documented: those four are below its reach.
13. **`-f` moves what is in the way aside** (a backup under `$WSM_STATE_HOME/backup/`), as ppm;
    **`wsm remove` does not restore** backups, which would need a record.
14. **`PSM_CLI_VERSION` is renamed `ANFS_SKILLS_CLI_VERSION`** (`anfs.conf`), read by `skill.sh`
    for psm and wsm. No shim.
15. **`depends:` on another source names its alias**, though aliases are per machine. Accepted.
16. **Skills in parent directories**: an agent started in a child space may load the parent's
    project skills if it looks up the tree. Plan 03 checks Claude Code's documentation and
    records the answer; nothing is built for it.
17. **An anfs-ui panel for a space's skills** is later, not this unit.
18. **Linker plan 03 adapts to this unit** when it is reviewed (see "linker").
19. **Plans 01 and 02 of the draft are one plan** (`01-spaces`): an interim tracker would be code
    written only to be deleted, and backward compatibility is ruled out.

## Decided here (within the decisions above)

- **An id's first segment is its source.** A spec or `depends:` entry whose first segment is the
  alias of a source with `spaces/` is qualified; otherwise it is a path resolved the way names
  resolve today (`resolve.sh`): the highest-priority source that has it, and for `depends:` the
  declaring source or a lower-priority one. A bare `depends: [org]` is `<this source>/org`, not a
  sibling: the sibling rule was a workspace rule. When an alias equals a top-level directory of
  another source, the alias wins.
- **A repository path may neither lie inside a child space nor contain one** (decision 8 says the
  first; the second keeps a child space from being linked into a clone).
- **"Installed"** is a link at `<root>/<source>/<path>/space.yml` to this space's `space.yml`
  (its link text, or what it resolves to). A link there into the source that is dangling makes
  the row `not in the source`.
- **A leftover link** is a dangling link, in a mirrored directory, into the source's `spaces/`.
  A live link to a source file in the wrong place is the `wrong` of the file at that place, not a
  leftover; a link to anything outside the source is not wsm's (decision 7).
- **`links:` and skills are part of `wsm install` and `wsm remove`** of the space; `links:`
  count in `partial`, skills do not (reading them takes `npx`; `wsm skill list` shows them).

## Performance

Measured 2026-10-21: 10 files across 7 sources, a full link check took 56 ms. The work scales
with the files in the sources, not with what is in `~/spaces`: a check reads the source tree
(one `find` per source), tests each expected link with bash's `-L`/`-e`, and reads every link's
text with one `readlink` per source (it takes many operands, on macOS and GNU), comparing strings.
Only a link whose text differs is resolved. No fork per file. The leftover walk lists only
directories that mirror a source directory (bash globs, no `find`), so a clone of any size inside
a space costs one directory entry. Completion reads only the sources' `spaces/` (ids) and tests
`-L` for installed; it never runs the check. `wsm ls` takes 240 ms today (AGENTS.md); plan 01's
log records it again.

## linker

- **Linker plan 01 (merged)** gives `anfs_link_plan`/`apply`, `anfs_relpath`, `anfs_link_owner`.
  Plan 01 here adds one read-only function to `link.sh`, `anfs_link_check` (the batched check
  above), because the roadmap says what spaces need from the linker is added there.
- **Linker plan 03 (`anfs owner`, `anfs mv`, `anfs status`) conflicts with decision 6 as written**: it
  has `wsm status` "supply the rows for spaces" and `wsm __file mv` rename "the tracker's entry".
  Decided (18): linker 03 adapts to this unit when it is reviewed. Spaces' rows come from `wsm
  list --json` (the per-file detail), and a move within a space is the file's move in the source
  plus a relink, with no tracker.
- **Linker plan 04 (`-n` on every command)** is not duplicated: `wsm install -n` and `wsm repo install
  -n` exist and are kept; the new verbs (`wsm skill install|remove`, `wsm remove`) get `-n` there.

## ovm

ovm reads `wsm ls --json` (`.space`, a trailing `/` trimmed, and `.dir`) and wsm's
`install`/`remove` events, and matches ids as strings. Path ids keep that working: a vault link to
`lgat/tech/web` is a link to that space. What changes: a link whose target was a workspace or
source level with no `space.yml` (`acme/main`) becomes `unknown` to ovm (no such space), until the
directory gets a `space.yml`. ovm's help and comments say "level ... source, source/ws or
source/ws/space"; plan 01 rewords them (stack is in its job). On this Mac ovm's one link targets
`anfs/`, a source that no longer exists here (core was `anfs`): unknown before and after.

## Plans

| # | Plan | What | Job repos |
| --- | --- | --- | --- |
| 01 | `01-spaces` | Spaces by `space.yml` at any depth, path ids, prefix specs, nearest-space ownership, `depends:` by id, the repository-id rule (not inside, not containing a child space); no tracker: `anfs_link_check`, the leftover walk, `wsm list` STATUS and per-file detail; one root, several specs per install, `-f` moves aside; `wsm status` removed; the readers (stacks.sh, `anfs src install`, anfs-ui, `anfs work` and `work.bats`, the acme fixture, ovm's help) | core, anfs-dev, stack |
| 02 | `02-repo-lib` | `lib/anfs/repo.sh` under `anfs src` and `wsm repo`; one clone, one URL parser; `wsm repo ssh` | core |
| 03 | `03-skills-and-links` | `lib/anfs/skill.sh`, psm on it (`ANFS_SKILLS_CLI_VERSION`); `wsm skill install\|list\|remove`; `skills:` and `links:` in `space.yml`; agent installs re-sync spaces' skills | core |
| 04 | `04-repo-push` | `anfs_repo_push` and one CLI helper in `repo.sh`: `anfs src push` and `wsm repo push` (names or `--all`, `-n`, a list of outcomes) | core |

01 comes first: 02 and 03 build on its ids and computed state. 02 and 03 do not depend on each
other and can run side by side (both edit wsm and core `AGENTS.md`, in different places;
whichever merges second rebases).

## Open questions

None left: the draft's four were answered on 2026-10-21 (decisions 13–17).

## Completion Criteria

- `grep -n '_is_level\|_each_level\|_is_named\|_unlink_orphans\|ANFS_NAME_PARTS\|WSM_INSTALLED'`
  in wsm and `lib/anfs` finds nothing; `wsm help` has no DIR, no `status`, no workspace.
- A source with `spaces/space.yml`, `spaces/a/b/c/space.yml` and a file in `spaces/a/` installs
  `<source>` and `<source>/a/b/c`, the file with `<source>`; without the root `space.yml` the file
  is not linked and is warned about once.
- `-f` moves a file in the way aside and links; `wsm remove` leaves the backup.
- `$WSM_STATE_HOME` holds nothing but `backup/`; deleting a file from the source makes its space `partial`
  (one leftover) and a re-install `ok`; `wsm list` on this Mac's 7 sources stays under the
  measured scale (log).
- `anfs src install a b` installs both sources' spaces; `anfs install` of a stack with `lgat/`
  reports it installed when every space under `lgat` is.
- `anfs src` and `wsm repo` clone through `repo.sh`; `grep -rn 'git clone' packages/anfs
  packages/wsm` finds only `repo.sh`.
- `wsm skill install lgat/tech` puts the space's skills under `~/spaces/lgat/tech/.agents/` (and
  the detected agents' directories); psm's suite passes on `skill.sh`, and `PSM_CLI_VERSION` is
  gone (`ANFS_SKILLS_CLI_VERSION`).
- `links:` with `~/.local/share/anfs/sources` makes `~/spaces/anfs-dev/sources` `ok` with no
  change on this Mac.
