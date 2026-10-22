---
plan: wsm-spaces/03-skills-and-links
job: wsm-03
date: 2026-10-21
---

# Log: wsm-spaces/03-skills-and-links

## What was built

- **`lib/anfs/skill.sh`** (new; bash floor, functions only, messages through log.sh). psm's engine
  with a scope, `-g` or a directory: `anfs_skill_cli` (`npx skills@$ANFS_SKILLS_CLI_VERSION`, stdin
  closed; with `-g` the flag goes before the first option, so names stay before the variadic `-a`;
  a directory scope runs the CLI there and **refuses an empty, relative or missing directory**, since
  `cd ""` would leave it in the cwd), `anfs_skill_require`, `anfs_skill_agents` (ppm trackers'
  `meta.agent`, csv), `anfs_skill_package_agents`, `anfs_skill_display` and its inverse
  `anfs_skill_id`, `anfs_skill_dir_skills`, `anfs_skill_expand`, `anfs_skill_is_local`,
  `anfs_skill_load` (rows `<name>|<agents>|<source>|<path>`), `anfs_skill_installed_agents`,
  `anfs_skill_has`, `anfs_skill_status`, `anfs_skill_sync_rows` (the counts in `ANFS_SKILL_*`; the
  summary wording stays the caller's), `anfs_skill_remove`, `anfs_skill_remove_for` (only some
  agents), `anfs_skill_unlink`.
- **`ANFS_SKILLS_CLI_VERSION`** replaces `PSM_CLI_VERSION` (`anfs.conf`, commented, `latest`). No
  shim.
- **psm on it** (`package.yml` 0.1.6 by the hook): `require_npx`, `skills_cli`, `_installed_agents`,
  `_agent_display`, `_dir_skills`, `_load_installed`, `_skill_agents`, `_has_skill`, `_sync_rows`
  became thin calls with `-g`. Commands, options, output, configs, exit codes and the CLI command
  lines (the stub's log) are unchanged. `install.sh`'s `_psm_package_agents` calls
  `anfs_skill_package_agents`.
- **wsm** (`WSM_VERSION` 0.7.0):
  - **`links:`** (`[{src, dest?}]`): `src` with `~` and `$NAME`/`${NAME}` expanded from the
    environment (no eval; an unset variable makes the entry `invalid`), a relative one from the
    space's directory in the tree, normalized and written absolute; `dest` defaults to the basename
    and must pass `_repo_path_ok`. States `ok`, `missing`, `missing src`, `wrong`, `in the way`,
    `invalid`. Install makes them after the clones (`missing src` is said, not made, exit 0); `wrong`
    and `in the way` fail unless `-f` moves them aside with `_backup`. `remove` deletes one only
    while its text is still `src`. `wsm ls` counts `missing`/`wrong`/`in the way` into the row
    (`missing src` and `invalid` do not); `wsm ls <id> --json` adds `links: [{dest, src, status}]`.
  - **`skills:`** (`[{repo, skills?}]`): a path repo (`.`, `./`, `../`, `/`, `~`) resolved from the
    source `space.yml`'s directory with `cd -P`, its skills by name when left out (`*` for a remote).
    Installed in the space's directory for the detected agents, after files, clones and links; a
    path's every run (force_local), a remote's when missing. No agent: `waiting`; no npx: one
    warning; neither fails. A space without `skills:` never runs npx. A child space never installs
    its parent's.
  - **`wsm skill install [SPEC] [-n]`, `wsm skill list [SPEC]`** (`SKILL STATUS REPO`, keys `skill
    status repo space name`), **`wsm skill remove SPEC...`** (a space or prefix: every declared
    skill; `<space id>/<name>`: that one, which must be declared; asks, `-y`).
  - `remove` and `implode` take a space's skills first, then its links, then its files.
  - `wsm add` skips (refuses when named) `skills-lock.json` and the files of the skills that lock
    names under `.agents/skills/<name>/` and `<.dir>[/<sub>]/skills/<name>/` (yq, no npx).
    `_skills_tidy` deletes an empty lock and empty skill directories (`rmdir` only).
  - `wsm new`'s template lists `skills:` and `links:` commented out; help documents both.
  - `links:` and `skills:` of every space a command needs are read with **one yq** over all their
    `space.yml` files (`_meta_load`; one at a time only if that fails).
  - **`_home_ok`**: every links:/skills: function acts only in a space directory that is set,
    strictly below a set root, without `..`, and a real directory (not a link). Added at the
    person's request mid-plan.
  - **`wsm_ppm_changed`** in `install.sh`, registered in `post_install`: an install run with a
    `meta.agent` package runs `wsm skill install`; a remove run unlinks each removed id that no
    installed package still declares, through the plumbing `wsm __agents_rm <id>...` (not in help).
- **`anfs rpc`**: `RPC_GROUPS` gains `skill`, so `wsm skill list` (and `anfs space skill ls`) is a
  read.
- **Docs**: core AGENTS.md (Configuration; "wsm: spaces": `links:`, `skills:` and all of the above;
  Post-run Callbacks; Lists and --json: the `wsm skill list` row and `wsm list`'s `links`; Lib
  Structure: `skill.sh`; Testing), `packages/psm/AGENTS.md` (psm on `skill.sh`, the project-scope
  findings, Claude Code's parent-directory lookup), `packages/wsm/README.md` (`space.yml` in full,
  `wsm skill`, links' states).

## Commits (branch wsm-03, not pushed)

core:
- `905d5ba` lib/anfs: skill.sh, psm's engine with a scope (-g or a directory); psm on it; ANFS_SKILLS_CLI_VERSION replaces PSM_CLI_VERSION
- `9e1e531` wsm: skills: and links: in space.yml; wsm skill install|list|remove; agent installs re-sync the spaces' skills (wsm_ppm_changed); anfs rpc: skill is a group; 0.7.0
- `290eb87` skills: never an unscoped remove in a space (it sweeps skills/<name> and other agents' dirs): -a for the agents it is on, wsm drops the shared copy and lock entry; a universal agent counts when the copy is in .agents/skills
- `e4ca231` docs: skill.sh, wsm's skills: and links:, wsm skill, the callback (AGENTS.md); the CLI's project scope and Claude Code's parent-directory lookup (psm AGENTS.md); wsm README
- `c3626d1` skills: the global scope is scoped too (remove -g -y swept ~/agent/skills/<name>): anfs_skill_remove removes with -a for the agents a skill is on, then the shared copy and lock entry; psm's remove, prune and implode and wsm on it (the fix asked for before merge, below)

The pre-commit hook bumped `packages/psm` to 0.1.6; anfs (0.2.14) and wsm (0.1.8) already carried
unpushed bumps. `WSM_VERSION` set to 0.7.0 by hand.

## The skills CLI with a project scope (re-checked, `skills@1.7.4`)

In a scratch directory under the job's TMPDIR with a temp `HOME` (no git, no `package.json`). The
plan's findings hold, plus four that changed the code:

- With only non-universal agents (`-a claude-code`) there is no `.agents`: the copy is
  `./.claude/skills/<name>/`.
- **`remove <name> -y` without `-a` sweeps every agent's project directory**, three of them not
  hidden: `skills/<name>`, `data/skills/<name>`, `agent/skills/<name>` (the CLI's agent table). It
  deleted a user's `skills/hello/SKILL.md` in the scratch run, and in the first end-to-end run it
  took the space's own linked `skills/hello/` (the link only: the source was intact). Hence the
  third commit: wsm never removes without `-a`, and deletes itself what a scoped remove keeps for
  universal agents (the `.agents/skills/<name>` copy and its lock entry). The global scope does
  the same (fix after review, below).
- **The project-scope list leaves Pi out** of a skill's agents (OpenCode, the other universal one,
  is listed), so a skill installed for Pi read `partial` forever. `anfs_skill_has` now counts a
  universal agent (`pi`, `opencode`) as installed when the listed path is in `.agents/skills/`.
  This also applies to psm (global scope), where it can only turn a `partial` into `ok`.
- The CLI lists a skill as installed for AstrBot when a `data/skills/<name>` of the user's exists.

Written into `packages/psm/AGENTS.md` ("With a project scope").

## Fix after review: the global scope sweeps too

The orchestrator confirmed in a temp `HOME` (1.7.4) that psm's `remove -g -y foo` deleted a user's
own `~/agent/skills/foo/` (it left `~/skills` and `~/data/skills`). Re-checked here: a scoped
`remove -g -a ...` leaves `~/agent/skills/<name>`; it keeps `~/.agents/skills/<name>` when a
universal agent is still listed (`-a pi claude-code`) and drops it for `-a opencode`. The global
lock is `~/.agents/.skill-lock.json` (`version: 3`, `skills.<name>`).

`anfs_skill_remove <scope> <name>...` now does it for both scopes (wsm's own version moved into
it): one `list`, then `remove <names> -y -a <ids>` for the agents each name is on (a name not
installed is skipped; one only on agents with no known id stays, with a warning, copy included),
then the `<base>/.agents/skills/<name>` copy (a real directory) and the lock entry
(`<dir>/skills-lock.json`, or `~/.agents/.skill-lock.json`) go. psm's `remove`, `prune` and
`implode` call it with `-g`; `agents rm` already used `-a`. **No `skills remove` without `-a`
is left in core.**

psm behaviour change: `psm remove <name>` of a skill the CLI does not list now runs nothing (was a
blind `remove -g -y`); the command lines carry `-a`. By hand with the real CLI (temp `HOME`,
agents pi and claude-code): `psm install -y acme/`, then `psm remove -y hello` left the user's
`~/agent/skills/hello/SKILL.md`, removed the copy and the lock entry, `psm ls` says `missing`.

Tests: psm 15 (new: every remove `psm remove`, `prune`, `agents rm` and `implode` make carries
`-a`, the shared copy goes, `~/agent/skills/<name>` stays), `skill.bats` 8 (new: both scopes, the
copy and both locks, a not-installed name skipped, an empty scope refused), wsm 96. **Full run
once: 448 passed, 0 failed**, 31 files, 48 s.

## Claude Code and parent directories (unit decision 16)

From code.claude.com/docs/en/skills (WebFetch, 2026-10-21): "Claude Code loads project skills from
`.claude/skills/` in the directory where you start it and in every parent directory up to the
repository root". Skills below the start directory load lazily, when Claude first touches a file
there; in a linked git worktree the walk stops at the worktree root. **Not verified**: what the
walk does outside any git repository (a space directory under `~/spaces` usually is not one), as
the page does not say; and whether `.agents/skills` is read (not mentioned). By the documented rule,
an agent started inside a clone in a space stops at the clone's root and does not see the space's
skills above it (inference, not tested). Recorded in `packages/psm/AGENTS.md`; nothing built.

## Decisions made during the work

- **Removal is scoped** (above): the plan said "the CLI removes each"; a bare remove would break
  decision 7. **wsm (and psm) delete the `.agents/skills/<name>` copy and the lock entry
  themselves** after a scoped remove: approved by the person before merge.
- **`missing src` for a link already in place** whose src vanished (text equals src): reported as
  `missing src`, not `ok`, and `remove` still takes it.
- **`invalid` links are not counted** in `wsm ls`'s NOTE (they fail install with a warning and show
  in `--json`'s `links`). Approved by the person before merge.
- **`wsm skill install` takes `-n`** (the plan names no option; `wsm install -n` already passes dry
  through, so it was free). `wsm skill remove` has no `-n` (linker plan 04's sweep).
- **`wsm skill list`'s `repo`** is the repo as the CLI is given it (a path resolved through links),
  not as written in `space.yml` (`./skills` means nothing outside the source).
- **`wsm skill list` without npx** exits 69 once a space under SPEC is installed, declares skills
  and an agent exists; install only warns.
- **The callback triggers only on `meta.agent` packages**, not on wsm's own install (unlike psm's):
  a space's skills are installed by `wsm install` of the space already.
- **`_written_by_skills`** looks at one or two directories above `skills/`, the first hidden (covers
  `.agents`, `.claude`, `.pi/agent`); a non-hidden agent directory (`skills/<name>`) is the space's
  own and is added as usual.
- **`PSM_CLI_VERSION` remains only in psm's test** that proves it is no longer read. The completion
  criterion's grep therefore finds that test.

## Tests (all through `anfs work run wsm-03 core …`, `</dev/null`)

- While working: `packages/anfs/tests/skill.bats` 7 (new: `-g` placement and version, a directory
  scope with no `-g` and the cwd kept, an empty/relative/missing scope refused, agents from trackers
  one and a list, status, universal agents, sync_rows), `packages/psm/tests` 14 (new: the version
  variable; prune with a `*` repo after the row gained `path`), `packages/wsm/tests/skills.bats` 17
  and `links.bats` 9 (new, the plan's Test Spec, plus the scoped remove, the lock and copy cleanup,
  a space directory replaced by a link), `rpc.bats` 21 (new: `wsm skill list` is a read, `skill
  install` a write). The 70 existing wsm tests unchanged and passing.
- **Once, at the end**: `packages/anfs/tests/all`: **446 passed, 0 failed**, 31 files (pmm and pcm
  off), 48 s; after the fix, once more: **448 passed, 0 failed**.
- anfs-ui: not run (`kinds.rb` unchanged).
- **By hand, temp HOME, the real CLI** (`skills@1.7.4`): a source with a space declaring
  `skills: [{repo: ./skills}]` and a `links:` entry, a ppm tracker for an agent package. With
  `claude-code`: install linked the files and the link, copied the skill to `.claude/skills/hello`,
  `wsm ls` ok, `wsm skill ls` ok, `wsm remove -y` left `~/spaces` empty and the source intact. With
  `[pi, claude-code]`: the same, the `.agents` copy and lock removed too.
- Performance: `wsm ls` on 10 spaces, best of 3: 142 ms (0.6.0) / 175 ms (0.7.0): one yq per run
  for the installed spaces' `links:`, and `skill.sh` sourced.

## Not run

- Deferred to the end of the roadmap (boot a box or a VM): `roundtrip debian`, `roundtrip-macos`.
- pmm's and pcm's suites are off (ROADMAP, 2026-10-19); this plan does not touch them.
- Nothing installed on this host; the real CLI ran only in temp dirs with a temp `HOME`.

## Open questions

None left: the three raised were answered before merge (the cleanup and `invalid` approved; the
global sweep confirmed and fixed above).

## By hand, for the person after merge

1. `ppm install core/anfs core/psm core/wsm` (links `skill.sh`, psm 0.1.6, wsm 0.7.0, registers
   `wsm_ppm_changed`). If `anfs.local.conf` sets `PSM_CLI_VERSION`, rename it
   `ANFS_SKILLS_CLI_VERSION`.
2. Add `links: [{src: ~/.local/share/anfs/sources}]` to anfs-dev's `space.yml`, then `wsm install
   anfs-dev`: the existing `~/spaces/anfs-dev/sources` (an absolute link to that path) should be
   `ok`, untouched (`wsm ls anfs-dev --json` → `links`).
3. `skills:` in one space, `wsm install <space>`, `wsm skill ls`.
