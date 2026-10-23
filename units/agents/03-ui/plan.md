# Plan 03 — anfs-ui on the bus

Written 2026-10-23 against core `5293c0c` (02a, 02b, 02c merged) in the job `agents-03` (core,
docs). Binding: `AGENT-NATIVE.md` (principles 5, 6 and 7 above all), `units/agents.md`, plan 02's
"What 03 gets from 02", toolkit-api/06's log (the CPU loop), and the decisions in the job's brief
(the bus is required; no backward compatibility; the 2 s guard stays; the toolkit API's `watch`
goes). Nothing below reopens them.

## What changes

The ui stops asking "did anything happen?" and listens instead. It reads every `io.anfs.*` event
from JetStream as the `reader` user and updates only the kind an event names. Polling stays as a
slow reconcile.

- **`lib/anfs/bus.rb`** (new): nats-pure (`~> 2.7`, 2.7.0 of 2026-10-07) connects as `reader`.
  It makes sure the durable consumer `ui` exists on stream `ANFS` (filter `anfs.>`, explicit ack,
  deliver `new` when it is first made) and pulls batches of up to 100 with a 30 s wait. Each
  message is a CloudEvent: it is handed to the poller, then acked. The client's own reconnect is
  off; the bus loop owns it. NATS unreachable, a refused login or a missing password file: the
  state goes `unreachable` (the error and the next try), backoff 1, 2, 5, 10, 30, then 60 s, one
  log line per change of state. Every log line and every state string has the password redacted.
- **What an event updates** (`Poller#apply`):

  | Event | Kind | How |
  | --- | --- | --- |
  | `container.started`, `.unpaused` | containers | from the data: the row named `data.pcm` becomes `running` |
  | `container.created`, `.died`, `.stopped`, `.paused` | containers | from the data: `stopped`, unless another container of that service is running (known from earlier events) |
  | `container.removed` | containers | from the data: no container left means `not created` |
  | `container.health`, `pod.*` | none | events panel only (pcm's list shows no health or pods) |
  | `podman.connected`, `.disconnected` | containers | one `pcm list` (02c: "a subscriber reconciles once by listing") |
  | `source.updated` | sources | one `anfs src list` (there is no `anfs src show`; the event names no row state) |
  | `report.*`, `run.*`, anything else | none | events panel only |

  No event re-lists another kind, and none re-lists everything. A read that an event causes goes
  through the per-kind 2 s guard. A re-render that an event causes goes through the same guard:
  the first one at once, then at most one per key every 2 s.
- **The reconcile:** each kind is listed at start, then every 5 minutes, and after a command run
  from its row (as today). Kinds with no events yet (stacks, packages, images, machines, spaces,
  repos, skills) get only this, plus the ui's own commands. The log lists them and why.
- **The watch goes.** The ui's `Watch`, `LogFollower` and `log_line`/`event` code goes, and the
  toolkit API's `{"watch":true}` goes from `anfs rpc` too (`_rpc_watch`, `_rpc_events`). The
  ui was its only client. A `watch` request then gets `unknown request`.
- **The events page** (`/events`, in the nav): the latest 100 events, newest first, with time,
  type, subject, space (`spacepath` or the scope id, `host`) and by (`actor`, `as` role when there
  is one). The bus state is at the top. The overview gets one line with the bus state and a link
  to it.

## Credentials (decided here)

- **The password is a mounted, read-only file.** compose.yml mounts core/nats's
  `${HOME}/.local/state/nats/users/reader` at `/run/secrets/anfs-bus-reader:ro`, allowed by
  `x-pcm.allow_mounts`. The ui reads it at each connect. It is never in the image (the dev image
  has no source, and `.containerignore` does not matter for a mount), never in the environment
  (`podman inspect` shows the environment), and never in a log. Rotating it needs no rebuild.
- **The address is in `.env.schema`**: `ANFS_UI_BUS_URL=nats://host.containers.internal:47222`,
  overridden in `~/.config/pcm/containers/ui/local.env` (config/04). The user is `reader`, fixed.
- On the host (`bundle exec puma`) the defaults are `nats://127.0.0.1:<~/.local/state/nats/port>`
  and `~/.local/state/nats/users/reader`. `ANFS_BUS_URL`, `ANFS_BUS_USER` and
  `ANFS_BUS_PASSWORD_FILE` override them.
- No pcm change: a `bus: read` key in `container.yml` (plan 02's idea) would do the same as these
  two lines for one container. It can come when a second container needs the bus.
- Linux: a rootless container reaching the host's `127.0.0.1:47222` is not settled here (no Linux
  host to check). It is recorded as open, as the toolkit API solved it with a socket.

## Files

- core `containers/ui`: `lib/anfs/{bus,coalescer,event_log}.rb` (new), `poller.rb`, `events.rb`
  (gone), `lib/anfs.rb`, `kinds.rb` (intervals), `config.ru`, `app.rb` (`/events`),
  `views/{events,layout,overview}.rb`, `Gemfile{,.lock}`, `compose.yml`, `.env.schema`,
  `container.yml` (0.2.0), README; specs.
- core `packages/anfs`: `bin/anfs` (watch removed), `tests/rpc.bats`; `AGENTS.md`.
- docs: this plan, `log.md`.

## Tests

- rspec, with doubles for the bus's connection: each container verb gives the right row status,
  and the right row when a name is shadowed. A container event runs no command. `source.updated`
  re-lists only sources, and `podman.connected` only containers. `report.*` and `run.*` re-list
  nothing. 200 events in a burst give one re-render, then one more at the end of the window. With
  the bus down there are retries on the backoff (fake sleeps) and the state says unreachable,
  while the reconcile still lists. The password is in no log line and no state string. Idle CPU
  stays under 0.2 s in 10 s.
- rspec against a throwaway `nats-server` (the pinned binary on PATH, a random port, a temp
  store, the reader's real permissions from `bus.sh`): a published CloudEvent reaches the poller
  and is acked. The consumer is made once and a restart resumes it. The server stopped, then
  started again, gives unreachable, then connected, with no CPU loop. Skipped with a message when
  no `nats-server` is on PATH.
- bats: `rpc.bats` `{"watch":true}` is an unknown request.
- `bundle exec rspec`, `bundle exec rubocop` on the host; `anfs work run agents-03 core --
  packages/anfs/tests/all </dev/null` once.

## By hand, after merge

In log.md: rebuild the dev image (a new gem), restart the ui, the events page while valkey goes
down and up, a scratch report filed and closed, and the API's request rate and `top` checked.
