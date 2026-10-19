---
---

# Plan 01 — `anfs rpc`: the toolkit's API as a filter

## Context — read these files first

- `units/toolkit-api.md` — the shape, the protocol, the decisions (questions 2–4 settled
  2026-10-19, below)
- core `packages/anfs/home/.local/bin/anfs` — how a command is declared (`cli_cmd`), the tools
  as `anfs <resource>`, `ANFS_TOOLS` and `ANFS_BIN_DIR`
- core `packages/anfs/home/.local/lib/anfs/cli.sh` — `anfs_confirm` (no terminal and no `-y`:
  exit 1 today), the `-y`/`--json` options, the EXIT trap
- core `packages/anfs/home/.local/lib/anfs/log.sh` — `anfs_log_start`, the `start:` line, `ANFS_LOG`
- core `packages/ppm/home/.local/lib/ppm/platform.sh` — `_system_sudo` (`sudo -n`, else a prompt)
- core `containers/ui/lib/anfs/{cli,jobs,poller}.rb` — what anfs-ui needs from the API

## Settled

- **What a caller may do**: a `run` is `read` or `write`. Read: a tool's `list|ls`, `show`,
  `status`, `path`, `validate`, `--version`, and `version`/`watch`. Everything else is write.
  `anfs rpc` is started with the caller's grant (`--grant read|write`, default `read`); a write
  request under `read` is refused. The grant comes from the container's `container.yml`
  (`toolkit: read|write`, plan 03); plan 02's socket passes it.
- **Confirmations**: a request may carry `"yes": true`, which becomes `-y` on the command line.
  Without it, a command that would ask fails as it does now.
- **Needs a terminal**: one exit status, **77**, for "this needs a person at a terminal" — a
  confirm with no terminal and no `-y`, and sudo that cannot run unattended. The API passes it
  through as `{"exit": 77, "needs": "terminal", "command": [...]}` so a UI can say "run this in a
  terminal".

## Implementation

### 1. `anfs rpc [--grant read|write]`

Reads JSON lines on stdin, writes JSON lines on stdout, until EOF. bash and `yq`/`jq` only: no
new interpreter. Requests:

| Request | Answer |
| --- | --- |
| `{"run": ["pmm","list"]}` | `{"exit":0,"data":<the tool's --json output, parsed>}`; a tool's non-JSON output as `{"exit":0,"out":"..."}` |
| `{"run": [...], "stream": true}` | `{"out":"<line>"}` per line of stdout and stderr as it comes, then `{"exit":N}` |
| `{"run": [...], "yes": true}` | as above, with `-y` |
| `{"watch": true}` | `{"event":"start",...}` / `{"event":"end",...}` per tool invocation, following `anfs.log`, until stdin closes |
| `{"version": true}` | `{"anfs":"<version>","tools":{"ppm":"0.2.5",...}}` (the installed ones) |
| anything else | `{"error":"..."}`, the connection stays open |

- **Only the toolkit's tools**: `run[0]` must be one of `anfs` and `ANFS_TOOLS` (and `pvm`),
  resolved in `$ANFS_BIN_DIR`, never `PATH`. The rest is passed as an argument array, never
  through a shell or `eval`. `anfs rpc` itself and `anfs implode` are refused.
- `--json` is added for read requests; each run has stdin `/dev/null`, `ANFS_LOG=off` for reads
  (a poll is not an event), a timeout (`ANFS_RPC_TIMEOUT`, default 60 s; a stream has none).
- A failed run: `{"exit":N,"error":"<the tool's last stderr line>"}`.

### 2. The `end:` log line

`cli.sh`'s EXIT trap writes `end: <id> <exit>` to `anfs.log`, matching the `start:` line's id
(add an id to `start:` if it has none: pid plus a counter is enough). `watch` turns the pair
into events.

### 3. Exit 77

- `anfs_confirm` with no terminal and no `-y`: exit 77 (was 1), message unchanged.
- `_system_sudo` and anfs's own sudo check (`bin/anfs`, the `sudo -n true || sudo -v` path):
  with no terminal and no `SUDO_ASKPASS`, exit 77 instead of prompting or failing with 1.
- Core `AGENTS.md`'s exit codes: 64 usage, 69 missing, 75 busy, **77 needs a terminal**.

### 4. Documentation

Core `AGENTS.md`: a "The toolkit API" section (the requests, the grant, 77, what is refused).

## Test Spec

`packages/anfs/tests/rpc.bats`, requests piped in, tools stubbed in a temp `ANFS_BIN_DIR`:

- each request kind; `data` is parsed JSON; a non-JSON answer is `out`
- refused: a tool not in the toolkit, a path (`../x`), `implode`, `rpc`; a write under
  `--grant read`; a malformed line answers `error` and the next line still works
- no shell: an argument with `$(...)`, `;` and spaces reaches the tool unchanged
- `yes: true` adds `-y`; a stub that calls `anfs_confirm` without it exits 77 and the answer says
  `needs: terminal`
- `stream` yields lines before the exit; the timeout kills a stub that hangs
- `watch` yields a start and an end event for a stub run in another process
- `log.bats`: the `end:` line and its id; `cli.sh`'s confirm exits 77

## Verification

- [ ] `packages/anfs/tests/all`
- [ ] `echo '{"run":["ppm","list"]}' | anfs rpc` on this Mac (read-only) prints one JSON line
