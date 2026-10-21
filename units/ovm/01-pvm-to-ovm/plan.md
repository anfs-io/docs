---
---

# Plan 01 — pvm leaves core as `ovm`, in `stack/obsidian`; vaults stop being a resource

Written 2026-10-21 against main

## Context — read these files first

The job is `anfs work new <job> core stack anfs-dev user docs`: stack for the tool, core for
what goes, anfs-dev for two lines of the round trips, user for `pjm`, docs for three documents.
Paths below are core's unless they name another repo.

- `units/ovm.md` — the inventory of every tie, the decisions, the open questions
- `packages/pvm/` — all of it: `home/.local/bin/pvm`, `home/.config/zsh/pvm.zsh`,
  `install.sh`, `package.yml`, `tests/{helper.bash,vaults.bats,links.bats,docs.bats}`
- stack `AGENTS.md` — "Tools shipped by packages" (the contract a stack tool meets, the keys
  table); stack `packages/ventoy/` (the model: `tests/ventoy.bats`' `setup`, `ANFS_TEST_LIB`);
  stack `packages/obsidian/` (what is there: nvim config, `obsidian.zsh`, `package.yml`)
- `packages/anfs/home/.local/bin/anfs` — the header comment (line 29), `install`'s help
  (line 89), `_anfs_tool vault pvm` (line 150), the rpc comment (line 618), `_rpc_run`'s
  allowlist (lines 676–677), `_rpc_version` (line 787)
- `packages/anfs/home/.config/sh/anfs.sh` — the `anfs` function's `case` (line 51)
- `packages/anfs/home/.local/lib/anfs/stacks.sh` — header (line 12), `STACK_KINDS`,
  `stack_tool`, `_stack_load`'s keys, `stack_install`'s `vaults` array and its two notes
- `packages/anfs/home/.local/lib/anfs/callbacks.sh` — the example in the header (line 27)
- `packages/anfs/tests/{registry,stacks,bash,system,callbacks,rpc}.bats` — the lines in
  "Test Spec"
- `containers/ui/lib/anfs/kinds.rb` (the `vaults` and `links` kinds), `spec/anfs/kinds_spec.rb`,
  `spec/anfs/rpc_spec.rb`; `views/overview.rb` reads `sourced`
- `stacks/system/stack.yml` — the comment
- `AGENTS.md` (core's) — "Stacks", "Repository Layout", "The toolkit API", "Callbacks",
  "Lists and --json", "The Command Line", "Bash" (the lines are in section 4)
- anfs-dev `packages/dev/tests/roundtrip` (lines 4, 145), `roundtrip-macos` (line 124)
- user `packages/pjm/` — `package.yml`, `home/.config/pjm/config.yml`, `home/.local/bin/pjm`
  (comments at 5 and 211)
- linker plan `units/linker/01-link-library/plan.md` section 3, last bullet: the one place it
  names pvm (another job, `linker-01`, is running it; see `units/ovm.md`, "linker")

## Overview

`pvm` becomes `ovm` and moves, whole, from `core/packages/pvm` into `stack/packages/obsidian`.
Its behaviour does not change beyond the name: marker `ovm.yml`, state `$XDG_STATE_HOME/ovm`,
`OVM_*` variables, `CLI_TOOL=ovm`. Then every place core treats a vault as a resource goes:
`anfs vault`, `vaults:` in a stack, `pvm` on the toolkit API, the two anfs-ui panels, the docs.
`user/pjm` follows the rename. A one-time `post_install` step carries an existing pvm install
over.

What a user sees: `ovm` where `pvm` was, `ppm install obsidian` to get it, `anfs vault` gone, a
stack with `vaults:` refused (64), anfs-ui without the Vaults and Vault links panels.

## Implementation

### 1. ovm in `stack/packages/obsidian`

- `git mv` is across repos, so: copy `core/packages/pvm/home/.local/bin/pvm` to stack's
  `packages/obsidian/home/.local/bin/ovm`, and the tests to `packages/obsidian/tests/`, then
  delete `core/packages/pvm` in core. One commit per repo.
- In the script, rename: `pvm` → `ovm` in messages, help and usage (`CLI_TOOL=ovm`,
  `anfs_tool_paths ovm`); `PVM_*` → `OVM_*` (`OVM_VERSION`, `OVM_STATE_HOME`, `OVM_REGISTRY`,
  `OVM_LOCK`, `OVM_LINKS`, `OVM_CONFIG=ovm.yml`, `OVM_HOME`, `OVM_PROBE_SECONDS`, `OVM_REACH`,
  `OVM_LOCK_HELD`); `pvm.yml` → `ovm.yml` everywhere, the template `init` writes included;
  temp-file prefixes `pvm-show`, `pvm-id` → `ovm-*`. The header says "Obsidian Vault Manager"
  and that a vault may be any directory of markdown, Obsidian's or not. `OVM_VERSION=0.2.0`.
- `_wsm_load`'s 69 says `wsm is required for links (ppm install core/wsm)`, unchanged.
- Nothing else in the script changes: commands, options, aliases, exit codes, list headers and
  keys (`ovm list`: `name id status path`; `link list`: `source target as status link vault`;
  `docs`: `id type title space path vault`; `validate --json`: `path finding`), registry and
  links file formats, the auto-register rule, `complete_vault`, `complete_spec`.
- `home/.config/zsh/ovm.zsh`: pvm.zsh renamed (`command -v ovm`, `zcomp ovm`). obsidian.zsh is
  left as it is.
- `package.yml`: `version: 0.2.0`, keep `categories: [pde]`, add `depends: [wsm]` with pvm's
  comment. No `yq`: it is anfs's base software, and no stack package declares it.
- `install.sh` (new for this package): pvm's, renamed: `post_install` registers
  `anfs_callback_register wsm "$PPM_CURRENT_PACKAGE" "ovm link apply --event"`, then runs the
  migration (below), then the `user_message`; `post_remove` unregisters and says what is left
  (`$XDG_STATE_HOME/ovm`, the `ovm.yml` files, the links).
- **The migration**, a function in `install.sh`, idempotent and quiet when there is nothing:
  when `$XDG_STATE_HOME/pvm` is a directory and `$XDG_STATE_HOME/ovm` does not exist, `mv` it;
  then for each line of `ovm/vaults` (`id<TAB>name<TAB>added<TAB>path`) whose path answers within
  3 s (the same `perl -e 'alarm ...; stat'` probe as `_reachable`) and has `pvm.yml` and no
  `ovm.yml`, `mv pvm.yml ovm.yml` and print one line per vault. A vault that does not answer is
  named with "rename its pvm.yml to ovm.yml by hand". `links` keeps its format: its paths and
  vault ids are unchanged. If both state directories exist, nothing is moved and the message
  says so.
- `README.md` for the package (stack's packages have one when they ship a tool): what a vault
  is, the commands, the links, the migration and its manual form.
- stack `AGENTS.md`, "Tools shipped by packages": `ovm` (obsidian) joins the "On it" list, and
  the keys table gets `ovm list`, `ovm link list`, `ovm docs`, `ovm validate`.

### 2. core: vaults out of anfs

- `bin/anfs`: delete `_anfs_tool vault pvm "Secrets"`; the header's tool line ends at
  `space (wsm)`; `install`'s help drops "vaults,". The rpc: `run[0]` is `anfs` or one of
  `ANFS_TOOLS` (the comment, the check at 676, its error `not a toolkit tool: x (anfs ppm pcm pmm
  psm wsm)`), and `_rpc_version` loops over `$ANFS_TOOLS` only.
- `sh/anfs.sh`: delete the `vault|pvm` case. `anfs vault ...` then reaches `command anfs`, which
  answers as for any unknown command (64), since no `anfs-vault` exists.
- `stacks.sh`: `STACK_KINDS="packages skills spaces containers machines"`; `stack_tool` and
  `_stack_load`'s `case` lose `vaults` (so `vaults:` is the existing "unknown key" 64, its
  message built from `STACK_KINDS`); `stack_install` loses the `vaults` array, the `continue`
  and both notes; the header example loses `vaults:`.
- `callbacks.sh`: the header example becomes `stack/obsidian`'s (`"ovm link apply --event"`).
- `stacks/system/stack.yml`: the comment reads "Not pmm, pmm-network or qemu".
- `packages/anfs/package.yml`, `README.md`, `packages/*/README.md`: no mention today (checked);
  check again on the branch.

### 3. anfs-ui (`containers/ui`)

- `lib/anfs/kinds.rb`: delete the `vaults` and `links` kinds. Keep `sourced: false` in `Kind`
  (the comment's "lists that are not of a source's resources" still describes a valid choice;
  no kind uses it after this, which `views/overview.rb`'s `sourced == true` tolerates). If
  rubocop or a spec objects to an unused value, drop it with its comment.
- Statuses only vaults and links used (`pending`, `dangling`, `conflict`, `unknown`) stay in
  `TONES` only if another kind can print them; check each against the remaining tools' lists
  (`dangling`/`conflict`/`pending` are wsm's too) and remove none that another kind uses.
- `spec/anfs/kinds_spec.rb`: drop `vaults`/`links` from the command map, the `links` source
  case and the `vaults` status case. `spec/anfs/rpc_spec.rb` line 60: `pvm` → `psm` (it only
  needs a second tool whose answer is a refusal).

### 4. core `AGENTS.md`

- "Stacks": the manifest example (line 48) and both key lists (57, 65) lose vaults; the
  "**vaults** are declared and shown" bullet (94–95) goes; `--build`'s "never a vault" (96–97)
  goes; "pmm, qemu and pvm are not in it" (79) → "pmm and qemu".
- "The toolkit API": "one of `ANFS_TOOLS` or `pvm`" (550) → "one of `ANFS_TOOLS`".
- "Repository Layout": `packages/{pcm,pmm,psm,wsm,pvm}` (615) and "Not pmm, qemu or pvm" (619).
- "Callbacks" (850): the example is `stack/obsidian`'s `ovm`.
- "Lists and --json": "pvm reads `wsm ls --json`" (878) → "stack's ovm reads"; rule 5's list
  (903): "pvm's" → "stack's ovm's".
- "The Command Line": the `anfs <resource>` table loses its `anfs vault` row (1005); "each of the
  six" (1012) → "five"; the verbs example `pvm link remove` (1039) → `wsm repo remove` or
  another core one; the completion example "pvm's `--to` runs `wsm __complete install ""`"
  (1102) stays as an example, as "stack's ovm's `--to`".
- "Bash": "the six tools" (1382) → "the five tools".
- Leave the 1Password `--vault` lines (1267, 1270) alone.

### 5. anfs-dev, user, docs

- anfs-dev round trips: `roundtrip` line 4 "(no pmm, qemu or pvm)" → "(no pmm or qemu)"; the
  check at 145 and `roundtrip-macos` 124 test `ovm` and `stack/obsidian` instead of `pvm` and
  `core/pvm` (a default install has neither). Not run here (deferred).
- user `packages/pjm`: `depends: [obsidian, fnox]` (with the comment: ovm, in stack's
  obsidian package); `config.yml`'s `[pvm, items, ...]` lines and the `pvm.yml` comment → `ovm`;
  `bin/pjm`'s two comments. `items` stays a command ovm does not have (the open question in
  `units/ovm.md`); say so in `log.md`. Bump pjm's version.
- docs: `units/stacks.md` (line 8 drops `vaults/` pvm; the kind lists at 40 and 61 drop
  vaults; add one line: vaults left the kinds in `ovm/01`); `BACKLOG.md` line 31 (the "pvm is a
  core tool now" note) is replaced by "pvm left core as stack's `ovm` (`units/ovm.md`)";
  `ROADMAP-apps.md` line 16 drops "vaults" from the primitives, and lines 41 and 102 (cloud
  credentials kept somewhere "after pvm leaves core") are removed: pvm was never for secrets
  (decided 2026-10-21); credentials are `op`/`fnox`'s. `ROADMAP.md` is not touched. Old plans and logs keep pvm:
  they are history.
- linker's plan 01 is another job's; do not edit it. `log.md` notes its pvm line.

## Test Spec

ovm, in stack (`packages/obsidian/tests/`):

- pvm's three suites, renamed (`vaults.bats`, `links.bats`, `docs.bats`, 52 tests) pass with
  `ovm`, `ovm.yml` and `$XDG_STATE_HOME/ovm`. `helper.bash`: `ovm_setup` takes the libraries
  from `ANFS_TEST_LIB` (stack's convention, as `ventoy.bats`), not `PVM_TEST_ANFS_LIB`;
  `REGISTRY` is `$XDG_STATE_HOME/ovm/vaults`. The `wsm` stub is unchanged.
- No test may mention `pvm` except the migration's.
- `install.bats` (new), sourcing `install.sh` with `anfs_callback_register`,
  `anfs_callback_unregister` and `user_message` stubbed:
  - `post_install` registers `ovm link apply --event` for `wsm` under `$PPM_CURRENT_PACKAGE`;
    `post_remove` unregisters.
  - Migration: a `pvm/` state with two vaults, one with `pvm.yml`, one already `ovm.yml` →
    state moved, the first renamed, the second untouched, ids and `links` unchanged; a second run
    does nothing and says nothing; both `pvm/` and `ovm/` present → nothing moved, said once;
    no `pvm/` → silent; a vault whose path is gone is skipped with its manual line;
    `OVM_PROBE_SECONDS=0`-style stub for an unreachable one is optional (the probe is pvm's,
    already tested).
- After the migration, `ovm list` shows the vault `ok` and `ovm link list` the link `ok`.

core (`packages/anfs/tests/`):

- `registry.bats`: "a tool that is not installed" drops its `anfs vault` half; a new case:
  `anfs vault list` exits 64 as an unknown command; help's Tools line has `skill, psm` and
  `space, wsm` and no `vault`.
- `stacks.bats`: the fixture manifest (line 45) loses `vaults:`; the two vault assertions (172,
  228–232) go; a new case: a `stack.yml` with `vaults: [x/y]` is 64, "unknown key 'vaults'".
- `bash.bats`: `pvm` leaves the entry-point list (line 83).
- `system.bats`: the `*pvm*` negative (line 91) goes (nothing can bring pvm now).
- `callbacks.bats`: the fixture `user/pvm` (167, 176) becomes `stack/obsidian`; any name works.
- `rpc.bats`: the comment at 50 says "a tool's `$(yq ...)`"; a new line in the "refused" case
  (86): `{"run":["pvm","list"]}` is `not a toolkit tool`.
- `isolation.bats` and `registry.bats`' table-header check scan core's packages; with pvm gone
  they have less to scan and should pass unchanged.

anfs-ui: `bundle exec rspec` and `bundle exec rubocop` pass with the two kinds gone.

user: `bats packages/pjm/tests` passes (its fixtures have `vault` fields, which are pjm's item
keys and stay).

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that
boots a VM or a box runs, and nothing is installed on this host (no `ppm install obsidian`, no
migration on the real `~/.local/state/pvm`).

- [ ] While working, the file you touch: in stack, `env ANFS_TEST_LIB=<job>/core/packages/anfs/home/.local/lib/anfs
      bats packages/obsidian/tests/<file>` (`anfs work path <job> core` gives the path; the
      job's core, so the tests run against the branch's libraries)
- [ ] stack: `bats packages/obsidian/tests` with that `ANFS_TEST_LIB`, then with the installed
      libraries (no `ANFS_TEST_LIB`): ovm needs nothing of the branch's core
- [ ] core: `bats packages/anfs/tests/{registry,stacks,bash,system,callbacks,rpc}.bats`
- [ ] core, anfs-ui: `anfs work run <job> core -- sh -c 'cd containers/ui && bundle exec rspec
      && bundle exec rubocop'`
- [ ] user: `bats packages/pjm/tests`
- [ ] `grep -rn 'pvm' packages stacks containers install.sh AGENTS.md README.md` in core:
      nothing; `grep -rn 'vault' ...` the same: only onboard's 1Password and wsm's `opm.yml`
      fixture. In stack: only the migration and its test, and the README's migration note.
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm remove user/pvm` (its source is gone,
  so if the hook cannot run, unregister by hand: `anfs callbacks wsm` must not list `user/pvm`),
  delete the dangling `~/.local/bin/pvm`, `ppm install obsidian`, then `ovm list` (apex `ok`),
  `ovm link list` (the `anfs/` link `ok`), `anfs callbacks wsm` (`stack/obsidian` `ok`).
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian`, `roundtrip-macos`
  with the changed checks.
