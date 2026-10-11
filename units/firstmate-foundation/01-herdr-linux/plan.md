---
---

# Plan 01: herdr via brew on Linux

## Context: read these files first

- `sources/ai/packages/herdr/package.yml`: currently `brew: {macos: [herdr]}`.
- `sources/anfs/CLAUDE.md`, the "Packages" section: how `brew:` lists and per-platform maps resolve.
- `sources/anfs/packages/anfs/tests/harness`: test boxes (`start --from bootstrap`, `exec`, `snapshot`).

## Overview

herdr is in homebrew-core (Apache-2.0). Installing it through brew on every platform is enough;
no `install_linux` hook is needed. The local formula JSON shows only a macOS bottle, so on Linux
brew may build from source with Rust. The first install could take minutes, so time it and then
snapshot the box.

## Implementation

1. In `package.yml`, change `brew: {macos: [herdr]}` to `brew: [herdr]`.
2. Bump `version` to `0.1.1`.
3. Leave `home/` (config.toml, aliases) unchanged.

## Test Spec

In a Debian box started from the `bootstrap` snapshot, with sources mounted:

```
ppm install ai/herdr
herdr --version
herdr status --json | jq -e '.client.protocol >= 16'
nohup herdr server >/tmp/herdr.log 2>&1 &
herdr session list
ppm remove ai/herdr
```

## Verification

- `herdr` is on PATH in a fresh login shell in the box.
- Record the protocol, the version, and whether a bottle was used or brew built from source (with wall time) in `log.md`.
- Snapshot the box as `herdr` for later plans: `harness snapshot debian herdr`.
- macOS host: `ppm install ai/herdr` is a no-op apart from the version bump.
