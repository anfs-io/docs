---
plan: toolkit-api/02-socket
job: socket
date: 2026-10-19
---

# Log — toolkit-api/02-socket

## Built

- `lib/anfs/listener.sh` (core/anfs): writes, loads, unloads and deletes the listener.
  - macOS `~/Library/LaunchAgents/io.anfs.rpc.plist`: `Sockets` on TCP `127.0.0.1:$ANFS_RPC_PORT`
    (47100), `inetdCompatibility` `Wait` false (one process per connection, the socket on its
    stdin and stdout), runs `anfs rpc --token`, `PATH` = the install's, stderr to
    `$ANFS_STATE_HOME/rpc/rpc.err`. Loaded with `launchctl bootstrap gui/<uid>`, unloaded with
    `bootout gui/<uid>/io.anfs.rpc`.
  - Linux `~/.config/systemd/user/anfs-rpc.socket` (`%t/anfs/rpc.sock`) and
    `anfs-rpc-write.socket` (`rpc-write.sock`): `Accept=yes`, `SocketMode=0600`; their
    `@.service` runs `anfs rpc --grant read|write` with `StandardInput/Output=socket`.
    `daemon-reload`, `enable --now`; removed with `disable --now`, then delete, `daemon-reload`.
  - Tokens: `$ANFS_STATE_HOME/rpc/token.{read,write}`, 64 hex, 0600 in a 0700 dir, made once.
    `rpc/port` records the port for pcm.
- `packages/anfs/install.sh` (new): `post_install` = install the listener, `post_remove` = remove
  it. It sources the lib from the package dir (post_remove runs after the unstow).
- `anfs rpc --token`: the first line must be `{"token":"..."}` (that key only); the matching token
  is the grant. Otherwise `{"error":"unauthorized: ..."}`, exit 1. Without `--token` a first token
  line is ignored (the Linux socket, a pipe).
- `anfs implode`: removes the listener and `$ANFS_STATE_HOME/rpc` first, after the confirm.
- pcm: `container.yml` `toolkit: read|write` (`PCM_MANIFEST_KEYS`, validate, show, list `--json`).
  macOS: override sets `ANFS_RPC_URL=tcp://host.containers.internal:<port>` and
  `ANFS_RPC_TOKEN: "${ANFS_RPC_TOKEN:-}"`; `compose_run` sets the real token only in compose's
  environment. Linux: the grant's socket mounted at `/run/anfs/rpc.sock`,
  `ANFS_RPC_URL=unix:///run/anfs/rpc.sock`. `core/containers/ui/container.yml`: `toolkit: write`.
- Core `AGENTS.md`: "The listener" under the toolkit API; pcm's `toolkit:` key.

## Commits (core, branch socket)

- `8ebb586` anfs: the toolkit API's listener, socket-activated
- `560882e` pcm: container.yml toolkit: read|write; core/ui asks for write
- `6cabdd6` listener: how to enable the sockets by hand without a user manager
- `1b05b73` docs: AGENTS.md

## Deviations / decisions the plan left open

- **One TCP port on macOS, not one per grant**: the token decides the grant (`anfs rpc --token`).
  Two tokens as the plan says, one listener. Linux keeps two sockets, as the plan says.
- **Unit files are written by post_install, not stowed**: launchd can't expand `$HOME` or `PATH`,
  and only one platform's files get written. They hold the install-time `PATH` (the tools need
  bash 5, yq, brew/mise). If that `PATH` changes, run `ppm install core/anfs` again.
- Tokens survive `ppm remove core/anfs` (reinstall runs post_remove then post_install, so it
  can't tell the two apart). `anfs implode` deletes them.
- Unload runs only when this `$HOME`'s unit file is there. The launchd label is per user, not per
  `$HOME`, so a test HOME can never boot out the real agent.
- Loading never fails an install (no GUI session, no user manager): it warns.
- A failed token check exits 1 (no special code).
- `PCM_TOOLKIT_OS` / `ANFS_RPC_OS` force the platform, for tests.
- `system.bats` also got `launchctl`/`systemctl` stubs on its PATH, since it runs implode.

## Tests

- `packages/anfs/tests/listener.bats`: 11 new (unit files per platform, plist lint, token 0600 and
  kept on reinstall, port, failures warn, post_remove, no launchctl without a plist, implode on
  both platforms). `rpc.bats`: 4 new for the token check (driven through `anfs rpc` directly).
- `packages/anfs/tests/all` through the job: **374 passed, 0 failed, 26 files (pmm and pcm off)**.
- Proved the inetd wiring once with a throwaway `socat` on a random 127.0.0.1 port in an isolated
  HOME: token + `{"version":true}` answered; no token got `unauthorized` and was closed. Killed
  by pid.

## Written, not run

- `packages/pcm/tests/toolkit.bats` (7): validate, macOS env and token only in compose's env,
  missing token warns, Linux mount, absent gives nothing, show/list never show the token.
- Checked by hand in an isolated HOME with podman stubbed: `pcm show ui` prints `toolkit write`;
  `pcm validate` passes `write` and errors on `toolkit: root`; `pcm list` `--json` has `toolkit`.
  `pcm down ui` with `PCM_COMPOSE` set to an env-printing stub: macOS gets
  `ANFS_RPC_TOKEN=<token>` in compose's env; Linux gets none. The override files never contain the
  token. This caught a missing newline after the Linux volume line, now fixed.

## Not verified

- A real launchd load: whether `StandardErrorPath` really takes stderr off the socket in inetd
  mode (if it doesn't, rpc's own stderr would reach the client), and whether launchd accepts the
  plist as written (`plutil -lint` passes).
- systemd on Linux (deferred box), podman-compose's `${ANFS_RPC_TOKEN:-}` interpolation in an
  override file, and a container reaching the endpoint (deferred podman).
- The round trips' HOME diff after implode.

## On this Mac (the person's step: it loads a LaunchAgent)

```sh
# with the socket branch merged (or core's source pointing at ~/work/socket/core):
ppm install core/anfs                     # writes ~/Library/LaunchAgents/io.anfs.rpc.plist, loads it
launchctl print gui/$(id -u)/io.anfs.rpc | head -20     # loaded, the socket listed
printf '{"token":"%s"}\n{"run":["ppm","list"]}\n' "$(cat ~/.local/state/anfs/rpc/token.read)" | nc 127.0.0.1 47100
printf '{"run":["ppm","list"]}\n' | nc 127.0.0.1 47100  # {"error":"unauthorized: ..."}
cat ~/.local/state/anfs/rpc/rpc.err                     # empty, or rpc's own errors
# undo the agent only: launchctl bootout gui/$(id -u)/io.anfs.rpc && rm ~/Library/LaunchAgents/io.anfs.rpc.plist
```
