---
---

# Plan 03: `firstmate` launcher

## Context: read these files first

- `sources/anfs/orchestrators-eval.md` §8: herdr findings (server env inheritance, `agent start` racing the shell, clean `session stop`).
- `~/evals/firstmate/docs/herdr-backend.md`: backend selection, `HERDR_ENV`, workspace placement.
- `~/evals/firstmate/docs/configuration.md` (`FM_HOME`) and `docs/fleet-ledger.md`.
- `~/evals/firstmate/bin/fm-inbox.sh`: `note [--request-id] [--json] -`.
- `~/evals/firstmate/bin/fm-claude-trust.sh`: `--lab-home`.
- `herdr --skill` and `herdr agent`: `agent start|prompt|wait|read|list`, `workspace create --cwd --label --env`.

## Overview

`home/.local/bin/firstmate` is a thin bash wrapper: one herdr session called `firstmate`, one
primary agent called `firstmate` in a `captain` workspace. It must not reimplement Firstmate;
every subcommand is a few lines over herdr or a Firstmate `bin/` script.

## Implementation

**Environment:**
- `FM_ROOT=$XDG_DATA_HOME/firstmate/repo`
- `FM_HOME=${FM_HOME:-$XDG_DATA_HOME/firstmate/home}`
- `FM_SESSION=${FM_SESSION:-firstmate}`

All herdr calls go through `hr() { herdr --session "$FM_SESSION" "$@"; }`. That function form
matters because zsh doesn't word-split a variable holding a command.

| Command | Behaviour |
|---|---|
| `up [--headless]` | 1. Seed `FM_HOME/{config,data,state,projects}`, write `config/backend` = `herdr`, touch `config/fleet-ledger`. 2. If the session isn't running, start it with an allowlisted env: `env -i PATH HOME USER LOGNAME SHELL TERM LANG TMPDIR FM_HOME [auth/XDG vars] herdr --session firstmate server` (backgrounded, log in `FM_HOME/state/herdr-server.log`), then wait until `herdr session list` shows it running. 3. If no `firstmate` agent exists: `workspace create --cwd "$FM_ROOT" --label captain --env FM_HOME=…`, wait for the shell prompt (`pane wait-output` or retry `agent start` until it succeeds, max 30 s), then `agent start firstmate --kind claude -- --dangerously-skip-permissions`. 4. Run `fm-claude-trust.sh --lab-home "$FM_HOME"` before the start. 5. Without `--headless`, `exec herdr --session firstmate`. |
| `attach` | `exec herdr --session "$FM_SESSION"` |
| `status` | `hr agent list` as one-line rows, plus `fm-crew-state.sh` for each `state/*.meta` |
| `prompt <text…>` | `hr agent prompt firstmate "$*"` |
| `note <file\|->` | `fm-inbox.sh note --request-id <sha of content> -`, then `prompt "Captain note waiting in your inbox."` so an idle primary wakes. Prints the request id. |
| `project add <path\|wsm-name>` | Resolve a wsm name with `wsm path`, `git clone` into `FM_HOME/projects/<name>`, and append `- <name> [local-only] - <description> (added <YYYY-MM-DD>)` to `data/projects.md`. That is the registry line format the first mate wrote in the spike; check it against `.agents/skills/project-management/`. |
| `doctor` | `FM_HOME=… fm-bootstrap.sh`; `herdr status --json` protocol ≥ 16; Claude auth present (`CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY` set, or `claude auth status` ok). Non-zero exit if anything is missing. |
| `down` | `herdr session stop "$FM_SESSION"`, then confirm no `fm-watch`/claude processes with that `FM_HOME` remain. |

Not included: `ledger`, `remote`. Add them only when a later unit needs them.

## Test Spec

On the macOS host (your own default herdr session must be left untouched):

- `firstmate doctor` exits 0.
- `firstmate up --headless`; `firstmate status` shows `firstmate` idle.
- `firstmate project add <throwaway repo>`.
- `firstmate prompt "dispatch a scout on <project>: list the files"` → within 3 min, `state/*.status` shows `done`.
- `firstmate note -` with a short spec → the primary acts on it.
- `firstmate down` → `pgrep -f "$FM_HOME"` is empty and `herdr session list` shows `firstmate` stopped.
- The default herdr session is still running throughout.

## Verification

- `ps eww` of the herdr server shows no `CLAUDE_CODE_*` variables.
- Running `up` twice is idempotent: no second workspace or agent.
