---
plan: agents/00-stack-go
job: stack-go
date: 2026-10-23
---

# Log: agents/00-stack-go

Go is for building `anfsd`, the daemon in `anfs-io/runtime`'s `go/` (agents/02).
It comes in through anfs-dev's `dev` package because only people working on anfs build it.
Users of anfs get prebuilt binaries and never need Go.

## What was done

- **stack `packages/go`** (new, 0.1.0, `categories: [pdt]`), modelled on `stack/rust` with
  `stack/ruby`'s `sh/` tier:
  - `.config/mise/conf.d/go.toml`: `go = "latest"` (pinned the way ruby and rust pin theirs), and
    `[env]` `GOPATH = ~/.local/share/go` (DATA, per core's XDG rule; never `~/go`) and
    `GOBIN = $GOPATH/bin`, so `mise exec` and a hook's `mise activate bash` get them too.
  - `.config/sh/go.sh`: the same two exports from `$XDG_DATA_HOME`, and `ensure_path "$GOBIN"`.
    GOBIN is GOPATH/bin, as ruby's local gems are `$XDG_DATA_HOME/gems/bin`; `~/.local/bin`
    stays anfs's.
  - `.config/nvim/lua/plugins/lsp/go.lua`: `gopls`, which mason installs (as rust's
    rust-analyzer config); so no `install.sh`.
  - No `brew:`/`system:`: mise downloads go.dev's binaries on macOS and Debian (and Fedora).
  - `README.md`, `tests/go.bats`.
- **anfs-dev `packages/dev/package.yml`**: `depends: [go]` (bare name, resolved across sources,
  as cnfs's rails depends on stack's ruby). The pre-commit hook bumps its patch.

## Tests

- `anfs work run stack-go stack -- bats packages/go/tests packages/ruby/tests </dev/null`:
  14/14. go.bats (temp HOME, `mise` a stub that fails and records any call): go.toml's tool and
  env lines; `go.sh` in bash and zsh sets GOPATH/GOBIN in DATA, GOBIN first on PATH, no mise
  call, nothing written to HOME; sourced twice, GOBIN on PATH once.
- `anfs work run stack-go anfs-dev -- bats packages/dev/tests </dev/null`: 38/38.
- Core's suite not run: nothing here is in core.

## By hand, after merge

```bash
ppm install stack/go     # or ppm install dev, which now pulls it in
exec $SHELL              # GOPATH, GOBIN and PATH from go.sh
go version
go env GOPATH GOBIN      # ~/.local/share/go, ~/.local/share/go/bin
```
