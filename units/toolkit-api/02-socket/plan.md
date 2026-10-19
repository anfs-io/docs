---
---

# Plan 02 — The socket: `anfs rpc` reachable from containers, and pcm's grant

Needs plan 01.

## Context — read these files first

- `units/toolkit-api.md` (open question 1: tested — a container on macOS reaches the host's
  loopback at `host.containers.internal`), plan 01 and its log
- core `packages/anfs/package.yml`, `packages/anfs/install.sh` (`post_install`, `post_remove`),
  `bin/anfs` `cmd_implode`
- core `packages/pcm/home/.local/lib/pcm/{manifest,compose,runtime}.sh` — `container.yml`, how a
  service's compose is run
- core `containers/ui/` (`compose.yml`, `container.yml`)

## Implementation

### 1. The listener, socket-activated

Nothing runs between connections; each connection starts one `anfs rpc` with it as stdin and
stdout.

| | macOS | Linux |
| --- | --- | --- |
| Unit | `~/Library/LaunchAgents/io.anfs.rpc.plist`, `Sockets` + `inetdCompatibility` | `~/.config/systemd/user/anfs-rpc.socket` (`Accept=yes`) + `anfs-rpc@.service` |
| Listens on | TCP `127.0.0.1:<port>` (`ANFS_RPC_PORT`, default 47100) | unix socket `$XDG_RUNTIME_DIR/anfs/rpc.sock`, mode 0600 |
| Who may call | anyone local who has the token | the user (file permissions) |

- `core/anfs` stows the unit files; its `post_install` loads them (`launchctl bootstrap gui/$UID`,
  `systemctl --user enable --now anfs-rpc.socket`); `post_remove` unloads them; `anfs implode`
  does the same before deleting anything.
- **The token (macOS)**: generated at install, `$ANFS_STATE_HOME/rpc/token`, mode 0600. The first
  request line on a TCP connection must be `{"token":"..."}`; otherwise the connection is closed.
  On Linux the socket's permissions are the check; a token line is accepted and ignored.
- **The grant**: one token per grant (`token.read`, `token.write`) on macOS, and on Linux two
  sockets (`rpc.sock` read, `rpc-write.sock` write); the unit starts `anfs rpc --grant <g>`.

### 2. pcm hands it to a container that asks

- `container.yml` key `toolkit: read|write`. Absent: the container gets nothing.
- pcm adds, when it runs the service: on macOS `ANFS_RPC_URL=tcp://host.containers.internal:<port>`
  and `ANFS_RPC_TOKEN` (the grant's token, from the env, never written into the compose file); on
  Linux the grant's socket mounted at `/run/anfs/rpc.sock` and
  `ANFS_RPC_URL=unix:///run/anfs/rpc.sock`.
- `pcm show` says `toolkit: write` (never the token). `pcm validate` checks the value.
- `core/ui`'s `container.yml` gets `toolkit: write`.

### 3. Documentation

Core `AGENTS.md`: the API section gains the listener, the token, the grant and pcm's key; the
pcm section the `toolkit:` key.

## Test Spec

- `packages/anfs/tests`: `post_install` writes the right unit files (stubbed `launchctl`,
  `systemctl`) for each platform; the token is 0600 and survives a reinstall; `post_remove` and
  implode unload and remove them; a connection without the token is closed (drive `anfs rpc`'s
  token check directly, no listener).
- pcm's tests (suite off: written, not run): `toolkit:` gives the env or the mount, absent gives
  nothing, the token never appears in `pcm show` or a compose file on disk.

## Verification

- [ ] `packages/anfs/tests/all`
- [ ] On this Mac (by the person or with consent, it loads a LaunchAgent): `ppm install core/anfs`
      again, then `printf '{"token":"%s"}\n{"run":["ppm","list"]}\n' "$(cat ~/.local/state/anfs/rpc/token.read)" | nc 127.0.0.1 47100`
- [ ] Deferred (podman): from inside `core/ui`, the same request through `ANFS_RPC_URL`
- [ ] Deferred (Linux box): the user socket, `systemctl --user status anfs-rpc.socket`
