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

- **Runs file reports by event** (from agents/02d's "Out of scope", 2026-10-24): inside a
  dispatched run, `anfs report new` cannot write its record in STATE (outside the sandbox). The
  run emits `report.filed` into its own spool instead; anfsd ingests and stamps it, and a consumer
  records it (the review's projection, `AGENT-NATIVE-REVIEW.md` 4.2), replacing `anfs report
  new`'s STATE write when it runs inside a run. A plan in `units/agents` after 02d.

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

## r-20261022-3 pcm remove: 'Error: image name or ID must be specified' for a definition with no built images

Status: fixed in core b2656a8 (2026-10-22)

Package: `core/pcm`

### What was expected

pcm remove completes without an error line when there are no built images to delete

### What happened

Prints 'Error: image name or ID must be specified' mid-output; the remove still completes

### Steps to reproduce

Local definition using a stock image (busybox:1.37, no build:) with x-pcm.depends_on glitchtip; pcm up it; pcm remove -y it

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
| stack | 5f6d56a | dirty, unpushed |
| core | b7be10b | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.3.0 |
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

Recent failures in the log (~/.local/state/anfs/anfs.log):

```
2026-10-22T17:12:57-0400 pcm[67643] start: 67643-1792703577 down postgres
2026-10-22T17:13:12-0400 pcm[67643] end: 67643-1792703577 1
2026-10-22T17:14:28-0400 pcm[70709] start: 70709-1792703668 exec glitchtip -- ./manage.py shell --no-imports -c 
2026-10-22T17:14:30-0400 pcm[70709] end: 70709-1792703668 1
2026-10-22T17:15:09-0400 pcm[18761] start: 18761-1792703709 provision --as probe-check glitchtip org=probe
2026-10-22T17:15:10-0400 pcm[18761] end: 18761-1792703709 1
2026-10-22T17:15:11-0400 pcm[20024] start: 20024-1792703711 provision --as probe-check glitchtip org=Probe project=x
2026-10-22T17:15:12-0400 pcm[20024] end: 20024-1792703711 1
2026-10-22T17:15:12-0400 pcm[21388] start: 21388-1792703712 provision --as probe-check glitchtip project=x bogus=1
2026-10-22T17:15:13-0400 pcm[21388] end: 21388-1792703712 1
2026-10-22T17:15:13-0400 pcm[22633] start: 22633-1792703713 provision --as probe-check glitchtip org=admin project=x
2026-10-22T17:15:14-0400 pcm[22633] end: 22633-1792703713 1
2026-10-22T17:15:17-0400 pcm[28039] start: 28039-1792703717 provision --as probe-check glitchtip org=Probe project=x
2026-10-22T17:15:18-0400 pcm[28039] end: 28039-1792703717 1
2026-10-22T17:15:18-0400 pcm[29320] start: 29320-1792703718 provision --as probe-check glitchtip project=x bogus=1
2026-10-22T17:15:19-0400 pcm[29320] end: 29320-1792703718 1
2026-10-22T17:15:19-0400 pcm[30329] start: 30329-1792703719 provision --as probe-check glitchtip org=admin project=x
2026-10-22T17:15:22-0400 pcm[30329] end: 30329-1792703719 1
```

## r-20261022-4 space setup: what partitia needs (fnox 1Password provider per space, non-interactive, shell activation)

Status: open

Package: `core/wsm`

Input for the upcoming `anfs space setup` / `wsm setup`, from partitia (cnfs). It isn't a bug.

partitia's `pt services` provisions pcm services for a Rails app. With `PARTITIA_SECRETS_STORE=fnox` it pipes secrets (a GlitchTip DSN now; OIDC client secrets later) to `fnox set` in the app's own `fnox.toml`. fnox then exports them as `PARTITIA_<ENGINE>_*`. For that to work in a space, space setup should provide:

1. **Space-root `fnox.toml`:** at the space root, defining a 1Password provider authenticated by the space's service-account token (`OP_SERVICE_ACCOUNT_TOKEN`). The provider name must be stable and discoverable, so a tool can target it (partitia reads it from `PARTITIA_FNOX_PROVIDER`; a convention or an exported var naming it would be enough). One vault per space.
2. **Inheritance:** app dirs below the space root inherit that provider. An app's own `fnox.toml` holds only its keys.
3. **Shell activation:** fnox's shell activation on cd, so `rails console`, runners and jobs see the secrets, not only commands wrapped in `fnox exec`.
4. **Non-interactive:** with the token set, no 1Password unlock prompt. Today, cd into a dir with a `fnox.toml` prompts for the 1Password password, which blocks agents.

Recorded in partitia's ROADMAP ("Outside this repo"). The person will manually test the fnox → 1Password path once space setup exists.

### What was expected

anfs space setup / wsm setup provides a per-space fnox 1Password provider usable by tools like pt

### What happened

Not yet implemented (input for the upcoming feature)

### Steps to reproduce

n/a (requirements input)

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
| stack | 29e13fb | unpushed |
| core | b2656a8 | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.3.1 |
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

Recent failures in the log (~/.local/state/anfs/anfs.log):

```
2026-10-22T17:33:03-0400 partitia[29190] start: 29190-1792704783 engine check
2026-10-22T17:33:03-0400 partitia[29190] end: 29190-1792704783 1
```

## r-20261022-5 pcm remove: podman 'no container / no such pod' Error lines when the service has no containers

Status: fixed in core 4d8cf0e (2026-10-22)

Package: `core/pcm`

### What was expected

pcm remove on a stopped service with no containers prints no podman Error lines

### What happened

Error: no container with name or ID "authentik_worker_1" found (x4, worker/server) and Error: unable to find pod "pod_authentik": no such pod; the remove and the postgres deprovision still succeed

### Steps to reproduce

pcm down authentik (containers gone); pcm remove -y authentik. Related to r-20261022-3 (that fixed --rmi local only)

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
| cnfs | 5031446 | unpushed |
| rws | 43a125f | dirty |
| user | 710b294 | dirty, unpushed |
| stack | 29e13fb | unpushed |
| core | b2656a8 | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.3.1 |
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

Recent failures in the log (~/.local/state/anfs/anfs.log):

```
2026-10-22T17:48:43-0400 pcm[16630] start: 16630-1792705723 deprovision authentik --as partitia/ptsvc slug=ptsvc redirect=http://localhost:3000/auth/callback
2026-10-22T17:48:44-0400 pcm[16630] end: 16630-1792705723 1
2026-10-22T17:48:44-0400 pcm[16928] start: 16928-1792705724 deprovision glitchtip --as partitia/ptsvc org=partitia project=ptsvc
2026-10-22T17:48:44-0400 pcm[16928] end: 16928-1792705724 1
2026-10-22T17:48:42-0400 partitia[14895] start: 14895-1792705722 services --down
2026-10-22T17:48:44-0400 partitia[14895] end: 14895-1792705722 1
```

- psm copies local-path skills (core/skills/*) rather than linking them, and `psm sync` re-copies
  them only with `--local`: an edited core skill stays stale for every agent until someone runs
  `psm sync --local core` (found 2026-10-22 via r-20261022-5: pcm-containers named the old
  `~/.config/pcm/env/` path). Link them instead, or re-copy on every sync, or run that sync in
  core/psm's post-merge hook.

## r-20261022-6 pcm logs: no public command; 'podman compose --pcm logs' is a zsh function only

Status: fixed in core f54fb97 (2026-10-22)

Package: `core/pcm`

### What was expected

pcm logs [-f] [-s NAME] <service> works from any shell or script, like exec/show

### What happened

Logs need 'podman compose --pcm logs', a function in ~/.config/zsh/pcm.zsh, which fails in non-interactive or bash shells (agents and tools like pt have to source the zsh file)

### Steps to reproduce

bash -c 'podman compose --pcm logs telemetry'

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
| cnfs | 44aa594 | unpushed |
| rws | 43a125f | dirty |
| user | 710b294 | dirty, unpushed |
| stack | 29e13fb | unpushed |
| core | 4d8cf0e | unpushed |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.1 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.3.2 |
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

Recent failures in the log (~/.local/state/anfs/anfs.log):

```
2026-10-22T18:09:41-0400 pcm[99278] start: 99278-1792706981 down valkey
2026-10-22T18:09:47-0400 pcm[99278] end: 99278-1792706981 1
2026-10-22T18:09:47-0400 pcm[7230] start: 7230-1792706987 down valkey
2026-10-22T18:09:53-0400 pcm[7230] end: 7230-1792706987 1
```

## r-20261022-7 infra tooling: what cnfs apps need for production deploy (provision parity with pcm, one app def across footprints, wildcard subdomains, per-tenant migrations)

Status: open

Package: `user/anfs`

Requirements input from cnfs (partitia + appwork apps) for anfs's generic infrastructure tooling (production deploy to EC2, local Proxmox VM, containers, later k3s). It isn't a bug. The user asked for our considered view of what would make deployment easy, in priority order:

1. **The same provisioning interface in production as pcm has in dev.** `cnfs services` (formerly `pt services`) calls `pcm provision <service> --as cnfs/<app> key=value…` and reads KEY=VALUE (e.g. authentik → ISSUER, CLIENT_ID; glitchtip → DSN). Engines declare what they need in gemspec metadata (`partitia.services`: authentik, telemetry, glitchtip). If production provisioning took the same args and returned the same keys for a given target, the app tooling would need no deploy-specific code. anfs maps a service name to its provider per footprint: the pcm container in dev; managed Postgres, a shared Authentik or a hosted error tracker in prod.
2. **One app definition, many footprints.** The app ships a Dockerfile (Rails 8 generates one) plus one small declaration: processes (web, jobs), health check (`/up`), release command (migrations), and the services it needs. Each footprint is a target with a profile (EC2, Proxmox VM, local containers, k3s); no per-footprint config in the app. Kamal under the hood for VM footprints, or Helm for k3s, is anfs's call; just no hand-written config per footprint.
3. **pcm-style commands, scriptable.** deploy, status, logs, exec (a production `rails console`), run (one-off), snapshot, rollback. Non-interactive, safe to rerun, `--json`. That lets agents operate it.
4. **Secrets from the space.** Values from the space's 1Password through fnox, delivered to the running app as environment variables, never baked into images. A rotation story would be nice. This extends r-20261022-4 to production.
5. **Wildcard subdomains.** partitia-tenanted resolves tenants from subdomains, so every footprint needs wildcard DNS and wildcard TLS (DNS-01) for `*.app.example.com`. Custom domains per tenant come later. Without this, multi-tenant apps can't deploy.
6. **Releases that know about data.** Snapshot, migrate, deploy behind the health check, roll back on failure. With one database per tenant, "migrate" is N migrations: a tracked job with progress and per-tenant failure handling. Backups per tenant, ideally continuous (e.g. Litestream if SQLite stays in production).
7. **Observability in every footprint.** An OTel collector, an error tracker and a log sink, with endpoints handed to the app as in #1, so partitia-telemetry works unchanged.
8. **A skill and a report path,** like pcm-containers and `anfs report`, so agents can use it from day one.

Open design question on our side that interacts with footprints: SQLite per tenant fits VM footprints with persistent disks, but is awkward on k3s (single writer, RWO volumes). Footprint choices may push production to Postgres. We'll settle that early in our D7 (production deploy).

Recorded in partitia's ROADMAP under D7.

### What was expected

anfs infrastructure tooling covers these app needs

### What happened

Requirements input; infra tooling under development

### Steps to reproduce

n/a (requirements input)

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
| cnfs | 9f39e64 | clean |
| rws | 43a125f | dirty |
| user | 710b294 | dirty |
| stack | eb4dce1 | clean |
| core | f54fb97 | clean |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.2 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.4.0 |
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
| stack/rails | 0.1.1 |
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

## r-20261023-1 wsm repo push: accept a space spec (cnfs/) like repo list; hint for a bare id with no repositories

Status: fixed in core 589bacf (2026-10-23)

Package: `core/wsm`

Feature request (wsm 0.10.0): `wsm repo push` should take a space spec, as `wsm repo list` already does.

**Today:**
- `wsm repo list cnfs/` lists every repository under the cnfs spaces.
- `wsm repo push -n cnfs/` fails with `wsm: no repository 'cnfs/' (wsm repo ls)`. push accepts only exact repository names (`cnfs/partitia/partitia`) or `--all`.
- `--all` spans every space: on this machine `wsm repo push -n --all` would also push 80 commits in anfs-dev's `docs`. Pushing "everything in this project" has no safe scoped form.

**Requested:**
1. `wsm repo push [--all] [-n] [spec|repository]...`. Each argument is an exact repository name (as `repo list` prints it) or a space spec resolved exactly as `repo list` resolves it: an id like `cnfs/partitia` (that space), or a prefix ending in `/` like `cnfs/` (every space below it). Push the repositories that are ahead of their upstream; `-n` shows what would be pushed, as today.
2. Keep the existing spec semantics (bare id = that space, trailing `/` = the subtree), consistent with `install` and `list`. Add a hint for the common mistake: a bare id naming a space with no repositories of its own, whose children have some (`wsm repo list cnfs` prints nothing today), should say `no repositories in space 'cnfs'; did you mean cnfs/?` rather than nothing or an error.

Use case: after a working session across cnfs/partitia, cnfs/appwork and cnfs/apps (testbed, showcase), `wsm repo push cnfs/` pushes exactly that project's repositories.

### What was expected

wsm repo push -n cnfs/ pushes the repositories under the cnfs spaces that are ahead of upstream

### What happened

wsm: no repository 'cnfs/' (push accepts only exact repository names or --all, which spans every space)

### Steps to reproduce

wsm repo list cnfs/   # works; wsm repo push -n cnfs/   # fails

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
| cnfs | 076e5b0 | dirty |
| rws | 43a125f | dirty |
| user | 710b294 | dirty |
| stack | 4cdcf1b | clean |
| core | f54fb97 | clean |

Installed packages:

| Package | Version |
| --- | --- |
| anfs-dev/dev | 0.1.15 |
| cnfs/rails | 0.1.2 |
| core/anfs | 0.2.16 |
| core/node | 0.1.0 |
| core/pcm | 0.4.0 |
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
| stack/rails | 0.1.1 |
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

Recent failures in the log (~/.local/state/anfs/anfs.log):

```
2026-10-23T00:13:11-0400 wsm[74868] error: no repository 'cnfs/' (wsm repo ls) (exit 1)
2026-10-23T00:13:11-0400 wsm[74868] start: 74868-1792728791 repo push -n cnfs/
2026-10-23T00:13:11-0400 wsm[74868] end: 74868-1792728791 1
```

- core `packages/anfs/tests/rpc.bats` line 193: a bare `! grep -q '"tool":"anfs"'` can never fail
  a bats test, so that watch assertion checks nothing (found in toolkit-api/06-watch-loop; the
  new test there counts instead). Grep the suites for other bare `! cmd` lines.

- Space ids (agents/01): only anfs-dev has one. To do by hand: `wsm id --new` for anfs-admin,
  lgat/*, roteoh, rws, user/evals (each commits in its own source repo; user and rws are the
  person's); cnfs/* asked of the cnfs session.

- If beads + firstmate + herdr are adopted (a big if, 2026-10-23): compare and contrast them with
  the agents unit (beads vs work items/records, firstmate vs dispatch/supervision/approvals,
  treehouse worktrees vs `anfs work` jobs), as a learning exercise first. Not before the original
  agents plan is done.

- core `rpc.bats` "a group's list is a read" failed once in the full suite during agents/03 (passed
  12 reruns, 8 in parallel): flaky, not tracked down.
- ppm, pmm, wsm and psm emit no events yet, so anfs-ui shows their changes made in a terminal
  only at its 5-minute reconcile (agents/03). A follow-up plan: emitters in those tools.

- anfsd, smaller findings from `AGENT-NATIVE-REVIEW.md` (F8, 2026-10-24), to fix when they bite:
  per-scope token buckets don't isolate scopes (`limiter.Wait` runs inside one batch, so a
  flooding scope stalls shipping for all); the report sink's `MaxAckPending: 1` with nak delays
  up to 5 minutes lets one failing event hold up all rendering for about 12 minutes; a consumer
  deleted and recreated with `DeliverNewPolicy` skips the gap (deliver from the last-known time
  instead, the `stale` guard makes it safe); `runs/`, the supervisor's in-memory indexes and the
  month files grow without limit (retention, AGENT-NATIVE's open question 1).
- An event catalog, `anfs events types [space]` (review §4.5): the types packages declare plus
  those seen in the log, each with its last-seen time and an example, so an agent landing in a
  space knows what is on the bus.
- Harness definitions as package YAML (review §4.6: argv template, permission mapping, summary
  extraction), so a new harness is a ppm package, not an anfsd release. Revisit when a harness
  changes its command line.
- The flexibility acceptance test (review §4.1, AGENT-NATIVE principle 9): a second, non-anfsd
  consumer (Pi Durable or Firstmate) handling one space end to end, with 02d's dispatcher-off
  switch and no core change. Roadmap, not the next slice.
