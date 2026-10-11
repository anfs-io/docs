---
---

# Plan 01: auth and the e2e box

## Context: read these files first

- `sources/stack/packages/firstmate/tests/roundtrip` (from `firstmate-foundation` plan 04): the box setup to reuse.
- `sources/core/packages/anfs/tests/harness`: `start` takes extra podman args after `--`. `exec` is a wrapper around `podman exec`.
- `sources/stack/packages/fnox` and `sources/user/packages/git/home/.config/fnox/gh.toml`: the fnox + `op://` pattern.
- `~/evals/firstmate/bin/fm-worker-account-lib.sh` (around line 65): worker-account pinning unsets auth variables, so don't enable it.

## Overview

Create `sources/stack/packages/firstmate/tests/e2e` with this plan's pieces: box setup, auth and
onboarding. Tiers come in plans 03–04.

## Implementation

1. **One-time, manual: create the token.**
   - Run `claude setup-token` on the host (Max plan).
   - Store the token in 1Password.
   - Add `sources/stack/packages/firstmate/tests/fnox.toml` mapping `CLAUDE_CODE_OAUTH_TOKEN = { value = "op://<vault>/claude-e2e/token" }`.
   - Document all of this in the README.
2. **Refuse to run without the token.** `tests/e2e` exits with a clear message unless `CLAUDE_CODE_OAUTH_TOKEN` is set.
3. **Every in-box command that can start Claude** runs as `podman exec -e CLAUDE_CODE_OAUTH_TOKEN -e DISABLE_AUTOUPDATER=1 …`, passing the variable by name only so the value never appears in argv or logs.
   - The herdr server is started by `firstmate up` inside that exec, so its panes inherit the variable.
   - Confirm in the box that `firstmate up`'s clean-environment start keeps the variable: add `CLAUDE_CODE_OAUTH_TOKEN` and `DISABLE_AUTOUPDATER` to the launcher's `env -i` allowlist.
4. **Pre-seed onboarding** before `firstmate up`. Write `~/.claude.json` with `hasCompletedOnboarding: true` and the theme set, merging if the file exists.
5. **Smoke test:** `claude -p "reply with OK"` in the box returns OK.

## Test Spec

- With the token: the smoke test passes.
- Without it: `tests/e2e` exits 2 and prints instructions.
- Run `grep -r` for the token value over `tests/results/` and over the output of `podman image save` for the box image. Both find nothing.

## Verification

- `log.md` records the token's scope and how to revoke it.
