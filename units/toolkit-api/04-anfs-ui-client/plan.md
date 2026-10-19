---
---

# Plan 04 — anfs-ui talks to the toolkit API (reduced)

Needs plans 01 and 02. Reduced from the outline: the code stays in core (`containers/ui/`), no
CI and no published image yet.

## Context — read these files first

- `units/toolkit-api.md` ("Settled 2026-10-19"), plans 01 and 02 and their logs
- core AGENTS.md, "The toolkit API" (the requests, the token line, `--grant`, exit 77) and pcm's
  `toolkit:` key (what a container gets: `ANFS_RPC_URL`, `ANFS_RPC_TOKEN` or the socket mount)
- core `containers/ui/`: `lib/anfs/cli.rb` (runs a tool, parses JSON), `jobs.rb` (commands that
  change something, run in the background), `poller.rb` (follows `anfs.log`), `kinds.rb`, the
  views that show errors, `spec/`
- core `stacks/system/stack.yml` and `packages/anfs/tests/system.bats`

## Implementation

1. **One client, two transports.** `lib/anfs/rpc.rb`: a connection to `ANFS_RPC_URL` (`tcp://host:port`
   or `unix:///path`), sends the token line when `ANFS_RPC_TOKEN` is set, then JSON-line requests.
   `CLI#run` sends `{"run": [...]}` and returns `data`; `Jobs` sends `{"run": [...], "stream": true,
   "yes": true}` and forwards each `out` line as it does now.
2. **Host mode stays.** With no `ANFS_RPC_URL`, today's behaviour: the tools executed from
   `ANFS_BIN` (for `bundle exec puma` on the host while developing). One switch, in one place.
3. **Events from `watch`.** The poller's log follower becomes a long-lived `{"watch": true}`
   connection in API mode (reconnect with backoff); a `start`/`end` event for a tool refreshes
   that tool's kinds, as a log line does now. Host mode keeps following the file.
4. **Errors.** The API's `error` is shown where a tool's stderr is today; `needs: terminal`
   (exit 77) shows "Run this in a terminal: `<command>`" instead of a failure; a refused write
   (read grant) says so. The "tools are not reachable" state stays for no URL and no tools.
5. **`system` starts it.** `core/stacks/system/stack.yml`: `containers: - core/ui: { status: running }`.
   `system.bats` and AGENTS.md updated (the "not yet in system" notes go).

## Test Spec

`containers/ui/spec/`: a fake API server in the spec (a `TCPServer` and a `UNIXServer` on temp
paths speaking the JSON lines): `CLI#run` both transports, the token line first, `data` parsed;
`Jobs` streams lines and sends `yes`; `watch` events trigger a refresh and a dropped connection
reconnects; 77 renders the terminal hint; host mode unchanged when `ANFS_RPC_URL` is unset.
`bundle exec rspec` and `rubocop` on the host.

## Verification

- [ ] `bundle exec rspec`, `rubocop`; core `packages/anfs/tests/all`
- [ ] By hand on the host: `ANFS_RPC_URL=tcp://127.0.0.1:<port>` against a throwaway
      `socat ... EXEC:"anfs rpc --token"` on a random loopback port, the dashboard shows real lists
- [ ] The person: `ppm install core/anfs` (loads the listener), `pcm up ui`, the dashboard in a
      browser shows this Mac's packages, and starting a container from it works
