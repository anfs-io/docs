---
objective: The toolkit has one API on a host, so everything built on it ships as a container — anfs-ui first, part of a default install
status: plans 01-02 written
---

Draft. The plans are written once the open questions are settled and the macOS transport has
been tried by hand.

anfs-ui (`~/spaces/user/dev/apps/anfs-ui`, plan in `~/spaces/user/dev/products/anfs-ui.md`) is a
dashboard over the tools: it runs their list commands with `--json`, follows `anfs.log` to know
when something changed, and runs `ppm install`, `pcm up`, `pmm up` and the like from its rows. It
does all of that by executing the tools on the host it runs on.

It should ship as a container, as every app built on the toolkit should: no interpreter on the
host, the same artifact wherever it is deployed. A container cannot reach the tools, their state
or the sources, and on macOS it is inside podman's VM. So the toolkit needs what docker has in
its daemon socket: one endpoint on the host that containers talk to.

Depends on: nothing. Independent of `machine-images`; its request examples use that unit's
command names.

## The shape

```
 container (anfs-ui, any app)            host
 ───────────────────────────             ─────────────────────────────────────────────
  one JSON line  ───────────────▶  socket (launchd / systemd --user)
                                     └─ starts `anfs rpc` for the connection
                                          └─ runs  <tool> --json <args...>
  JSON lines     ◀───────────────  result, or a stream of output and events
```

- **`anfs rpc`** is a command of `anfs`: bash, no new interpreter, no build. It reads one request
  from stdin and writes its answer to stdout. It is not a server.
- **Socket activation** does the listening. launchd on macOS and a systemd user unit on Linux
  each own a socket and start `anfs rpc` per connection with the connection as its stdin and
  stdout. Nothing runs between requests. `core/anfs` stows the two unit files and its
  `post_install` loads them.
- **It is the toolkit's, not the app's.** One endpoint per host, whoever the callers are:
  anfs-ui, a dashboard watching several hosts, the deploy tool, an agent.

## The protocol (to be fixed in plan 01)

One JSON object per line, each way.

| Request | Answer |
| --- | --- |
| `{"run": ["pmm", "list"]}` | `{"exit": 0, "data": [...]}`: the tool's `--json` output, parsed. On failure `{"exit": 1, "error": "pmm: ..."}` |
| `{"run": ["pmm", "image", "build", "debian-13"], "stream": true}` | `{"out": "..."}` per line of output as it happens, then `{"exit": 0}`. For what takes minutes |
| `{"watch": true}` | `{"event": "start", "tool": "pcm", "args": [...], "id": ...}` and `{"event": "end", "id": ..., "exit": 0}` for every tool invocation on the host, until the caller hangs up |
| `{"version": true}` | the toolkit's version and the tools installed |

- **Only the toolkit's tools run**: the first word is checked against the installed tools, and
  the rest is an argument array handed to it as it is. No shell string is ever evaluated.
- **`watch` is the log.** Every invocation already writes a `start:` line to `anfs.log`. This unit
  adds the matching `end:` line with the exit status (`cli.sh`, on exit), because a caller in a
  container cannot watch a pid the way anfs-ui's log follower does now. `watch` follows the file.
- **The API's own read calls stay out of the log** (`ANFS_LOG=off` for a `run` that is a list),
  as anfs-ui does today, or every poll would be an event.

## Decisions

- **Apps are containers; the toolkit has one API.** The alternative, each app as a host service,
  needs an interpreter on the host per app and a deployment per app. That is the edge case this
  avoids, at the cost of one small host service that is part of `core/anfs`.
- **bash and socket activation, not a daemon in another language.** The tools already answer in
  JSON, so there is nothing to translate; a compiled agent would add a toolchain and a release
  per architecture to a project that has neither.
- **A `Containerfile` lives with the app's code.** The app's repo builds and publishes the image
  (`ghcr.io/anfs-io/anfs-ui:<version>`); a source ships the service that pulls it
  (`containers/anfs-ui/compose.yml`). A default install builds nothing.
- **What anfs-ui found missing is fixed in the tools, not papered over in the API.** Its notes
  list: `--json` works for lists only (`show`, `validate`, `status` print text), and errors are
  never JSON. The API reports an error as the tool's stderr line for now; making `show --json`
  real is each tool's own work and is listed below.

## Settled 2026-10-19

- **Who may call (question 2)**: a container asks in its `container.yml`, `toolkit: read|write`;
  pcm gives it the socket (Linux) or the address and that grant's token (macOS). Nothing for a
  container that does not ask.
- **What a caller may do (question 3)**: read by default (lists, `show`, `status`, `path`,
  `validate`, `version`, `watch`); everything else needs `write`. `core/ui` declares `write`.
- **Commands that ask (question 4)**: a request may carry `"yes": true` (`-y`); a command that
  needs a person at a terminal (a confirm without `-y`, sudo that cannot run unattended) exits
  **77**, and the API reports it as `needs: terminal`.
- The app's code lives in core, `containers/ui/`, for now (not its own repo): the `dev` target
  builds the gems and the code is mounted from the source; `release` copies it in.

Plans: `toolkit-api/01-rpc`, `02-socket`; then anfs-ui's client (plan 04, reduced).

## Open Questions

1. ~~Does a container on macOS reach a socket on the host's loopback?~~ **Yes** (tested
   2026-10-13, podman 6.1.1, applehv). A listener bound to `127.0.0.1` only is reached from a
   container at `host.containers.internal:<port>`, and refused on the Mac's LAN address. The
   listener sees the caller as `127.0.0.1`, so it cannot tell a container from any other local
   process: the token in question 2 is needed. Transport: TCP on loopback on macOS, a unix
   socket on Linux.
2. **Who may call it?** On Linux, a unix socket with the user's permissions, mounted into the
   container. On macOS, TCP on loopback is open to every local process, so a token: generated at
   install, kept in `$ANFS_STATE_HOME`, given to a container by pcm. How pcm hands it over
   (an `x-pcm` key that says "this service uses the toolkit's API") is the convention to design.
3. **What may a caller do?** Everything the user can, or read-only unless a service is granted
   more (anfs-ui has `ANFS_UI_REMOTE_ACTIONS` for the same reason). A default install that
   listens for commands deserves a conservative default.
4. **Commands that ask.** `ppm install` can need sudo, and several commands confirm without
   `-y`. With no terminal they fail. A UI needs to tell "needs a terminal" from any other
   failure; check whether the tools' exit codes already distinguish it, and give it one if not.
5. **More than one host.** A dashboard over several machines, or the deploy tool reaching a
   remote toolkit, is the same protocol over ssh (`ssh host anfs rpc`), with no listener at all.
   Note it; build nothing for it here.
6. **Is this the surface agents use?** An MCP server over the same requests is a thin layer.
   Out of scope here; the protocol should not make it harder.

## Out of Scope

- Authentication beyond one host's own user (no accounts, no TLS: nothing leaves loopback).
- `show --json` and JSON errors in every tool. Listed as a follow-up unit; this one works with
  what the tools print today.
- Rewriting anfs-ui's views or polling model beyond swapping how it reaches the tools.

## Plans (outline)

1. `01-rpc` — `anfs rpc` (`run`, `stream`, `watch`, `version`), the `end:` log line, bats tests
   that pipe requests into it. No socket yet: it is a filter, and tested as one.
2. `02-socket` — the launchd agent and the systemd user socket in `core/anfs`, loaded by
   `post_install`, removed by `post_remove` and `anfs implode`; the token; the round trips check
   that an install leaves the endpoint answering and an implode leaves nothing.
3. `03-pcm-convention` — how a service declares that it uses the API, and what pcm gives it
   (the address, the socket mount or the token).
4. `04-anfs-ui-client` — in the app's repo: `lib/anfs/cli.rb` calls the API instead of executing
   tools; the log follower becomes a `watch` connection; a `Containerfile`; CI publishes the image.
5. `05-default-install` — `containers/anfs-ui/` in core, pulling the published image; whether
   `ppm install core/anfs` starts it or `anfs onboard` offers it.

## Completion Criteria

- On a fresh machine after `install.sh`, a browser on that machine shows the dashboard, served
  by a container, with no Ruby on the host.
- Starting a machine from the dashboard starts it on the host, and the row changes without a
  reload.
- `echo '{"run":["ppm","list"]}' | anfs rpc` prints one line of JSON, on macOS and Linux.
- Nothing is listening that the user's own account cannot explain: `anfs implode` removes the
  socket units, and the round trips' `$HOME` diff is clean.
- A request naming anything but a toolkit tool is refused, and no request can run a shell string.
