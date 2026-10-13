# Roadmap: images, one API, deployment

Written 2026-10-13, after the pim audit. The order to take the units in, what each step needs
from you first, and what is still undecided. Update the Status column as steps finish.

The direction in one paragraph: `pmm` owns machines and their images (pim goes away), `pcm` gets
the same commands, the toolkit gets one API so apps ship as containers, a deploy tool puts
images on targets, and hardware installs from the same image definitions.

## The order

| # | Unit / plan | What it does | Needs from you first | Status |
| --- | --- | --- | --- | --- |
| 1 | `machine-images` / `01-fold-pim-into-pmm` | pim becomes `pmm image`; `run`, `exec`, the `snapshot` group; `machines/images/` and `machines/networks/` | Read `units/machine-images.md` and plan 01. Agree the command list in **step 1b** | built on branch `machine-images`, awaiting review; round trips, Linux and tart not run (see `log.md`) |
| 2 | `container-images` / `01-run-exec-snapshot` | pcm gets `run` and the `snapshot` group, so the two tools match | Read the plan and its six open questions | plan drafted 2026-10-13, awaiting review |
| 3 | `machine-images` / `04-iso-commands` | One `iso` group in `anfs`, `pmm` and `ventoy` | Nothing | built on the branch 2026-10-13 (see `log.md`); no real download, stick or Linux run |
| 4 | `machine-images` / `02-image-definition` | Everything a build runs is in the image's directory; `show` prints the plan; `image new` | Skim the plan after step 1; it will read differently then | in progress (workflow, 2026-10-13) |
| 5 | `machine-images` / `03-provisioners` | The provisioner contract; `sh` and `anfs` behind it | Nothing: the step variables are `ANFS_IMAGE_*` | in progress (workflow, 2026-10-13) |
| 5a | `dev-source` / `01-move`, `02-work` | `core/dev`, the test boxes, the round trips, the fixture and the docs space move to the `anfs-dev` source; bats suites stay in core. Then `~/work/<job>/` and `ppm work` | Merge the `machine-images` branch first (review, Linux checks). Answer the plan's question 2 | plan written |
| 5b | `machine-images` / `05-pmm-optional` | pmm and qemu leave the default install; pcm stays | Nothing | plan written |
| 6 | `container-images` / `02-pcm-image` | `pcm image list\|build\|pull\|export\|remove\|prune` | Plan not written yet. Revisit the placeholder on provision steps in container builds | plan to write |
| 7 | `toolkit-api` / `01`–`05` | `anfs rpc`, the socket, anfs-ui as a container in a default install | The macOS test below, then settle the unit's open questions 2 and 3 | draft unit |
| 8 | `deploy` / `01-targets`, `02-proxmox-publish` | The deploy tool; a build becomes a template on a PVE host | Read `units/deploy.md` (not read yet). Name the tool (`pdm` is a placeholder) | plans written |
| 9 | `deploy` / `03-deployments` | Deployments over OpenTofu | Details are written after step 8, against the real host | outline |
| 10 | `metal` | Network install from an image definition; the old laptop; pcs | Plans are written after steps 5 and 8 | outline |
| A | `linker` / `01`–`03` | Our own linker replaces GNU stow in ppm, wsm and the bootstrap; per-platform files in `package.yml`; `anfs owner`, `anfs mv`, `anfs status` | Read `units/linker.md` (it answers the three questions on stow) | plans written |
| B | `profiles` / `01-profiles` | Hosts and roles in a source; `anfs profile apply` | Read `units/profiles.md` and its three open questions | plan written |

## For an agent running this list

- **Work only in the branch worktrees.** Core: `~/anfs-dev/machine-images`. Stack:
  `~/anfs-dev/machine-images-stack`. Both are on branch `machine-images`. The checkouts under
  `~/.local/share/anfs/sources/` are the live install (the tools on PATH are linked from them):
  never edit them, and never run `ppm install`, `anfs install`, or any `implode` on this host.
- **Never touch real state.** `~/.local/share/{pim,pmm}`, `~/.local/state/{pim,pmm}` and tart's
  store hold a running machine. The bats suites and `tests/e2e` isolate themselves; anything run
  by hand gets its `*_HOME` variables pointed at a temp dir.
- **One plan per agent, in the order above.** Read the unit file and the plan first. Commit on
  the branch as you go; do not push. Write `log.md` beside the plan when it is done.
- **The gate for the next step** is the plan's Verification list: `bats packages/<tool>/tests
  </dev/null` for every suite touched, and `packages/pmm/tests/e2e debian` where the plan
  changes a build or a machine. A step whose gate fails stops the list.
- **Stops for a person**: after step 1 (review before more is built on it); before any step
  whose "Needs from you first" is not done; before anything that needs hardware or a PVE host.
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
- **The tooling steps (5a, 5b, the AGENTS.md rename) come before more feature work**: every
  later job then runs in `~/work` with its own ports and sockets, and can run beside another.
- **A and B are lettered, not numbered: they depend on nothing above.** `linker` touches ppm,
  wsm and `install.sh`, which the `machine-images` branch does not, so it can run as its own job
  in parallel. `profiles` is better after `linker` plan 02 and step 5b.
- **Deploy and metal last**: they build on finished images, and you have not reviewed them.

## Small things to do when convenient

- ~~macOS transport test~~ done 2026-10-13: a container reaches the Mac's `127.0.0.1` through
  `host.containers.internal` (see `toolkit-api`, question 1).
- **The slow amd64 PC**: check whether its build said `(tcg)`. If so, the user was not in the
  `kvm` group and the build was emulated. Step 1 adds a warning for this.
- **Before installing step 1**: run `pim implode -y` and delete tart's `pim-*` VMs. Nothing is
  migrated.
- **Commit the docs.** Everything under `units/` from this session, and this file, is uncommitted.

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
