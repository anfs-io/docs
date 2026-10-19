---
plan: toolkit-api/01-rpc
job: rpc
date: 2026-10-19
---

# Log — toolkit-api/01-rpc

## Built

- `anfs rpc [--grant read|write]` (core `bin/anfs`): JSON lines in, JSON lines out, until EOF.
  `run` (data / out / error), `stream`, `yes`, `watch`, `version`; a bad line answers `error` and
  the next is read. bash + yq only (jq is not a dependency: only mentioned in ppm's platform.sh).
- Only `anfs`, `ANFS_TOOLS` and `pvm` from `$ANFS_BIN_DIR`; argv passed as an array. Refused:
  `anfs rpc`, any `implode`, a write under `--grant read`. Read = `list|ls show status path
  validate --version version`, after skipping leading options, anfs's tool names (`anfs package
  list`) and one group word (`pmm image list`).
- Each run: stdin `/dev/null`, `ANFS_RPC=1`, timeout `ANFS_RPC_TIMEOUT` (60 s; exit 124; kills the
  run and its direct children). Reads get `--json` and `ANFS_LOG=off`.
- `log.sh`: `start: <id> <args>` (id `<pid>-<epoch>`), and `end: <id> <exit>` from an EXIT trap set
  in `anfs_log_start`, chained with the tool's existing EXIT trap (ppm's, wsm's).
- Exit 77: `anfs_confirm` (`CLI_EX_TERMINAL`), ppm `_system_sudo`, anfs `implode --all`'s sudo.
- Core `AGENTS.md`: "The toolkit API (anfs rpc)", exit codes (77), the log's end line.

## Commits (core, branch rpc)

- `6507bf2` log.sh: an id on start: and an end: line
- `663884c` exit 77, needs a terminal
- `967a692` anfs rpc
- `5e33d83` docs: AGENTS.md

## Deviations / decisions the plan left open

- rpc lives in `bin/anfs`, not a `lib/anfs/rpc.sh`: bin/anfs reads the installed lib from
  `$HOME`, so the worktree's anfs could be verified against the real HOME.
- Every tool's `implode` is refused, not only `anfs implode`.
- A `stream` request gets no `--json` (its lines are text) and no timeout, as planned.
- A run with no command word (`["ppm"]`, its help) counts as a read.
- `-y` goes before a `--` in the argv, else last; a command without `-y` answers 64.
- "No terminal" for sudo: `ANFS_RPC` set, or stdin not a terminal and `/dev/tty` cannot be
  opened (not just `! -t 0`, so `curl | bash` installs still prompt on the tty). `anfs_confirm`
  keeps `! -t 0`.
- `watch` follows from the log's size at the request (`tail -c +N -F`), so nothing is lost
  between the request and tail starting; the connection keeps answering other requests.
- The JSON detection for `data`: only when stdout starts with `[` or `{` and yq parses it.
- The `end:` line is lost for a tool that sets its own EXIT trap later, inside a command (pvm's
  lock and tmp traps, pmm's build traps): watch sees a start with no end there.

## Tests

- `packages/anfs/tests/rpc.bats`: 16 tests (stubbed ppm in a temp `ANFS_BIN_DIR`), all pass.
- `log.bats`: 3 new (end line and id, chained trap keeps exit status, confirm exits 77); start-line
  test updated for the id. 16 pass.
- `packages/anfs/tests/all`: 357 passed, 0 failed, 25 files (pmm and pcm off).

Tests changed for 77: `anfs/tests/iso.bats` (prune, remove without -y),
`pcm/tests/remove.bats` ("remove: without a terminal it needs -y") — **pcm's written, not run**.
No pmm test expected 1 from a confirm.

## Verification

`echo '{"run":["ppm","list"]}' | ANFS_LOG=off ~/work/rpc/core/packages/anfs/home/.local/bin/anfs rpc`
on this Mac (real HOME, read-only; `ANFS_LOG=off` so the rpc's own start/end did not touch
`~/.local/state`): one line, 4186 bytes, exit 0.

```
{"exit":0,"data":[{"package":"anfs-dev/dev","status":"outdated","version":"0.1.10"},{"package":"roteoh/tmuxinator","status":null,"version":"0.1.0"},... 67 packages]}
```

## Not verified

- `_system_sudo` on Linux with a real sudo, and `implode --all`'s 77 path (logic only; the
  ppm side has a stub-sudo test).
- `watch` across log rotation (`tail -F` should follow it).
- No socket, launchd or systemd: plan 02.
