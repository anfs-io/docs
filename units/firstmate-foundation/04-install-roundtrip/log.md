---
status: complete
summary: tests/roundtrip debian passes from a cold box in ~2.5 min — install, doctor, up, herdr detects the agent, idempotent up, down, no processes, remove, clean $HOME
started_at: 2026-10-07
completed_at: 2026-10-07
deviations: The $HOME baseline is taken after installing firstmate's dependencies, so the diff covers only firstmate. Added --keep and printing of failing-check output. The launcher's server start changed from a login shell to an allowlisted env (see below).
---

- `sources/ai/packages/firstmate/tests/roundtrip [--keep] [debian|fedora]`. It reuses the anfs harness and `debian-bootstrap`, and copies the anfs/ai/pde/pdt sources from `$ANFS_SOURCES_HOME`.
- Result: 12/12 checks ok and `$HOME` is clean. 164s from a cold box.
- **Bug found and fixed in the launcher.** `env -i … bash -lc 'exec herdr …'` failed in the box: a bash login shell there has neither brew nor mise on PATH, because no bash shell-integration package is installed (the host worked only because of zsh's login setup). The server now starts as `env -i PATH="$PATH" HOME … <allowlist> herdr --session firstmate server`. Re-checked on the host: 0 `CLAUDE_CODE_*` variables in the server env.
- **Bug found and fixed in the launcher.** An unauthenticated `claude` makes `herdr agent start` return `agent_not_ready`. The launcher now counts an existing `firstmate` agent as started instead of retrying into a name clash.
- `IGNORE_RE` additions, all state owned by dependencies, not firstmate:
  - `.local/state/gh` (gh `device-id`, written when `fm-bootstrap.sh` runs `gh auth status`)
  - `.config/herdr/{.plugins.lock,sessions}` (herdr)
  - `.claude*` (claude)
  - `.bash_history` (the interactive shells herdr opens for panes)
- A broken `meta.firstmate.ref` (all zeros) fails at "repo checked out at meta.firstmate.ref" with exit 1. The tool check also fails in that case: `ppm_fail` in `post_install` stops the rest of that package's install.
- **Not covered:** Fedora (not run), and the `~/.treehouse` worktree pool, which the round trip never creates because it dispatches no tasks. The pool is treehouse's and lives outside `FM_HOME`, so it's left alone by design. Note it for the e2e unit.
