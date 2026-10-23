# Plan 02c — `podman events` into the bus

Written 2026-10-23, to be built after 02a is merged (02b is not needed). Part of [plan 02](plan.md).
Mergeable alone: after it, containers and pods starting and stopping are events in the host
scope, which is what 03 moves the ui onto.

## Context — read these files first

The job is `anfs work new agents-02c runtime core docs`.

- docs: `AGENT-NATIVE.md` (principle 6: real event sources, polling only as a slow reconcile),
  [plan.md](plan.md), [02a](02a-bus.md) and its log
- core `AGENTS.md`: "pcm: containers" (the podman machine on macOS, names and labels pcm sets),
  "Events"; `containers/ui/lib/anfs/poller.rb` (what the ui refreshes today)
- runtime: `go/internal/{cloudevent,spool}`

## What anfsd does (`go/internal/podman`)

- Runs `podman events --format json --filter type=container --filter type=pod` (no `--since`: it
  starts from now) as a child, reads one JSON object per line.
- **Kept** (container): `create`, `start`, `stop`, `died`, `remove`, `pause`, `unpause`, and
  `health_status` only when the status changes. (pod): `create`, `start`, `stop`, `remove`.
  Everything else (`exec`, `attach`, `cleanup`, `mount`, `sync`, ...) is dropped.
- **Written** through the spool (02a's writer), into the host scope: type `container.<status>` or
  `pod.<status>` (`container.started`, `container.stopped`, `container.died`, `container.removed`,
  `container.created`, `container.health`; `pod.started`, ...), subject the container's (or pod's)
  name, data `id` (12 characters), `image`, `pod`, `exit` (died), `health`, and `pcm` (the
  container's pcm service name, from the labels pcm and compose set) when there is one. Actor
  `podman`.
- **Guards**: the same container, status and second coalesce into one event; at most 20 events
  per second overall, the rest counted and logged once a minute (a restart loop cannot flood
  the bus).
- **Podman away** (no podman, or on macOS the machine stopped): the child exits; anfsd starts it
  again after 5, 10, 30, then every 60 s, one log line per change. When it comes back, one
  `podman.connected` event (host scope), so a subscriber reconciles once by listing; a
  `podman.disconnected` when it goes. No polling of `podman ps`.
- `ANFSD_PODMAN=off` in `anfsd.local.env` turns it off. `anfs bus status` gains `podman`
  (connected, or the last error and the next try).

## Files

- runtime: `go/internal/podman`, its wiring in `cmd/anfsd`, `testdata/podman/` (recorded `podman
  events` lines from this Mac: container and pod lifecycles, a health check, exec noise)
- core: `anfs-bus` (the `podman` row); `anfsd.env` (`ANFSD_PODMAN`); `lib/anfs/events.sh`
  (`container.*`, `pod.*` in the completion types); `AGENTS.md` ("Events": the host scope's
  container events); `packages/anfsd/README.md`
- docs: `02c-log.md`

## Test Spec

- Go, with a fake `podman` script that prints recorded lines (and exits, or hangs, as told): each
  kept status gives one valid spool line with the right type, subject and data; the dropped
  statuses give none; a burst is coalesced and capped; the fake exiting brings backoff and a
  restart with `podman.disconnected` and `podman.connected`; `ANFSD_PODMAN=off` starts nothing;
  idle CPU stays under 0.2 s in 10 s.
- e2e bats: the fake podman, the throwaway nats-server: `container.started` is in the stream on
  `anfs.host.container.started`.
- core bats: the `podman` row of `anfs bus status` (connected, away).
- No test starts a container or the podman machine. Recording the fixture lines is done once by
  hand in the job (`podman events --format json` while `pcm up`/`down` of a test container in the
  job's environment), and saved.

## Verification

- `anfs work run agents-02c runtime -- sh -c 'cd go && go test -race ./...'` and the e2e bats
- `anfs work run agents-02c core -- bats packages/anfsd/tests </dev/null`; core's `tests/all` once

## By hand, for the person after merge

1. Runtime: merge, push, tag; core: the new sums (this session for you); `anfs src update core &&
   ppm install core/anfsd`.
2. `anfs bus status`: `podman` connected. `pcm down ui && pcm up ui`, then `anfs events ls --type
   'container.*' --since 5m`: `container.stopped` and `container.started` for it.
