---
status: complete
summary: firstmate launcher (up/attach/status/prompt/note/project add/doctor/down) drives a full note → ship → approve → local merge on the macOS host
started_at: 2026-10-07
completed_at: 2026-10-07
deviations: Trust is written directly to ~/.claude.json, because fm-claude-trust.sh --lab-home refuses any home without fm-lab-home.sh's marker. `down` also deletes the session, so a restart doesn't restore a stale captain workspace.
---

Run on the macOS host against a throwaway repo:

- **doctor:** exit 0 (repo, herdr 0.8.2, bootstrap info lines only, claude auth).
- **up --headless:** works, and running it twice does nothing extra (one agent). The herdr server env has 0 `CLAUDE_CODE_*` variables. The default herdr session was untouched throughout.
- **project add:** clones the repo and registers it as `- fm-demo [local-only] …`.
- **prompt:** a scout was dispatched, then status went `done` in about 90s. `status` showed it through `fm-crew-state.sh`.
- **note spec.md:** queued as a durable inbox note, followed by a nudge.
  - About 60s later the ship was `ready in branch fm/fm-demo-shout-s2`.
  - The first mate asked "should I merge…? I won't merge until you say so".
  - `prompt "Approved…"` produced ledger `task.merged {via: local}` and then `task.cleaned_up`, and `main` advanced.
  - The ledger fields used were `event` and `task`, with `via` on `task.merged`.
- **down:** no processes reference `FM_HOME`, the session is deleted, and default is still running.
- **Leftover:** `~/.treehouse/<project>-<hash>`, treehouse's reusable worktree pool, which lives outside `FM_HOME`. Plan 04 decides whether `post_remove` should prune it or `IGNORE_RE` should allow it.
- **Gotcha for test scripts:** in zsh, an unmatched glob such as `state/*.status` before the first task exists aborts the command. Run waits under bash, or use `nullglob`.
