---
---

# Plan 03 — `lib/anfs/skill.sh`, `wsm skill`, and `skills:` and `links:` in `space.yml`

Written 2026-10-21 against main (which holds `linker-01` and `ovm-01`), and plan 01 of this
unit (independent of plan 02)

## Context — read these files first

The job is `anfs work new <job> core`. Paths are core's.

- `units/wsm-spaces.md` — decisions 7, 10, 11; "Decided here"; decision 2026-10-21/5 (skills in
  parent directories); plan 01's `log.md`
- `packages/psm/home/.local/bin/psm` — all of it: `require_npx`, `skills_cli`,
  `_package_agents`, `_installed_agents`, `_agent_display`, `_dir_skills`, `_anfs_skill_sources`,
  `_config_sources`, `_load_installed`, `_skill_agents`, `_has_skill`, `_sync_rows`, `cmd_list`,
  `cmd_install`, `cmd_agents_remove`, `cmd_implode`
- `packages/psm/AGENTS.md` — "What the skills CLI actually does" (found 2026-09-15 against the
  global scope); `packages/psm/install.sh` (`psm_ppm_changed`); `tests/sources.bats` (the npx stub)
- `packages/wsm/home/.local/bin/wsm` — after plan 01: `_install_space`, `cmd_remove`, `cmd_list`,
  `cmd_implode`, `_add_one`, the `repo` group (the model for `skill`); `packages/wsm/install.sh`
- `packages/ppm/home/.local/lib/ppm/` — `ppm_register_callback` and how a package's callback is
  called (`AGENTS.md` "Post-run Callbacks")
- `packages/anfs/home/.local/bin/anfs` — `RPC_GROUPS`; `tests/rpc.bats`
- `packages/anfs/home/.config/anfs/anfs.conf` (where `ANFS_SKILLS_CLI_VERSION` goes);
  `grep -rn PSM_CLI_VERSION` in core for every reader of the old name
- `AGENTS.md` — "Lists and --json", "Post-run Callbacks", "wsm: spaces", "Lib Structure"

## Overview

psm's engine (agents, the CLI, rows, what is installed) moves into `lib/anfs/skill.sh` with a
scope: global, or a directory. psm uses it with the global scope and behaves as before. wsm uses
it with a space's directory: `space.yml`'s `skills:` are installed into the space, `wsm skill
install|list|remove` manage them, and installing an agent package re-syncs them. `space.yml`'s
`links:` put absolute links into the space.

## What the skills CLI does with a project scope

Checked 2026-10-21 against `skills@latest` (1.7.4) in a scratch directory with no git and no
`package.json`; re-check in this plan and write the result into `packages/psm/AGENTS.md` beside
the global findings:

- Without `-g` the scope is the **current directory**, with no project marker needed. `add
  <path> -y -a claude-code opencode` wrote `./.agents/skills/<name>/` (the copy), a link
  `./.claude/skills/<name> -> ../../.agents/skills/<name>` for a non-universal agent, and
  `./skills-lock.json` (`version`, `skills.<name>.{source, sourceType, computedHash}`, the source
  written relative to the cwd even when given absolute).
- `list --json` (no `-g`) answers for the cwd only: `[]` from a subdirectory. Rows carry `name`,
  `path`, `scope: project`, `agents` (display names; the universal ones, OpenCode and Pi, listed
  though not asked for), and for a local source its absolute path (the global scope said
  `source: none`).
- `remove <name> -y` removes the copy and the agent links, and leaves empty `.agents/skills`,
  `.claude/skills` and a `skills-lock.json` with `"skills": {}`.
- Inside an agent session it prints `Agent detected — installing non-interactively`.

So a space's skills land in the space's directory, in dot-directories wsm did not link, and
`skills-lock.json` is a file wsm did not link.

## Implementation

### 1. `lib/anfs/skill.sh`

Functions only, the bash floor. The scope is `-g` or a directory:

```
anfs_skill_cli        <scope> <args...>     npx skills@$ANFS_SKILLS_CLI_VERSION, stdin closed; -g added, or run in <dir>
anfs_skill_require                          npx (node through mise), or 1 with the message psm prints today
anfs_skill_agents                           the agent ids of installed ppm packages (meta.agent), csv
anfs_skill_package_agents <repo/pkg>        one package's ids
anfs_skill_display    <id>                  the CLI's display name, or 1
anfs_skill_dir_skills <dir>                 <name>/SKILL.md directly under <dir>
anfs_skill_load       <scope>               what is installed there: ANFS_SKILL_INSTALLED rows
anfs_skill_has        <name> <agents csv>   installed for every one of them (after load)
anfs_skill_sync_rows  <scope> <force_local> <detected>   install rows from stdin: "<label>|<repo>|<skills csv>|<agents csv>"
anfs_skill_status     <name> <agents csv>   ok, partial, missing (after load)
anfs_skill_remove     <scope> <name>...     remove for every agent
anfs_skill_unlink     <scope> <agent> <name>...   remove for one agent (names before -a)
```

`PSM_CLI_VERSION` is renamed `ANFS_SKILLS_CLI_VERSION` (decided 2026-10-21): set in `anfs.conf`
(commented, default `latest`), read by `skill.sh` for psm and wsm alike. No shim: psm no longer
reads `PSM_CLI_VERSION`; `grep -rn PSM_CLI_VERSION` in core finds nothing after (AGENTS.md's
"Configuration" example names the new one).

### 2. psm on it

psm keeps its commands, options, output, configs, exit codes and `psm_ppm_changed`; its functions
become calls with scope `-g`. `packages/psm/tests/sources.bats` passes; the one behaviour change is the setting's name
(`ANFS_SKILLS_CLI_VERSION`). `psm`'s
AGENTS.md gains the project-scope findings above and points at `skill.sh` for the engine.

### 3. `skills:` in `space.yml` and `wsm skill`

- Rows from a space's `skills:`: `repo` as the CLI takes it (`owner/repo`, a URL, a path); a
  relative path is from the `space.yml`'s directory in the source, resolved through links
  (`cd -P`: the CLI does not follow linked directories); `skills` optional (every skill when left
  out: for a path, the names `anfs_skill_dir_skills` finds, so each gets a real status; for a
  remote, `*`). The agents are the detected ones (`anfs_skill_agents`).
- Only the declaring space (decision 10): a child space's directory is never a scope for its
  parent's skills.
- **`wsm skill install [SPEC]`**: for each installed space under SPEC, `anfs_skill_sync_rows`
  with the space's directory as scope. Path skills are copied again every run (they are the
  space's own, and edits arrive that way); remote ones when missing for an agent. No agent yet:
  `waiting`, not a failure, as psm. No npx: one warning, not a failure.
- **`wsm skill list [SPEC]`**: `SKILL STATUS REPO`, keys `skill` (`<space id>/<name>`, or
  `<space id>/*` for a whole remote repo), `status` (`ok`, `partial`, `missing`, `waiting`, `*`,
  null when the space is not installed), `repo`, `space`, `name`. One `list --json` per installed
  space.
- **`wsm skill remove SPEC...`**: a space (or prefix) removes every skill its `space.yml`
  declares from it; `<space id>/<name>` removes that one. Asks first (`-y`).
- **`wsm install`** of a space ends with its skills (after files, clones and `links:`); **`wsm
  remove`** of a space removes them first; **`wsm implode`** for every space. Skills it did not
  declare (a user's `npx skills add` in the space) are never removed (decision 7).
- What the CLI writes is wsm's to step around: `wsm add` refuses or skips `skills-lock.json`,
  `.agents/skills/<name>/` and `<agent dir>/skills/<name>` for the names in that
  `skills-lock.json` (read with yq, no npx); `remove` deletes `skills-lock.json` when its
  `skills` is empty and the empty skill directories, `rmdir` only.
- `wsm list` does not run the CLI: skills are not in its STATUS ("Decided here").
- The `RPC_GROUPS` of `anfs rpc` gain `skill`, so `anfs rpc wsm skill list` is a read
  (`rpc.bats`).

### 4. Agents come and go

`packages/wsm/install.sh` registers `wsm_ppm_changed` with ppm (`ppm_register_callback`), next to
psm's and on the same trigger: on `install`, when a package in the run declares `meta.agent`,
`wsm skill install` (every installed space); on `remove`, for each removed agent id no installed
package still declares, `anfs_skill_unlink <space dir> <id> <names>` in each installed space with
skills. wsm's own, not psm's: psm does not need to know wsm, and wsm does not need psm installed.
Both use `anfs_skill_package_agents` for the trigger.

### 5. `links:` in `space.yml`

- Each entry: `src` (`~` and `$NAME`/`${NAME}` expanded from the environment, no `eval`; an unset
  variable makes the entry invalid; relative is from the space's directory); `dest` optional (the
  basename of `src`), relative to the space's directory and required to stay inside it (the
  `_repo_path_ok` rule: no `/` first, no `..`).
- `wsm install` makes `<space dir>/<dest>` an **absolute** symlink to `src`, after the files and
  clones. States: `ok` (a link whose text is `src`), `missing` (nothing there, `src` exists),
  `missing src` (`src` does not exist: not made, not an error, said once), `wrong` (a link
  elsewhere), `in the way` (a file or directory). `wrong` and `in the way` fail the entry; `-f`
  moves them aside as plan 01's `-f` does (no restore).
- `wsm remove` deletes a `links:` link only while its text is still `src`.
- In `wsm list`: `missing`, `wrong`, `in the way` count into the row's counts (and `partial`);
  `missing src` does not. `wsm list <id> --json` adds `links: [{dest, src, status}]`.
- ovm's vault links into a space are separate and stay (unit, decision 11); a `links:` dest and an
  ovm link at the same path meet as `in the way` on whichever comes second.
- `wsm new`'s template lists `skills:` and `links:` commented out, with the schema's comments.

### 6. Documentation

Core `AGENTS.md`: "wsm: spaces" (`skills:`, `links:`, `wsm skill`, what the CLI writes and how
wsm steps around it, re-sync on agent installs); "Post-run Callbacks" (wsm registers too); "Lists
and --json" (the `wsm skill list` row; `wsm list`'s `links`); "Lib Structure" (`skill.sh`);
"Testing" (the wsm and psm lines). `packages/wsm/README.md`: `space.yml` in full (decision 11's
block), `wsm skill`. `packages/psm/AGENTS.md`: the project-scope findings; psm runs on
`skill.sh`.

## Test Spec

The npx stub of psm's suite (logs its arguments, `list` answers `$NPX_LIST`) is shared by wsm's
tests; it also logs its working directory.

- psm: `tests/sources.bats` passes (only a test that sets the CLI version, if any, renames the
  variable); `ANFS_SKILLS_CLI_VERSION` reaches the `npx skills@<version>` call; `PSM_CLI_VERSION`
  is not read.
- `packages/anfs/tests/skill.bats`: `anfs_skill_cli` with `-g` and with a directory (the stub
  sees `-g`, or no `-g` and that cwd); `anfs_skill_agents` from a ppm tracker with `meta.agent`
  (one and a list); `anfs_skill_status`.
- wsm `skills.bats` (new): install of a space with `skills:` runs `add <abs path> -y -s <names> -a
  <agents>` in the space's directory, and `add owner/repo ... -s '*'`; a relative `repo` is
  resolved from the source's `space.yml`; a child space's install does not run its parent's; no
  agent is `waiting` and exit 0; `skill list` statuses from `$NPX_LIST`; `skill remove` asks,
  `-y` removes; `wsm remove` removes the space's skills first and leaves a skill it did not
  declare; `add` of a directory skips `skills-lock.json` and the skill copies; `implode` removes
  every space's skills; `wsm_ppm_changed install` with an agent package re-syncs, without one does
  nothing; `remove` unlinks only an agent nothing else declares.
- wsm `links.bats` (new): `~` and `$VAR` expanded, an unset variable invalid; relative `src`;
  default `dest`; a `dest` with `..` or a leading `/` refused; each state; `missing src` is exit 0;
  `-f` backs up what is in the way; `remove` leaves a link the user repointed; `ls <id> --json`
  has `links`; `partial` counts.
- `rpc.bats`: `wsm skill list` is a read through `anfs rpc`.

## Verification

Every command runs in the job: `anfs work run <job> core -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs, nothing is installed on this host, and the real skills CLI is not run by the
tests.

- [ ] While working, the file you touch: `bats packages/wsm/tests/{skills,links}.bats`,
      `bats packages/anfs/tests/skill.bats`, `bats packages/psm/tests`
- [ ] The project-scope findings, re-checked once with the real CLI in a temp directory (no `-g`,
      a local skill, `-a claude-code`), and written into `packages/psm/AGENTS.md`
- [ ] Decision 2026-10-21/5 of the unit: how Claude Code finds project skills in parent directories,
      from its documentation, written into the same section; nothing changed for it
- [ ] `bats packages/wsm/tests`, `bats packages/psm/tests`, `bats packages/anfs/tests`
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm install core/anfs core/psm core/wsm`;
  add `links: [{src: ~/.local/share/anfs/sources}]` to anfs-dev's `space.yml` and `wsm install
  anfs-dev`: the existing `~/spaces/anfs-dev/sources` is `ok` untouched; `skills:` in one space
  and `wsm skill list`.
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian`, `roundtrip-macos`.
