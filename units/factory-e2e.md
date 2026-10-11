---
objective: You can trust it — a repeatable, outcome-checked test proves spec/PRD → Firstmate → herdr → Claude worker → merged code, inside a container
status: pending
---

The end-to-end proof for the factory path. A test feeds a markdown task spec (Tier 1) or a
markdown PRD (Tier 2) to Firstmate running in a disposable Debian box. The test asserts on
*outcomes*: events in the fleet ledger, workers placed in herdr, branches that exist, acceptance
tests that pass, and merges that happen only after approval. It never checks model wording.

Depends on: `firstmate-foundation` unit complete.

## Decisions

- **Tests run in a container.** That reuses `core/packages/anfs/tests/harness` and its snapshots. Day-to-day work belongs in a VM (unit `factory-vm`), not here.
- **Auth uses a subscription OAuth token.**
  - Create it once with `claude setup-token` on the host and store it in 1Password.
  - `fnox exec -- tests/e2e` passes it into the box with `podman exec -e CLAUDE_CODE_OAUTH_TOKEN`.
  - It is never baked into an image or written to disk in the box.
  - Agents inside the box can read it, so it must be revocable.
- **Inputs are plain markdown** (no chorus):
  - `fixtures/spec-shout.md`: one task.
  - `fixtures/prd-greet.md`: several tasks, including one dependency.
- **Two tiers:**
  - Tier 1 is the gate: one task, mostly deterministic.
  - Tier 2 is the factory claim: the PRD is decomposed and run in parallel. It runs manually or overnight, and its pass rate is tracked.
- **Acceptance comes from bats tests in the fixture repo that fail at the start.** Code is "done" when they pass on `main`. Whatever the model says about its own work is ignored.
- **The authority gate is tested.** The first mate must ask before merging. The harness approves through `firstmate prompt`.

## Out of Scope

- VM runs (unit `factory-vm`).
- Harnesses other than Claude.
- GitHub or PR delivery modes (local-only only).

## Plans

1. `01-auth-and-box`: token injection, Claude onboarding pre-seed, box lifecycle for e2e
2. `02-fixtures`: `greet` fixture repo with failing bats tests, spec and PRD
3. `03-tier1-gate`: spec → ship → herdr placement → approval → merge, asserted
4. `04-tier2-factory`: PRD → parallel decomposition → dependency order → full suite green
5. `05-hygiene`: timeouts, failure dumps, `--keep`, pass-rate log

## Completion Criteria

- `fnox exec -- sources/stack/packages/firstmate/tests/e2e debian --tier 1` passes twice in a row from a cold box.
- `--tier 2` passes at least 2 times out of 3, with wall time and token usage recorded.
- A deliberately bad input produces a results dump under `tests/results/<ts>/` and a non-zero exit.
- No token value appears in any image layer, results dump or log.
