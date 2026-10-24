---
plan: agents/06-coordinator (status line)
job: claude-status
date: 2026-10-24
---

# Log: the coordinator's status line

The ask: with Claude Code's status line, the session that is a space's coordinator shows `coordinator: <space path>` (`coordinator: lgat/next`); every other session shows nothing from anfs. Built on core `0eb40c8` (06b2, unpushed).

## What the docs say

Checked against https://code.claude.com/docs/en/statusline.md and settings.md (fetched 2026-10-24). Claude Code 2.1.309 is installed; it was not run, and its binary did not show the statusLine schema to `strings`.

- Shape: `"statusLine": {"type": "command", "command": "...", "padding": N, "refreshInterval": N, "hideVimModeIndicator": bool}`. There is no `args`: "The `command` field runs in a shell". So it has no exec form, unlike hooks.
- Stdin: JSON with top-level `session_id`, `cwd` and `workspace.current_dir` (the same value), `workspace.project_dir`, and more (model, cost, context window).
- When it runs: at session start and resume, then on each new assistant message, `/compact`, a permission-mode or vim-mode change, a change to the command, and `refreshInterval` if set. It is debounced at 300 ms. An update cancels a run still in flight.
- Output: each stdout line is a row; ANSI and OSC 8 are allowed. stderr is only logged under `claude --debug`. Empty output shows nothing.
- Gates: workspace trust (the status line stays blank until it is accepted), `disableAllHooks`, `allowManagedHooksOnly`.
- A configured status line also hides most footer hints (`esc to interrupt`, `? for shortcuts`). That is why the person was first going to opt in.
- **Running sessions pick up a change without a restart.**
  - settings.md: "Claude Code watches your settings files and reloads them when they change, so it applies most edits to the running session without a restart". statusLine is not among the keys it lists as read only at start.
  - statusline.md lists "You change the `command` in your `statusLine` settings" as an update trigger: "A change to the `command` itself skips the debounce: Claude Code runs the new command right away."
  - So a session that is already open shows the line as soon as wsm writes it. Whether removing the key also clears the row at once is not stated; a reload implies it, but this is not verified.

## What changed

**core/anfs** (`lib/anfs/claude/hook`, `lib/anfs/config.sh`):

- A new entry, `hook statusline`, in the same script. It has the same `WSM_SPACES_HOME` gate first; then the space at or above the input's `cwd` (it must have an id); then one read of `leases/<id>/coordinator.json`. It prints `coordinator: <space path>` when the lease's session is this `session_id` and `anfs_lease_status` says listening or working (the TTL judged on lease.sh's clock). Otherwise it prints nothing. Exit 0 always.
- It branches before the process walk and the policy read: no `ps`, no wsm, no yq, no event. When `WSM_SPACES_HOME` is unset, it exits at once instead of taking the gate's `wsm config get` slow path.
- `ANFS_RUN`, no input, a tty on stdin, or input without a session: nothing, as for the other entries.
- Measured on macOS (a scratch HOME, 50 runs): about 18 ms per call in a space, 12 ms outside.
- `config.sh`: after `set` or `unset` writes the file, it calls a tool's `anfs_config_changed <names...>` if the tool defines one.

**core/wsm**:

- `wsm.env` gains `#WSM_CLAUDE_STATUSLINE=on`.
- `__claude install` writes `"statusLine": {"type":"command","command":"'<abs hook>' statusline"}`. It does so only when the file has no statusLine or has wsm's own: a command ending in the adapter's path plus `statusline`. Ours is assigned in place, so its position in the file is kept.
- The path is single-quoted, with any `'` written as `'\''`. A test runs the written command through `sh -c`.
- `off`, and `__claude remove`, take only wsm's own statusLine out. Any other value warns and counts as off.
- A statusLine of the person's is never touched. Install prints one line:
  ```
  claude settings: statusLine is yours (<cmd>): kept; for the coordinator's line, have it also run '<hook>' statusline with the same stdin and print what that prints
  ```
- `wsm config set|unset WSM_CLAUDE_STATUSLINE` rewrites the settings at once. wsm's `anfs_config_changed` re-reads the effective value (the environment still wins, and config.sh warns so) and runs `_claude_install`. Any other setting leaves the file alone.
- README: the status line, turning it off, and composing with your own (an example script). AGENTS.md: the status line under "Claude Code settings", the `statusline` entry under the adapter, and the tests.

**Versions**: anfs 0.2.19, wsm 0.1.13 and `WSM_VERSION` 0.13.0 were already bumped in this unpushed series. The pre-commit hook said so and did not bump again.

## Decisions

1. **On by default.** This is the person's change during the job; the brief had it opt-in. `WSM_CLAUDE_STATUSLINE=off` leaves it out and takes wsm's own out.
2. **A quoted shell command, not exec form.** The docs give statusLine no `args`. wsm's own is recognised by the command's tail (`/anfs/claude/hook` with an optional `'`, whitespace, then `statusline`), as the hooks are by `/anfs/claude/hook$`.
3. **The same script, not a sibling.** It reuses the gate, the input parsing, `anfs_events_space_at` and lease.sh's reader and judge. The libraries are sourced (paths.sh's one `cd -P` subshell included); the cost is in the measurement above.
4. **No `wsm config get` when `WSM_SPACES_HOME` is unset.** The brief says no wsm on this path, and the status line runs often. So a Claude Code started outside a `sh/wsm.sh` shell shows nothing. See open question 1.
5. **The holder's start time is not compared** (`anfs_lease_status ... false`: no `ps`). The pid is still checked with `kill -0` and the TTL is judged. A pid reused after a resume with the same session id would show the line until the TTL passes. See open question 2.
6. **The space path is the current install path** (`anfs_events_space_at`), not the lease's `spacepath`, so a `wsm move` shows at once.
7. **stderr is left alone.** Claude Code shows only stdout, and `--debug` keeps the hook's warnings visible.
8. **A config callback in config.sh** (`anfs_config_changed`), rather than asking the person to rerun a step. It is generic and a no-op for tools that do not define it. 06b2 had a similar callback for `WSM_SPACES_HOME` and dropped it when that design changed.

## Tests

Every run was `anfs work run claude-status core -- <cmd> </dev/null`, with the wsm helper's temp HOME and every `XDG_*`, `ANFS_*_HOME`, `WSM_*_HOME`, `ANFSD_STATE_HOME` redirected. `ANFS_WORK` was never set. No real claude was run.

- `claude-hook.bats`: **25** (2 new).
  - The holding session shows `coordinator: acme`, also from a subdirectory; another session and an unknown one show nothing.
  - Nothing outside the spaces, under `ANFS_RUN`, without `WSM_SPACES_HOME`, or with no input. Nothing once the lease has expired (fake clock) or been released.
  - Stub `ps` and `wsm` that leave a mark are never called. No event is written.
  - A nested space shows `coordinator: acme/lgat/next`. A dead holder shows nothing.
- `claude-settings.bats`: **13** (3 new).
  - On by default and quoted for `sh -c`; idempotent; moved within the file, still up to date. Off and invalid values take it out and keep the hooks; remove deletes the file.
  - The person's statusLine is kept, with the compose line, through on, off and remove.
  - `wsm config set|unset` rewrites the file, the environment wins, and another setting leaves the file alone.
- Together with `config.bats`, `settings.bats` and `isolation.bats`: 61 passing.
- `packages/anfs/tests/all` (pmm and pcm off): **839 passed, 0 failed**, 60 files, 132 s.
- Not run: a real Claude Code session, and Linux.

## Commits (branch claude-status, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| core | `9706dfa` | anfs: `hook statusline`, config.sh's `anfs_config_changed`, tests |
| core | `84dd757` | wsm: the statusLine in `__claude install/remove`, `WSM_CLAUDE_STATUSLINE`, the config reaction, tests, README |
| core | `2cba808` | AGENTS.md |
| docs | (this) | this log |

## By hand after merge (no migration code)

1. Merge `claude-status` (core, docs) after 06b2; rebase if main moved.
2. `anfs src update core`, then `ppm install core/anfs core/wsm`.
   - Expect `claude settings: written (~/.claude/settings.json: the coordinator's hooks and the status line)`.
   - If you have a statusLine of your own, it says so and keeps it instead.
3. A running coordinator session should show `coordinator: <space>` at its next update (the next assistant message), with no restart.
4. Off: `wsm config set WSM_CLAUDE_STATUSLINE=off`. Back on: `wsm config unset WSM_CLAUDE_STATUSLINE`.

## Open questions

1. **A Claude Code started without `WSM_SPACES_HOME`** (a GUI app, a shell without `sh/wsm.sh`) shows no status line, while its hooks still work through `wsm config get`. **Recommendation:** accept. If it matters, have the hook read wsm's two env files directly (bash, no fork) for the root.
2. **No pid start-time check**: after a resume that keeps the session id, a reused pid could show a stale line until the TTL. **Recommendation:** accept; it is rare, and the next takeover corrects it.
3. **Being on by default hides most footer hints** in every session on the machine, spaces or not, because the row exists even when it is empty. Not verified: whether an empty output also hides those hints. **Recommendation:** the person checks in the first session; `WSM_CLAUDE_STATUSLINE=off` if it bothers.
4. **Unverified until a real session:** that removing the key clears the row in a running session, and that the status line's command runs with the session's cwd as `$PWD` (the gate reads `$PWD`, as for hooks).
