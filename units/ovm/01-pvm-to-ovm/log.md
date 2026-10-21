---
plan: ovm/01-pvm-to-ovm
job: ovm-01
date: 2026-10-21
---

# Log: ovm/01-pvm-to-ovm

## What was built

- **`ovm` in `stack/packages/obsidian`** (obsidian 0.2.0): pvm's script, completion and tests,
  renamed. `pvm` → `ovm` and `PVM_*` → `OVM_*` throughout (`OVM_VERSION=0.2.0`, `CLI_TOOL=ovm`,
  `anfs_tool_paths ovm`, `OVM_CONFIG=ovm.yml`, the `init` template, the `ovm-show`/`ovm-id` temp
  prefixes). The header says "Obsidian Vault Manager" and that any directory of markdown is a
  vault. Nothing else changed: commands, options, exit codes, list keys, registry and links
  formats. `_wsm_load`'s hint still says `ppm install core/wsm`.
  - `home/.config/zsh/ovm.zsh` (`zcomp ovm`); `obsidian.zsh` untouched.
  - `package.yml`: `depends: [wsm]` with pvm's comment; no `yq`.
  - `install.sh` (new): `post_install` registers `ovm link apply --event` as a wsm listener and
    gives the usage message; `post_remove` unregisters and says what is left. **No migration**
    (see Decisions): nothing in stack names pvm.
  - `README.md` (new): ovm and its links into spaces.
  - stack `AGENTS.md`: `ovm` (obsidian) on the "On it" list; `ovm list`, `ovm link list`,
    `ovm docs`, `ovm validate` in the keys table.
- **core: vaults out of anfs.** `packages/pvm` deleted. `bin/anfs`: no `_anfs_tool vault pvm
  "Secrets"`, the header's tool line and `install`'s help without vaults; the rpc runs `anfs` or
  `ANFS_TOOLS` only (the comment, the check, its error, `_rpc_version`). `sh/anfs.sh`: no
  `vault|pvm` case. `stacks.sh`: `STACK_KINDS` without vaults, `stack_tool`, `_stack_load`'s
  keys, `stack_install`'s `vaults` array and both notes gone, the header example too.
  `callbacks.sh`'s example is ovm's. `stacks/system/stack.yml`'s comment: "Not pmm, pmm-network
  or qemu". `AGENTS.md`: every line in the plan's section 4.
- **anfs-ui**: the `vaults` and `links` kinds deleted. `Kind`'s `sourced: false` kept (no kind
  uses it now; rubocop and the specs do not object). TONES loses `pending`, `unknown`,
  `conflict` and `dangling`: checked against every remaining kind's list (ppm, pcm, pmm, pmm
  image, wsm, wsm repo, psm, anfs, anfs src), none prints them. (The plan guessed wsm prints
  `dangling`/`conflict`/`pending`; it does not, in any list.) Specs follow; `rpc_spec`'s refusal
  case uses `psm`.
- **anfs-dev**: `roundtrip` line 4 "(no pmm or qemu)"; the system checks in `roundtrip` and
  `roundtrip-macos` test `! command -v ovm` and no `stack/obsidian` installed.
- **user/pjm** (0.1.1): `depends: [obsidian, fnox]`, the comment says ovm is in stack's
  obsidian package and has no `items` yet; `config.yml`'s sources call `ovm items`, the comment
  names `ovm.yml`; `bin/pjm`'s two comments say `ovm items --json`. **`items` is still a command
  ovm does not have**: pjm cannot pull, exactly as before with pvm (open question 1 of
  `units/ovm.md`, plan 02).
- **docs**: `units/stacks.md` (no `vaults/` pvm, no vaults in the kind order or `--build`'s list,
  one line that vaults left the kinds in `ovm/01`); `BACKLOG.md` line 31 replaced by "pvm left
  core as stack's `ovm`"; `ROADMAP-apps.md`: no vaults among the primitives, and the two credential
  lines removed (see Decisions).

## Commits

core (ovm-01):
- `9861059` pvm leaves core: it is stack's ovm now (stack/obsidian)
- `e2db220` anfs-ui: no Vaults or Vault links panels (the hook bumped ui to 0.1.2)
- `2fc5ead` anfs: vaults are not a resource (anfs 0.2.13, stacks/system 0.1.1)

stack (ovm-01):
- `72307b1` obsidian: ovm, the Obsidian Vault Manager (pvm from core, renamed) (obsidian 0.2.0)
- `14b6efa` obsidian: no migration from pvm (no backward compatibility)

anfs-dev (ovm-01):
- `c3c1928` dev: the round trips check for no ovm (stack/obsidian), not pvm (dev 0.1.13)

user (ovm-01):
- `0747fec` pjm: on ovm (stack/obsidian), not pvm (pjm 0.1.1)

docs (ovm-01):
- `1651224` ovm/01: vaults out of the stack kinds and the apps roadmap; pvm's credential lines go
- `2495b44` this log
- the migration reversed in `units/ovm.md` and the plan, and this log updated

## Decisions taken during the work

- **No migration code (decision 3 reversed by the person, 2026-10-21, during the job).** The
  first version had `_ovm_migrate` in `post_install` (state moved, `pvm.yml` renamed per
  registered vault, one line each) and 5 tests for it (`72307b1`); `14b6efa` removes the
  function, its tests and the README's "From pvm" section. The person's one vault (apex) is
  moved by hand after merge (below). `units/ovm.md` decision 3 and the plan say so now.

- **ROADMAP-apps.md 41 and 102, verified before deleting**: 102 is the "Still open" row "Where
  cloud credentials are kept after `pvm` leaves core"; 41 is the second line of the
  **Credentials** bullet ("where they are kept once `pvm` leaves core"). Deleting 41 alone would
  leave the bullet's first line ending in "; where", so the bullet now ends at "as containers'
  are." Nothing names op/fnox there: the decision (credentials are op/fnox's) is in
  `units/ovm.md`.
- **registry.bats' rule-5 exemptions**: pvm's three headers (`NAME ID STATUS PATH`, `ID TYPE
  TITLE SPACE PATH VAULT`, `PATH FINDING`) are gone from the test's `others` list: no core tool
  prints them now. ovm's lists are stack's, outside that check.
- **AGENTS.md's verbs example**: `pvm link remove` became `anfs src remove` (the plan suggested
  `wsm repo remove`, which does not exist).
- **AGENTS.md's "the six tools"** (Bash, entry points) became "the five tools" as the plan says;
  with anfs that list is six entry points, which the sentence already meant before pvm came in.
- `user/pjm`'s `projects:` example (`#  user/dev: pvm`) is left: it maps a space to a Firstmate
  project that happens to be called `pvm`, not a call to the tool.

## Deviations

- **The grep criteria are not literally empty.** `grep -rn 'pvm\|vault'` in core's packages
  finds, besides onboard's 1Password and wsm's `opm.yml` fixture, the plan's own new negative
  tests: `registry.bats` (`anfs vault list` and `anfs pvm list` are 64; help names no vault or
  pvm), `stacks.bats` (`vaults:` is 64), `rpc.bats` (`{"run":["pvm","list"]}` is refused). In
  stack, `grep -rn pvm` finds nothing.

## Tests

All through the job (`anfs work run ovm-01 <repo> -- ... </dev/null`).

- stack `bats packages/obsidian/tests`: **53 passed** (pvm's 52 renamed + 1 in `install.bats`:
  the listener registered by `post_install`, unregistered by `post_remove`), 0 skipped, both with
  `ANFS_TEST_LIB=<job>/core/packages/anfs/home/.local/lib/anfs` and with the installed libraries
  (ovm needs nothing of the branch's core). Before the migration was removed: 58 (with its 5).
- stack `grep -rn pvm` (outside `.git`): nothing.
- core `bats packages/anfs/tests/{registry,stacks,bash,system,callbacks,rpc}.bats`: **117
  passed**.
- core anfs-ui (`containers/ui`): `bundle exec rspec` **63 examples, 0 failures**; `bundle exec
  rubocop` **29 files, no offenses**. Bundler had everything (`bundle check`); nothing installed.
- user `bats packages/pjm/tests`: **13 of 14**. `pjm.bats` 12 "dispatch sends only what is ready
  and tagged" fails at line 155 (`pjm dispatch </dev/null` expected 1). **It fails on the base
  commit too** (checked with the changes stashed); pjm's dispatch code is not touched here.
- **Once, at the end**: `packages/anfs/tests/all`: **342 passed, 1 failed**, 25 files (pmm and
  pcm off), 49 s. The failure is wsm `manage.bats` 4 "repo ls lists the repositories space.yml
  files declare", which fails on main (its `--json` keys gained `ahead`); not touched.

## Not run

- pmm's and pcm's suites (off since 2026-10-19; nothing of theirs changed).
- Deferred to the end of the roadmap (boots a VM): `roundtrip debian` and `roundtrip-macos` with
  the changed "no pmm, qemu or ovm" checks.
- Nothing installed on this host: no `ppm install obsidian`, nothing moved in the real
  `~/.local/state/pvm`, no `ppm remove user/pvm`.

## Merging

- `linker-01` also edits core `AGENTS.md` and `packages/anfs/tests/bash.bats` (different lines),
  and its commits bumped `packages/anfs` to 0.2.13 and anfs-dev's `packages/dev` to 0.1.13, the
  same numbers this branch's hook chose: whichever merges second rebases and takes the next
  version.
- linker's plan 01 (section 3, last bullet) and its log still say pvm (`pvm link`, the stale
  `user/pvm` tracker). Not edited here: another job's. After both merge, the word is ovm's.

## Open questions

1. **`ovm items`** (unchanged from `units/ovm.md`): pjm's sources call `ovm items --json
   --space ...` and its config expects an `exclude` key in the vault's marker; ovm has neither.
   Was `items` a pvm feature lost in the move to core, or still to write? Plan 02 (a stub) waits
   on the answer.

## By hand, after merge (this Mac)

1. `ppm remove user/pvm`. Its source is gone, so if the hook cannot run, unregister by hand:
   `anfs callbacks wsm` must not list `user/pvm` (or `core/pvm`).
2. Delete the dangling `~/.local/bin/pvm` if it is still there.
3. `ppm install obsidian` (registers the wsm listener; nothing else).
4. Move the state: `mv ~/.local/state/pvm ~/.local/state/ovm` (ids, registry and `links` keep
   their formats).
5. In the apex vault (on iCloud): `mv pvm.yml ovm.yml`.
6. `ovm scan` (or `ovm scan <apex dir>`), then check: `ovm list` shows `apex` `ok`; `ovm link
   list` shows its link into `anfs/` as `ok`; `anfs callbacks wsm` lists `stack/obsidian` `ok`
   and nothing stale; `anfs vault` is 64.
7. anfs-ui: restart it (`pcm up ui`) to drop the two panels.
8. Tick `ovm/01-pvm-to-ovm` in `ROADMAP.md`; then `anfs work remove ovm-01`.
