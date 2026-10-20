# Roadmap: images, one API, deployment

Written 2026-10-13, after the pim audit. The order to take the units in, what each step needs
from you first, and what is still undecided. Update the Status column as steps finish.

The direction in one paragraph: `pmm` owns machines and their images (pim goes away), `pcm` gets
the same commands, the toolkit gets one API so apps ship as containers, a deploy tool puts
images on targets, and hardware installs from the same image definitions.

## The checklist

The master list: what is next, in the order it will be done. Tick a line when it is done. The
table below it says what each unit is; the plans say how.

### Then: tooling

- [x] `dev-source/02-work` — the `anfs <name>` seam, the tools as `anfs <resource>` commands,
      `anfs hooks`, `anfs work`. Merged into main in core, anfs-dev and docs 2026-10-19 (see
      `log.md`); `anfs-dev/dev` reinstalled here. VM and box checks deferred to "Last"
- [x] Two jobs at once through `anfs work`, its first real test (2026-10-19: 6 and 12 minutes, merged):
  - [x] `dev-source/01-move` — the rest of the move (the package and the docs space are done)
  - [x] `cli/01-one-shape` — psm without `skills`, `wsm repo`, every list in one shape
- [x] Remove the old worktrees under `~/anfs-dev` (merged) and the directory (2026-10-19; the
      leftovers archived in `~/.local/state/anfs-repair-2026-10-19/anfs-dev-leftovers.tgz`)
- [x] CLAUDE.md becomes AGENTS.md in core, stack and anfs-dev (2026-10-19); each CLAUDE.md is
      `@AGENTS.md`, since Claude Code reads only CLAUDE.md

### Then: stacks (`units/stacks.md`, decided 2026-10-19)

- [x] `stacks/01-stacks` and `stacks/03-container-manifest`, as two jobs at once (merged
      2026-10-19; see their `log.md`)
- [x] Small jobs once 01 and 03 are merged (2026-10-19, merged):
  - [x] `ppm list`: `PACKAGE STATUS VERSION` (`installed`, `outdated`, `-`), as `anfs list` is
        for stacks; its `--json` readers (01's stack record) in the same change
  - [x] stack's containers: each README keeps only what cannot be derived (why, first run,
        gotchas); data dirs, ports and image come from `compose.yml` through `pcm show`,
        description and attribution from `container.yml`
- [x] `stacks/02-system` — the toolkit is the `system` stack; implode is `anfs remove system`
- [ ] anfs-ui into core, before boxes (decided 2026-10-19):
  - [x] `containers/ui` in core (merged 2026-10-19): a Containerfile that builds the gems (`dev`
        target; `release` copies the code), the code mounted read-only from the source so it is
        developed on the host. Not in `system` yet (`pcm install` starts a service): `pcm up ui`
  - [x] `toolkit-api/01-rpc` — `anfs rpc` (merged 2026-10-19)
  - [x] `toolkit-api/02-socket` — the launchd / systemd socket, the token, pcm's `toolkit:` grant
        (merged 2026-10-19; loading it on this Mac is `ppm install core/anfs`, see its `log.md`)
  - [x] `toolkit-api/04-anfs-ui-client`, reduced: the app calls the API instead of running the
        tools; `status: running` in `system`; every panel on the current lists; `.env.schema` for
        where it listens (loopback by default). Merged 2026-10-19; for you to try (its `log.md`)
- [ ] `setup` (`units/setup.md`, decided 2026-10-19): onboarding becomes ordinary commands. `-i`
      asks for what is missing; a manifest's `setup:` lists what to run; offboarding is its reverse
  - [ ] `setup/01-interactive` — `-i, --interactive` on every command
  - [ ] `setup/02-setup` — `setup:` in `stack.yml`, the runner, its record and reverse — plan to write
  - [ ] `setup/03-steps` — the onboard steps as `add` commands; `anfs onboard` goes — plan to write
  - [ ] open questions 1 and 2 (a space's setup), then `04-spaces`, `05-forms`
- [ ] `stacks/04-boxes` — a stack onto a container or VM; replaces `dev-source/03-boxes`
- [ ] `stacks/05-roundtrips`

### Then: the linker, then wsm on it (no stow)

- [ ] `linker/01-link-library` — first read `units/linker.md`
- [ ] `linker/02-platform-files`
- [ ] `linker/03-owned-files` — `anfs mv`, `anfs owner`, `anfs status`; `ppm move` goes
- [ ] `wsm` without levels — plan to write after the linker, on it: anything spaces need from
      the linker is added there, not worked around. A space is any directory with a `space.yml`
      (and a source's `spaces/`), at any depth; ids are paths (`lgat/tech/web`); a space's files
      are those not inside a child space; no LEVEL, no workspace/space naming, no two-level limit

### Then: features

- [ ] `container-images/01-run-exec-snapshot` — first read the plan and its six questions
- [ ] `container-images/02-pcm-image` — plan to write
- [ ] `toolkit-api` — first settle its questions 2 and 3; plans to write
- [ ] `deploy` — moved to `ROADMAP-apps.md` section 2 (2026-10-19), as `pim`
- [ ] `metal` — plans to write

### Last

- [ ] **Test `pcm up` and `pcm machine resize` on this Mac** (merged untested 2026-10-19, with
      `stacks/03`): an idle podman machine too small for what starts is resized, then started;
      with containers running, `pcm up` refuses with the numbers; `pcm machine resize` by hand
      (and its confirm). Needs a container with `memory:` in its `container.yml`
- [ ] `audit`, first pass: the "first hour" and "why not what exists" lenses (moved here
      2026-10-19)
- [ ] **How pmm's and pcm's tests are run from now on (you and me).** Their bats suites are off
      since 2026-10-19: they took most of the test time and ran on every change. Decide how you
      run them by hand (`ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pmm pcm`) or how CI/CD
      runs them, and when (before a merge that touches pmm or pcm, nightly, before a release).
      Until then no agent runs them, and each plan's log says they were not run
- [ ] **Review testing together (you and me), before any VM or long-running test is launched.**
      The test categories (unit bats with stubs, the e2e scripts, the round trips, boxes, tart,
      Linux hosts, CI), and for each: what it proves, how it runs, when (while working, before a
      merge, nightly, before a release) and where (this Mac, the Debian box, CI). The outcome
      decides how the deferred tests below are run
- [ ] **The deferred VM tests** (decided 2026-10-19: none run until everything above is built,
      they take too long). Every plan's Verification items that boot a VM or a test box: `pmm`'s
      e2e (Debian, Fedora), two jobs' e2e at once, the round trips (Debian, Fedora, macOS), the
      tart build, `anfs work install` into boxes, a Linux host. Each plan's log lists its own
      as deferred; collect them here when they are run

- [ ] `audit`, in full
- [ ] `tutorial` — "From dotfiles to deployment"

### Done

- [x] Testing cut down (2026-10-19): `packages/anfs/tests/all` runs every bats file in parallel;
      pmm's and pcm's suites are off (core 1066363), so the full run is about 30 s

- [x] `machine-images`: tested (the walk, the tart build, the Debian box, the amd64 PC: most
      checks run, all passed), merged into main in core, stack and docs, pushed (2026-10-19)
- [x] This Mac repaired after the merge (2026-10-19): pim's VM, disks and links gone; the core
      source renamed `anfs` → `core` here; `anfs-dev/dev` installed and the git hooks wired
      again; docs at `~/spaces/anfs-dev/docs`, infra and website at `~/spaces/anfs-admin/`.
      Kept aside in `~/.local/state/anfs-repair-2026-10-19/`
- [x] By hand: `core/dev` moved to `anfs-dev` with the docs space (core 15d1aaa); `pvm` moved
      from the user source to core (450241b)

- [x] `machine-images/01-fold-pim-into-pmm`, `04-iso-commands`, `02-image-definition`,
      `03-provisioners`: built on the branch 2026-10-13, Mac only

## The units

| # | Unit / plan | What it does | Needs from you first | Status |
| --- | --- | --- | --- | --- |
| 1 | `machine-images` / `01-fold-pim-into-pmm` | pim becomes `pmm image`; `run`, `exec`, the `snapshot` group; `machines/images/` and `machines/networks/` | Read `units/machine-images.md` and plan 01. Agree the command list in **step 1b** | merged 2026-10-19 (see `log.md`; what was not run is listed there) |
| 2 | `container-images` / `01-run-exec-snapshot` | pcm gets `run` and the `snapshot` group, so the two tools match | Read the plan and its six open questions | plan drafted 2026-10-13, awaiting review |
| 3 | `machine-images` / `04-iso-commands` | One `iso` group in `anfs`, `pmm` and `ventoy` | Nothing | merged 2026-10-19 (see `log.md`; what was not run is listed there) |
| 4 | `machine-images` / `02-image-definition` | Everything a build runs is in the image's directory; `show` prints the plan; `image new` | Skim the plan after step 1; it will read differently then | merged 2026-10-19 (see `log.md`; what was not run is listed there) |
| 5 | `machine-images` / `03-provisioners` | The provisioner contract; `sh` and `anfs` behind it | Nothing: the step variables are `ANFS_IMAGE_*` | merged 2026-10-19 (see `log.md`; what was not run is listed there) |
| 5a | `dev-source` / `02-work`, `01-move`, `03-boxes` | The `anfs <name>` seam and the tools as `anfs <resource>`, `anfs hooks`, `~/work/<job>/` and `anfs work`; then the rest of the move to `anfs-dev` (bats suites stay in core); then (03, superseded by `stacks/04-boxes`) | `03-boxes` needs step 2 | `core/dev` and the docs space moved by hand 2026-10-17; `02-work` merged 2026-10-19 (VM checks deferred); `01-move` (the rest) next |
| 5c | `cli` / `01-one-shape` | psm without its `skills` group; `wsm repo`; every list's first column the resource as `source/name`, `STATUS` everywhere | Nothing | merged 2026-10-19 |
| 5d | `stacks` / `01`–`05` | `stacks/`: manifests of resources applied to a target; `anfs install\|remove\|list\|show -t`; `system` is the toolkit; `container.yml`; boxes; the round trips as stacks | Nothing | plans written 2026-10-19 |
| 5b | `machine-images` / `05-pmm-optional` | pmm and qemu leave the default install; pcm stays | Nothing | deprecated 2026-10-19: a new install strategy replaces it |
| 6 | `container-images` / `02-pcm-image` | `pcm image list\|build\|pull\|export\|remove\|prune` | Plan not written yet. Revisit the placeholder on provision steps in container builds | plan to write |
| 7 | `toolkit-api` / `01`–`05` | `anfs rpc`, the socket, anfs-ui as a container in a default install | The macOS test below, then settle the unit's open questions 2 and 3 | draft unit |
| 8 | `deploy` / `01-targets`, `02-proxmox-publish` | The deploy tool; a build becomes a template on a PVE host | Read `units/deploy.md` (not read yet). Name the tool (`pdm` is a placeholder) | plans written |
| 9 | `deploy` / `03-deployments` | Deployments over OpenTofu | Details are written after step 8, against the real host | outline |
| 10 | `metal` | Network install from an image definition; the old laptop; pcs | Plans are written after steps 5 and 8 | outline |
| A | `linker` / `01`–`03` | Our own linker replaces GNU stow in ppm, wsm and the bootstrap; per-platform files in `package.yml`; `anfs owner`, `anfs mv` (files, and a package between sources: `ppm move` goes), `anfs status` | Read `units/linker.md` (it answers the three questions on stow) | plans written |
| B | `profiles` / `01-profiles` | Hosts and roles in a source; `anfs profile apply` | Read `units/profiles.md` and its three open questions | plan written |
| Y | `audit` | Clean agents with none of our context judge the toolkit through seven lenses (first hour, why not what exists, architecture, safety, agent as user, ...); we answer each finding | A first pass (two lenses) can run once `machine-images` is merged; the full one before Z | placeholder |
| Z | `tutorial` | "From dotfiles to deployment": the whole toolkit as one story in chapters, a team and their agents taking a todo app from idea to deployment | Everything above built | placeholder |

## For an agent running this list

- **An orchestrator makes the job; an agent works only in it.** `anfs work new <job> <repo...>`
  (a source's alias: `core`, `stack`, `anfs-dev`; or a space's repository: `docs`) makes
  `~/work/<job>/` with a worktree of each repo on the branch `<job>`, and gives the job its number.
  `anfs work add <job> <repo>` brings in another. The agent is given the job's path (`anfs work
  path <job>`) and edits nothing outside it. The checkouts under `~/.local/share/anfs/sources/`
  are the live install (the tools on PATH are linked from them) and stay on main: never edit them,
  never `git worktree add` by hand, and never run `ppm install`, `anfs install`, `wsm install` or
  any `implode` on this host.
- **Every test runs through the job**: `anfs work run <job> core -- bats packages/<tool>/tests
  </dev/null` (the repo after the job is the worktree it runs in; without one, the job's
  directory, which holds the worktrees and no `packages/`). That sets the job's variables (`anfs work env <job>` prints them): its own
  `PMM_SOCK_DIR`, ssh port range, `TMPDIR` and `ROUNDTRIP_SRC_ROOT`, so two jobs' tests never
  meet each other or the live install. anfs-dev's suites: `anfs work run <job> anfs-dev -- bats
  packages/dev/tests </dev/null`.
- **Never touch real state.** `~/.local/share/pmm`, `~/.local/state/pmm`, `/tmp/pmm-<uid>` and
  tart's store hold running machines. The bats suites and `tests/e2e` isolate themselves, and
  core's `isolation.bats` fails a suite that names the live socket directory, points a `*_HOME`
  at a real home, writes under the real `~` or kills by pattern. Anything run by hand gets its
  `*_HOME` variables pointed at a temp dir. The ISO store (`~/.cache/anfs/isos`) is shared and
  read-only. No broad `pkill`; kill only the pids you started.
- **One plan per agent, in the checklist's order.** Read the unit file and the plan first. Commit on
  the branch as you go; do not push. Write `log.md` beside the plan when it is done.
- **pmm's and pcm's tests are off until further notice** (2026-10-19). Their helpers skip
  every test and `tests/all` leaves them out. Do not set `ANFS_TESTS_PMM_PCM`; a plan that
  changes pmm or pcm says in its log that their suites were not run. See "Last".
- **Tests while working: only the file you touch** (`bats <file> -f <name>` for one test).
- **The full run, once, at the end of a plan**: `anfs work run <job> core --
  packages/anfs/tests/all` (every bats file but pmm's and pcm's, several at once: about 30 s). A reviewer reads the code and runs only what it needs to check a finding; it
  does not run the full set.
- **Nothing that boots a VM or a test box runs** (pmm's e2e, the round trips, the harness, tart):
  those are deferred to the end of the list, after the testing review there. The log names each
  deferred item.
- **Stops for a person**: before a branch merges (then `anfs work remove <job>`, which refuses
  unmerged work); before any step whose "Needs from you first" is not done; before anything that
  needs hardware or a PVE host.
- bats is Homebrew's `bats-core`. Run it with `</dev/null`.

## Why this order

- **Step 1 first**: every other step uses its command names and directory layout.
- **Step 2 straight after**: step 1 changes pmm's snapshot commands. pcm should not stay on the
  old form for long, and the change is small.
- **Step 3 before 4**: it is small, nearly independent, and step 4 uses catalogue ids for ISOs.
- **Step 7 can move.** `toolkit-api` depends on nothing. It can run alongside steps 3 to 6, or
  jump ahead if the dashboard matters sooner. Its request examples use step 1's command names.
- **Step 5a when `machine-images` is merged**: that branch moves the images and rewrites the
  tests, so the split waits for it. It is the natural point to merge anyway: the unit is done.
  If step 2 has not run by then, its `ppm container` changes are made in `anfs-dev`.
- **`anfs work` is the first thing after the merge**, ahead of the move it was planned behind:
  agents do the work, and it is what gives each job its own trees, ports and sockets. The rest
  of the tooling (the move, the AGENTS.md rename, 5b) follows as jobs run through it.
- **A and B are lettered, not numbered: they depend on nothing above.** `linker` touches ppm,
  wsm and `install.sh`, which the `machine-images` branch does not, so it can run as its own job
  in parallel. `profiles` is better after `linker` plan 02 and step 5b.
- **Deploy and metal last**: they build on finished images, and you have not reviewed them.

## Decisions already made (so they are not reopened by accident)

- Packer is not used. pim's mechanism stays; it measured 151 s for Debian on Apple silicon.
- `pmm image` is the reference API; `pcm image` follows it. One list (no `builds`), `test` not
  `verify`, `export` not `publish`, `prune` not `clean`.
- `run <image>` is ad hoc, `up <machine>` is from a definition. `create` is gone
  (`run --no-start`). No command writes a machine or network definition.
- A machine made by `run` is recorded with its disk in `$PMM_DATA_HOME`, never in `~/.config`.
- A tool owns one top-level directory of a source: networks and images are under `machines/`.
  `containers/images/` is reserved, not used.
- `pmm network create` makes a network and records it in pmm's data, as `run` does a machine.
- A `Containerfile` lives with the code it builds; a source ships the service that pulls the image.
- Apps ship as containers and reach the host through the toolkit's one API.
- pmm never creates or changes a host bridge. Installed hosts are born with one (`metal`).
- What a provision step sees is `ANFS_IMAGE_*` (tool-neutral); pmm's own settings stay `PMM_IMAGE_*`.
- Packer and UTM are out of stack.
- Dev and test assets live in the `anfs-dev` source (`user.list`, not a default); each tool's
  bats suite and e2e stay beside its code in core.
- No backwards compatibility with pim.

## Still open

| Question | Where | Blocks |
| --- | --- | --- |
| What may an API caller do by default: read-only, or everything | `toolkit-api` question 3 | step 7 |
| The deploy tool's name | `deploy` | step 8 |
| Where amd64 images for Proxmox are built (the Mac emulates) | `deploy` plan 02 | step 8 |
| Does pcs deploy through the deploy tool | `metal` | step 10 |
