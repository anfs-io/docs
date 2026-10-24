# 06b2: the coordinator's hooks in the user's Claude settings

2026-10-24, from the person: every space will have a coordinator, so the hooks belong in `~/.claude/settings.json` (user scope) rather than in each space's `.claude/settings.json`, and the hook must be cheap and silent outside the spaces so other tools that use Claude are not affected.

## Decisions

1. **One place.** core/wsm's install merges its hook entries (the same ones 06b writes per space: SessionStart session-start and listen, Stop listen, UserPromptSubmit prompt or inbox, SessionEnd session-end) into `~/.claude/settings.json`, and its removal takes them out. Only entries whose command is `…/anfs/claude/hook` are touched; herdr's hook and the person's preferences stay. The file is a real file Claude Code also writes (`/config`): merge, never replace; a link is refused, saying so.
2. **Cheap outside spaces.** The hook's first lines, before sourcing any library: exit 0 unless `$PWD` is `$WSM_SPACES_HOME` or under it. Then, as today, exit 0 unless the directory is in an installed space.
   - Revised during the job (the person): `WSM_SPACES_HOME` is wsm's setting (`wsm config get`), not the shell's. core/wsm's `sh/wsm.sh` exports it once per shell (a value already in the environment wins; never empty), so Claude Code and its hooks inherit it. When it is unset (Claude Code started outside such a shell, e.g. a GUI app), the hook asks `wsm config get WSM_SPACES_HOME` once in that call: the slow, rare path. A config change reaches new shells.
3. **The policy moves to runtime.** (Built as `$ANFS_STATE_HOME/leases/<space id>/coordinator.policy`, one line: `claude hooks`, `claude task` or `none`; see `06b2-log.md`.) The per-space settings carried the policy (`--arm task`, coordinator on or off). Now the hook reads it per space: `wsm install` records the space's parsed `coordinator:` policy where the hook already resolves the space (beside its id), and the hook reads that small file, no yq. Every installed space has a coordinator by default; `coordinator: none` turns it off (the inbox line stays). `{harness, wake, window, arm}` keep their meaning.
4. **wsm stops writing `.claude/settings.json` into spaces.** Its code for that goes; no code removes the old files (by hand: `rm ~/spaces/anfs-dev/.claude/settings.json`). `wsm ls`'s claude column reports the user-scope hooks' state (ok, outdated, invalid, refused) instead, or goes if that reads better; the agent picks and says why.
5. **Unchanged:** dispatched runs (`--setting-sources=` and the `ANFS_RUN` exit), subagent hooks, `ANFS_COORDINATOR=off`, `ANFS_INBOX_HINT=off`, the lease.

## By hand after merge

`ppm install core/anfs core/wsm`, a new shell (for `WSM_SPACES_HOME`), `wsm install` (each space), `rm ~/spaces/anfs-dev/.claude/settings.json`, then start Claude in a space and in `~` and check the brief appears only in the space.
