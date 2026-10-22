---
plan: config/05-xdg-rule
job: config-05
date: 2026-10-22
---

# Log: config/05-xdg-rule

## The plan against current main

Written against core `9e8cb73`; the job branched from core `427a9bf`, stack `385827b` (config
01–04 merged). Re-checked every finding against those trees before changing anything:

- **Finding 1** (install.sh's `anfs.local.conf` seed): gone with plan 01. Dropped here, recorded.
  What was left of `install_configs` was finding 2's `user.list` seed alone.
- **Finding 5**: config/04 had moved the writer to `containers/cloudflared/local.env` as an
  interim (as planned); this plan moves it on to STATE. `PCM_ENV_HOME` no longer exists, so the
  plan's "if 04 has not merged" branch does not apply.
- **Line numbers** moved (stack `ssh/install.sh` is line 12 now, the edge's `write_token` 98–111);
  the findings themselves still held.
- **Finding 10**: the plan says "documented in firstmate's README"; it was not. Added.
- **Finding 11** is already in docs' `BACKLOG.md` (lines 21–24). Nothing to add.
- Nothing else had changed under the findings.

## What was built

core:
- **AGENTS.md "Where files go (the XDG rule)"**, after Repository Layout (whose paragraph now
  points to it): a table CONFIG / STATE / DATA / CACHE / RUNTIME (holds, where, examples), the
  rules (a side effect never writes CONFIG; a file another command needs is made by the command
  that writes it and readers take a missing one as empty; fetched secrets are STATE, 0600, read
  below the person's file; a config location someone else dictates gets a static linked file that
  includes the tool's STATE), the accepted exceptions with reasons, and how it is held (review,
  this log, the tests). Also: the pcm bullet (`state.env`), pmm's sockets and `ssh_config`
  bullets, Key Files (`state.env`, `ssh_config`, `user.list` made by `anfs src add`), Testing's
  isolation rule. `paths.sh`'s header layout says the same in five lines.
- **install.sh**: `install_configs` gone (and its two calls); header steps 7–8, the files list,
  `--script-only`'s line and `install_repo`'s comment and warning say so. onboard's message.
- **pcm**: `install.sh` makes only `~/.local/share/pcm/volumes` (DATA). `pcm_state_env <id>` =
  `$PCM_STATE_HOME/containers/<name>/state.env`; `export_env_overrides` reads `local.env` then
  `state.env`, each name only when unset; `bin/pcm` exports `PCM_STATE_HOME` (the edge is its own
  process; without it `${PCM_STATE_HOME:?}` stops it); the cloudflare-tunnel edge's `write_token`
  writes `state.env` (directory 0700, file 0600 through a temp file, other lines kept); `pcm show`
  prints `state <file>`; `pcm remove` lists it in its summary and deletes
  `$PCM_STATE_HOME/containers/<name>/`; help text.
- **pmm**: `install.sh` has no `post_install` (no `machines/{images,networks}`). `sock_dir_default`:
  `$XDG_RUNTIME_DIR/pmm` on Linux when that is a directory, else `/tmp/pmm-<uid>`;
  `sock_dir_path` refuses under bats both `/tmp/pmm-<uid>` and `$XDG_RUNTIME_DIR/pmm`, set or
  defaulted. `PMM_SSH_CONFIG` defaults to `$PMM_STATE_HOME/ssh_config`; core/pmm ships
  `home/.ssh/config.d/05-pmm.conf` (comments and `Include ~/.local/state/pmm/ssh_config`, linked
  per file); implode's comment; help line; the test helper's `PMM_SSH_CONFIG`.
- **ppm**: the message file is `mktemp "${TMPDIR:-/tmp}/ppm-messages.XXXXXX"` (core.sh is bootstrap,
  3.2-safe); implode sweeps `${TMPDIR:-/tmp}`.

stack:
- **ruby**: `ruby.toml`'s `MISE_RUBY_DEFAULT_PACKAGES_FILE` = `{{ env.HOME }}/.local/share/ruby/default-gems`;
  comments in `install.sh` and `ruby.sh` (`install_gem` itself only follows the variable).
- **cloudflared**: compose.yml, `.env.schema` and README name the state file (and that a
  `TUNNEL_TOKEN` in `local.env` would win over it).
- **firstmate** README: the one write into `~/.claude.json` and why. **ssh** README: the `Include`
  line as the accepted edit; fragments are linked files (op's, pmm's).

## The audit (re-run 2026-10-22 against the job's trees)

Greps: every `mkdir|touch|tee|cp|mv|ln|add_to_file|>|>>` naming `.config/`, `*CONFIG_HOME`,
`XDG_CONFIG`, `.ssh/`, `PCM_CONTAINERS_HOME`, `PMM_{MACHINES,IMAGES,NETWORKS}_HOME` in core's
install.sh, every core and stack `install.sh`, `bin/`, `lib/`; then each hit read.

| # | Where | What | Outcome |
| --- | --- | --- | --- |
| 1 | install.sh `install_configs` | `anfs.local.conf` seed | **Dropped**: plan 01 removed it before this plan |
| 2 | install.sh `install_configs` | `user.list` with one comment | **Fixed**: no seed. Every reader takes a missing list (`gitsrc_collect`, `gitsrc_list`, `remove`, `ssh` check `-f`; `src add` touches it; ppm's check needs either list and `system.list` is linked; onboard's init goes through `gitsrc_add` first). The round trips and pmm's `anfs` provisioner write their own |
| 3 | core/pcm `post_install` | empty `~/.config/pcm/containers/` | **Fixed**: dropped (catalogue, completion, `check_local_dirs` glob with `-f`/`-d` guards) |
| 4 | core/pmm `post_install` | empty `~/.config/pmm/machines/{images,networks}` | **Fixed**: hook gone (`image new` makes its `dest`; catalogue and networks glob with guards) |
| 5 | pcm cloudflare-tunnel `write_token` | `TUNNEL_TOKEN` into `containers/cloudflared/local.env` (config/04's interim) | **Fixed**: `$PCM_STATE_HOME/containers/cloudflared/state.env`, 0600/0700, read below `local.env` (decision 9) |
| 6 | pmm `ssh_config_write` | `~/.ssh/config.d/05-pmm.conf` on every up/down | **Fixed**: `$PMM_STATE_HOME/ssh_config`; a static linked `05-pmm.conf` includes it |
| 7 | stack/ruby `post_install`, `install_gem` | `~/.config/ruby/default-gems` | **Fixed**: `~/.local/share/ruby/default-gems` (DATA) |
| 8 | stack/ssh `post_install` | `Include config.d/*.conf` appended to `~/.ssh/config` | **Kept** (decision 14), documented: AGENTS.md exceptions, ssh README |
| 9 | stack/gam `post_install` | `gam.cfg` via `gam config ... verify` | **Kept** (decision 14), in AGENTS.md's exceptions |
| 10 | stack/firstmate `trust_root` | `hasTrustDialogAccepted` in `~/.claude.json` | **Kept**, now documented in firstmate's README and the exceptions |
| 11 | anfs-dev hooks | `core.hooksPath`; code under `~/.config/git/ppm-hooks` | **Out of scope** (decision 13): already in docs' BACKLOG.md |
| 12 | core/anfs listener | launchd plist, systemd user units | **Kept** (platform) |
| 13 | pcm `pcm_podman_ready` | podman's machine config | **Kept** (platform) |
| 14 | explicit commands | `anfs src add/remove/ssh`, onboard steps, `iso add/remove --entry`, `ppm file *`, `psm repo add/remove`, `pmm image new`, `<tool> config set/unset`, stack's `remotefs add/edit/remove`, `gem-local src add/remove` (`~/.config/ruby/local-gems`), `git account`, git's and op's onboard steps (git config, `fnox/gh.toml`), `opm create/link`, ssh onboard's `authorized_keys` | **OK** |
| R1 | pmm sockets | `/tmp/pmm-<uid>` everywhere | **Fixed**: `$XDG_RUNTIME_DIR/pmm` on Linux |
| R2 | pmm pid files | `$PMM_STATE_HOME/run/<name>/pid` | **Kept**, a named exception |
| R3 | ppm message file | `/tmp/ppm-messages.*` | **Fixed**: `$TMPDIR` |

Also seen, OK: install.sh's `mkdir -p "$ANFS_CONFIG_HOME"` (the directory the base packages link
into); pmm's `anfs` provisioner writes the *guest's* `user.list`; tailscale, rpi-imager, nfs only
print or read their config; solana deletes its own on remove. Nothing DATA-like under a
`*_CONFIG_HOME` but ruby's list (fixed). The final grep over core's `packages/*/install.sh` finds
nothing (`xdg.bats` holds it); over stack's, only findings 8 and 9 and the onboard steps.

## Commits (branch config-05, not pushed)

core:
- `49787eb` the fixes and tests (install.sh, pcm, pmm, ppm; `xdg.bats`, `pmm-runtime.bats`,
  `pcm-local-env.bats`, `isolation.bats`, ppm `messages.bats`; pcm `edges.bats`, pmm helper)
- `2789609` docs: AGENTS.md

stack: `4309752` ruby's default gems to DATA (`default-gems.bats`); cloudflared's docs; firstmate and ssh READMEs

Versions: the pre-commit hook bumped nothing. core's anfs (0.2.15), pcm (0.1.10), pmm (0.1.5),
ppm (0.2.8) and stack's cloudflared (0.1.1), firstmate (0.1.3), ruby (0.1.2), ssh (0.1.3) already
carry their bump for the unpushed series.

## Decisions made during the work

- **`pcm remove` deletes the whole `$PCM_STATE_HOME/containers/<name>/`** (only `state.env` lives
  there), and lists it in the summary as "deleted; fetched again on up".
- **`pcm show`'s line is `state <path>`**, never a value.
- **The `Include` names `~/.local/state/pmm/ssh_config` literally**: ssh expands `~` but not
  `$XDG_STATE_HOME`. A moved `XDG_STATE_HOME` needs the fragment claimed (said in the file and
  AGENTS.md). See open question 1.
- **Under bats, both live socket directories are refused whatever the platform** (so a Linux CI
  with a real `XDG_RUNTIME_DIR` is covered even though the helper stubs darwin).
- **`isolation.bats` gets a second pattern**: `$XDG_RUNTIME_DIR/pmm`, `${XDG_RUNTIME_DIR}/pmm`
  (with a `:-` default too) and `/run/user/<anything>/pmm`; a test's own `$T/run/pmm` and
  `pmm-bats` do not match (probe test).
- **ruby.toml uses `{{ env.HOME }}/.local/share`**, not `XDG_DATA_HOME` (deviation, small): the
  file's other path is spelled that way, and mise's template has no XDG default. Same path on
  every machine that does not move `XDG_DATA_HOME`.
- **The new tests live in `packages/anfs/tests`** (`pmm-runtime.bats`, `xdg.bats`; pcm's in
  `pcm-local-env.bats`), as config/04 did: pmm's and pcm's own suites are off. pcm's `edges.bats`
  and pmm's helper were updated so they stay true when turned back on; no case was duplicated there.
- `pmm-runtime.bats` builds `/tmp/pmm-<uid>` with `printf` to compare against: isolation's pattern
  is there to stop a test from using the live directory, and this one only compares strings
  (commented in the file).

## Tests (all through `anfs work run config-05 …`, `</dev/null`)

- While working: `pcm-local-env.bats` 15/15 (new: `pcm_state_env`; the edge writes `state.env`
  0600 in a 0700 dir, keeps other lines, replaces the old token, makes the directory, writes
  nothing under `$PCM_CONFIG_HOME`; `bin/pcm` exports `PCM_STATE_HOME`; environment > `local.env`
  > `state.env`; `pcm show`'s `state` line without the value; `pcm remove -y` deletes it and keeps
  `local.env` and another container's), `pmm-runtime.bats` 6/6 (PMM_SOCK_DIR wins; Linux stubbed
  with `XDG_RUNTIME_DIR` a directory, with a trailing `/`; unset, not a directory, macOS; both
  live defaults refused under bats; bin/pmm's default and the shipped `Include`;
  `ssh_config_write` writes the state file, nothing under `~/.ssh`, drops a stopped machine),
  `isolation.bats` 8/8 (2 new), `xdg.bats` 3/3 (pcm's and pmm's hooks make nothing under
  `~/.config`, volumes still made; no core hook writes config or `~/.ssh`; install.sh seeds no
  `user.list`), ppm `messages.bats` 2/2.
- **Once, at the end**: `packages/anfs/tests/all`: **590 passed, 0 failed**, 43 files (pmm and pcm
  off), 63 s.
- stack, `ANFS_TEST_LIB` = the job's core lib: `packages/ruby/tests` **9/9** (new
  `default-gems.bats` 2: the toml's path, nothing names `.config/ruby/default-gems`; `install_gem`
  appends to the DATA file, sorted, once each, nothing under `~/.config`). No other stack code
  changed (READMEs, a compose comment, a schema comment).
- `bash -n` on every changed script (pcm, pmm, ppm, install.sh; ppm's `core.sh` under `/bin/bash`).

## Not run

- **pmm's and pcm's suites** (off since 2026-10-19). Changed there and not run: pcm `edges.bats`
  (the token now in `state.env`), pmm `helper.bash` (`PMM_SSH_CONFIG`), and everything they cover
  in `runtime.sh`, `inspect.sh`, `remove.sh`, the edge, `core.sh`, `interfaces.sh`, `implode.sh`.
- **shellcheck**: not installed on this host; `bash -n` only.
- **Deferred to the end of the roadmap**: pmm's e2e on Debian (sockets under `$XDG_RUNTIME_DIR/pmm`,
  `ssh <machine>` through the state file), `roundtrip debian` and `roundtrip-macos` (a fresh
  install.sh with no `user.list` seed; core/pmm linking `~/.ssh/config.d/05-pmm.conf`).
- Nothing installed, no machine or container touched; `~/.config`, `~/.local/state`, `~/.ssh` only
  read.

## Deviations from the plan

- Finding 1 dropped (done by plan 01). Finding 10's README line added (the plan assumed it).
- Tests in `packages/anfs/tests` rather than pmm's and pcm's `tests/` (above).
- ruby.toml's path spelled with `env.HOME` (above).
- Beyond the plan: `pcm remove`'s summary line; stack/ssh's README paragraph; AGENTS.md's Testing
  line names Linux's socket directory.

## Open questions (for the person)

1. **`05-pmm.conf` hard-codes `~/.local/state/pmm/ssh_config`.** Fine wherever `XDG_STATE_HOME` is
   the default (this Mac, the test boxes). Accept, or should pmm instead write a one-line fragment
   on install (a side effect the rule forbids) or ship per-platform variants?
2. **`ANFS_CONFIG_HOME` is still made by install.sh's `clone_anfs`** (the base packages link into
   it). Recorded as OK; say if you want it left to the linker.

## By hand, for the person after merge (in this order)

Read on this Mac 2026-10-22: `~/.ssh/config.d/05-pmm.conf` is a plain file pmm wrote (Oct 20),
holding `Host dj-tester` (port 2223), and `dj-tester` is running (qemu pid 2782);
`~/.ssh/config`'s first line is `Include config.d/*.conf`; `~/.config/pmm/machines/images` and
`networks` are empty, `machines/debian-13/machine.yml` is yours; `~/.config/ruby/default-gems`
(0600, 22 gems) exists and `~/.local/share/ruby` does not; `~/.config/ruby/local-gems` is
gem-local's (stays); no `~/.config/pcm/containers/cloudflared/` and no `~/.config/pcm/env`;
`~/.config/anfs/user.list` is a plain file (stays).

1. Pull: `anfs src update` (core, stack). macOS: pmm's sockets stay in `/tmp/pmm-501`, so the
   running `dj-tester` is unaffected.
2. pmm's ssh entries move to STATE. Keep the running machine's entry, then let ppm link the new
   fragment in place of the written one:

   ```bash
   cp ~/.ssh/config.d/05-pmm.conf ~/.local/state/pmm/ssh_config
   rm ~/.ssh/config.d/05-pmm.conf
   ppm install -f core/pmm
   ls -l ~/.ssh/config.d/05-pmm.conf              # a link into sources/core/packages/pmm
   ssh -G dj-tester | grep -E '^(port|user) '     # port 2223, user pmm
   ```

3. The empty directories the old pmm hook made (nothing reads them; `pmm image new` makes
   `images/` again when used):

   ```bash
   rmdir ~/.config/pmm/machines/images ~/.config/pmm/machines/networks
   ls -A ~/.config/pmm/machines                    # debian-13
   ```

4. ruby's default gems move to DATA (`ruby.toml` is a linked file: the pull already points mise there):

   ```bash
   mkdir -p ~/.local/share/ruby
   mv ~/.config/ruby/default-gems ~/.local/share/ruby/default-gems
   exec zsh -l
   echo "$MISE_RUBY_DEFAULT_PACKAGES_FILE"         # ~/.local/share/ruby/default-gems, expanded
   ls -A ~/.config/ruby                             # local-gems only
   ```

5. The tunnel token: nothing on this Mac (no cloudflared settings). On a machine that runs
   cloudflared, take the token out of the person's file and let `pcm up` fetch it into STATE (a
   line left in `local.env` would win over it):

   ```bash
   f=~/.config/pcm/containers/cloudflared/local.env
   [ -f "$f" ] && sed -i.bak '/^TUNNEL_TOKEN=/d' "$f" && rm "$f.bak" && { grep -q . "$f" || rm "$f"; }
   pcm up cloudflared
   ls -l ~/.local/state/pcm/containers/cloudflared/state.env   # -rw-------
   ```

6. Nothing else needs a reinstall (`pcm`'s and `ppm`'s changes arrive with the pull; pcm's hook
   change only matters on a fresh install).
7. On a Linux host running pmm machines: `pmm down` each one **before** pulling (their sockets move
   to `$XDG_RUNTIME_DIR/pmm`; a running one would be unreachable by the new pmm), then step 2.
8. Run pcm's and pmm's suites: `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm pmm`.
