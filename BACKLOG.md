# Backlog

Ideas captured on 2026-10-13, not yet units. Each has the note that came with it and what it
touches. `ROADMAP.md` holds the master checklist (what is next, in order); an item moves there when it
gets a unit.

## Toolkit

- **Rename the `stack` source** (anfs-io/stack): with `stacks/` a resource, `stack/ruby` and
  `stacks/` read alike. Later; live with it until then (2026-10-19).

- **CLAUDE.md becomes AGENTS.md** in every repo. Do it when the `machine-images` branch is
  merged (agents are editing CLAUDE.md on it now). Check first that every agent in use reads
  AGENTS.md on its own; if one does not, leave a CLAUDE.md that only imports it.

- **The pre-commit hook and a pathspec commit** (`git commit -- <path>`): the version bump lands
  in the commit, but the index keeps the old `package.yml`, so the file shows as staged and
  modified. Seen 2026-10-19 in core. The hook should `git add` what it changed only when the
  commit is not pathspec-limited, or re-stage after.

- **anfs-dev's git hooks out of `~/.config`** (from `units/config.md`, plan 05's audit,
  2026-10-21): hook code is shipped under `~/.config/git/ppm-hooks` (code, not config), and
  `ppm install anfs-dev/dev` sets `core.hooksPath` in every source's `.git/config` as a side
  effect. Move the hooks to `~/.local/lib/anfs-dev/hooks`; set `hooksPath` only from `anfs hooks
  install` (and `anfs work new`). An anfs-dev job.

## Now units

- pmm out of the default install (pcm stays, for the dashboard): `machine-images` plan 05.
- Our own linker instead of GNU stow; per-platform files under `stow:`; `anfs mv`: `units/linker.md`.
- Worktrees for parallel jobs (`~/work`, `anfs work`): `dev-source` plans 02 and 03.
- The tools' command lines alike (psm groups, `wsm repo`, one list shape): `units/cli.md`.
- Profiles for more than one machine (hosts, roles, software): `units/profiles.md`.

## Sources

- pvm left core as stack's `ovm` (`units/ovm.md`).

- `anfs-admin` (`anfs-io/admin-anfs`, private): anfs.io's own infrastructure. The website, the
  demo cloud instances, anything not public. Nothing of the toolkit lives there.
- **Tests failing on main (2026-10-22):** stack `op/tests/ssh-hook.bats` 25, 31, 34, 35 and
  `tailscale/tests/tsm.bats` 61; user `pjm/tests/pjm.bats` 12 ("dispatch sends only what is
  ready"). Found by linker/02 and ovm/01; not caused by them.
- **anfs-dev after linker/02:** `roundtrip-macos`'s bootstrap parse check lists the new `lib/anfs`
  files (`manifest.sh`, `files.sh`, `platform.sh`); `move.sh` still uses stow and `PPM_GROUP_ID`
  (goes with `linker/05-move`).
- **Port clash (2026-10-22):** `cnfs/gemstash` and `core/ui` both publish host port 9292
  (`pcm validate` warns). Pick another port for one of them.
- **pcm's own suite** has not run since config/04 changed `packages/pcm/tests/helper.bash` and
  `edges.bats`: `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm` (for the person; see "Last").
- **`anfs config list --all`** treats any `~/.config/<d>/<d>.env` as an installed tool; an unrelated
  file of that shape would be listed. Accepted for now (2026-10-22); could read installed packages
  instead.


- pcm suite (`ANFS_TESTS_PMM_PCM=1`), two old failures found in pcm/01-run: `manifest.bats` 55
  expects exit 1 but a confirm without a terminal returns 77 since setup/01; `toolkit.bats` 83
  uses a shell function as the compose runner, which pcm can't `exec`. One-line fixes each.
- `units/container-images.md` plan 01 still proposes `pcm run <image>`; decided against
  2026-10-22 (`units/pcm.md`): edit that draft when the unit is refreshed.

- psm: every skill is `partial`: none reaches the antigravity and antigravity-cli agents (Claude
  Code, OpenCode and Pi have them). So `anfs report check`'s skill row fails here, accurately.
  Find out whether the skills CLI supports those agent ids or psm should stop listing them.
- `anfs report status` for a GitHub issue closed without a `Fixed in` comment says `fixed`;
  `closed` may be truer (not planned is not fixed). Decide on first real GitHub use.

- stack obsidian `init.bats`' sync test failed once in a full ovm-02 run (passed alone and in two
  more full runs). Flaky; not tracked down.
- apex: `ovm validate` gives 254 findings in 257 documents (123 missing `id`, 112 no
  frontmatter, 19 `type`): the person's call whether to fix the documents or loosen `required:`.

## r-20261022-1 pcm show/ps: 'Exited (0) 292 years ago' when podman has no FinishedAt

Status: fixed in core 0c3fe35 (2026-10-22)

Package: `core/pcm`

`pcm show authentik` (and `podman ps -a` from a terminal) shows its containers as `Exited (0) 292 years ago`.

Root cause: podman recorded the containers as exited but never set FinishedAt (`podman inspect authentik_server_1` gives `started=2026-10-11 23:32:25 … finished=0001-01-01 00:00:00 +0000 UTC`), most likely because they stopped with the podman machine rather than through `pcm down`. podman builds its Status string from that zero time. pcm prints the string as is.

Suggested fix: when `.State.FinishedAt` is the zero time, pcm shows the state alone (`exited`), or `exited (unclean)`, and doesn't pass podman's Status string through. This applies everywhere pcm prints container status (show, ps, list).

### What was expected

A sensible status for a container that exited without a recorded finish time

### What happened

Exited (0) 292 years ago

### Steps to reproduce

Stop the podman machine while a service is running; pcm show <service>

### Diagnostics

- anfs: 0.2.16
- platform: macos (Darwin 27.0.0, arm64; macOS 27.0.1)
- bash: 5.3.15(1)-release
- git: 2.55.0
- podman: 6.1.1

Sources:

| Source | Commit | Status |
| --- | --- | --- |
| anfs-dev | 77ff15d | clean |
| anfs-admin | 4413964 | clean |
| lgat | e3ae6b4 | clean |
| roteoh | a9ec35b | clean |
| cnfs | 32e4bdf | unpushed |
| rws | 43a125f | dirty |
| user | 710b294 | dirty, unpushed |
| stack | 018c496 | unpushed |
| core | 2cbc7f9 | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.2.0 |
| core/pmm | 0.1.5 |
| core/podman | 0.1.1 |
| core/ppm | 0.2.8 |
| core/psm | 0.1.6 |
| core/qemu | 0.1.1 |
| core/varlock | 0.1.2 |
| core/wsm | 0.1.10 |
| pdt/pcs | 0.1.0 |
| pdt/python | 0.1.0 |
| rws/git | 0.1.0 |
| spaces/cnfs | 0.1.0 |
| spaces/rjayroach | 0.1.0 |
| spaces/rws | 0.1.0 |
| stack/antigravity | 0.1.1 |
| stack/bash | 0.1.1 |
| stack/claude | 0.1.1 |
| stack/cli | 0.1.0 |
| stack/firstmate | 0.1.0 |
| stack/fnox | 0.1.1 |
| stack/ghostty | 0.1.1 |
| stack/git | 0.1.2 |
| stack/herdr | 0.1.1 |
| stack/nvim | 0.1.1 |
| stack/obsidian | 0.6.0 |
| stack/op | 0.1.5 |
| stack/opencode | 0.1.1 |
| stack/pi | 0.1.1 |
| stack/pikvm | 0.1.2 |
| stack/python | 0.1.0 |
| stack/rails | 0.1.0 |
| stack/remotefs | 0.1.1 |
| stack/ruby | 0.1.2 |
| stack/ssh | 0.1.3 |
| stack/tmux | 0.1.0 |
| stack/ventoy | 0.1.3 |
| stack/zsh | 0.1.4 |
| user/anfs | 0.1.0 |
| user/claude | 0.1.0 |
| user/git | 0.1.0 |
| user/pjm | 0.1.0 |
| user/pvm | 0.1.0 |
| user/ssh | 0.1.1 |
| utils/ventoy | 0.1.0 |

## r-20261022-2 pcm provision/deprovision: run a service's provision hooks for a consumer that isn't a pcm definition

Status: fixed in core b7be10b (2026-10-22)

Package: `core/pcm`

Feature request (pcm). Provision hooks only run when another pcm definition lists the service under `x-pcm.depends_on`. A consumer that is not a pcm definition can't use them. For example, a Rails app run on the host by the `partitia` tool needs an authentik OIDC client, a glitchtip DSN or a postgres database. Running `./provision` directly bypasses varlock and pcm's environment (PCM_PROJECT, PCM_SERVICE…), and the hooks aren't meant to be run that way.

Requested public commands:

- `pcm provision <service> [key=value...]`: start the service if it isn't running, wait until it's healthy (PCM_HEALTH_TIMEOUT), then run its provision hook under varlock exactly as `up` does for a dependent. Print the hook's KEY=VALUE lines on stdout. With --json, print them as an object. Set PCM_DEPENDENT to a caller-supplied name (e.g. `--as partitia/shop`) or something like `external`.
- `pcm deprovision <service> [key=value...]`: run the deprovision hook with the same args and env. If the service isn't running, say what was left in place, as `remove` does.

Exit non-zero if the service has no provision/deprovision hook. The partitia `pt services` command would be the first consumer; the stack authentik and glitchtip definitions would gain hooks for it.

### What was expected

A public pcm command to provision/deprovision for an external consumer

### What happened

Hooks only run via x-pcm.depends_on

### Steps to reproduce

n/a (feature request)

### Diagnostics

- anfs: 0.2.16
- platform: macos (Darwin 27.0.0, arm64; macOS 27.0.1)
- bash: 5.3.15(1)-release
- git: 2.55.0
- podman: 6.1.1

Sources:

| Source | Commit | Status |
| --- | --- | --- |
| anfs-dev | 77ff15d | clean |
| anfs-admin | 4413964 | clean |
| lgat | e3ae6b4 | clean |
| roteoh | a9ec35b | clean |
| cnfs | 32e4bdf | unpushed |
| rws | 43a125f | dirty |
| user | 710b294 | dirty, unpushed |
| stack | 018c496 | unpushed |
| core | 2cbc7f9 | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.2.0 |
| core/pmm | 0.1.5 |
| core/podman | 0.1.1 |
| core/ppm | 0.2.8 |
| core/psm | 0.1.6 |
| core/qemu | 0.1.1 |
| core/varlock | 0.1.2 |
| core/wsm | 0.1.10 |
| pdt/pcs | 0.1.0 |
| pdt/python | 0.1.0 |
| rws/git | 0.1.0 |
| spaces/cnfs | 0.1.0 |
| spaces/rjayroach | 0.1.0 |
| spaces/rws | 0.1.0 |
| stack/antigravity | 0.1.1 |
| stack/bash | 0.1.1 |
| stack/claude | 0.1.1 |
| stack/cli | 0.1.0 |
| stack/firstmate | 0.1.0 |
| stack/fnox | 0.1.1 |
| stack/ghostty | 0.1.1 |
| stack/git | 0.1.2 |
| stack/herdr | 0.1.1 |
| stack/nvim | 0.1.1 |
| stack/obsidian | 0.6.0 |
| stack/op | 0.1.5 |
| stack/opencode | 0.1.1 |
| stack/pi | 0.1.1 |
| stack/pikvm | 0.1.2 |
| stack/python | 0.1.0 |
| stack/rails | 0.1.0 |
| stack/remotefs | 0.1.1 |
| stack/ruby | 0.1.2 |
| stack/ssh | 0.1.3 |
| stack/tmux | 0.1.0 |
| stack/ventoy | 0.1.3 |
| stack/zsh | 0.1.4 |
| user/anfs | 0.1.0 |
| user/claude | 0.1.0 |
| user/git | 0.1.0 |
| user/pjm | 0.1.0 |
| user/pvm | 0.1.0 |
| user/ssh | 0.1.1 |
| utils/ventoy | 0.1.0 |
