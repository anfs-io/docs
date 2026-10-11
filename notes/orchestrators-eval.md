---
objective: Choose how agent orchestrators plug into ANFS as interchangeable front ends
status: proposed
---

# Orchestrator Evaluation: Firstmate and Paperclip

Evaluated 2026-10-04 against `kunchenguid/firstmate` and `paperclipai/paperclip` (clones in
`~/evals`). This combines a static read of both repos with a hands-on spike that ran a real
task through each one.

**Verdict.** Both tools work and both can sit on top of ANFS. Neither should become part of
ANFS.
- **Firstmate** fits ANFS's own approach almost exactly: bash, plain files, one operator, tmux
  or herdr, and skills discovered natively. Make it the default interactive front end,
  shipped as the `stack/firstmate` ppm package.
- **Paperclip** is a heavy control plane that owns its own database, tasks and skill
  projection. Treat it as an optional pcm service for governed, long-running "company"-style
  work. Expect to write an adapter layer for skills and tasks.

Before shipping either, write down the **orchestrator contract** in section 2, so that
"front-end agnostic" is something we can test rather than an aspiration.

---

## 1. Where an orchestrator sits in ANFS

ANFS already owns everything an orchestrator consumes. It has no orchestrator of its own:

| ANFS provides | Mechanism | An orchestrator should... |
|---|---|---|
| Harness binaries and auth | `ai/<harness>` packages (`meta.agent`), installed via mise | launch them, not bundle its own |
| Skills | psm → `~/.agents/skills`, then projected into harness-native dirs (`~/.claude/skills/*` symlinks) | let harnesses discover them natively |
| Workspaces / repos | wsm (`.wsm/id`, `spaces/*/space.yml`) | treat them as projects |
| Services | pcm (`containers/<name>/compose.yml`, shared postgres via `x-pcm.depends_on`) | run there if it needs a server or DB |
| Sandboxes | pim VMs and pcm containers | (future) use them as execution targets |
| Units of work | chorus `units/<unit>/NN-slug/plan.md`; `log.md` means done | take plans as intent and write completion back |

The orchestrator owns only dispatch, supervision and its own runtime state.

## 2. Proposed orchestrator contract

An orchestrator is **ANFS-compatible** when all of these hold:

1. **Install:** it ships as a ppm package (plus an optional pcm container) and uninstalls cleanly with `ppm remove`.
2. **Harnesses:** it drives the harness binaries ANFS installed, using the operator's existing auth, and does not vendor its own copies.
3. **Skills:** the `chorus-hello` probe is discoverable *and invokable* by its workers without extra copying (`stack/skills/chorus-hello`).
4. **Work in:** a chorus `plan.md` can be dispatched as a task by script, not only through a UI.
5. **Work out:** completion is observable by script (file, JSONL or REST), so a bridge can write chorus `log.md`.
6. **Isolation:** worker changes land on a branch or worktree. The project's primary checkout is never switched.
7. **State:** all runtime state lives under declared homes. Teardown leaves no processes behind.
8. **Privacy:** outbound telemetry is off, or can be switched off by config the package sets.

Items 3, 6 and 7 are covered by a smoke test against a throwaway repo. That is the same
procedure as this spike (section 5) and is worth keeping as `anfs` test tooling.

## 3. Scorecard against the contract (from the spike)

| # | Contract item | Firstmate | Paperclip |
|---|---|---|---|
| 1 | Install as a package/container | ✅ Clone plus 7 tools. Fits `stack/firstmate` (axi tools via mise `npm:` backend) | ⚠️ `npx paperclipai` works natively. The container needs an onboarding/provision step (see 5.2) |
| 2 | Uses ANFS harnesses | ✅ Runs `claude` on PATH with host auth | ⚠️ Native: yes. Container: bundles its own `claude-code@latest`, `codex`, `opencode`, `gemini` and `kimi` (4.8 GB image) |
| 3 | Skills (`chorus-hello`) | ✅ Discovered and invoked natively by workers | ❌ Lists psm skills as "external, read-only" but **does not project them into runs**. Importing from the ANFS sources dir is refused (`skill_workspace_boundary_denied`) |
| 4 | Work in by script | ✅ `bin/fm-brief.sh` + `bin/fm-spawn.sh`. A chorus plan maps onto "Captain's intent". *(Mapping not exercised.)* | ✅ REST `POST /api/companies/:id/issues`. A bundled `paperclip-converting-plans-to-tasks` skill exists. *(Not exercised.)* |
| 5 | Work out by script | ✅ `state/<id>.status` lines, `data/<id>/report.md`, opt-in `state/fleet-ledger.jsonl` (versioned contract) | ✅ REST issue status and comments, heartbeat runs with per-run cost, MCP server |
| 6 | Isolation | ✅ Treehouse worktree per task. Primary checkout untouched | ❌ With default project settings the agent **checked out its branch in the primary checkout** (`executionWorkspacePolicy: null`) |
| 7 | State under homes; clean teardown | ⚠️ Worktrees go to `~/.treehouse/` (outside `FM_HOME`) and scratch to `/tmp/fm-<id>`. The first mate and supervisor **survive `tmux kill-session`** and re-arm themselves; use `bin/fm-teardown.sh` | ⚠️ All state under `PAPERCLIP_HOME`. Killing the CLI **orphaned the embedded postgres** |
| 8 | Telemetry | ✅ None found | ⚠️ Opt-out. Set `PAPERCLIP_TELEMETRY_DISABLED=1` (and/or `DO_NOT_TRACK=1`) |

## 4. The two tools

### 4.1 Firstmate: an "agent distro"

- **What it is:** an `AGENTS.md` of about 7k words plus about 210 bash scripts. Together they turn any supported CLI session into a "first mate". The first mate never edits projects. It writes briefs and dispatches crewmates into isolated worktrees, supervises them without spending tokens through a bash watcher (`bin/fm-watch.sh`), and reports back. Merging and anything destructive need the captain's explicit word, enforced by scripts that refuse.
- **Stack:** about 112k lines of bash in `bin/`, about 220k lines of shell tests, plus jq, git and gh. It also needs the author's own tools: treehouse (worktree pool), no-mistakes (validation pipeline), and `gh-axi`, `tasks-axi`, `quota-axi` and `lavish-axi`.
- **Harnesses:** 14 or more (claude, codex, opencode, pi, grok, kimi, cursor, omp, gemini …), each a hard-coded launch template in `bin/fm-spawn.sh`.
- **Seams:**
  - Session backends `bin/backends/{tmux,herdr,zellij,orca,cmux}.sh`. herdr is already an ANFS package.
  - A task store via tasks-axi (markdown or beads).
  - `fleet-ledger.jsonl`.
  - Per-home `config/` flags and `FM_HOME`.
  - No HTTP, MCP or plugin API.
- **Maturity:** 883 commits since 2026-06, 93 contributors, about 67% of commits by the author, MIT. Agent-driven churn is high, and most recent commits fix edge cases.
- **Risks:**
  - Workers run with approvals bypassed (`--dangerously-skip-permissions`, `--yolo` …), isolated only by worktrees, as your user.
  - Liveness checks scrape vendor TUIs and are pinned to specific CLI versions (VISION.md calls this "standing debt").
  - Its toolchain and its contributors are concentrated in one author.

### 4.2 Paperclip: "the company" for agents

- **What it is:** a control plane built on Express 5, React 19 and Postgres. It models companies, goals, an org chart (with a CEO agent that hires), heartbeat-driven agents, budgets with hard stops, approval gates and an audit log.
- **Stack:** about 2.3M lines of TS/TSX plus a Rust runner, Drizzle with 282 migrations, embedded Postgres for dev, Node 24 or later, and 12 patched dependencies.
- **Harnesses:** 12 or more through a clean `ServerAdapterModule` (`packages/adapter-stack/src/types.ts`). It also supports external adapter packages, `http` and `process` adapters (the easiest way to put a foreign runtime behind it), and sandbox-provider plugins (E2B, Daytona, Modal, k8s …).
- **Seams:** REST/OpenAPI, an MCP server, a plugin SDK (UI slots, jobs, webhooks, tools), and company import/export.
- **Maturity:** 4,775 commits since 2026-02, 226 contributors, about 62% by the top author, MIT, CalVer releases with signed images. Hosted Paperclip Cloud code paths already exist.
- **Risks:**
  - It wants to be the **system of record**: tasks, org, budgets and skills all live in its database, and chorus would at best be a mirror.
  - Telemetry is on by default.
  - The harness SDK pins in its own image lag upstream.
  - Plugin UI is not sandboxed.
  - Its 30k-line `heartbeat.ts` makes forking impractical.

## 5. Spike log

Throwaway repo: `greet.sh`, no remote. Same two tasks for both tools: a review/scout that also
probes `chorus-hello`, and a "ship" that adds a `--shout` flag, bash 3.2 compatible, local
branch, no push or merge. Every tool was installed into the session scratchpad, not globally.

### 5.1 Firstmate (tmux backend, claude primary and workers)

- **Bootstrap:** `bin/fm-bootstrap.sh` reported 7 missing tools and `NEEDS_GH_AUTH`.
  - Installed into a scratch prefix: treehouse v3.1.2 (release tarball), no-mistakes v1.84.0 (`NO_MISTAKES_INSTALL_DIR`), and `npm -g --prefix` for gh-axi, tasks-axi, quota-axi, lavish-axi and chrome-devtools-axi.
  - Skipped `setup hooks` for the axi tools, because those write global harness hooks.
  - **GitHub auth was not needed** for a local-only project.
- **Run:** one natural-language captain order. The first mate loaded its project-management and harness-adapters skills, registered the project as local-only with merge requiring approval, scaffolded two briefs and spawned both workers.
  - The ship finished about 10 s after spawn: commit `ab7c917` on `fm/demo-shout-flag` in a treehouse worktree.
  - The scout finished about 55 s after spawn, with a thorough 8-finding report at `data/<id>/report.md`.
  - The first mate was woken by its Stop hook / watcher, reported both results, and **asked before merging**.
  - Total primary time was about 2.5 min.
- **Skills:** the worker saw `chorus-hello` in its native skill list (via `~/.claude/skills` → `~/.agents/skills`) and invoked it. It output `HELLO-WORLD SKILL ACTIVE`.
- **Footprint:**
  - `~/.claude/settings.json` was unchanged and the firstmate clone stayed clean.
  - Writes outside `FM_HOME`: `~/.treehouse/<repo>-<hash>/N/` worktrees, `/tmp/fm-<task>/`, and a folder-trust entry in `~/.claude.json`.
- **Teardown gotcha:** killing the tmux session left the primary `claude` running, re-parented to the tmux server. Its `fm-claude-stop-autoarm` → `fm-supervision-host` → `fm-watch-arm` → `fm-watch` chain kept re-spawning until the primary itself was killed. The package's remove hook must call `bin/fm-teardown.sh` and stop the primary.

### 5.2 Paperclip

- **Container (pcm path):**
  - `ghcr.io/paperclipai/paperclip:latest` is 4.8 GB.
  - `local_trusted` mode refuses to start in a container ("local_trusted requires server.bind=loopback"), so the container must use `authenticated` mode.
  - In authenticated mode the server is healthy, but creating the first admin (`auth bootstrap-ceo`) requires a `config.json` that the env-only quickstart doesn't write.
  - So a pcm definition needs a `provision` hook that runs `onboard`/`bootstrap-ceo`, or prepares the config, and it should use the shared pcm postgres (`DATABASE_URL`) instead of embedded.
- **Native (what we actually ran):**
  - `PAPERCLIP_HOME=… PAPERCLIP_TELEMETRY_DISABLED=1 npx paperclipai onboard --yes --no-install-service` came up in about 10 s on `127.0.0.1:3100`, with embedded postgres and 282 migrations.
  - Created a company, a project (workspace `cwd` set to the repo), a `claude_local` agent with a $3 budget, and an issue, all through the REST API.
  - Assigning the issue woke the agent. It finished in **2 min 16 s** and set the issue to `done`. Its comment covered the implementation and a bash 3.2 verification table.
  - Reported cost: $0.91 (34k input, 10k output, 616k cached tokens; billed as subscription).
- **Skills:**
  - `GET /api/agents/:id/skills` lists every psm skill as `external, user_installed, readOnly`.
  - The run itself used an isolated runtime-skills root that contains only `paperclip`. The worker reported "`Skill(chorus-hello)` → Unknown skill" and fell back to reading `SKILL.md` off disk.
  - `skills/sync {mode: add}` marks the skill desired but "missing".
  - `skills/import` from `sources/ai/skills/chorus-hello` is denied by the workspace boundary.
- **Isolation:** the agent ran `git checkout -b` in the **project's primary checkout**. An `executionWorkspacePolicy` that uses worktrees must be set per project.
- **Teardown:** killing the CLI left the embedded postgres (port 54329) running. It had to be killed separately.

### 5.3 Side effect to note

During the scratch `npm install`, the terminal printed `mise uninstall aqua:anthropics/claude-code@2.1.280`.
- Only that old, unused version was removed. `claude` 2.1.286 (current) was unaffected.
- npm's debug log shows no lifecycle script calling mise, so the trigger is unconfirmed. It is probably mise pruning old `latest` installs, not the evaluated tools.

## 6. Recommendation and next steps

1. **Adopt the contract (section 2)** as the definition of "pluggable front end". Turn the spike into a repeatable smoke test (`chorus-hello` probe + throwaway repo + process and teardown checks).
2. **Firstmate becomes the default front end**, as `stack/packages/firstmate`:
   - `package.yml` installs treehouse and no-mistakes from their release tarballs, and the axi tools through mise's `npm:` backend in `home/.config/mise/conf.d/firstmate.toml`.
   - Set `FM_HOME=~/.local/share/firstmate` (XDG-style) and a `firstmate` launcher.
   - Default the backend to herdr, which is already packaged (section 8). Pin herdr to Firstmate's verified range, and start the herdr server from a launcher with the full ANFS PATH.
   - The remove hook runs `bin/fm-teardown.sh` and kills the primary.
   - Track upstream as a source clone, not a fork. Customize through `FM_HOME/config` and `.agents/skills` overlays.
3. **Paperclip is optional**, as `core/containers/paperclip` (pcm) on the shared postgres, with `PAPERCLIP_TELEMETRY_DISABLED=1` in `.env.schema`. It needs adapter work before it meets the contract:
   - a skills bridge (register psm's `~/.agents/skills`, or each source repo, as an approved skill source per company);
   - a worktree `executionWorkspacePolicy` by default;
   - a provision hook that bootstraps the board user.
   - Alternatively run it natively via a ppm package, so it uses ANFS harnesses instead of the copies bundled in its image.
4. **Chorus bridge (both tools):** a small script that turns a `plan.md` into a Firstmate brief (`## Captain's intent`) or a Paperclip issue, and writes `log.md` on completion. The inputs are the fleet ledger / status files and Paperclip issue status. Chorus stays the system of record for plans, and orchestrators stay replaceable.
5. **Security follow-up:** both tools run workers with approvals bypassed as your user. Running crewmates in pim VMs or pcm containers is where ANFS can add value that neither tool has locally: a Firstmate session backend, or a Paperclip sandbox-provider plugin.

## 7. Project goals (from their own sites, 2026-10-04)

| | Firstmate | Paperclip |
|---|---|---|
| Tagline | "Talk to one agent. Ship with a crew." | "The app people use to manage AI agents for work". Also: "Manage business goals not pull requests" |
| Problem it starts from | The author kept losing focus switching between parallel agent sessions | The creator was running more than 20 Claude Code windows for an automated hedge fund |
| Stated goal | An *experience*: "not a smart workflow, or a useful tool… to create an experience", namely "a sense of peacefulness that was taken away by the noisy agents" | "Autonomous companies" / "zero-human companies" with an org chart, budgets and governance. "If OpenClaw is an *employee*, Paperclip is the *company*" |
| Primary user | One developer, who is the captain | A builder or entrepreneur running business functions (marketing, ops, engineering) |
| Unit of work | A brief, then a branch or report | A goal, then issues, then heartbeats, then approvals |
| Ownership | "100% user-owned agent distro": you clone it, own it and fork it | Self-hosted control plane |
| Commercial | MIT; no company | MIT. The creator is pseudonymous with no disclosed company or funding. A third-party hosted service (paperclip.inc, €10/mo) closed on 2026-10-02 |
| Outside critique | (none notable) | "Productivity theater" (agents creating work for other agents), a "telephone" effect across delegation layers, and immaturity. It is praised as an orchestration and cost-control layer |

**How they differ:** Firstmate makes *one developer* more effective at software work and keeps the
human as the decision-maker. Paperclip aims to replace the human org with agents for business
work, with the human as a "board". For ANFS, an agent-native *development* stack, Firstmate's goal
is the one that matches. Paperclip's goal sits next to ANFS rather than inside it.

**Does Firstmate add real value going forward?** Yes, but the value is in its operating model, not
its code:
- one conversational supervisor;
- written briefs that separate intent from spec;
- explicit authority gates for merge and destructive actions;
- disk state that survives restarts;
- a worktree per task;
- supervision that spends no tokens.

These ideas survive any change of harness. The bash implementation is the part at risk:
- it scrapes vendor TUIs and is pinned to CLI versions;
- one author dominates its commits and its toolchain;
- harness vendors are shipping native multi-agent features (subagents, agent teams, cloud agents) that will overlap with it.

Adopt it behind the contract (section 2). Keep chorus as the system of record, and don't build
ANFS features that depend on Firstmate internals. Then you get the value now and can swap it
later.

## 8. Firstmate on herdr (ANFS `stack/herdr` package, herdr 0.8.2)

Setup:
- The ppm package was already installed, and the herdr Claude integration hook was in `~/.claude/hooks/herdr-agent-state.sh`.
- The test ran in an isolated named session (`herdr --session fm-eval`), with the first mate started via `herdr agent start … --kind claude` and prompted via `herdr agent prompt`.
- Same two tasks as the tmux run: ship 13 s, scout 66 s. That is no speed difference.

**What herdr added over tmux:**
- **Push events instead of screen polling.** Firstmate ran `bin/backends/herdr-eventwait.py`, subscribed to `pane.agent_status_changed`, so "worker is blocked/asking" arrives as an event.
- **Typed agent state.** `herdr agent list` gives idle/working/blocked/done per pane as JSON, and `herdr agent wait/prompt/read` makes the crew scriptable. This is useful to ANFS even without Firstmate, for example for the chorus bridge or a smoke test.
- **A visible task per workspace.** Each task got its own sidebar workspace (`└ demo-shout-flag-b2`), and Firstmate removed it on completion.
- **Clean teardown.** `herdr session stop` killed the first mate, the workers and the supervisor chain together. Under tmux, `kill-session` left the first mate running and its supervisor re-spawning (5.1).
- **Survives restarts.** After a herdr session restart, a fresh first mate reconciled the queued backlog from `FM_HOME` and ran it without being re-briefed.
- **Remote crews.** Firstmate's remote secondmates *require* herdr (`docs/remote-secondmates.md`). That is the route to crews on other machines or pim VMs (section 6, item 5).

**What it cost or exposed:**
- **Env comes from the herdr server, not the caller.** Panes get the server's PATH and environment. The first attempt failed because `treehouse` wasn't on the server's PATH. Starting the server from inside Claude Code also leaked `CLAUDE_CODE_*` variables into every pane. The first mate diagnosed the PATH failure correctly and stopped to ask rather than work around it. For ANFS this means the herdr server must be started from a login shell that has the full ANFS PATH, for example a launcher in the `stack/firstmate` package.
- **Version drift.** herdr 0.8.2 (protocol 20) is newer than Firstmate's verified range (0.7.1–0.8.0). It worked, but nothing guarantees it.
- **The worker's state came from screen matching, not the Claude hook integration.** `herdr agent explain` showed `rule: live_prompt_box`, from a remotely updated detection manifest. So herdr also reads the screen for Firstmate's workers; it does it better and centrally.
- **Rough edges:**
  - `agent start` failed silently once, because the login shell wasn't ready yet;
  - restarting the session restored a stale empty workspace;
  - there is an extra AGPL dependency, and `python3` is needed for the events.

**Verdict:** worth making the default backend for `stack/firstmate`. Speed is the same. The gains are
reliability (events, clean teardown), scriptable crew state, and the remote/VM path. Pin the herdr
version the package installs to Firstmate's verified range, and start the server from the package
launcher with a known environment.

Suggested chorus unit `orchestrators`, as an outline:
- `01-contract`
- `02-smoke-test`
- `03-firstmate-package`
- `04-chorus-bridge`
- `05-paperclip-pcm` (optional)
- `06-sandboxed-workers`
