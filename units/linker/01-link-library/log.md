---
plan: linker/01-link-library
job: linker-01
date: 2026-10-21
---

# Log: linker/01-link-library

## What was built

- **`lib/anfs/link.sh`** (core/anfs, bash 3.2, functions only): `anfs_link_plan`,
  `anfs_link_apply`, `anfs_unlink`, `anfs_unlink_tree`, `anfs_link_one`, `anfs_link_owner`,
  `anfs_relpath` (wsm's `_relpath` moved here, without its `dirname` forks).
  - The plan prints `<class>\t<path>`, plus `\t<detail>` for `other` (the owner, as
    `anfs_link_owner` says it) and `conflict` (`a file`, `a directory`, `a link to X`,
    `a file where a directory goes: X`, `a link out of the target on the way: X`). It returns 1
    on any `other` or `conflict` and 2 when it refuses (see Decisions).
  - The link text is stow's: `../` for each directory between the link and `<to>`, then
    `<to>`→`<from>` relative, then the file (`../../.local/share/anfs/sources/...`). It is not
    the shortest relative path. That is what makes it byte for byte stow's, and it is what the
    live links on this Mac look like.
  - Paths are compared canonically: the link's directory is resolved, then the link text is
    applied lexically, so dangling links get classified too. `anfs_link_owner` also names a
    local-path source (a link in the sources dir) by its alias.
  - `keep` is the fast path: the link text equals the expected text, a string compare with one
    `readlink`. Only a link whose text differs gets resolved.
- **ppm** (`installer.sh`, `file.sh`, `implode.sh`, `bin/ppm`):
  - `stow_subdir` runs the plan. Under `-f` it deletes `other` links and backs up `conflict`s
    with `_backup_file`, then `anfs_link_apply`s. Without `-f` it prints stow's wording with every
    file listed and returns 1 (`WARNING! stowing <repo/pkg> would cause conflicts: ... All
    operations aborted.`). `remove_dangling_links`, `force_remove_conflicts`, `_link_points_into`,
    `_stow_ignore_arg`, `is_stow_ignored` and `PPM_IGNORE_ARGS` are gone.
  - `PPM_SKIP_FILE` is a temp file of exact paths. `_reset_skip_file` replaces
    `_reset_ignore_args` and seeds it from `protected.yml`; each layer appends its files. core.sh's
    EXIT trap deletes it with the message file.
  - `meta_cleanup_stale` is now `anfs_unlink` over the tracker's old files minus the new ones.
    `STOWED_FILES` keeps its three readers.
  - `unstow_package` is `anfs_unlink` over the tracker's files, or `anfs_unlink_tree` when there
    is no tracker.
  - `package_links` lists files and links (stow linked links in a package too; the old `-type f`
    left them out of the tracker), never `.DS_Store`.
  - `file.sh`: `_file_claim` (add, claim) and `_file_reset` link the one file with
    `anfs_link_one`. `_file_reset`'s rebuilt ignore list is gone.
  - `implode.sh`: `_implode_sweep_links` decides with `anfs_link_owner`. "Kept:" no longer names
    stow.
  - **`ppm install -n` / `--dry-run`**: `_install_plan` prints, per layer, `link`, `keep`,
    `replace`, `skip`, `other`, `conflict`, and what `-f` would do with what is in the way. It
    skips the source pull, `brew update`, the declared deps, hooks, links and the tracker, and it
    does not ask before a `repo/` or `@category`. Exit 1 when the install would stop.
  - `bin/ppm` and `wsm` source `~/.local/lib/anfs/link.sh`, or the core clone's copy when it is
    not linked yet (see "After merge").
- **wsm**: `_stow`, `_stow_ignores`, `STOW_IGN`, `WSM_STOW_HOME`, `WSM_STOW_IGNORE` and the
  "needs GNU stow" check are gone. `_link_unit` plans and applies `_unit_files`.
  `_unlink_unit` is just the loop over the recorded files. `_add_one` uses `anfs_relpath`. The
  messages and help say link/unlink. `wsm implode` already deleted all of `$WSM_STATE_HOME`, so
  an old `stow/` there goes with it (a comment says so).
- **install.sh**: no stow in `base_formulas`, in the check after `setup_brew` or at the top of
  `stow_base`. `stow_base`'s pre-pass is gone. It is `_reset_skip_file` plus `stow_package` per
  base package. `source_libs` sources `ANFS_BOOTSTRAP_LIBS="link.sh"` (from
  `ANFS_LIB_SUBDIR=packages/anfs/home/.local/lib/anfs`) before the ppm libs, in all three cases:
  the checkout, the clone, and the download. The header, `--script-only` and `brew_env` comments
  follow.
- **Docs**: core AGENTS.md (the table, Repository Layout, wsm, Source Precedence, a new
  "Linking (lib/anfs/link.sh)" section, Claiming and Protecting, Shell Integration, Key Files,
  Package Structure, Dependencies, Lib Structure, the bootstrap set, Testing), README.md,
  ppm's and wsm's READMEs, and the comments in `resolve.sh`, `sources.sh`, `system.list`,
  `packages.sh`, `installer.sh`, `implode.sh` and `packages/anfs/package.yml`. Nothing declared
  stow: neither `packages/ppm/package.yml` nor `stacks/system/stack.yml` named it.
- **anfs-dev**: `roundtrip`'s `build_bootstrap` runs `brew install -q yq mise`, and its header and
  AGENTS.md's line no longer mention stow. `work.bats`' `docs_space` has no "no stow" skip.

## Commits

core (linker-01):
- `ab5e334` lib/anfs/link.sh: link a list of files from one directory into another, as stow
  --no-folding did (the hook bumped anfs to 0.2.13)
- `973afc7` ppm and install.sh link with lib/anfs/link.sh; ppm install -n; no stow in the
  bootstrap (ppm 0.2.7)
- `bff4354` wsm links with lib/anfs/link.sh: no stow, no fake HOME, no ignore list (wsm 0.1.8)
- `ecac598` docs: the linker in AGENTS.md (Linking), the READMEs and comments; stow is not a
  dependency
- `45dd44d` ppm tests: give bats its EXIT trap back after core.sh, so a skip is reported

anfs-dev (linker-01):
- `dd89e90` roundtrip's bootstrap snapshot brews no stow; work.bats needs none (dev 0.1.13)

Three fixes the person approved after the first report (stack was added to the job with
`anfs work add linker-01 stack`):

core:
- `e9f3dd9` ppm: the tracker quotes each file name, so any name reads back as written
- `0ef478e` wsm tests: repo ls --json has ahead (as main's src list and repo list do)

stack (linker-01):
- `10a49e2` nvim: README.md at the package root, not under home/ (nvim 0.1.2; there was no root
  README to merge with)

## Decisions and deviations

- **What `replace` covers.** A link into `<from>` that is absolute, points at another of its
  files, or dangles is replaced, and so is a dangling link anywhere under `$ANFS_SOURCES_HOME`.
  A dangling link anywhere else is the user's, so it is a `conflict`. Before, ppm's
  `remove_dangling_links` behaved the same way. `install.sh`'s old pre-pass deleted *any*
  dangling link at a base package's paths, and now that is a conflict that stops the bootstrap.
  A machine with links left from a much older layout (sources outside
  `~/.local/share/anfs/sources`) would hit it.
- **`keep` is any relative link that resolves to the file**, not only stow's exact text. So
  links that `wsm add` and `ppm file add/claim` write (`anfs_link_one`, the shortest relative
  path) are kept on the next install, with no rewrite. Only an absolute link to the file is
  `replace` (wsm 0.3's).
- **`anfs_link_one` writes the shortest relative path**, as wsm's `_add_one` always did. It does
  not know `<to>`, so it cannot write stow's form. That is why implode's sweep pre-filters on
  `-lname "*anfs/sources/*"` (the sources dir's last two components) and not on
  `.local/share/anfs/sources`: every relative link into the sources contains the former.
- **A name with a tab is refused too**, not only one with a newline (deviation). The plan's
  lines are tab-separated, so a tab in a name would make the path ambiguous. A path in the list
  that `<from>` lacks is also refused (exit 2), since that is how a newline-cut list shows itself.
- **Obstacles on the way are never cleared by `-f`.** "A file where a directory goes" or "a link
  out of the target on the way" (e.g. a `~/.config` that is a link to a dotfiles dir) stop the
  install with or without `-f`. Before, stow refused these too, and `-f` only ever handled the
  file itself.
- **wsm keeps one rule of its own**: a recorded link (`OLD_FILES`) that the plan calls `other` or
  `conflict` is still wsm's to replace, wherever it points. That was the old `_link_unit`
  behaviour, and the plan leaves what to do with `other` and `conflict` to the tool.
- **ppm's function names stay** (`stow_package`, `stow_subdir`, `unstow_package`, `STOWED_FILES`):
  "stow" means "linked into place", as the plan says for help text.
- **The link.sh fallback.** A tool that sources `lib/anfs` from `~/.local/lib/anfs` would break
  right after a pull, because `link.sh` is not linked yet and ppm could no longer install
  core/anfs to link it. So ppm and wsm read the clone's copy when the linked one is missing. The
  plan did not ask for this.
- **`.DS_Store` is never linked, and not tracked by ppm either.** With stow, ppm linked a package's
  `.DS_Store` (stow's built-in list does not include it).
- `backup.bats` and `plan.bats` put bats' EXIT trap back after sourcing core.sh, which sets its
  own. Without that, a `skip` (the stow test, with no stow) was reported as nothing at all: the
  run said "Executed 15 instead of 16". `teardown` deletes the message and skip files.

## ppm move is broken (accepted)

anfs-dev's `packages/dev/home/.local/lib/ppm/move.sh` calls `unstow_package` (still there, but
now it unlinks by tracker), `PPM_IGNORE_ARGS`, `_stow_ignore_arg` (gone) and `stow` (no longer
installed). From this plan on, `ppm move` does not work. As agreed (2026-10-21), there are no
shims. Plan 03 decides its fate. anfs-dev's AGENTS.md prose on move still describes stow (also
plan 03's).

## Tests

All runs went through the job (`anfs work run linker-01 ...`, `</dev/null`).

- `packages/anfs/tests/link.bats` (new, 21): each plan class; two-plus conflicts reported with
  nothing changed; names (space, `+`, `(`, `[`, leading `-`, dot file, a nested one with all of
  them); newline refused once, tab refused; a directory on the way that links out (a conflict) or
  inside (followed); a file where a directory goes; byte for byte against `stow --no-folding`,
  stow's links planned as `keep`, `stow -D` removing ours (skipped without stow); `anfs_unlink`
  leaving a file, a link elsewhere and another package's; `_tree` with no list; `anfs_link_owner`
  (live, dangling, local-path source, not ours); `anfs_link_one`; `anfs_relpath`; plan, apply,
  keep and unlink under `/bin/bash` 3.2. `bash.bats` parses `link.sh` under 3.2 and checks
  `ANFS_BOOTSTRAP_LIBS`.
- `packages/ppm/tests/backup.bats` (16; its stow skip is gone, the cases kept on the new names)
  adds: every conflict named without `-f`; a dangling link into the sources replaced without `-f`;
  layers skipped by exact name with `+` and `.`; `README.md` and `.gitignore` linked and
  `.DS_Store` not; a stow-made install taken over with the inodes unchanged (skips without stow);
  `unstow_package` by tracker and by tree; `meta_cleanup_stale`.
- `packages/ppm/tests/plan.bats` (new, 4): `ppm install -n` on a fresh, an installed and a
  conflicting package, plus layers. The target tree (with link texts), the tracker and the hooks'
  marker file are all unchanged, and `-n -f` changes nothing either.
- **No stow on PATH** (a temp bin dir of links to `/opt/homebrew/bin/*` without `stow`, in place
  of `/opt/homebrew/bin`; `command -v stow` checked empty first):
  `bats packages/anfs/tests packages/ppm/tests packages/wsm/tests`: 361 tests, 360 ok (1 of them
  the stow comparison, skipped), 1 failed: wsm `manage.bats` "repo ls lists the repositories",
  which **also fails on main** (checked with the changes stashed). Its `--json` keys now include
  `ahead` (commit `06b4307`) and the test expects `repository status url`. Not touched here.
- anfs-dev `bats packages/dev/tests/work.bats`, no stow on PATH: 21 passed (`docs_space` runs
  wsm from the job's core).
- **Once, at the end**: `packages/anfs/tests/all`: 424 passed, 1 failed (the same `manage.bats`
  test), 30 files (pmm and pcm off), 54 s.
- **After the three fixes**: `backup.bats` gained two cases (a tracker with `[x]`, `-lead`,
  `*star`, `&amp`, `!bang`, `#hash`, `a: b`, quotes, a backslash, `123`, `true` reads back
  exactly, and fails without the fix; an old unquoted tracker still reads). `manage.bats` and
  `backup.bats` 41/41. `packages/anfs/tests/all`, once: **427 passed, 0 failed**, 30 files
  (pmm and pcm off), 56 s.
- A by-hand check of `_implode_sweep_links` in a temp HOME: it removed a live link, a dangling
  link and a shortest-form link into the sources, and left a link elsewhere. There is no bats case
  for it.
- **Every installed package plans only keep, on an isolated copy.** A copy under the scratchpad,
  never the real `$HOME`: the sources without `.git` (core from the job's worktree), `~/.config/anfs`,
  ppm's trackers, and whatever the real `$HOME` has at each of the 222 paths the installed names
  ship (202 present, copied with `cp -P`). Run with `env -i HOME=<copy> PATH=...`, so no `*_HOME`
  variable reached a real directory. 47 trackers: 7 have no source any more (pdt/pcs,
  pdt/python, spaces/cnfs, spaces/rjayroach, spaces/rws, user/pvm, utils/ventoy) and could not be
  planned. The other 40 installed layers, each name's layers in priority order with `-s`, gave
  **200 keep, 5 link, 2 other, 4 skip, 0 replace, 0 conflict**, and the copy was unchanged
  afterwards (a tree listing with link texts and mtimes, compared before and after). Every link
  ppm made on this Mac is kept as it is. What is not `keep`:
  - `skip`: `user.list` and `system.list` are protected (plain files); core/anfs's `anfs.conf`
    and user/git's `.config/git/config` are a higher layer's.
  - `link ~/.local/lib/anfs/link.sh` (core/anfs): new in this job.
  - `link ~/.local/bin/git-account` (stack/git), `~/.local/bin/ssh-key` (stack/ssh),
    `~/.local/bin/opm` (stack/op): files added to stack since they were last installed. A real
    install links them, as stow would have.
  - `link ~/README.md` (stack/nvim shipped `home/README.md`): the intended change, `README.*`
    under `home/` is linked now. It would have put a `README.md` in `$HOME`; fixed in stack
    (`10a49e2`, the README moved to the package root).
  - `other ~/.config/anfs/anfs.conf` (user/anfs ships one; the live link is core/anfs's) and
    `other ~/.config/git/config` (rws/git ships one; the live link is user/git's). The state on
    this Mac does not follow the layer order today, and stow would have refused these the same
    way. `-f` would relink them to the higher layer.
  - Planning only the installed layers matters: a bare `ppm install -n anfs git ...` also plans
    layers that are not installed (rws/anfs), which a real install would install too.
  - Run time of the plan for all 40: a few seconds, including 35 `ppm path` calls to order the
    layers.

- **One slip:** the first version of `plan.bats` made a real install, to plan over it, without
  stubbing `update_brew_if_needed`. With `PPM_CACHE_HOME` unset, that ran the host's
  `brew update` once (16:07; Homebrew's metadata only, nothing was installed or upgraded). The
  test now stubs the pull, `brew update` and the brew/cask/system lookups.

## Not run (deferred to the end of the roadmap)

- `roundtrip debian` from a bootstrap snapshot with no stow, and its `$HOME` diff. **The cached
  snapshot `localhost/anfs-test:<distro>-bootstrap` still has stow**, because it is built once
  and reused. Remove it first (`podman rmi localhost/anfs-test:debian-bootstrap`, and fedora's)
  or the round trip does not test "no stow".
- `roundtrip fedora`, `roundtrip-macos` (tart), the harness, and pmm's e2e. Nothing that boots a
  box or a VM was run.
- pmm's and pcm's suites are off (ROADMAP, 2026-10-19) and were not run. This plan does not
  touch them.
- A real `install.sh` (piped or from a checkout) was not run. `source_libs`' download path
  (fetching `link.sh` from GitHub's raw URL) only works once this is on main.
- `ppm file add/claim/reset` on the new code has no test of its own: `anfs_link_one` is tested in
  link.bats, and `file.sh` only parses under 3.2.

## Open questions

- **The `other` drift on this Mac** (`anfs.conf`: user/anfs over core/anfs; `.config/git/config`:
  rws/git over user/git). `ppm install anfs git` stops on these, as it would have with stow.
  `-f` relinks them to the higher layer. Is that what you want for rws/git's `config`?
- Fixed after review approval: the tracker now writes each name double-quoted (`e9f3dd9`), and
  `manage.bats` expects `ahead` (`0ef478e`; yq lists keys in insertion order:
  `repository status url ahead`).

## After merge, on this Mac (by hand)

- After `anfs src update` (or a pull of core), ppm and wsm run with `link.sh` from the clone
  until core/anfs is installed again. Then `ppm install core/anfs` links
  `~/.local/lib/anfs/link.sh` (not `-r`: see AGENTS.md). It is worth running `ppm install -n
  core/anfs` first.
- Try `ppm install -n` on a few packages, then a real `ppm install` of one with stow off PATH
  (e.g. `PATH=${PATH//\/opt\/homebrew\/bin:/}` in a subshell).
- stow stays installed here (nothing uninstalls it). `anfs implode --all` still removes it,
  because this machine's bootstrap record lists it.
- `ppm move` does not work until plan 03.
