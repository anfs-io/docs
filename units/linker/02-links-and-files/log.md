---
plan: linker/02-links-and-files
job: linker-02
date: 2026-10-22
---

# Log: linker/02-links-and-files

Built against the plan with its "Update 2026-10-21 (after review)": `src:` allowed in
`package.yml`, per-tool opt-in of entry kinds, `platform()`/`os()` in `lib/anfs/platform.sh`,
`system.list` stays a link (`files:` ships with its tests only), `anfs src list` shows the clone's
origin.

## What was built

- **`lib/anfs/link.sh`**
  - **Directory entries**: a path in a list may be a directory of `<from>`; plan, apply, check and
    unlink treat it as one link. A real directory at its place is `conflict` (`a directory`), or
    `a directory, with links into another package: <owner>` when it holds a link into the sources
    that is not this entry's (`_anfs_link_holds`, one `find`, only in that case).
  - **On the way into the sources**: `_anfs_link_dir` follows a link on the way only when it
    resolves inside `<to>` and not into `$ANFS_SOURCES_HOME`; otherwise `inside a directory link:
    <rel> -> <owner>`. `anfs_link_check` now loads the sources too, so it says `in the way` there.
  - **`anfs_link_text`** (relative, or the target itself when only `/` is shared); the plan's
    stow-form text and the check's expected text are absolute in that case, and an absolute link
    is `keep`/`ok` (unmarked) there.
  - **`anfs_link_one`** uses it and refuses (1) when anything is at the path.
  - **`anfs_link_src_state`** (`ok`, `missing`, `missing src`, `wrong`, `in the way`) and
    **`anfs_unlink_src`** (only while it resolves to `src`), on canonical paths.
  - `anfs_unlink` never removes through a directory link into the sources (a source's own link).
- **`lib/anfs/manifest.sh`** (new, bash 3.2, bootstrap set, yq): `anfs_manifest_read <yml>...`
  (one yq over all; per file only when that fails, a malformed one as an `error` row; rows
  `<i>\t<kind>\t<path>\t<dest>\t<platforms>`, kinds `dir|path|src|file|error|extra`;
  `ANFS_MF_YQ_EXTRA` rides along in the same yq, which is how wsm reads `skills:`),
  `anfs_manifest_plan <rows> <tree> <platform> [files-file]` (`ANFS_MF_LINKS`, `_SRCS`, `_COPIES`,
  `_OFF`, `_DIRS`, `_ERR`; 2 on any refusal), `anfs_manifest_match`. `ANFS_MF_KINDS` is the
  per-tool opt-in (default every kind; ppm and wsm take all), `ANFS_MF_SRC_BASE` the base of a
  relative `src` (wsm sets it; ppm does not, so it is refused). `_expand_vars`/`_norm` moved here
  (`_anfs_mf_expand`, and link.sh's `_anfs_link_norm`).
- **`lib/anfs/files.sh`** (new, bash 3.2, bootstrap set, no yq): `anfs_files_plan`/`_apply`,
  copy-if-missing, states `missing`, `present`, `differs`, `in the way`, `skip`; `cp -p` to a temp
  name beside the target, then a hard link into place (fails on an existing name, so a file that
  appeared meanwhile is never overwritten). Its header says it is plan 03's home.
- **`lib/anfs/platform.sh`** (new, bootstrap set): `platform()` and `os()`, moved unchanged from
  ppm's `platform.sh` and `core.sh` (`PPM_OS_RELEASE` kept as is).
- **ppm** (0.2.8)
  - `stow_package <dir> [read]` reads the manifest; invalid: `ppm_fail` with every error, returns
    2. Then **one plan** (`_pkg_plan`) over links, copies and links out of the tree; every
    `other`, `conflict`, `in the way` and `wrong` in the one WARNING block; `-f` clears what it may
    (`_backup_file`); `differs` never. Apply: links, copies, `{src}` links (`anfs_link_one`;
    `missing src` said, not made). `STOWED_FILES` (a directory link one path), `STOWED_SRCS`
    (tracker `links: [{dest, src}]`), everything placed joins `PPM_SKIP_FILE`.
  - `install_single_package` checks the manifest before any hook: invalid, the package is skipped
    with every error and the run goes on; a conflict stops the run with `exit 1` (it did before,
    through `set -e`; now said).
  - `stow_subdir`/`plan_subdir` gone; `plan_package` is `-n` (copies as `copy`/`present`/`differs
    (yours; left as is)`/`in the way`, `{src}` links as `keep`/`link`/`missing src`/`wrong`, and
    `off  <path>  (<platforms>)`; an invalid manifest prints its errors and makes `-n` exit 1).
  - `unstow_package`: tracker files + tracker `links:`; without a tracker the manifest's lists;
    with an invalid manifest too, the tree. `meta_cleanup_stale` also drops `{src}` links whose dest
    left. `_tracker_add_file`'s fallback uses `_pkg_links`.
  - mise: `PPM_MISE_TOMLS` collects the tomls each layer actually linked (a directory link that
    covers `.config/mise` included); `_mise_install_declared` reads only those.
  - `ppm show` prints a "Placed:" block: each `links:`/`files:` entry, its platforms, and when
    installed its state (`off here` for another platform's); an invalid manifest's errors.
  - `file.sh`: `claim`, `add` and `protect` refuse a path inside a directory link ("~/x is inside
    core/anfs's directory link ~/.local/lib/anfs").
  - `PPM_CORE_KEYS` gains `links files`; `bin/ppm` sources `platform link manifest files` from
    `~/.local/lib/anfs`, no fallback.
- **install.sh**: `ANFS_BOOTSTRAP_LIBS="platform.sh link.sh manifest.sh files.sh"` (all three
  `source_libs` cases go through the variable); `anfs.local.conf` seeded with its header line only;
  comments (anfs.conf linked, system.list linked, `--script-only` needs yq).
- **sources.sh**: `gitsrc_hook_system_writable` and `GITSRC_SYSTEM_LIST_HINT` gone; `src ssh` on a
  system-list https entry: `<s>: origin -> git@...; system.list's https URL is only used for the
  first clone`, or `<s>: https in system.list, only used for the first clone` when there is no
  clone to switch. `src list`'s `url` is the clone's `origin` when cloned (one `git remote get-url`
  per cloned source), else the list's; no `list_url` (the readers, anfs-ui's sources view, only
  display it). `src remove`'s wording unchanged (the update made the original rewording moot).
- **wsm** (`WSM_VERSION` 0.8.0, package 0.1.9): `_meta_load` reads every needed space's
  `links:`/`files:` and `skills:` in one `anfs_manifest_read`; `_mf_plan` plans a space over its own
  files, plus wsm's refusals (a directory link that holds/is/lies in a child space or overlaps a
  repository; a directory link at a hidden directory when `skills:` is declared; `space.yml` in
  `files:` or a path entry). `_files_of` is the manifest's list (a directory link one entry),
  `_state_load` loads the manifests of the sources it checks in one yq. Invalid space: `wsm ls`
  STATUS `invalid` with the first error as NOTE; install fails it with every error. `{src}` links:
  `anfs_link_src_state`, made with `anfs_link_one` (relative), removed with `anfs_unlink_src`.
  `files:` planned with the links (one in the way fails the space before anything is linked; `-f`
  moves it aside with `_backup`), copied after the links; missing/in the way count in `wsm ls`,
  `differs` not; `wsm ls <id> --json` adds `copies`. `_add_one` uses `anfs_link_one`. Clone
  fallbacks for `link.sh`/`skill.sh` gone. Template and help: the vocabulary and `files:`.
- **Packages**: core `anfs` (`links: [.local/lib/anfs/]`), `pcm`, `pmm` (their lib dirs); stack
  `op` (`macos/` and `linux/` files moved to `home/`, two path entries with platforms; README,
  AGENTS.md, package.yml comment), `tailscale` (`.local/lib/tsm/`), `rpi-imager`
  (`.local/lib/rpi-imager/`), `bash/install.sh` header, `ssh/install.sh` comment (a line in the
  user's file, a candidate for `setup:`); user `host` (`platforms: [macos]`, `macos/` → `home/`),
  `ssh` (`authorized_keys` → `home/`, `links: [{path: .ssh/authorized_keys, platforms: [macos]}]`).
  No package in core, stack or user has a `macos/`, `linux/`, `debian/` or `fedora/` directory.
- **ovm** (stack obsidian): sources `link.sh`; `_link_make` uses `anfs_link_one`; README says
  relative.
- **Docs**: core AGENTS.md (Package Structure, package.yml with the table and refusals, mise,
  Declared Resources, anfs.conf, wsm: spaces, `anfs src list`'s url, Key Files, Claiming,
  Linking, Lib Structure, the bootstrap set, `-r core/anfs`, Testing); ppm, wsm, anfs READMEs.
  `packages/pcm/legacy/README.md` still names `$PPM_GROUP_ID`: legacy, left.

## Commits (branch linker-02, not pushed)

core:
- `6877962` lib/anfs/link.sh: directory entries, the on-the-way rule into the sources,
  anfs_link_text (absolute across /), anfs_link_one's guard, anfs_link_src_state/anfs_unlink_src
  (anfs 0.2.15)
- `d8132a4` lib/anfs: manifest.sh and files.sh; bootstrap set parses them
- `3c89eaf` ppm and install.sh on manifest.sh and files.sh; PPM_GROUP_ID gone; mise on the linked
  tomls; platform.sh to lib/anfs; core anfs/pcm/pmm directory links; src ssh / src list (ppm 0.2.8,
  pcm 0.1.10, pmm 0.1.5)
- `07e84c5` wsm on manifest.sh and files.sh; 0.8.0 (package 0.1.9)
- `9192a38` docs (AGENTS.md, READMEs)
- `ac3c7de` docs: stow_package, not stow_subdir (two stale mentions found by the closing grep)

stack:
- `b2a4037` obsidian: ovm on anfs_link_one; tests (package already 0.2.1 unpushed; `OVM_VERSION`
  kept equal to it)
- `2408e71` op, tailscale, rpi-imager: links:; bash and ssh comments (bash 0.1.2, op 0.1.5,
  rpi-imager 0.2.2, ssh 0.1.3, tailscale 0.1.3)

user:
- `710b294` host: platforms: [macos], files in home/; ssh: authorized_keys in home/, macOS only
  (host 0.1.1, ssh 0.1.1). Only those two packages; nothing else in the person's repository.

The commit order is the plan's (1–8); the one extra docs commit in core is at the end of core's
list, so every stopping point is still clean.

## Decisions made during the work

- **`{path: x}` without platforms is refused** like a `links:` string without `/` (it says
  nothing the default does not). The plan lists only the string.
- **A directory link over a directory holding another package's links is never cleared by `-f`**
  (the test spec: "a conflict in either install order, and -f does not clear it"). A directory of
  only this package's own links (or nothing of the sources) is `a directory`, which `-f` moves aside
  whole — the by-hand case of `~/.local/lib/{pcm,pmm}`.
- **`anfs_unlink` never removes through a directory link into the sources.** Without it, the first
  install after a package gains a directory link could unlink a symlink that is a file *of the
  checkout* (an old per-file tracker entry under the new directory link). Not in the plan.
- **`_ANFS_ABS` fixed**: it was set for every link whose text was not the expected one (the
  relative prefix was added before the test), not only absolute ones. Now only an absolute text is
  marked, and none where absolute is expected.
- **An unset variable in a `src` makes the whole space (or package) invalid**, as every refusal
  does; before, only that entry was `invalid` and the rest installed. The plan's refusal rule
  ("every error listed, nothing of that package or space is installed") read literally.
- **`wsm ls <id> --json`: copies are `copies: [{path, status}]`**, not `files:` as the plan says:
  `files` already holds the linked files' states (`[{file, status}]`), and a reader of one shape
  must not get two.
- **`ppm show` has no `--json`** (it is a detail view, never was one), so "gains links and files"
  became the human "Placed:" block.
- **wsm's "hidden agent directory" rule** is: a directory link whose first component starts with
  `.`, when the space declares `skills:`.
- **yq quirks** met on the way (4.53.6): `!=` against a `"!!..."` string is always true, and `or`
  binds looser than `==` unless parenthesised; the reader uses `==`, `not` and parentheses.
  Recorded in manifest.sh.
- bash 3.2: a `local a=x b=$a` sees no `a` (files.sh had one; fixed, the 3.2 test caught it).

## Tests (all through `anfs work run linker-02 …`, `</dev/null`)

- While working: `link.bats` 35 (8 new: directory entries — link/keep/replace/conflict/unlink and
  a file added upstream; a directory holding another package's links; the check on a directory
  entry; through a directory link into the sources a conflict, inside the target followed;
  `anfs_link_text` and absolute plans; `anfs_link_one` refusals; `anfs_link_src_state` and
  `anfs_unlink_src`; unlink never through a directory link), `manifest.bats` 14 (new),
  `files.bats` 7 (new), `bash.bats` 10, `sources.bats` 18 (2 rewritten: ssh never edits the system
  list; a system source's clone switched and `src list` showing the origin), ppm 67 (17 new in
  `ppm/tests/manifest.bats`: directory links in both orders and `-f`, the old per-file directory
  under `-f`, protect/add refused, remove with and without a tracker, platform entries on macOS,
  Debian, Fedora with `mise` stubbed and its log asserted, a higher layer's toml, a `macos/`
  directory and `PPM_GROUP_ID` ignored, `{src}` links, `missing src`, `off`, relative `src` refused,
  `files:` copied/differs/recopied/left on remove, in the way in the same block and `-f`, protected
  skipped, an invalid package skipped while its dependent installs, `ppm show`), wsm 103 (96 old,
  4 of them updated to relative links and the new invalid rule; 7 new: a directory link, invalid
  directory links, a platform entry, `files:` twice, `space.yml` in `files:`, `wsm add`).
- **Once, at the end**: `packages/anfs/tests/all`: **501 passed, 0 failed**, 34 files (pmm and pcm
  off), 56 s.
- stack, with `ANFS_TEST_LIB` at the job's core lib: `packages/obsidian/tests` **59/59** (one new:
  an absolute link made before is `ok` and `link rm` takes it; the link texts asserted relative).
  `packages/op/tests`, `packages/tailscale/tests`, `packages/ssh/tests`: 64 passed, **5 failed —
  the same 5 on the base commit** (`950cc21`, checked from a `git archive` of it under the job's
  TMPDIR, with the job's lib and with main's): `ssh-hook.bats` 25, 31, 34, 35 and `tsm.bats` 61.
  Not touched here; worth a look on its own.
- anfs-ui: not run (unchanged).
- Greps: `PPM_GROUP_ID` in core and stack finds only `pcm/legacy/README.md`, the AGENTS.md line
  saying it is gone and ppm's test that it changes nothing; no `stow_subdir`, `plan_subdir`,
  `gitsrc_hook_system_writable` or `GITSRC_SYSTEM_LIST_HINT` left.

### This Mac, copied (temp HOME under the job's TMPDIR; the real side only read)

Built as plan 01's log describes: the 10 sources without `.git` (core, stack and user from the
job's worktrees), `~/.config/anfs`, ppm's trackers, `cp -P` of what the real `$HOME` has at each
of the 234 tracked paths (216 present), and the links under `~/spaces`. Run with `env -i
HOME=<copy> PATH=<copy>/.local/bin:/opt/homebrew/bin:/usr/bin:/bin TMPDIR=<job's>`.

- **Right after the pull, ppm cannot start**: `~/.local/lib/anfs/platform.sh: No such file or
  directory` (the per-file directory has no link for the new libraries). This is by-hand step 1.
- With the three new libraries linked into the copy's per-file directory for the moment,
  `ppm install -n core/anfs core/pcm core/pmm`: `conflict ~/.local/lib/anfs (a directory; -f moves
  it aside)`, the same for `~/.local/lib/pcm` and `~/.local/lib/pmm`; `skip
  ~/.config/anfs/system.list` (protected here); everything else `keep`.
- After step 1 on the copy (the directory held only links; removed, one link made), every
  installed layer (40; 7 trackers have no source any more: user/pvm, pdt/pcs, pdt/python,
  spaces/cnfs, spaces/rjayroach, spaces/rws, utils/ventoy), each name's layers in priority order
  with `-s`: **144 keep, 3 link, 2 replace, 2 conflict, 2 other, 4 skip, 1 off**, and the copy
  unchanged (375 paths with link texts and mtimes, before and after):
  - `keep ~/.local/lib/anfs` (core/anfs: one link now).
  - `conflict ~/.local/lib/pcm`, `~/.local/lib/pmm` (`a directory; -f moves it aside`): by-hand
    step 3.
  - stack/op: `replace ~/.ssh/config.d/99-op.conf` (its link points into `macos/`), `off
    ~/.config/mise/conf.d/op.toml (linux)`, `link ~/.local/bin/opm` (new in stack since its last
    install, as in plan 01's log); no `mise install` is planned (`-n` never runs one; the real
    rehearsal below ran none either).
  - user/ssh: `replace ~/.ssh/authorized_keys`, `keep ~/.ssh/config.d/99-defaults.conf`. Its
    tracker also lists `10-local.conf` and `90-mac.conf`, dangling links into `macos/` since those
    files left the repository (user `28d29a2`); `-n` does not show them, a real install's
    `meta_cleanup_stale` removes them (seen in the rehearsal).
  - The rest as plan 01 found it: `link ~/.local/bin/git-account` (stack/git), `link
    ~/.local/bin/ssh-key` (stack/ssh), `other ~/.config/anfs/anfs.conf` (user/anfs ships one; the
    live link is core/anfs's), `other ~/.config/git/config` (rws/git over user/git), `skip` for
    the protected `user.list` and `system.list` and the higher layers' `anfs.conf` and git config.
- Not installed here, so all `link`: stack/tailscale (`~/.local/lib/tsm` as one entry),
  stack/rpi-imager (`~/.local/lib/rpi-imager` as one entry), user/host (3 files, macOS).
- **A rehearsal of by-hand steps 2–4 on the copy** (`-c`, so no hooks and no declared software;
  `brew`, `mise` and `git` stubbed first on PATH, the source marks fresh: the stub log stayed
  empty): `ppm install -c -s core/anfs`, `ppm install -c -f -s core/pcm core/pmm` (moved the two
  per-file directories to ppm's backup, then one link each), `ppm install -c -s stack/op user/ssh`
  (99-op.conf and authorized_keys relinked from `home/`, the two dangling `config.d` links gone,
  the tracker lists 2 files). Every layer planned again: 149 keep, 2 link (git-account, ssh-key),
  2 other, 4 skip, 1 off; nothing in the way.
- **`wsm ls`** on the copy: the same 11 rows as before; `anfs-dev`'s `sources` link (absolute)
  reads `ok` once it points at the copy's own sources (copied as is, it pointed at the real ones,
  so it read `wrong` in the copy only). Best of 3: **215 ms** (the main branch's wsm 0.7.0 on the
  same copy: 185 ms; one yq per run in both: 0.8.0 reads every space of the checked sources in it,
  not only the installed ones, and sources three more libraries).

## Not run

- Deferred to the end of the roadmap (boot a box or a VM): `roundtrip debian` (op's `op.toml`
  linked and `mise install op` run, its macOS file absent; `~/.local/lib/anfs` one link from the
  bootstrap), `roundtrip fedora`, `roundtrip-macos`, the harness, pmm's e2e.
- pmm's and pcm's suites are off (ROADMAP, 2026-10-19) and were not run; `pcm` and `pmm` change
  only in `package.yml` (`links:`).
- A real `install.sh` (piped or from a checkout): `source_libs`' download path fetches the three new
  libraries from GitHub's raw URL, which works only once they are on main.
- anfs-dev is not in the job: `roundtrip-macos`'s check that "the bootstrap set parses under
  /bin/bash" lists `~/.local/lib/ppm/{core,platform,...}.sh` and should gain
  `~/.local/lib/anfs/{platform,link,manifest,files}.sh`; `move.sh` still loops over
  `${PPM_GROUP_ID:-}` and calls stow (broken since plan 01; plan 03's). Follow-ups.
- Nothing was installed on this host; the real `$HOME`, `~/spaces`, `~/.local` and `~/.config`
  were only read, to copy them.

## Open questions

1. **`wsm ls` 185 → 215 ms on this Mac's copy.** The manifest reader covers every space of the
   checked sources (what each links depends on it), not only installed ones. Acceptable, or should
   a space with no `links:`/`files:` key skip the read (a grep per `space.yml` first)?
2. **An unset variable in a `src` invalidates the whole space or package** (decision above). On a
   machine where the variable is legitimately unset (a vault path set only on one Mac), the space
   would not install at all. Keep it, or let an unset variable be a per-entry `missing src`
   instead?
3. **`copies` instead of `files`** in `wsm ls <id> --json` (decision above): fine?
4. **`{path: x}` without platforms refused** (decision above): fine?
5. **The five stack test failures** in `op/tests/ssh-hook.bats` and `tailscale/tests/tsm.bats`
   exist on the base commit too; nobody's plan touches them.
6. `ppm` and `anfs` still source `bash.sh` "only if it exists", a leftover of per-file linking;
   AGENTS.md says so. Make it unconditional now that `lib/anfs` is one link?

## By hand, for the person after merge (in this order)

ppm and wsm source the new libraries from `~/.local/lib/anfs`, which today is a directory of
per-file links, and there is no fallback any more: after the pull, `ppm` stops at `platform.sh: No
such file or directory`. So the directory link comes **first** (it can be done before the pull too:
the clone's `lib/anfs` and the installed links are the same files until then).

1. **Make `~/.local/lib/anfs` one link** (only if it holds nothing but links, and the target is
   there; else it prints why and changes nothing):

   ```bash
   d=~/.local/lib/anfs t=../share/anfs/sources/core/packages/anfs/home/.local/lib/anfs
   if [ -n "$(find "$d" ! -type l ! -type d -print 2>/dev/null)" ]; then echo "not only links in $d: look first"
   elif [ ! -d ~/.local/share/anfs/sources/core/packages/anfs/home/.local/lib/anfs ]; then echo "no core clone's lib/anfs"
   else rm -r "$d" && ln -s "$t" "$d" && ls "$d/"; fi
   ```

2. `anfs src update` (pulls core and stack; the user source is merged on its own, below).
3. **pcm and pmm** (their directories hold only links; checked read-only 2026-10-22):

   ```bash
   for x in pcm pmm; do d=~/.local/lib/$x
     if [ -z "$(find "$d" ! -type l ! -type d -print 2>/dev/null)" ]; then rm -r "$d"; else echo "not only links in $d: look first"; fi
   done
   ppm install -n core/anfs core/pcm core/pmm   # keep for ~/.local/lib/anfs, link for lib/pcm and lib/pmm
   ppm install core/anfs core/pcm core/pmm
   ```

   (Instead of the `rm -r`, `ppm install -f core/pcm core/pmm` moves the two directories to
   `~/.local/state/ppm/backup/files/.local/lib/{pcm,pmm}`; a later `ppm remove` would put them
   back, so delete them and their two `backup/manifest` lines if you go that way.)
4. `ppm install -n stack/op` (`replace ~/.ssh/config.d/99-op.conf`, `off …op.toml (linux)`), then
   `ppm install stack/op`.
5. Once the user branch is merged and pulled: `ppm install -n user/ssh` (`replace
   ~/.ssh/authorized_keys`), then `ppm install user/ssh`; it also drops the two dangling
   `config.d` links (`10-local.conf`, `90-mac.conf`) its tracker lists. `user/host`,
   `stack/tailscale` and `stack/rpi-imager` are not installed here: nothing to do.
6. `sed -i '' '/^PPM_GROUP_ID=/d' ~/.config/anfs/anfs.local.conf` (it holds `PPM_GROUP_ID=macos`;
   nothing reads it now).
7. **The two absolute links on this Mac** (`find ~/spaces -maxdepth 4 -type l -lname '/*'` lists
   exactly these today), made relative:
   - `rm ~/spaces/anfs-dev/sources && wsm install anfs-dev`
   - `rm ~/spaces/anfs-dev/apex && (cd "$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/apex" && ovm link apply)`

Not a step of this plan (the update's note): `~/.config/anfs/system.list` is protected here, so it
is a plain file. To have it a link again: `ppm file unprotect ~/.config/anfs/system.list`, `rm
~/.config/anfs/system.list`, `ppm install core/anfs`.
