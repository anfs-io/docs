---
plan: toolkit-api/06-watch-loop
job: ui-rpc-loop
date: 2026-10-23
---

# Log: toolkit-api/06-watch-loop

## What

A feedback loop between anfs-ui's poller and the toolkit API burned the CPU. Fixed at both
ends, plus a guard so no future event can spin the poller.

## Why

`Poller#event` refreshed on any watch `start` whose args didn't match `READ_ONLY`, and
`nudge("anfs")` wakes every kind. Every refresh is a new API connection (`anfs rpc --token`), and
the server reported that connection's own start as an `anfs` event with args `rpc --token`. That
isn't read-only, so it nudged every kind again, forever. `log_line` (host mode, anfs.log) had the
same hole. The reads a connection runs log nothing (`ANFS_LOG=off`), so the connection line was
the whole loop.

Measured on the Mac, 2026-10-23:

- About 440 `anfs rpc --token` start lines a minute in `~/.local/state/anfs/anfs.log`, about 7 a
  second.
- Load average about 20 on 10 cores, 60% system time: every connection forks bash pipelines
  (`anfs --json list`, `ppm/psm/pmm --json list`, ...).
- Stopping the ui container took the CPU from 0–5% idle to 95% idle at once.

## Built

- Server (`packages/anfs` `bin/anfs` `_rpc_events`, core/anfs 0.2.17): a start line for
  `anfs [options] rpc ...` is not an event, and neither is the end with its id. The line stays in
  anfs.log (it is how the measurement above is made). What a connection runs for a client still
  logs and is still an event, so watchers still hear about real writes.
- Client (`containers/ui/lib/anfs/poller.rb`, anfs-ui 0.1.4): `change?(tool, args)` is false for
  `READ_ONLY` and for an `anfs` API connection (`CONNECTION`, with or without the log line's id),
  for both watch events and log lines. That covers an older API and host mode. The poller's own
  requests can't be told apart from another client's (the log line has no client id). Since its
  reads don't log and its connections are now ignored, they cause nothing.
- Guard: nudges coalesce per kind. The first nudge after a quiet spell refreshes at once. Any
  nudge within `COALESCE` (2 s) of the last refresh waits out the rest of the window, and every
  nudge in between rides along: at most one refresh per kind per 2 s, whatever arrives.

## Tests

- `containers/ui/spec/anfs/poller_spec.rb`: 200 rounds of connection start/end, read-only events
  and `rpc --token` log lines cause only the first poll. A `ppm install` start refreshes packages.
  50 nudges inside a window after an immediate one give exactly one more refresh, at the window's
  end. `log_line` and `event` ignore connections (`rpc --token`, `-d rpc --grant write`) but not
  `anfs src install`.
- `packages/anfs/tests/rpc.bats`: with a watch open, a read connection and a `-d rpc --grant write`
  connection running `ppm install` are both in anfs.log. The watcher gets no `anfs` event, only
  the install's start and end. This fails without the server change.

## By hand, after merge

```sh
pcm up ui
# a minute later: should be well under ~40 (was ~440)
grep -c "$(date +%Y-%m-%dT%H:%M).*rpc --token" ~/.local/state/anfs/anfs.log
top -l 1 | grep 'CPU usage'   # idle back near 95%
```
