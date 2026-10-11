---
status: complete
summary: ai/firstmate installs the repo at meta.firstmate.ref plus every tool fm-bootstrap.sh checks for, via existing packages and mise
started_at: 2026-10-07
completed_at: 2026-10-07
deviations: Read-only re-check of ref-bump and remove deferred to plan 04's round trip
---

- `package.yml`:
  - `depends: [ai/herdr, ai/claude, anfs/node, pde/git, pdt/python]`
  - `brew: [jq]`
  - `system: perl` (Debian) / `perl-Digest-SHA` (Fedora)
  - `meta.firstmate.{repo,ref}`
- `firstmate.toml`: `github:kunchenguid/treehouse` 3.1.2 and `github:kunchenguid/no-mistakes` 1.84.0. The mise `github:` backend works with their release asset names. `npm:` tasks-axi 0.2.6, quota-axi 0.1.59, gh-axi 0.1.35, chrome-devtools-axi 0.1.39.
- `install.sh`:
  - `post_install` clones if the repo is absent, fetches only if the ref is unknown, then does a detached checkout of the ref. The ref is read with yq from `$pkg_dir/package.yml`.
  - `pre_remove` runs `firstmate down`.
  - `post_remove` removes `~/.local/share/firstmate`.
- macOS host results:
  - HEAD is at 42dd906.
  - All 6 tools resolve through mise shims, and jq comes from brew.
  - `~/.claude/settings.json` and `~/.claude/hooks` are unchanged, so no axi hooks were installed.
- `fm-bootstrap.sh` reports only `PRESENTATION_UNAVAILABLE: lavish-axi` (lavish-axi was deliberately left out) and `NEEDS_GH_AUTH`.
