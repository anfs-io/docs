---
plan: report/01-anfs-report
job: anfs-report
date: 2026-10-22
---

# Log: report/01-anfs-report

## What was built

- **`anfs report`** (`lib/anfs/report.sh`, declared by `anfs_report_cli` in `bin/anfs`): a group
  whose default sub-command is `new`, so `anfs report --title ...` works as decided.
  - `new`: `--package`, `--title` (required), `--expected`, `--actual`, `--steps`, `--body-file
    F|-` (one of the four required), `-n`, `-y`, `--json`. Body: `Package:`, the body file's
    text, `### What was expected` / `### What happened` / `### Steps to reproduce` (those given),
    then the diagnostics. Prints the id.
  - `diag`: anfs's version (and `ANFS_WORK`), platform (`platform`, `uname -sr`, arch, the OS's
    own version), bash, git, podman; a table of sources (short commit, `anfs_repo_status`); a
    table of ppm trackers' versions; the log's last 10 failures (`error:` lines, and a failed
    run's `start:` with its non-zero `end:`). `--json`: `{"diagnostics": ...}`.
  - `check`: `CHECK STATUS DETAIL` rows: `report-to` (value and FROM/file via config.sh), then
    `file` (writable, or will be created) or `gh`, `gh-auth` (`gh auth status`), `origin core`,
    `origin stack`; then `skill` (anfs-bug-report through `lib/anfs/skill.sh`: installed for every
    detected agent, partial, missing, or no npx). Exit 1 on any fail; `--json` is the table.
  - Ids: a file's `r-<YYYYMMDD>-<n>` (heading `## r-... <title>`, then `Status: open`), GitHub's
    `owner/repo#N` (from the URL `gh issue create` prints).
  - `close <id> --commit SHA [--source S]`, `status <id>`, `wait <id> [--timeout] [--interval]`:
    as in the plan ("Following the fix").
  - Routing: `ANFS_REPORT_TO` = `github` (default) or an absolute path (`~/` expanded). GitHub:
    the owner source's origin (`git@github.com:`, `ssh://git@github.com/`, `https://github.com/`),
    else 1 naming `ANFS_REPORT_TO`; duplicates by `gh issue list --search "<title> in:title"`
    (with `--jq ... @tsv`, so no jq needed), shown (stderr under `--json`), then `anfs_confirm`,
    then `gh issue create --body-file`.
- **`cli.sh`**: `cli_group <name> <summary> --default <sub>`: in `_cli_find`, an option where the
  sub-command goes (not `-h`/`--help`) resolves to the default; completion offers the default's
  options after `report -`; the group's help says where options go.
- **`anfs.env`**: `#ANFS_REPORT_TO=github` with a comment (settings.bats holds it).
- **Skill** `skills/anfs-bug-report/SKILL.md`: when it triggers (one line), check mode
  (`/anfs-bug-report check` → `anfs report check`, relayed), reproduce minimally, run `anfs
  report`, 77 → ask the person before `-y`, keep and cite the id, `anfs report wait` as a
  background command (carry on or delegate meanwhile), whoever merges runs `anfs report close`.
  Never chooses the destination, never calls gh.
- **Docs**: core `AGENTS.md` ("Reporting a bug: anfs report", `--default` under "The Command
  Line", rule 5 and the keys table for `report check`, Lib Structure, Testing); `README.md`
  ("Reporting a Bug"); `packages/anfs/README.md` (the commands, the skill, how to install it).

## Commits (branch anfs-report, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| docs | `cf07d04` | the plan; `units/report.md` |
| core | `53c6d0e` | `cli.sh`: a group's default sub-command; `registry.bats` |
| core | `3ac2aae` | `anfs report`, `report.sh`, `report.bats`, `anfs.env`, the skill, docs (the hook bumped core/anfs to 0.2.16) |
| docs | (this) | the plan's additions (check; ids, close, status, wait); this log |

## Decisions made during the work

1. **`report` is a group with a default sub-command (`new`)**, which needed `cli_group --default`
   in `cli.sh`. A group could not also be a command, and the decided `anfs report --title ...`
   had to keep working beside `anfs report diag|check|status|close|wait`. `anfs report` alone is
   the group's help.
2. **`-n` makes no `gh` call**, not even the duplicate search: a dry run works offline. On GitHub
   it therefore has no id (`id: null` in `--json`); on a file it shows the id it would take.
3. **A file target never asks**; only `gh issue create` does (77 without a terminal or `-y`).
   `close` never asks (it closes; nothing is lost).
4. **Duplicates under `--json`** go to stderr as text, so an agent that hits 77 still sees them.
5. **ppm's trackers are read as files** for the package versions and to resolve a bare
   `--package` (as `onboard.sh` does), not `ppm list --json`: a report matters most when ppm is
   what is broken, and it costs a yq per package.
6. **`status` on a GitHub issue closed with no `Fixed in` comment is `fixed`** (exit 1, "closed
   with no commit named"): there is no commit to look for, so `wait` runs to its timeout. A `gh`
   that cannot answer is 2 for `status`, and a warning plus another try for `wait`.
7. **`close` records the sha as given** (7 to 40 hex digits); it does not check that the commit
   exists anywhere. `--source` defaults to core for a file, and to the source whose origin is the
   issue's repo for GitHub.
8. **`wait` defaults**: `--timeout 3600`, `--interval 60`.
9. **`platform.sh` and `skill.sh` are sourced by report.sh only when `diag` or `check` runs**, so a
   tab press pays only for report.sh itself (about 3 ms: 63 against 60 ms for `anfs __complete iso
   pull ""`).
10. **`check`'s skill row runs the skills CLI** (`npx skills list -g --json`, read-only), as `psm
    list` does; with `@latest` that may fetch the CLI from npm on a real run. The test stubs npx.

## Tests (all through `anfs work run anfs-report core …`, `</dev/null`)

- `packages/anfs/tests/report.bats` (new): 26 tests, all passing. The interface (help, the
  default, 64s, completion), `diag` (versions, sources with a dirty one, trackers, the log's
  failures, `ANFS_LOG=off`, `--json`), `new` to a file (ids, `Status: open`, earlier text kept, the
  directory made, `--json`, a relative path), `--body-file` (file and stdin), `-n` (both targets,
  no gh call), GitHub with no duplicate and with one (77, then `-y`), `--json`, `--package`
  (source/name, installed, in a source only, unknown), a non-GitHub origin; `close`/`status`/`wait`
  for file ids (one Status line changed, open → fixed → available with a real temp repo,
  unknown 2, 64s) and GitHub ids (comment and close, the source by origin, OPEN, CLOSED fixed or
  available, gh failing 2); `wait` (timeout 1, available 0, polling until a merge lands, unknown
  2); `check` (all ok, gh auth failing + non-GitHub origin + skill missing, no gh on PATH, a file
  writable / to be created / read-only, the skill partial, `--json`). gh, podman and npx are
  stubs; no network.
- `registry.bats`: one new test (a group's default sub-command), and `CHECK STATUS DETAIL` among
  the lists that are not of resources: 43 passing. `settings.bats`, `bash.bats`: passing.
- **The full run, once, at the end**: `packages/anfs/tests/all`: 649 passed, 0 failed, 47 files
  (pmm and pcm off), 88 s. An earlier run, before report.sh stopped sourcing `platform.sh` and
  `skill.sh` at start, failed `iso.bats`'s tab-press budget once (108 ms against 100, under the
  parallel load; it passed alone, and the measured cost of report.sh is about 3 ms).

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19) and were not run; neither changed.
- `ANFS_WORK`'s job worktree counting for `status`/`wait` (`anfs_work_job`): untested, since a
  test may not set `ANFS_WORK` (this job's rule). Read by hand only.
- The real `gh` and GitHub: never called. `gh issue list --search ... --jq`, `issue create`,
  `issue view --json state,comments --jq`, `issue comment`, `issue close` and `auth status` were
  tested against a stub only; the `--jq` expressions have not met real gh output.
- The real skills CLI for `check`'s skill row (stubbed npx).
- Nothing installed on this host; `~/.config/anfs` only read.

## By hand, for the person after merge

1. `anfs src update` (pulls core; `lib/anfs` is one link, so `report.sh` and the new `cli.sh` are
   live; `anfs.env` is a linked file, so its new line is there too).
2. Send this machine's reports to the backlog:

   ```bash
   anfs config set ANFS_REPORT_TO=~/spaces/anfs-dev/docs/BACKLOG.md
   # or by hand: echo 'ANFS_REPORT_TO=~/spaces/anfs-dev/docs/BACKLOG.md' >> ~/.config/anfs/anfs.local.env
   ```

3. Make the skill available the way the other core skills are (psm syncs core's `skills/`):

   ```bash
   psm install core/anfs-bug-report     # or: psm sync core
   psm list | grep anfs-bug-report      # ok
   ```

4. `anfs report check`: every row `ok` (`report-to` from `local`, `file` writable, `skill`
   installed for your agents).
5. Optional, against the real file: `anfs report --title "test report" --actual "ignore me" -n`
   (prints the id it would take); a real one, then `anfs report close <id> --commit <a core sha>`
   and `anfs report status <id>`, and delete the section by hand afterwards.
6. Optional, with `ANFS_REPORT_TO=github` for one command: `ANFS_REPORT_TO=github anfs report
   check` to see `gh auth status` and the origins against real gh.

## Open questions

1. `wait`'s defaults (an hour, every minute) are a guess; an agent waiting on a merge that comes
   the next day needs `--timeout`.
2. A GitHub issue closed with no commit named never becomes `available`; should `status` say
   something other than `fixed` for it (e.g. `closed`)? Kept to the three states asked for.
