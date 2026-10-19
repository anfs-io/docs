---
plan: toolkit-api/04-anfs-ui-client
job: ui-client
date: 2026-10-19
---

# Log: toolkit-api/04-anfs-ui-client

## Built

- `containers/ui/lib/anfs/rpc.rb`: `Anfs::RPC`, the toolkit API client over `tcp://host:port` or
  `unix:///path`. Each request gets its own connection. The `{"token":...}` line goes first only
  when `ANFS_RPC_TOKEN` is non-empty. `call(request)` returns the answer, and with a block it
  yields each streamed `out` line. `watch` yields events and raises when the connection drops.
  `RPC.from_env` is the one switch: it returns nil without `ANFS_RPC_URL`, which means host mode.
- `CLI#run` sends `{"run":[tool,*args]}` and takes rows from `data`. The API adds `--json` and
  `ANFS_LOG=off` itself. `error` and `exit` go where stderr and the exit code went. `Result`
  gains `command` and `needs_terminal?`.
- `Jobs` sends `{"run":..., "stream":true, "yes":true}` with the kind's own `-y` taken out, and
  appends each `out` line as it arrives. For exit 77 the job status is `needs a terminal` (not
  failed), plus the line "Run this in a terminal: <command>". A write refused under a read grant
  gets the API's error plus a line saying the dashboard may only read.
- `lib/anfs/events.rb`: `Watch` (a `{"watch":true}` connection that reconnects with a
  1/2/5/10/30s backoff, reset by an event) and `LogFollower` (the old anfs.log tail, moved here).
  `Anfs.events` picks one of them, and Poller takes `events:` (it was `log:`). A watch `start`
  for a non-read command nudges that tool, and its `end` (matched by id) nudges it again. This
  replaces the pid wait the log mode uses.
- Views: a panel with a 77 shows "Run this in a terminal:" and the command. A row and the
  activity list show the last output line of a job that needs a terminal. `needs a terminal` and
  `outdated` use the warn tone.
- `stacks/system/stack.yml`: `containers: - core/ui: { status: running }`. `system.bats` checks
  that the manifest names core/ui and that install runs `pcm install core/ui` and then
  `pcm up core/ui`. AGENTS.md: the "not yet core/ui" notes are gone, and anfs-ui's two modes are
  described. README.md and compose.yml are updated to match.

## Commits (core, branch ui-client)

- `3ceaf44` anfs-ui: the toolkit API client (ANFS_RPC_URL): lists, streamed commands with yes, watch events
- `da1a734` system: starts the dashboard, core/ui running, now that it reaches the tools through the API
- `b0d12cc` anfs-ui: packages read ppm list's status and version, as ppm prints them now

The hook did not bump `containers/ui/container.yml` ("new at 0.1.0, not bumping until it is pushed").

## Deviations

- **No half-close.** The client keeps its write side open until it has the answer, then closes.
  With `close_write`, socat dropped the connection 0.5s after the EOF, before a slow `ppm list`
  could answer. A relay is not the listener, but not half-closing works with both.
- **Packages kind fixed (not in the plan).** It read an `installed` key that `ppm list --json` no
  longer prints, so every package showed as available and offered Install. It now reads
  `status` (installed|outdated|null) and `version`. This is a separate commit.
- The kind's `-y` is left out in API mode because `yes: true` adds it. Host mode is unchanged.

## Tests

- `bundle exec rspec`: 56 examples, 0 failures (40 before, 16 new in `spec/anfs/rpc_spec.rb`,
  against `spec/fake_api.rb`: a TCPServer or a UNIXServer on a temp path). `rubocop`: 28 files,
  no offenses. spec_helper unsets `ANFS_RPC_URL` and `ANFS_RPC_TOKEN`, so the specs run in host
  mode and never reach a real listener.
- `anfs work run ui-client core -- packages/anfs/tests/all`: 374 passed, 0 failed, 26 files
  (pmm and pcm off).

## By hand (Verification 2)

The listener was `ANFS_LOG=off socat TCP-LISTEN:<random>,bind=127.0.0.1,fork,reuseaddr
EXEC:"<worktree>/packages/anfs/home/.local/bin/anfs rpc --grant read"`, with no token. The app
was `ANFS_RPC_URL=tcp://127.0.0.1:<port> bundle exec puma -b tcp://127.0.0.1:<random>`. Only
reads were made, and both pids were killed afterwards.

- `nc` with `{"run":["ppm","list"]}` returned `{"exit":0,"data":[{"package":"anfs-dev/dev","status":"outdated",...`
- `nc` with `{"run":["ppm","install","x"]}` returned `{"error":"refused: a write request (install) under --grant read"}`
- `/packages`: 67 rows (30 installed, 11 outdated, 26 available), for example `rws/git 0.1.0`
  installed. `/sources`: 9 rows.
- `/containers` showed the API's error in the panel:
  `pcm list --json failed (exit 127)` / `~/.local/lib/pcm/sources.sh: line 72: manifest_fields:
  command not found`. This is a fault in the installed pcm on this Mac, not in the app.
- The puma log had `anfs-ui: watch: logging is off (ANFS_LOG=off)` once, then backoff retries.
  This was expected because the listener ran with `ANFS_LOG=off`.

## Not verified

- A real `watch` event stream from the listener (the listener ran with logging off). This is
  covered by specs only.
- Streamed commands and 77 against the real `anfs rpc` (no writes against the real toolkit).
  This is covered by the fake API only.
- The unix socket against a real listener (Linux), and the token against the real macOS
  listener.
- The container itself (no podman).

## The person's steps

1. `ppm install core/anfs` loads the listener (launchd on macOS, the user sockets on Linux).
2. `pcm up ui` (or `anfs install system`, which now starts it).
3. Open http://localhost:9292. Packages should list this Mac's packages, and starting a
   container from Containers should work, with its output in Activity.
