---
objective: It works — `ppm install ai/firstmate` gives a working Firstmate-on-herdr on macOS and Debian, and `ppm remove` leaves `$HOME` clean
status: complete
---

Packages [Firstmate](https://github.com/kunchenguid/firstmate) as the default ANFS orchestrator
front end, running on the herdr backend. Background, spike results and the orchestrator contract
are in `sources/anfs/orchestrators-eval.md`. Chorus as an orchestrator is out of scope for this
unit.

Depends on: `ai/herdr`, `ai/claude`, `anfs/node`, `pde/git`, `pdt/python` packages (all exist).

## Decisions

- **Prefer existing packages.** Firstmate depends on existing ppm packages. A `system:`/`brew:`
  line is used only where no package exists: `jq` (brew) and `perl` with `shasum`
  (Debian/Fedora `system:`, since macOS ships both). curl needs no declaration:
  `anfs/install.sh` guarantees it on Linux and macOS ships it.
- **herdr installs through brew on every platform**, not through an `install_linux` hook. On
  Linux there may be no bottle, in which case brew builds it from source (Rust). brew can't pin
  versions, so the e2e unit catches herdr drift.
- **The Firstmate version is pinned in `package.yml` under `meta.firstmate.ref`.** `install.sh`
  reads it with yq from `$pkg_dir/package.yml`, the same way psm reads `meta.agent`. Bumping
  `ref` and reinstalling is how you upgrade.
- **Firstmate's tool dependencies are installed through mise:**
  - `github:` backend: treehouse and no-mistakes.
  - `npm:` backend: tasks-axi, quota-axi, gh-axi, chrome-devtools-axi.
  - lavish-axi is skipped (optional presentation boards).
  - The axi tools' `setup hooks` are **never run**, because they write global harness hooks.
- **A thin `firstmate` launcher owns the herdr session.** It starts the herdr server with an
  allowlisted environment: the caller's PATH plus a few named variables. In the spike, panes
  inherited the server's env, which caused a PATH failure and a leak of `CLAUDE_CODE_*` variables.
  A login shell was tried first, but in a box without the shell-integration packages it has
  neither brew nor mise on PATH. `firstmate down` uses `herdr session stop`,
  the only teardown proven to leave no processes behind.

## Out of Scope

- Driving Firstmate with an LLM in tests (unit `factory-e2e`).
- Running in a VM (unit `factory-vm`).
- Chorus integration and a plan→brief bridge.
- Paperclip.

## Plans

1. `01-herdr-linux`: herdr via brew on Linux
2. `02-firstmate-package`: `ai/firstmate` package (deps, pin, mise tools, install/remove hooks)
3. `03-launcher`: `firstmate` launcher (up/attach/status/down/prompt/note/project/doctor)
4. `04-install-roundtrip`: no-LLM install/up/down/remove round trip in a Debian box

## Completion Criteria

- `ppm install ai/herdr` works on Debian 13, and `herdr status --json` reports protocol ≥ 16.
- `ppm install ai/firstmate` on macOS and Debian installs the repo at the pinned ref plus every tool `fm-bootstrap.sh` checks for. `firstmate doctor` shows no `MISSING:` lines.
- `firstmate up --headless` leaves a `firstmate` agent visible in `herdr agent list`.
- `firstmate down` leaves no herdr, claude or `fm-*` processes.
- `ppm remove ai/firstmate` followed by a `$HOME` diff is clean.
- `tests/roundtrip debian` passes.
