# Claude Code settings on this Mac: a review

2026-10-23, for the person, before dispatch (agents/02b) is turned on. What is there, where it
comes from, what each part does day to day and to a dispatched agent, and a proposal for who
should own what. Facts about Claude Code are from its docs (links at the end); values that look
like secrets are not reproduced here.

## How Claude Code combines settings (the facts that matter)

- **Scopes**: managed (none on this Mac), the `--settings` file a command passes, local
  (`<project>/.claude/settings.local.json`), project (`<project>/.claude/settings.json`), user
  (`~/.claude/settings.json`). Lists merge across them.
- **Permissions: deny beats allow, from any scope.** Rules are checked deny → ask → allow. A deny
  in the run's own settings can't be undone by an allow elsewhere. But a Bash deny matches only
  the command forms it names (`sh -c "git push"`, `/usr/bin/git push` may slip past): the docs say
  a Bash deny "isn't a security boundary". The **sandbox** is the hard stop.
- **`dontAsk` mode** (what dispatch uses): anything no rule allows is denied; a bare `Bash` allow
  allows every command that no deny matches.
- **Hooks from user and project settings run in `-p` (headless) mode**, even in a folder never
  trusted, unless `--bare` is passed.
- **`env` from user settings applies in `-p`**; a project's `env` waits for folder trust.
- **`~/.claude/settings.local.json` is local scope for the project `~`**: it applies only to
  sessions started in the home directory itself, not in `~/spaces/...` or a job.
- **Narrowing flags**: `--setting-sources user,project,local` picks which file scopes load;
  `--bare` skips hook auto-discovery; `--restricted` loads only managed settings and `--settings`
  but also removes Bash (too much for an agent that fixes code).

## What is there

| File | Owner today | Contents |
|---|---|---|
| `~/.claude/settings.json` (user) | **nobody in anfs**: Claude Code writes it (`/config`), herdr's integration added its hook | herdr `SessionStart` hook (`~/.claude/hooks/herdr-agent-state.sh`, "managed by herdr"); `skipDangerousModePermissionPrompt: true`; `editorMode: vim`; `theme: dark`; `tui: fullscreen`; `agentPushNotifEnabled: true`; effort `high` for opus-5; plugin `ruby-lsp` |
| `~/.claude/settings.local.json` (local, project `~`) | the person's **user repo** (`user/packages/claude`, linked) | `allow`: Read, Edit, Write, Glob, Grep, WebSearch, WebFetch, bare `Bash`; `env`: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| `~/spaces/user/evals/firstmate/.claude/settings.json` (project) | firstmate's repo | firstmate's hooks: SessionStart, PreToolUse checks, UserPromptSubmit |
| `stack/claude` | ships **no** settings | — |

No repo of core, stack, anfs-dev or cnfs has a `.claude/settings*.json` today.

## Each part, day to day and for a dispatched agent

| Setting | Day to day (you) | A dispatched agent (`claude -p` in `~/work/<job>`) |
|---|---|---|
| herdr `SessionStart` hook | herdr shows agent state in its panes | **runs** in every dispatched session too (user scope, `-p`); harmless if it only reports state, but it is code we don't own running in unattended sessions |
| `skipDangerousModePermissionPrompt` | no warning when you start in bypass mode | none (dispatch never uses bypass; `args:` refuses it) |
| vim, theme, fullscreen, effort | your editor and look | effort applies (cost); the rest is moot headless |
| `agentPushNotifEnabled` | phone notifications from your sessions | possibly notifications from dispatched sessions too: to check on the first real run |
| plugin `ruby-lsp` | Ruby navigation | loads in dispatched sessions (slower start, harmless) |
| `settings.local.json` allow-all | **only in sessions started in `~`**: there, nothing asks. In `~/spaces/...` it does **not** apply (a surprise worth knowing) | does **not** apply (a job is not `~`) |
| `AGENT_TEAMS=1` env | experimental agent teams, but only in `~` sessions | does not apply |
| firstmate hooks | only inside that evals project | only if a job's repo carried them; none do |

So the widening I worried about is smaller than feared: the allow-all is local to `~`, and the
run's denies win anyway. What does reach a dispatched agent is the **user scope**: herdr's hook,
the plugin, effort, possibly push notifications. Plus any project settings a job's repo might
carry in future, whose hooks would run untrusted.

## Proposal (for the person to decide)

1. **Dispatched runs load only what anfs gives them**: the run's `--settings` file, plus the
   sandbox (already: `failIfUnavailable`, no unsandboxed commands), with
   `--setting-sources local` (a job has no local file, so effectively none) and `--bare` (no hook
   auto-discovery). Verify on the first real run with `/status`'s "Setting sources" line. Hooks a
   role needs come from the run's settings, declared by anfs, not inherited.
2. **Give the user scope an owner.** Split it: personal preferences (vim, theme, fullscreen,
   effort, push notifications, plugins) move into the user repo's `claude` package, and the
   integrations (herdr's hook) are declared by the package that installs them (herdr's), so
   `ppm remove` removes them. Claude's `/config` writes the file, so it stays a real file managed
   by a merge, or a link the user repo tracks (your call).
3. **Move the allow-all deliberately.** Today it only works in `~`. Decide what you want in your
   own sessions in `~/spaces/...` (likely a curated allow list per space, in each `space.yml`'s
   future roles), and drop the `~`-only file or make it what you meant.
4. **A rule for repos**: no `.claude/settings*.json` with hooks in source repos without review,
   since `-p` runs project hooks untrusted. Add it to AGENTS.md.

## Questions for the person

1. Dispatched runs: only anfs's settings (`--setting-sources local --bare`), yes?
2. User scope: preferences into the user repo, integrations owned by their packages, yes?
3. The allow-all in `settings.local.json`: did you mean it for every session? It applies only in
   `~` today.

## Decided 2026-10-24

1. **Superseded by 02d** (`units/agents/02-daemon/02d-safe.md`, section 3 and its open question
   1): dispatched claude runs with `--setting-sources local --strict-mcp-config` plus
   `claudeMdExcludes`, keeping the subscription login; `--bare` is opt-in
   (`ANFSD_CLAUDE_BARE=on`, with an API key), since it drops that login. Not `--bare` always, as
   proposed above.
2. **Yes.** Personal preferences (vim, theme, fullscreen, effort, push notifications, plugins)
   move into the user repo's `claude` package; integrations (herdr's `SessionStart` hook) are
   owned by the packages that install them, so removing the package removes them. Moving the
   preferences is the person's by-hand work in their own user repo: agents never edit the user
   repo.
3. **Keep the allow-all scoped to `~`**, as it is (`~/.claude/settings.local.json`, sessions
   started in `~` only).

## Sources

- https://code.claude.com/docs/en/permissions.md
- https://code.claude.com/docs/en/settings.md and settings-reference.md
- https://code.claude.com/docs/en/headless.md
- https://code.claude.com/docs/en/hooks.md
- https://code.claude.com/docs/en/cli-reference.md
