---
---

# Plan 01 — `anfs report`: an issue with its diagnostics, filed where the machine says

Written 2026-10-22 against core main `78aac47`. Decided with the person 2026-10-22 (the ROADMAP
"Next" item). Two additions from the person the same day, while it was being built: `anfs report
check` with the skill's check mode, and ids with `close`, `status` and `wait`, so the reporter
learns when its bug is fixed. Both are folded in below.

## Context — read these files first

The job is `anfs work new anfs-report core docs`. Paths are core's unless they name docs.

- `AGENTS.md`: the bootstrap header of an entry point, "The Command Line" (`cli_cmd`,
  `cli_group`, `-y`, `-n`, `anfs_confirm`, exit codes), "Lists and --json" (rule 5),
  "Messages, Debugging and the Log", "Configuration: `<tool>.env`", "Where files go"
- `packages/anfs/home/.local/bin/anfs`: where anfs's groups are declared (`anfs_iso_cli`,
  `anfs_config_declare`) and its libraries sourced
- `lib/anfs/`: `cli.sh` (`_cli_find`, `_cli_complete`, `cli_help_command`), `config.sh`
  (`_anfs_config_collect`: a setting's value, FROM and file), `paths.sh` (`anfs_conf_expand`),
  `repo.sh` (`anfs_repo_status`), `sources.sh` (`gitsrc_collect`, `gitsrc_dir_reply`),
  `platform.sh`, `log.sh` (the log's line format), `skill.sh` (`anfs_skill_load`,
  `anfs_skill_agents`, `anfs_skill_status`), `table.sh` (`anfs_table`, `anfs_result`)
- `skills/pcm-containers/SKILL.md`: a core skill's form
- `packages/anfs/tests/{config,registry,settings}.bats`: a temp `HOME` with the tree linked in

## Overview

`anfs report` builds an issue in markdown: a title, what was expected, what happened, the steps
to reproduce, and a diagnostics block it gathers itself. Where it goes is the machine's business
(`ANFS_REPORT_TO`), so an agent only describes the bug. `anfs report check` says whether this
machine can file one. A core skill, `anfs-bug-report`, is how an agent is told to use it.

## The interface

`report` is a group with a **default sub-command**: options with no sub-command go to `report
new`, so the decided `anfs report --title ...` works and the group still lists what it holds.
That is one addition to `cli.sh` (below); a group that is also a command is otherwise impossible.

| Command | Does |
| --- | --- |
| `anfs report [new] [--package PKG] [--title T] [--expected TEXT] [--actual TEXT] [--steps TEXT] [--body-file F] [-n] [-y]` | builds the report and files it where `ANFS_REPORT_TO` says |
| `anfs report diag` | prints only the diagnostics block (markdown) |
| `anfs report check` | `CHECK STATUS DETAIL`, one `ok`/`fail` row per item; exit 1 when one fails |
| `anfs report status <id>` | `open`, `fixed` or `available`; exit 0 available, 1 open or fixed, 2 an unknown id |
| `anfs report close <id> --commit SHA [--source S]` | records the fix on the report and closes it |
| `anfs report wait <id> [--timeout SECS] [--interval SECS]` | polls `status`: 0 when available, 1 at the timeout (default 3600, every 60) |
| `anfs report` | the group's help (nothing given) |

- `--title` is required, and so is something to say: one of `--expected`, `--actual`, `--steps`,
  `--body-file` (64 otherwise). `-i` asks for them, as on every command.
- `--body-file F` (`-` is stdin): free text placed first in the body, before the sections. Its
  kind is `file` (completion offers paths).
- `--package PKG`: `source/name`, or a bare name resolved to the installed package
  (`$XDG_STATE_HOME/ppm/installed/*/<name>.yml`, the first in source order), else the first
  source with `packages/<name>`; else 1. Only the source part decides the owner, so
  `stack/postgres` (a container) works too. None given: the owner is `core`. Its kind is
  `package`: `complete_package` lists the installed trackers' names (file names only).
- `-n`: prints the report, then `would go to: github <owner/repo>` or `would be appended to:
  <file>`. No `gh` call at all, not even the duplicate search: `-n` works offline.
- `-y`: no confirmation before `gh issue create`. A file target never asks (appending destroys
  nothing).
- `new` prints the report's **id**: `owner/repo#N` (github), `r-<YYYYMMDD>-<n>` (a file).
- `--json`: `new` prints one object: `id` (null for `-n` on github), `to` (`github`, `file`),
  `repo`, `file`, `url` (the issue made; null for a file or `-n`), `title`, `filed` (`true`/`false`), `duplicates` (an array of
  `{number, state, title, url}`), `report` (the markdown). `diag --json` is
  `{"diagnostics": "<markdown>"}`. `check --json` is the table's array (`check`, `status`,
  `detail`).

### The report

```
Package: `core/pcm`                      (only with --package)

<the body file's text>

### What was expected
### What happened
### Steps to reproduce
### Diagnostics
```

A section with no text is left out. The title is not in the body (GitHub has it; the file target
puts it in the heading).

### The diagnostics block (`report diag`)

- `anfs`: the running anfs's version (`cli_package_version`), the job's when `ANFS_WORK` is set
  (and then `ANFS_WORK=<job>` on its own line)
- `platform`: `platform` (lib/anfs/platform.sh), `uname -sr`, `uname -m`, and the OS's own
  version: `sw_vers -productVersion` on macOS, `PRETTY_NAME` from os-release on Linux
- `bash` (`$BASH_VERSION`), `git`, `podman`: each only when present (`podman --version`, which
  starts no machine)
- **Sources**: a table, each collected source: alias, short commit, `anfs_repo_status` (`clean`,
  `dirty`, `unpushed`, `no upstream`, `local`, `missing`): "whether it has uncommitted changes" and
  whether the commit is on the remote at all
- **Installed packages**: a table, each ppm tracker: `source/name`, its `version:`
- **Recent failures in the log**: when `ANFS_LOG` is on and the file has any, the last 10 of: an
  `error:` line, or a non-zero `end:` line preceded by the `start:` line of the same id (the
  command that failed), in a fenced block. Only `anfs.log`, not `.1`. None: the section is left
  out.

Reading another tool's state: the trackers are read as files, as `onboard.sh` and
`ppm_callback_state` already do. Running `ppm list --json` would cost a yq per package and fail
when ppm is broken, which is when a report is wanted.

### Where it goes: `ANFS_REPORT_TO`

In `anfs.env`, documented and commented (`#ANFS_REPORT_TO=github`); a machine sets it in
`anfs.local.env` (`anfs config set ANFS_REPORT_TO=~/notes/bugs.md`). The value is `github` or a
path: a leading `~/` expanded (`anfs_conf_expand`; the loader does it already for the files, this
covers the environment), then it must be absolute (else 1, `ANFS_REPORT_TO is 'x': github or an
absolute path`).

- **A file**: appended (`>>`): a blank line (and a newline first when the file does not end with
  one), `## r-<YYYYMMDD>-<n> <title>`, a blank line, `Status: open`, a blank line, the body. `<n>`
  is one past the highest of that day's ids in the file. The directory is made when missing.
  Prints `r-...: appended to <file>`. Only `close` changes a report afterwards, and only its
  `Status:` line.
- **github**: the owner source's clone (`gitsrc_dir_reply`), its `origin` (`git remote get-url`).
  `git@github.com:o/r(.git)`, `ssh://git@github.com/o/r(.git)`, `https://github.com/o/r(.git)`
  give `o/r`; anything else (another host, a local source, no clone) is 1: `<alias>'s origin
  (<url>) is not on GitHub: set ANFS_REPORT_TO to a file (anfs config set
  ANFS_REPORT_TO=~/bugs.md)`. Then `gh` (`anfs_require gh`, 69), `gh issue list --repo o/r
  --state all --search "<title> in:title" --limit 10 --json number,state,title,url --jq ...`
  (tab-separated rows, so no jq on the host); the duplicates are shown (`possible duplicates on
  o/r:` and one line each), then `anfs_confirm "report files an issue on o/r" "File it on o/r?"`
  (77 without a terminal and `-y`), then `gh issue create --repo o/r --title T --body-file <tmp>`;
  the id `o/r#N` (from the URL) and the URL are printed. A failing `gh` is 1 with its message.

### Following the fix: `close`, `status`, `wait`

- `close <id> --commit SHA [--source core|stack]`: `--commit` is required (7 to 40 hex digits;
  else 64). A file id: that section's `Status:` line becomes `Status: fixed in <source> <sha>
  (<YYYY-MM-DD>)`; nothing else in the file changes (a temp file beside it, then `mv`); `--source`
  defaults to core. A GitHub id: `gh issue comment N --body "Fixed in <source> <sha> (anfs report
  close, <date>)"`, then `gh issue close N`; `--source` defaults to the source whose origin is that
  repo (none: 1, name it). No confirmation: closing loses nothing.
- `status <id>`: a file id reads its section's `Status:` line (the file must be the current
  `ANFS_REPORT_TO`); a GitHub id reads `gh issue view N --json state,comments` (the first line of
  each comment; the last `Fixed in <source> <sha>` names the fix). `open` (exit 1); closed with a
  commit that is an ancestor of HEAD (`git merge-base --is-ancestor`) in the source's live
  checkout (`gitsrc_dir_reply`, repo.sh's path) or, with `ANFS_WORK`, in the job's worktree of the
  source (`anfs_work_job`): `available` (0); otherwise `fixed` (1), the detail saying `anfs src
  update <source>` (or `closed with no commit named`). An id of neither form, one not in the file,
  or a `gh` that cannot answer: 2.
- `wait <id>`: `status` every `--interval` until `available` (0) or `--timeout` (1, with the last
  state); an unknown id (2) at once; a `gh` that cannot answer is a warning, and it asks again.
- `--json`: `{id, status, source, commit, detail}` for `status` and `wait`, `{id, status: fixed,
  source, commit[, file]}` for `close`.

### `anfs report check`

One row per item, `CHECK STATUS DETAIL` (rule 5 of "Lists and --json": not a source's resources,
STATUS second). Exit 0 when every row is `ok`, else 1 (the table is printed either way).

| CHECK | ok when | DETAIL |
| --- | --- | --- |
| `report-to` | the value is `github` or an absolute path | the value and where it is from: `default`, `shipped`, `local`, `env`, and the file (config.sh's FROM; `built in` when no `anfs.env` declares it) |
| `file` (file target) | the file is writable, or does not exist and its nearest existing directory is writable | `writable`, `will be created`, or why not |
| `gh` (github) | `gh` on PATH | its path, or `not on PATH (brew install gh)` |
| `gh-auth` (github) | `gh auth status` exits 0 (the only network call; stubbed in tests) | `logged in`, or its first line of output |
| `origin core`, `origin stack` (github) | the source's origin is a GitHub URL | `o/r`, or the URL and why not |
| `skill` | `anfs-bug-report` is installed (skills CLI, global scope, read-only: `anfs_skill_load -g`) for every detected agent | the agents it is installed for; `partial`/`missing` with `psm install core/anfs-bug-report`; `npx not found` |

### `cli.sh`: a group's default sub-command

`cli_group <name> <summary> [--default <sub>]` records `CLI_GROUP_DEFAULT[<name>]`. In
`_cli_find`, a group whose next word is an option (`-*`, but not `-h`/`--help`) resolves to `<name>
<sub>` and takes one word. Completion: `anfs report -<TAB>` offers the default's options.
`cli_help_command` for such a group says `Options with no command go to <name> <sub>`. Nothing else
changes; no existing group declares one.

### The skill: `skills/anfs-bug-report/SKILL.md`

Short, in the style of `pcm-containers`. The description says when it triggers, in one line: an
anfs tool (anfs, ppm, pcm, pmm, psm, wsm, or a stack tool) does something wrong. The body:

1. Reproduce it minimally: the smallest command line that shows it, in a temp `HOME` when it can
   be; note what was expected, what happened (exact output, exit code), the steps.
2. Run `anfs report --package <source/name> --title ... --expected ... --actual ... --steps ...`
   (or `--body-file -`). Never choose where it goes, never call `gh`, never write the issue
   anywhere yourself.
3. Exit 77 means it wants a person's yes (the GitHub target): show them what it printed (where
   it goes, any duplicates) and rerun with `-y` only when they say so. A duplicate that is the
   same bug: tell the person, do not file.
4. Keep the id it prints and name it in later messages, plans and commits about the bug.
5. Run `anfs report wait <id>` as a background command, so the harness says when it exits; carry
   on, or delegate, meanwhile. Whoever merges a fix runs `anfs report close <id> --commit <sha>`.
6. **Check mode**: invoked with no bug in hand (`/anfs-bug-report check`, or just to test it):
   run `anfs report check` and relay its rows as they are.

## Files

- `packages/anfs/home/.local/lib/anfs/report.sh` (new): the declarations (`anfs_report_cli`),
  the three commands, the diagnostics, the routing
- `packages/anfs/home/.local/bin/anfs`: sources `report.sh` and `skill.sh`, calls
  `anfs_report_cli`; the header's command list
- `packages/anfs/home/.local/lib/anfs/cli.sh`: `--default`
- `packages/anfs/home/.config/anfs/anfs.env`: `#ANFS_REPORT_TO=github` with its comment
- `skills/anfs-bug-report/SKILL.md` (new)
- `packages/anfs/README.md`, `AGENTS.md` (a section "Reporting a bug: anfs report"; rule 5's list;
  Lib Structure; Testing), `README.md` (one line)
- `packages/anfs/tests/report.bats` (new), `registry.bats` (the default sub-command; `CHECK STATUS
  DETAIL` among the lists that are not)

## Test Spec

`packages/anfs/tests/report.bats`: a temp `HOME` with core's `lib/anfs` and `anfs` linked in,
`anfs.env` linked, two sources (`core`, `stack`) as local git repos with an `origin` set to a URL
(nothing fetched), a ppm tracker or two, `gh`, `podman` and `npx` stubs on PATH that log their
arguments and answer from files the test writes. `ANFS_WORK` unset; no network.

- `diag`: versions (bash, git, the stub podman), platform, each source with its commit and
  status (one made dirty shows `dirty`), each tracker's version; the log section only when the log
  has a failure, and then the start line of a failed run and an `error:` line; `ANFS_LOG=off`: no
  section. `diag --json` is one object.
- `new` with a file target (`ANFS_REPORT_TO` in `anfs.local.env`, with `~/`): appended as a dated
  `##` section, twice gives two sections, earlier content byte-for-byte kept; no `gh` call;
  `--json` has `to: file`. A relative path: 1.
- github, no duplicate: `gh issue list` searched on the owner repo, `gh issue create` called with
  the title and a body file holding the sections and the diagnostics; the URL printed.
- github, a duplicate found: listed before the confirmation; without `-y` and no terminal: 77 and
  no `create`; with `-y`: filed.
- `--package stack/x` goes to stack's repo; a bare installed name resolves through its tracker;
  an unknown one is 1.
- A non-GitHub origin: 1 with the message naming `ANFS_REPORT_TO`; no `gh` call.
- `-n`: the report and where it would go, no `gh` call (the stub's log is empty), nothing appended.
- `--json` on github: `to`, `repo`, `url`, `duplicates` as an array.
- Usage: no title, or nothing to say: 64. `--body-file -` reads stdin.
- `anfs report --title ...` (no sub-command) is `report new`; `anfs report` alone is the help.
- File ids: two reports the same day are `-1` and `-2`. `close` changes exactly one line (a diff
  of two lines), the other report stays `open`; `status` is `open` (1), then `fixed` with the
  commit on a branch (1), then `available` once the branch is merged into the checkout (0);
  `--json`. An unknown id, a malformed one: 2; `close` without `--commit` or with a non-sha: 64.
- GitHub ids: `close` comments `Fixed in stack <sha>` and closes, the source found by the origin;
  a repo no source has: 1. `status`: `OPEN` is open; `CLOSED` with a commit not in the checkout is
  fixed, with one that is (a real temp repo) available; `gh issue view` failing: 2.
- `wait`: `--timeout 0` on an open report is 1; a report whose fix is in the checkout is 0 at
  once; the fix merged into the checkout while it waits is 0; an unknown id is 2.
- Not tested: `ANFS_WORK`'s job worktree counting for `status` (a test may not set `ANFS_WORK`;
  the person's rule for this job).
- `check`: file target writable (0), not writable (1); github with gh missing (1, `gh` fail),
  `gh auth status` failing (1), a non-GitHub stack origin (1), all ok (0); the skill installed for
  every detected agent (ok), for some (fail, partial), missing (fail); `--json`.
- `registry.bats`: a group's default sub-command takes options with no sub-command, `-h` still
  shows the group, completion offers the default's options.
- `settings.bats` (unchanged) holds `anfs.env` to the new `ANFS_REPORT_TO` line.

## Verification

Every command through the job, `</dev/null`. pmm's and pcm's suites stay off; nothing is installed
on this host; no `gh` call reaches GitHub (the real `gh` is never run).

- [ ] While working: `anfs work run anfs-report core -- bats packages/anfs/tests/report.bats
      </dev/null`, `registry.bats`, `settings.bats`, `bash.bats`
- [ ] Once, at the end: `anfs work run anfs-report core -- packages/anfs/tests/all </dev/null`
- By hand, after merge (in `log.md`): `anfs src update`; `ANFS_REPORT_TO=~/spaces/anfs-dev/docs/BACKLOG.md`
  in `~/.config/anfs/anfs.local.env`; `psm install core/anfs-bug-report`; `anfs report check`;
  a file report, `close` and `status` against BACKLOG.md by hand.
