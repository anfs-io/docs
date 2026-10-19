---
---

# Plan 01 — Stacks: the manifest and anfs's commands on the host

## Context — read these files first

- `units/stacks.md` — the decisions; this plan builds its "What a stack is", "Commands" and
  "Reversible" sections, for the host target only
- core `packages/anfs/home/.local/bin/anfs` — `cmd_install` (the loop over `ANFS_TOOLS`, the
  plan print and confirm), `cmd_implode`, the `cli_cmd` declarations
- core `packages/anfs/home/.local/lib/anfs/paths.sh` (`ANFS_TOOLS`, `anfs_tool_dir`),
  `sources.sh` (`GITSRC_DIRS`, `gitsrc_*`, how a `source/name` resolves), `resolve.sh`,
  `catalog.sh`, `table.sh`
- core `packages/ppm/home/.local/lib/ppm/packages.sh` — dependency resolution and the layering
  rule (a source depends only on same or lower priority): stacks use the same rule
- core CLAUDE.md: "Lists and --json", "The Command Line", the anfs section

## Implementation

### 1. `stacks/` is a resource directory

- `stacks` joins the directories a source may provide (`anfs src list` PROVIDES shows it). It is
  anfs's own, not a tool's: `ANFS_TOOLS` is unchanged, and stacks are not in `anfs src install`.
- A stack is `stacks/<name>/stack.yml`. Resolved like any resource: `source/name`, or a bare name
  through the priority order.

### 2. The manifest

```yaml
version: 0.1.0
description: ...
stacks: [core/system]
packages: [stack/ruby]
skills: [lgat/rails-conventions]
vaults: [lgat/dev]
spaces:
  - lgat/: { status: installed }
containers:
  - core/anfs-ui: { status: running }
machines: []
```

- Keys: `version`, `description`, and the resource keys `stacks packages skills vaults spaces
  containers machines`. Anything else is an error (`anfs show` and `install` exit 64 naming it).
- A resource entry is a string, or a one-key map whose value may hold `status:`. Validate
  `status` against the kind: containers and machines `running|stopped`; spaces
  `installed|cloned`. Other kinds take none.
- `stacks:` resolve recursively with ppm's layering rule; a cycle exits 1 naming the chain. The
  resolved order is depth-first, each stack once, dependencies before dependants.

### 3. Commands

| Command | What |
| --- | --- |
| `anfs list` (`ls`) | `STACK STATUS VERSION`: every stack in every source; STATUS `installed`, `partial` (a record whose resources are not all there), `-` |
| `anfs show <stack>` | the resolved resources by kind in install order, each with its declared and actual status, and which stack brought it in |
| `anfs install <stack>...` | prints the plan (as `cmd_install` does now), confirms (`-y`), installs kind by kind through the tools, then applies `status:` (`pcm up\|stop`, `pmm up\|stop`, wsm's `installed` vs `cloned`) |
| `anfs remove <stack>...` (`rm`) | the reverse, from the record (section 4), confirm, `-y` |
| `anfs path <stack>` | the stack's directory |
| `anfs src install <source>...` | today's `cmd_install`, moved under the `src` group unchanged |

- `--json` on `list` and `show` (keys `stack`, `status`, `version`; `show`: `kind`, `resource`,
  `declared`, `actual`, `from`).
- `-t/--target` is declared now and accepts only `host` (anything else exits 64 "plan 04").
- `--build` installs only packages and skills and records it; a stack with `vaults:` under
  `--build` still installs no vault (say so).
- `anfs install <source>` with a source name, not a stack: exit 64, "did you mean anfs src
  install <source>". No alias for the old form.
- Each tool is run once per kind with every resource of that kind (`ppm install -y a b c`), as
  `cmd_install` does today. A tool not installed for a kind the stack needs: exit 69 with
  `ppm install core/<tool>` before anything is installed (checked during the plan).

### 4. The record

`$ANFS_STATE_HOME/stacks/<source>/<name>.yml`, written as the install goes (so a failed install
leaves a record of what it did):

```yaml
stack: lgat/rails
version: 0.2.0
installed_at: ...
build: false
resources:
  - { kind: packages, resource: stack/ruby, from: lgat/rails, before: false }
  - { kind: packages, resource: stack/git, from: core/system, before: true }
```

- `before: true` when the resource was already installed before this stack's install (ask the
  tool: `ppm list --installed --json`, `pcm list --json`, `wsm list --json`, `psm list --json`,
  `pmm list --json`). Read the keys as `cli/01-one-shape` made them.
- `anfs remove <stack>` removes a resource only if `before: false` and no other record holds it.
  It lists what it removes and what it keeps (and why) before the confirm.
- A resource removed by hand since is skipped, not an error. The record is deleted at the end.
- Re-running `install` on an installed stack installs what is missing and updates the record; it
  never flips `before` from true to false.

### 5. Documentation

Core CLAUDE.md: a "Stacks" section (the manifest, the commands, the record and the remove rule),
`stacks/` in the resource-directories list and the key table of "Lists and --json"; the anfs
section's command list. The anfs skill if it lists commands.

## Test Spec

`packages/anfs/tests/stacks.bats`, with the tools stubbed (a fake `ppm`, `pcm`, `psm`, `wsm` on
PATH that log their argv and answer `list --json` from a file), no real installs:

- the manifest: unknown key 64; a bad `status` 64; a cycle 1 with the chain; dependency order
- `install`: the tools are called in kind order with all of a kind's resources at once; `status:`
  calls follow; the record has `before` right for a pre-installed resource; a missing tool 69
  before anything runs; `--build` calls only ppm and psm; a source name 64 with the hint
- `remove`: removes only `before: false` resources no other record holds; reverse order; the
  record is gone; a resource removed by hand is skipped
- `list` and `show`, terminal and `--json`; `registry.bats` keeps every header to the shape
- `anfs src install` does what `anfs install <source>` did (the existing tests move to it)

## Verification

- [ ] `packages/anfs/tests/all`
- [ ] By hand in an isolated HOME with stubbed tools: a two-level stack (`b` depends on `a`),
      install, list, show, remove `b`, `a`'s resources stay
