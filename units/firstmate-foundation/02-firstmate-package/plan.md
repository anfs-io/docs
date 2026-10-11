---
---

# Plan 02: `ai/firstmate` package

## Context: read these files first

- `sources/anfs/CLAUDE.md`, the "Packages" section: `package.yml` keys, install.sh hooks, mise `conf.d`, `meta`.
- `sources/anfs/packages/psm/install.sh`: reads `meta` with `yq` (`_psm_package_meta`).
- `sources/anfs/packages/ppm/home/.local/lib/ppm/installer.sh`: hooks are sourced with `$pkg_dir` in scope.
- `sources/anfs/packages/node/home/.config/mise/conf.d/node.toml`: an example of the `npm:` backend.
- `~/evals/firstmate/bin/fm-bootstrap.sh`: the tool list (`COMMON_TOOLS`, plus `herdr jq treehouse` for the herdr backend) and the version floors (`NO_MISTAKES_MIN=1.46.0`, `GH_AXI_MIN=0.1.29`).
- `sources/anfs/orchestrators-eval.md` §5 and §8: spike findings.

## Overview

Create `sources/ai/packages/firstmate/`. It installs the Firstmate repo at a pinned ref plus every
tool Firstmate checks for at startup, using existing packages and mise instead of custom
downloads.

## Implementation

### `package.yml`

```yaml
version: 0.1.0
author: rjayroach
depends: [ai/herdr, ai/claude, anfs/node, pde/git, pdt/python]
brew: [jq]
system:
  debian: [perl]
  fedora: [perl-Digest-SHA]
meta:
  firstmate:
    repo: https://github.com/kunchenguid/firstmate
    ref: 42dd906d02f672652606057a193a6386b956c3f3   # 2026-10-04, the commit evaluated
```

### `home/.config/mise/conf.d/firstmate.toml`

```toml
# firstmate: tools fm-bootstrap.sh checks for. setup hooks are deliberately never run.
[tools]
"github:kunchenguid/treehouse" = "3.1.2"
"github:kunchenguid/no-mistakes" = "1.84.0"
"npm:tasks-axi" = "0.2.6"
"npm:quota-axi" = "0.1.59"
"npm:gh-axi" = "0.1.35"
"npm:chrome-devtools-axi" = "0.1.39"
```

### `install.sh`

```bash
FIRSTMATE_REPO_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/firstmate/repo"

_fm_meta() { yq -r ".meta.firstmate.$1" "$pkg_dir/package.yml"; }

post_install() {
  local repo ref
  repo=$(_fm_meta repo) ref=$(_fm_meta ref)
  if [[ ! -d "$FIRSTMATE_REPO_DIR/.git" ]]; then
    git clone --quiet "$repo" "$FIRSTMATE_REPO_DIR" || { ppm_fail "firstmate: clone failed"; return; }
  fi
  git -C "$FIRSTMATE_REPO_DIR" fetch --quiet origin \
    && git -C "$FIRSTMATE_REPO_DIR" checkout --quiet --detach "$ref" \
    || ppm_fail "firstmate: cannot check out $ref"
}

pre_remove()  { command -v firstmate >/dev/null && firstmate down || true; }
post_remove() { rm -rf "${XDG_DATA_HOME:-$HOME/.local/share}/firstmate"; }
```

Notes:
- The checkout is detached at the pinned ref, so a reinstall after a `ref` bump moves it.
- `FM_HOME` lives under `$XDG_DATA_HOME/firstmate/home` (owned by plan 03), so `post_remove` also drops operator state. Say so in the README.

### `README.md`

Cover:
- What it installs.
- How to bump `ref`.
- That the axi `setup hooks` are skipped on purpose.
- That herdr version drift is caught by the `factory-e2e` unit.

## Test Spec

- macOS host: `ppm install ai/firstmate`, then:
  - `git -C ~/.local/share/firstmate/repo rev-parse HEAD` equals `ref`;
  - `mise which treehouse no-mistakes tasks-axi`;
  - `FM_HOME=$(mktemp -d) ~/.local/share/firstmate/repo/bin/fm-bootstrap.sh` prints no `MISSING:` lines (`NEEDS_GH_AUTH` is acceptable).
- Change `ref` to an older commit and reinstall. HEAD moves.
- `ppm remove ai/firstmate`: the repo dir is gone, and `mise ls` no longer lists the tools after the conf.d unstow.

## Verification

- `~/.claude/settings.json` and `~/.claude/hooks` are unchanged after install (no axi hooks).
- `ppm show ai/firstmate` lists the stowed `firstmate.toml`.
