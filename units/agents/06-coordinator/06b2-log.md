---
plan: agents/06-coordinator (06b2)
job: coord-06b2
date: 2026-10-24
---

# Log: agents/06b2, the coordinator's hooks in the user's Claude settings

The plan is `06b2-global.md` (copied into this job's docs from main's working tree, where it was untracked, with decision 2's revision and the by-hand steps updated). Built on core `d179042` (06b, unpushed).

## What changed

**core/wsm** (`packages/wsm`):

- `~/.claude/settings.json` (user scope) holds the coordinator's hooks, the same for every space: SessionStart `session-start` (10 s) and `listen --max 86340` (`asyncRewake`, 86400), Stop `listen`, UserPromptSubmit `prompt` (5 s), SessionEnd `session-end` (3 s).
- `wsm __claude install|remove` (plumbing, not in help) merges them in or takes them out; 06b's merge (`_CLAUDE_MERGE`, handlers whose command ends `/anfs/claude/hook`) is reused, so the person's keys and hooks (herdr's) stay, theirs first.
- `install.sh`: `post_install` runs `__claude install` (a failure is `ppm_fail`, the install goes on); `post_remove` runs `__claude remove`. Both run the package's own `home/.local/bin/wsm`, since `post_remove` runs after the unstow (core/anfs's install.sh does the same with its libs).
- Refused, saying so (exit 1): a file that is not a JSON object, and `settings.json` as a link. A `~/.claude` that is itself a link is not refused (the file in it is still a file). Remove deletes the file when only wsm's hooks were in it, never `~/.claude/`. Mode 600.
- `coordinator:` is now `claude` when absent (every installed space has a coordinator); `none` or an invalid form (warned) turns it off.
- `wsm install` records the parsed policy in `$ANFS_STATE_HOME/leases/<space id>/coordinator.policy`: one line, `claude hooks`, `claude task` or `none` (`coordinator: claude hooks` / `(up to date)` in its output; `-n`: `would record`). `wsm remove` deletes it. A space with no id gets none (the hook already ignores such a space).
- wsm no longer writes, removes or reads any space's `.claude/settings.json`: `_space_claude`, `_claude_unset` and `wsm add`'s exception for that file are gone.
- `wsm ls`: NOTE `coordinator outdated` (STATUS `partial`) when the recorded line is not what space.yml says now; `--json`'s `claude` key is replaced by `coordinator` (`ok`, `outdated`, `""` without an id). The user-scope hooks' state is a warning on stderr (outdated, invalid, refused), not a column.
- `sh/wsm.sh` exports `WSM_SPACES_HOME` once per shell from `command wsm config get WSM_SPACES_HOME` (after the `command -v wsm` guard; a value already in the environment wins; an empty one is never exported).
- README: the Claude Code row and a paragraph on `WSM_SPACES_HOME` in the shell.
- Tests: `claude-settings.bats` rewritten (10): the five entries, the person's settings kept, remove, refusals, `sh/wsm.sh` in bash and zsh, install.sh's hooks, the recorded policy, invalid forms, no id and dry run, listen's wake set. `manage.bats`: the `--json` keys.

**core/anfs** (`lib/anfs/claude/hook`):

- First lines, before any library is sourced: exit 0 unless `$PWD` is `$WSM_SPACES_HOME` or below it. When `$PWD` does not match, `cd -P .` (a builtin: no fork) gives the physical directory and that is tried too. A leading `~` or `$HOME` in the value is expanded; a value that is not absolute is nothing.
- `WSM_SPACES_HOME` unset (a GUI-started Claude Code): the hook runs `~/.local/bin/wsm` (else `wsm` on PATH) `config get WSM_SPACES_HOME` once, stdin from `/dev/null`.
- The policy is read from `coordinator.policy` with one `read` (no file: `claude hooks`). `none`: no process walk, no claim, no listener, the inbox line only. `claude task`: `listen` exits 0 at once, the brief and the prompt line as before.
- The `inbox` entry and `--arm` are gone (no compatibility: an old per-space file's `hook inbox` just exits 0).
- Tests: `claude-hook.bats` (23): `arm: task` and `none` through the recorded policy; new: the gate (a copy of the hook whose `bash.sh` leaves a mark: outside, a sibling `<root>-other`, a relative value, none of them sources; the root and below do; unset falls back to `wsm config get`), and no recorded policy is a coordinator.

**core/AGENTS.md**: `coordinator:` (at run time), "Claude Code settings" (user scope), `WSM_SPACES_HOME` in the shell, the adapter's gate and the tests.

**Versions**: wsm 0.1.13, anfs 0.2.19 and `WSM_VERSION` 0.13.0 were already bumped in this unpushed series (the pre-commit hook said so); not bumped again.

## Decisions made during the work

1. **Where the policy lives: the lease's directory**, `$ANFS_STATE_HOME/leases/<space id>/coordinator.policy`. That is "beside its id" read literally: the hook already has the space id and `ANFS_LEASES_HOME` (lease.sh), so no new path variable, and the file follows the id through `wsm move` as the lease does. `$WSM_STATE_HOME/...` was the alternative (implode would clean it); I picked the lease's because the hook does not resolve wsm's homes.
2. **No file means a coordinator.** The plan makes it the default, so a space installed before this change coordinates at once; `wsm ls` still says `coordinator outdated` until `wsm install` records it.
3. **`wsm ls`'s claude column goes**, replaced per space by the recorded policy's state (`coordinator`), because the hooks are now the machine's, not a space's: the same value on every row would read as a per-space fact. Their state is one stderr warning instead, naming the fix (`ppm install core/wsm`). It is silent when none of wsm's hooks is in the file (Claude Code may not be used, and a test HOME has none).
4. **The spaces root (decision 2), three iterations.** I first had wsm write the resolved root into each hook's args (`--spaces DIR`), rewritten by `wsm config set|unset` through a new config.sh callback. The person replaced that with `sh/wsm.sh` exporting `WSM_SPACES_HOME` and a slow-path `wsm config get` in the hook. The args and the callback are gone; config.sh is unchanged.
5. **`$PWD`, not the input's `cwd`, gates** (as the plan says): the input is not read before the libraries.
6. **`_coord_policy` reads the space's id from its space.yml**, not `_ids_load`'s cache: `wsm new` installs a space made after the cache was filled (found by manage.bats).
7. **Hook outputs**: `__claude install` prints `claude settings: written (~/.claude/settings.json: the coordinator's hooks)` or `up to date`; ppm shows it.

## Tests

Every run was `anfs work run coord-06b2 core -- <cmd> </dev/null`; the tests use the wsm helper's temp HOME and every `XDG_*`, `ANFS_*_HOME`, `WSM_*_HOME`, `ANFSD_STATE_HOME` redirected; `ANFS_WORK` was never set; the fake Claude Code is bash under another name.

- `packages/wsm/tests` with `anfs/tests/{cd,settings,config}.bats`: **190 passing**.
- `claude-settings.bats` **10**, `claude-hook.bats` **23**, all passing.
- `packages/anfs/tests/all` (pmm and pcm off): **834 passed, 0 failed**, 60 files, 130 s. A run before it had 2 failures: `isolation.bats` caught a literal `~/` in my new test (fixed: the `~` goes through a variable), and `rpc.bats` 3 ("anfs package list is ppm list") failed once under the parallel load; it passed 3 runs alone and in the final full run, and nothing here touches rpc.
- The `sh/wsm.sh` test runs bash and zsh (`zsh -f`); zsh is present on this machine.
- Not run: a real Claude Code session, Linux, any install against the real HOME.

## By hand after merge (no migration code)

1. Merge `coord-06b2` (core, docs) after 06b; rebase if main moved.
2. `anfs src update core` and `ppm install core/anfs core/wsm`. Expect `claude settings: written (~/.claude/settings.json: the coordinator's hooks)`. If it says the file is a link or not JSON, fix that and run it again.
3. Open a new shell (so it has `WSM_SPACES_HOME`); check `echo $WSM_SPACES_HOME`.
4. `wsm install anfs-dev/` (and each other space or prefix); each prints `coordinator: claude hooks` (or `none`). `wsm ls` should have no `coordinator outdated`.
5. Take 06b's per-space files away: `rm ~/spaces/anfs-dev/.claude/settings.json` (and an empty `.claude/`). Any other space installed since 06b has one too (the inbox entry): `ls ~/spaces/**/.claude/settings.json`; delete each that holds only wsm's hooks, or take them out of one you edited: `jq '.hooks |= (map_values(map(.hooks |= map(select(.command | test("/anfs/claude/hook$") | not))) | map(select(.hooks | length > 0))) | with_entries(select(.value | length > 0)))' f`.
6. Start Claude in `~/spaces/anfs-dev` and in `~`: the brief appears only in the space; `wsm lease holder coordinator` names the first.
7. Optional: anfs-dev's `spaces/space.yml` `coordinator: claude` is now the default; the line may stay or go (your commit).

## Open questions

1. **Sessions below a space's root now coordinate.** The user-scope hooks run in every session, and the space is the one at or above `cwd`, so a session started in a clone (`~/spaces/anfs-dev/core`) can become anfs-dev's coordinator if it is first. 06b's open question 5 (sub-root sessions get no hooks) is reversed. **Recommendation:** decide whether the coordinator should be claimed only at the space's root (a one-line check in `session-start`/`prompt`/`listen`: `HOOK_SDIR == cwd`), keeping the inbox line everywhere below it.
2. **A symlinked spaces root** whose logical path neither `$PWD` nor the physical directory matches (root set through a link, Claude started from the physical path) is treated as outside. Rare; canonicalizing would cost a fork per hook outside the spaces. **Recommendation:** accept.
3. **Every Claude session on the machine** now starts the hook two times a turn (prompt, Stop) and three times per session (two at start, one at end): outside the spaces each is a bash start and a string compare. **Recommendation:** accept; measure if it shows.
4. **`wsm config set WSM_SPACES_HOME`** now warns that the environment wins (the shell's export) until a new shell. **Recommendation:** accept, documented.
5. **The policy file of a vanished space** (its space.yml gone from the source) is left in the lease directory, inert. **Recommendation:** accept.
