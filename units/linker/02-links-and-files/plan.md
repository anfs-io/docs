---
---

# Plan 02 — `links:`, `files:` and `platforms:`: one vocabulary in `package.yml` and `space.yml`

Rewritten 2026-10-21 against main (linker 01, wsm-spaces 01–03 and ovm 01 merged), after the
review with the person (`units/linker.md`, "Decided with the person, 2026-10-21"). It replaces
the earlier `02-platform-files` (a `stow:` map and `link_dirs:`).

## Context — read these files first

The job is `anfs work new <job> core stack user`. core holds the work; stack has ovm, three
packages and a comment; **user is the person's own repository**: its worktree is on the job's
branch like the others, only the two packages below change there, and the person merges it
separately (the live checkout has uncommitted work and a local commit of its own; neither is the
job's business). anfs-dev is not in the job (see "Not in this plan").

Paths are core's unless they name another source.

- `units/linker.md` (the Decisions, all of them); plan 01's `log.md`; `units/wsm-spaces.md`
  (decisions 4, 7, 11) and `wsm-spaces/03-skills-and-links/log.md` (`links:` as built)
- `packages/anfs/home/.local/lib/anfs/link.sh` — all of it: `_anfs_link_classify` (the stow-form
  text `"$up$base/$f"`), `_anfs_link_dir` (a link on the way that resolves inside `<to>` is
  followed), `anfs_link_check` (`_ANFS_ABS`), `anfs_unlink_tree`, `anfs_link_one` (fails only via
  `ln`; a directory at `<to-path>` would receive the link inside it), `_anfs_relpath` (goes up to
  `/` and stays relative)
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `stow_package`, `unstow_package`,
  `stow_subdir`, `plan_subdir`, `_install_plan`, `package_links`, `meta_cleanup_stale`,
  `_mise_install_declared` (reads every `home/.config/mise/**/*.toml` and `$PPM_GROUP_ID/...`),
  `_backup_file`. Bootstrap set (bash 3.2)
- `packages/ppm/home/.local/lib/ppm/packages.sh` — `PPM_CORE_KEYS`, `meta_supported` (how
  `platforms:` matches: `linux` covers every distro), `meta_deps` (the per-platform map reader),
  `_tracker_add_file` (falls back to `package_links "$pkg_dir/home"`), `cmd_show`
- `packages/ppm/home/.local/lib/ppm/file.sh` — `_file_claim`, `_file_protect` (refusals added)
- `packages/ppm/home/.local/lib/ppm/platform.sh` — `platform()`
- `packages/ppm/home/.local/bin/ppm` — the `link.sh` fallback (lines 37–43)
- `install.sh` — `ANFS_BOOTSTRAP_LIBS`, `source_libs`, `stow_base`, `install_configs`
  (`PPM_GROUP_ID` seed at ~359; the "anfs.conf and system.list come from linking" comment)
- `packages/anfs/home/.local/lib/anfs/sources.sh` — `anfs_gitsrc_env`'s
  `GITSRC_SYSTEM_LIST_HINT` ("ppm file protect ~/.config/anfs/system.list"), `gitsrc_ssh` and
  `gitsrc_hook_system_writable`, the "tool-managed" messages; `tests/sources.bats` 98, 246, 255
- `packages/wsm/home/.local/bin/wsm` — the fallbacks for `link.sh` and `skill.sh` (lines 47–60);
  `_files_load`, `_unit_files`, `_state_load`, the leftover walk; `_meta_load` (one yq for every
  space's `links:` and `skills:`); `_expand_vars`, `_norm`, `_links_eval`, `_space_links`
  (`ln -s` ~1200), `_remove_links`; `_add_one` (`ln -s` with `anfs_relpath`, ~1946 and ~1957);
  `_written_by_skills`; `wsm new`'s template; help
- stack `packages/obsidian/home/.local/bin/ovm` — `_link_state`, `_points_at`, `_link_make`
  (`ln -s` ~841), `_link_drop`; `tests/links.bats` (asserts absolute link texts)
- `AGENTS.md` (core): "Linking (lib/anfs/link.sh)", "Package Structure" / "package.yml",
  "Declared Resources", "Configuration: anfs.conf", "wsm: spaces", "Claiming and Protecting
  Files", "Key Files", "Lib Structure", the bootstrap set, "Testing"
- The packages this plan converts: stack `op` (`macos/`, `linux/`), user `host` (`macos/` only),
  user `ssh` (`macos/`), core `anfs`, `pcm`, `pmm`, stack `tailscale`, `rpi-imager` (library
  directories), stack `ssh/install.sh` (the `add_to_file` Include line, unchanged), stack
  `bash/install.sh` (the header comment)

## Overview

Today a package says nothing about how its files are placed: everything under `home/` is linked
file by file, and a sibling directory named by `PPM_GROUP_ID` is linked too. A space's
`space.yml` has `links:` of its own shape, made with an absolute `ln -s`; ovm makes absolute
links too. There is no way to link a directory whole, to copy a default the user then owns, or to
say "macOS only" about one file.

After this plan both manifests share one vocabulary, read by one library:

```yaml
platforms: [macos]                 # the whole package (or space); exists for packages already
links:
  - .local/lib/anfs/               # a directory, as one link
  - {path: .ssh/config.d/99-op.conf, platforms: [macos]}
  - {path: .config/mise/conf.d/op.toml, platforms: [linux]}
  - {src: ~/.local/share/anfs/sources, dest: sources}   # a link out of the tree
files:
  - .config/anfs/system.list       # copied when missing, then the user's
  - {path: .config/foo/local.conf, platforms: [debian]}
```

`home/` is the only directory of a package that is linked; anything not named in `links:` or
`files:` is linked file by file everywhere, exactly as now. Every link ppm, wsm and ovm make is
relative, computed on canonical paths, except one whose only common ancestor with its target is
`/`. `lib/anfs` is one link, so a library added upstream is there after a pull.

**One plan, not 02a/02b.** The package conversions in stack and user are six small edits (moves
and keys) that can only be verified with this plan's ppm (`ppm install -n` from the job), and
core's own packages (`anfs`, `pcm`, `pmm`) must convert in the same change as the code, because
the bootstrap installs core/anfs with it. Split off, 02b would be a session of moves waiting on a
merge. The commit order below puts the stack and user packages last, so a run that has to stop
early stops at a clean point (after commit 6) and says so in its log.

## Implementation

### 1. The vocabulary (both manifests)

**`links:`** — an entry is a string or a map:

| Entry | Means |
| --- | --- |
| `<dir>/` (a string ending in `/`) | the directory in the tree as **one link** at the same path in the target |
| `{path: <in the tree>, platforms: [...]}` | only on those platforms. A file: that file. A directory without a trailing `/`: every file under it, per file. With a trailing `/`: the directory as one link, on those platforms |
| `{src: <outside the tree>, dest: <optional>}` | a link at `dest` (default: the basename of `src`) to `src`. `~` and `$NAME`/`${NAME}` expanded (wsm's `_expand_vars`, moved into `manifest.sh`); an unset variable makes the entry invalid. Relative `src`: from the space's directory (as today) in a `space.yml`; refused in a `package.yml` |

**`files:`** — an entry is a path in the tree (a string) or `{path, platforms}`. Each is a file
(not a directory). It is copied to the target when nothing is there, and never again: once there
it is the user's. Removing the package or space leaves it.

**`platforms:`** — `macos`, `linux`, `debian`, `fedora`, matched as `meta_supported` matches a
package's today (`linux` is any distro). On an entry it filters the entry; on the package it
filters the package (unchanged).

**What is linked per file** (the default set): every file under the tree (for a package,
`home/`; for a space, wsm's own files of the space, nearest-space ownership as now), minus the
files under a directory link, minus `files:` paths, minus the files of path entries whose
platforms do not match. Directory links that match are added as single entries.

**Refused** (the manifest is invalid; every error listed, nothing of that package or space is
installed):

- a `links:` string without a trailing `/` (it says nothing the default does not; "for some
  platforms, `{path: ..., platforms: [...]}`")
- a `path` or `files:` entry not in the tree; a trailing `/` on something that is not a directory;
  a `files:` entry that is a directory; an absolute path or `..`
- a directory link inside another directory link; **a platform entry or a `files:` entry inside a
  directory link** (the link brings the whole directory; it cannot leave one file out)
- the same path twice (in `links:`, in `files:`, or across them)
- an unknown platform, or `platforms: []`
- a map with both `path` and `src`, or with neither; a key other than `path`, `src`, `dest`,
  `platforms`
- `dest` outside the target (`..`, absolute): for a package `$HOME`, for a space its directory
  (wsm's `_repo_path_ok`, as now)
- wsm only: a directory link that is or contains a child space's directory or a repository path;
  a directory link at a directory the skills CLI writes when the space declares `skills:`
  (`.agents/`, a hidden agent directory: the CLI would write into the source); `space.yml` in
  `files:`

There is no `ppm validate` today; the checks run wherever the manifest is read: `ppm install`
(and `-n`), `ppm show`, `wsm install`, `wsm ls` (the space's row `invalid`, as a `links:` entry
is today).

### 2. `lib/anfs/manifest.sh` (new)

Bash 3.2, functions only, in the bootstrap set (install.sh's `stow_base` installs core/anfs,
which declares a directory link and a copy). Needs yq, which `stow_base` already has (and
`--script-only` already says it needs). It knows the two manifest shapes and nothing about
trackers, layers or backups.

```
anfs_manifest_read  <yml>...                       one yq over all of them (per file only if that fails, as
                                                   _yq_metas does); rows "<i>\t<kind>\t<path>\t<dest>\t<platforms csv>",
                                                   <i> the yml's index, <kind> dir|path|src|file
anfs_manifest_plan  <rows> <tree> <platform> [files-file]
                                                   the plan input for one manifest; [files-file] is the candidate
                                                   per-file list (wsm's), else every file under <tree>
anfs_manifest_match <platform> <csv>               true when the csv names <platform> (linux: any distro)
```

`anfs_manifest_plan` sets, newline-separated (no associative arrays):

- `ANFS_MF_LINKS` — paths for `link.sh`, relative to `<tree>`: the per-file set and the directory
  links (as plain paths, no trailing `/`)
- `ANFS_MF_SRCS` — `<dest>\t<src>` (expanded, normalised, absolute)
- `ANFS_MF_COPIES` — paths for `files.sh`
- `ANFS_MF_OFF` — `<path>\t<platforms>` filtered out here (for `ppm show`, `-n`)
- `ANFS_MF_DIRS` — the directory links alone (ppm's unlink fallback, wsm's leftover walk)
- `ANFS_MF_ERR` — one line per refusal; return 2 when not empty

The platform is the caller's argument: ppm passes `platform()`; wsm sources
`~/.local/lib/ppm/platform.sh` for it (ppm is a base package and `.local/lib/ppm` stays per file,
so the path is stable). See open question 1.

wsm reads every space's manifest in one yq, as `_meta_load` does now; that call moves into
`anfs_manifest_read` (with `skills:` read alongside, or by `_meta_load` beside it, whichever keeps
one yq per run). `wsm ls` on this Mac stays in its measured range (175 ms, wsm-03's log).

### 3. `lib/anfs/files.sh` (new)

Bash 3.2, bootstrap set (core/anfs copies `system.list` on the first install), no yq. Real files
in a target; for now one operation, copy-if-missing. Its header says it is the home of plan 03's
file operations (the `file` group).

```
anfs_files_plan  <from> <to> <files-file> [skip-file]   "<state>\t<path>[\t<detail>]"; 1 on any in the way, 2 refused
anfs_files_apply <from> <to> <files-file> [skip-file]   the plan, then each missing file copied; nothing on in the way
```

- `missing` (nothing there: apply copies it), `present` (a regular file with the same bytes),
  `differs` (a regular file with other bytes: information only, never touched, `-f` or not),
  `in the way` (a directory, a link of any kind, or an obstacle on the way: the same rule as
  `link.sh`'s `_anfs_link_dir`, which `files.sh` uses), `skip` (the skip set).
- A copy keeps the mode (`cp -p`), is written beside the target under a temp name and moved into
  place only if nothing appeared there meanwhile. Parents are made. Names as data, a tab or newline
  refused, as in `link.sh`.
- One `cmp` per file that exists; copies are few. No fork for `missing`.

### 4. `lib/anfs/link.sh`

- **Directory entries.** A path in a list may be a directory of `<from>`: plan, apply, check and
  unlink treat it as one link. A real directory at its place is `conflict` (`a directory`; ppm
  `-f` moves the whole directory aside with `_backup_file`), a link to it is `keep`.
- **On the way into the sources is a conflict.** `_anfs_link_dir` follows a link on the way only
  when it resolves inside `<to>` *and not into `$ANFS_SOURCES_HOME`* (canonical, local-path
  sources included, `_anfs_link_owned`). Otherwise another package's file at
  `~/.local/lib/anfs/x` would be linked *into core's checkout*. Detail: `inside a directory link:
  <owner>`. ppm's `-f` never clears it (an obstacle on the way, as now); the files.sh rule is the
  same.
- **Relative, but absolute across `/`.** One helper, `anfs_link_text <link-dir> <target>` (both
  canonical): the relative path, or the absolute target when the only common ancestor is `/`
  (`/Volumes/x`, a source on another volume). The plan's stow-form text follows the same rule
  (when `<to>` and `<from>` share only `/`, the text is `<cfrom>/<f>`), and so does the check's
  expected text. `keep`/`ok` for an absolute link stay as they are where absolute is expected;
  elsewhere an absolute link into `<from>` stays `replace` (plan 01's rule). `anfs_relpath` itself
  is unchanged.
- **`anfs_link_one`** uses `anfs_link_text` and refuses (1, nothing made) when anything is at
  `<to-path>`, a link or a directory included: today `ln -s` into an existing directory would
  make the link inside it.
- **A link to a source outside the tree** — `anfs_link_src_state <link> <src>`: `ok` (resolves to
  `src`, relative or absolute), `missing`, `missing src`, `wrong`, `in the way`: wsm's states, moved
  here so ppm and wsm share them. `anfs_unlink_src <link> <src>` removes it only while it still
  resolves to `src`. An absolute link to `src` is `ok`: it is not rewritten (decision 2: by hand).
- `anfs_unlink_tree` is left as is; its callers pass the manifest's list instead when there is
  one (a directory link is not found by walking the files).

### 5. ppm

- **`stow_package`** reads the package's manifest (`anfs_manifest_read` + `_plan` over `home/`
  with `platform()`). Invalid: `ppm_fail` with every error, the package is not installed, the run
  goes on as for any failed package. Then **one plan over links, copies and external links**
  before anything changes: `anfs_link_plan` over `ANFS_MF_LINKS`, `anfs_files_plan` over
  `ANFS_MF_COPIES`, `anfs_link_src_state` per `ANFS_MF_SRCS`, all with `PPM_SKIP_FILE`. Every
  `other`, `conflict`, `in the way` and `wrong` is listed in the one `WARNING! ... would cause
  conflicts` block; nothing is touched unless there is none, or `-f` cleared them (`_backup_file`,
  as now; `differs` is never cleared). Then apply: links, copies, external links.
- **`STOWED_FILES`** (the tracker's `files`) is the links made, the directory link as one path
  (`.local/lib/anfs`). External links go in the tracker as `links: [{dest, src}]`. Copies are not
  recorded: they are the user's once made. All three kinds of target path join `PPM_SKIP_FILE`, so
  a lower layer skips them.
- `PPM_GROUP_ID` goes from `stow_package`, `unstow_package`, `_install_plan` and
  `_mise_install_declared`; the `for subdir in home ${PPM_GROUP_ID:-}` loops become one pass over
  `home/`. A `macos/` or `linux/` directory is ignored, without a warning.
- **`unstow_package`**: the tracker's files with `anfs_unlink` (directory links included: they
  resolve into the package), its `links:` with `anfs_unlink_src`; without a tracker, the
  manifest's `ANFS_MF_LINKS` (not `anfs_unlink_tree`, which would miss a directory link) and
  `ANFS_MF_SRCS`. Copies stay. `meta_cleanup_stale` is unchanged (its list comes from the new
  `STOWED_FILES`).
- **`_tracker_add_file`**'s fallback (`package_links "$pkg_dir/home"`) uses the manifest's list.
- **mise**: `_mise_install_declared` reads the `*.toml` under `.config/mise/` among the files each
  layer actually linked this run (and under a directory link that covers `.config/mise`), collected
  in `stow_package`, not a `find` over every resolved layer. So stack/op's Linux-only `op.toml` is
  never installed on macOS, and a lower layer's toml that a higher one replaced is not read.
- **`-n`** (`_install_plan`, `plan_subdir`): prints the copies too (`copy`, `present`, `differs`
  (`yours; left as is`), `in the way`) and the external links, and what is filtered out for this
  platform (`off  <path>  (macos)`). An invalid manifest prints its errors and makes the run exit 1.
- **`ppm show`**: the package's `links:` and `files:` with their platforms and, when installed,
  each one's state; the manifest's errors. `--json` gains `links` and `files`.
- **`file.sh`**: `protect`, `add` and `claim` refuse a path inside a directory link ("inside
  `<owner>`'s directory link `~/.local/lib/anfs`"). Plan 03 builds the rest.
- `PPM_CORE_KEYS` gains `links` and `files`.
- **`bin/ppm`** sources `~/.local/lib/anfs/link.sh`, `manifest.sh` and `files.sh` with no fallback
  to the clone (the fallback is deleted; `lib/anfs` is a directory link, so a pulled library is
  there at once). The first time on this Mac is a by-hand step (below).
- **install.sh**: `ANFS_BOOTSTRAP_LIBS="link.sh manifest.sh files.sh"` (all three cases of
  `source_libs`; `bash.bats`' two checks follow). `install_configs` no longer writes
  `PPM_GROUP_ID`: a new `anfs.local.conf` gets only its header line. The comment "anfs.conf and
  system.list come from linking core/anfs" becomes "anfs.conf is linked from core/anfs, system.list
  copied". `stow_base` itself is unchanged (it calls `stow_package`).
- **`sources.sh`**: `system.list` is the user's file now. `anfs_gitsrc_env` defines
  `gitsrc_hook_system_writable` true when it is a regular file (so `anfs src ssh` rewrites its https
  entries), and the hint about `ppm file protect` goes. The "(system.list is anfs-managed)" wording
  of `src remove` becomes "(in system.list: edit it, or add the source to user.list to override
  it)". Open question 2. `sources.bats`' three cases follow.

### 6. wsm

- **Fallbacks**: the clone fallbacks for `link.sh` and `skill.sh` (lines 47–60) are deleted; wsm
  sources `manifest.sh` and `files.sh` too, and `~/.local/lib/ppm/platform.sh`.
- **`space.yml`'s `links:`** take the whole vocabulary. `{src, dest}` entries are as today but made
  relative (`anfs_link_one`), their state from `anfs_link_src_state`, removed with
  `anfs_unlink_src`; `_expand_vars` and `_norm` move into `manifest.sh`. A directory entry (`docs/`)
  is one link in the check (`anfs_link_check` takes it as a path), in install and remove. A path
  entry with platforms filters the space's own files. All through the space's `ANFS_MF_*`.
- **`files:`** in a space: copied when missing at install (after the files, before clones), never
  touched again, left by remove (so the space's directory stays when it holds one). `wsm ls`
  counts a `missing` or `in the way` copy into `partial`, not `differs`. `wsm ls <id> --json` adds
  `files: [{path, status}]`.
- **`-f`** moves a link's or a copy's obstacle aside with `_backup`, as today for files and links.
- **The leftover walk** never descends into a directory link (a bash glob `*/` follows a linked
  directory: test `-L` first).
- **`_add_one`**: `anfs_link_one` instead of `ln -s` with `anfs_relpath` (both places). A path
  inside a directory link is already "in the source" (`_context`); no change needed beyond a test.
- `wsm new`'s template and help: the vocabulary, `files:`; "absolute links" wording goes.
- No backward compatibility: an existing absolute `links:` link that resolves to its `src` reads
  `ok` and is not rewritten (decision 2); the person replaces it by hand.

### 7. ovm (stack `packages/obsidian`)

- Sources `~/.local/lib/anfs/link.sh`. `_link_make` uses `anfs_link_one "$LS_DEST" "$LS_PATH"`
  instead of `ln -s`. The registry keeps recording `LS_DEST` (absolute, the vault path);
  `_points_at` already accepts a link that resolves to it, so a relative link is `ok` and an old
  absolute one stays `ok` until replaced by hand.
- `tests/links.bats`: the link texts asserted become the relative ones (and one case where the
  vault is on another volume is absolute: a fixture under a path that shares only `/` with
  `$HOME` is not available in a test, so this case is covered in `link.bats` with
  `anfs_link_text`, not here).

### 8. The packages

core (with the code, same commits as the libraries they need):

- `anfs`: `links: [.local/lib/anfs/]`, `files: [.config/anfs/system.list]`. `package.yml`'s
  comment unchanged otherwise.
- `pcm`: `links: [.local/lib/pcm/]`; `pmm`: `links: [.local/lib/pmm/]`.

stack:

- `op`: `macos/.ssh/config.d/99-op.conf` and `linux/.config/mise/conf.d/op.toml` move to `home/`
  (`git mv`); `links:` gets the two path entries with `platforms: [macos]` and `[linux]`. The
  comment "Linux: the CLI comes from mise (linux/.config/mise)" and the README's and AGENTS.md's
  file tables follow.
- `tailscale`: `links: [.local/lib/tsm/]`; `rpi-imager`: `links: [.local/lib/rpi-imager/]`. Their
  library lookups (`~/.local/lib/<x>`, then `$here/../lib/<x>`) work through a directory link
  unchanged.
- `bash/install.sh`: the header's "instead of letting stow abort the install or `-f` delete it"
  becomes that the install would otherwise stop on it as a conflict, and `-f` would only move it
  to ppm's backup; the hook keeps the user's file beside it as `.pre-ppm`. The hook stays.
- `ssh/install.sh`: unchanged. Its `add_to_file ~/.ssh/config "Include config.d/*.conf"` is a
  line in the user's file, a third kind; a comment says it is a candidate for `setup:`.

user (the person's repository, its own branch in this job):

- `host`: `platforms: [macos]`; `macos/` moves to `home/` (`git mv`, no rename of the package).
- `ssh`: `macos/.ssh/authorized_keys` moves to `home/.ssh/authorized_keys`;
  `links: [{path: .ssh/authorized_keys, platforms: [macos]}]`.

After this no package in any source on this Mac has a `macos/`, `linux/`, `debian/` or `fedora/`
directory (anfs-dev, lgat, rws and the others have none today; checked 2026-10-21).

### 9. Documentation

- Core `AGENTS.md`: "package.yml" (the vocabulary, with the table and the refusals; `home/` is the
  only directory linked; the mise bullet: the linked tomls); "Declared Resources" (`links` and
  `files` are ppm's); "Linking" (`manifest.sh` and `files.sh`, directory links, the on-the-way
  rule, relative everywhere and the `/` exception, `anfs_link_text`, `anfs_link_src_state`; the
  fallback bullet goes); "Configuration: anfs.conf" (`anfs.local.conf` is no longer seeded with
  `PPM_GROUP_ID`); "wsm: spaces" (`links:` and `files:`, relative links); "Key Files"
  (`system.list` is a copy, yours once made); "Claiming and Protecting Files" (refusals inside a
  directory link); "Lib Structure" and the bootstrap set (the two new libraries); "Testing".
- READMEs: `packages/ppm/README.md`, `packages/wsm/README.md` (`space.yml` in full; "absolute
  links" goes), `packages/anfs/README.md` (`system.list`), stack `op`'s README and AGENTS.md,
  stack `obsidian`'s README if it says the links are absolute.
- `packages/pcm/legacy/README.md` mentions `$PPM_GROUP_ID`: one word, left (legacy) — say so in
  the log.

### 10. Commits, in order

1. `link.sh`: directory entries, the on-the-way rule, `anfs_link_text`, `anfs_link_one`'s guard,
   `anfs_link_src_state`/`anfs_unlink_src`; `link.bats`
2. `manifest.sh` and `files.sh`; `manifest.bats`, `files.bats`; `bash.bats`
3. ppm and install.sh on them; `PPM_GROUP_ID` gone; core's `anfs`, `pcm`, `pmm` converted;
   `sources.sh`; ppm's bats
4. wsm on them; wsm's bats
5. docs (core)
6. stack: ovm on `anfs_link_one`; its tests
7. stack: `op`, `tailscale`, `rpi-imager`, the bash comment, the ssh comment
8. user: `host`, `ssh`

The pre-commit hooks bump the package versions; `WSM_VERSION` is set by hand (minor).

## Test Spec

`packages/anfs/tests/manifest.bats` (new, temp dirs, yq):

- Each entry form gives the right lists: a directory string, a path for a matching and a
  non-matching platform (file, directory without `/`, directory with `/`), `src` with `~`,
  `$VAR`, `${VAR}`, a default `dest`; `files:` strings and maps; `linux` matching `debian` and
  `fedora`, `debian` not matching `fedora`.
- Every refusal in section 1, each with its message, and several at once all listed; exit 2.
- One read of three manifests gives three manifests' rows (one yq; a malformed one named, the
  others read).
- Parses and runs under `/bin/bash` 3.2.

`packages/anfs/tests/files.bats` (new):

- `missing` → copied with its mode; `present`; `differs` reported and left (bytes unchanged after
  apply); `in the way` for a directory, a link, a link on the way into the sources; a skip.
- Apply with one `in the way` copies nothing; two `in the way` both listed.
- A copy is never written over a file that appeared between plan and apply (the temp-and-move).
- Names (space, `+`, leading `-`); a tab refused. Bash 3.2.

`packages/anfs/tests/link.bats` (added cases):

- A directory entry: `link`, `keep`, `replace` (dangling), `conflict` (a real directory there);
  `anfs_unlink` removes it; `anfs_link_check` says `ok`/`missing`/`in the way` for it.
- A file whose path runs through a directory link into the sources: `conflict` (`inside a
  directory link`), and through a link that resolves inside `<to>` but not into the sources:
  followed, as before.
- `anfs_link_text`: relative for a shared ancestor below `/`; absolute when only `/` is shared
  (two canonical paths, no filesystem needed); the plan's text the same way.
- `anfs_link_one` refuses an existing directory, an existing link, a file; makes a relative link.
- `anfs_link_src_state`: each state; an absolute link to `src` is `ok`; `anfs_unlink_src` leaves a
  link the user repointed.

ppm (`packages/ppm/tests`; `backup.bats` and `plan.bats` extended, or a new `manifest.bats` there):

- **A directory entry**: one link at `~/.local/lib/x`, the tracker has the one path; a file added
  to the package's directory afterwards is at `~/.local/lib/x/new` with no reinstall; another
  package (or a higher layer) shipping `.local/lib/x/y` is a conflict, in either install order,
  and `-f` does not clear it; `ppm file protect ~/.local/lib/x/a` is refused; remove takes the
  one link; remove with no tracker takes it too.
- **Platform entries** (stub `platform`): a `macos` file linked on macOS and absent on Debian,
  `linux` on Debian and Fedora; a directory without `/` per file; `ppm install -n` shows `off` for
  the filtered ones. **mise**: a package with `home/.config/mise/conf.d/x.toml` under
  `platforms: [linux]` runs no `mise install x` on macOS (stub `mise`, assert its log) and does on
  Debian; a higher layer's toml replacing a lower one's: only the higher one's tools.
- **`src`/`dest`** in a `package.yml`: linked relative, in the tracker's `links`, `missing src`
  said and not made, removed on remove only while it points at `src` (if section 5 keeps it; else
  the refusal is tested).
- **`files:`**: copied once; edited, then reinstalled: left, `-n` says `differs`; deleted, then
  reinstalled: copied again; `in the way` (a directory) stops the install with the links' conflicts
  in the same block; `-f` moves it aside and copies; remove leaves the copy; a protected path is
  skipped.
- **Leftovers**: a package with a `macos/` directory is linked from `home/` only, no warning;
  `PPM_GROUP_ID=macos` in the environment changes nothing.
- **Validation**: an invalid `links:` stops that package's install with every error, others in the
  same run install.

wsm (`packages/wsm/tests/links.bats`, `install.bats`):

- A directory entry in a space: one link, `wsm ls` `ok`; a file added in the source's directory is
  there with no reinstall; the leftover walk does not enter it; a directory link containing a child
  space, or `.agents/` with `skills:`, is `invalid`.
- A path entry with `platforms: [linux]` on macOS (stub): the file is not linked, and not counted
  as missing.
- `files:`: copied, then left; `differs` does not make the space `partial`; `missing` does; remove
  leaves it.
- `{src}` links are relative; an absolute one to `src` made by hand reads `ok`; remove takes a
  relative one.
- `wsm add` makes a relative link through `anfs_link_one`.
- The existing 96 wsm tests pass.

ovm (stack `packages/obsidian/tests/links.bats`): `link add` makes a relative link that resolves
to the vault directory; an existing absolute one is `ok`; `link remove` takes either.

`bash.bats`: the bootstrap set includes `manifest.sh` and `files.sh` and parses under 3.2.

## Verification

Every command runs in the job: `anfs work run <job> core -- <cmd> </dev/null` (stack's suites:
`anfs work run <job> stack -- bats packages/obsidian/tests </dev/null`). Nothing that boots a VM
or a box runs; nothing is installed on this host.

- [ ] While working, only the file touched: `bats packages/anfs/tests/link.bats`, `manifest.bats`,
      `files.bats`, the ppm file, `packages/wsm/tests/links.bats`, ovm's `links.bats`
- [ ] Once, at the end: `packages/anfs/tests/all` (pmm's and pcm's suites stay off; the log says
      they were not run, though `pcm` and `pmm` change only in `package.yml`)
- [ ] On an isolated copy of this Mac's `$HOME` state, built as plan 01's log describes (sources
      without `.git`, core, stack and user from the job's worktrees, `~/.config/anfs`, ppm's
      trackers, `cp -P` of what the real `$HOME` has at each path the installed packages ship;
      `env -i HOME=<copy> PATH=...`, no `*_HOME` reaching a real directory):
  - `ppm install -n core/anfs` plans `conflict ~/.local/lib/anfs (a directory)` (the per-file
    links of today; the by-hand step below) and `present` or `differs` for `system.list`
    (protected here: `skip`)
  - `ppm install -n stack/op`: `link` for nothing new, `replace` for
    `~/.ssh/config.d/99-op.conf` (its link points into `macos/`, which moved), `off` for
    `op.toml`; no `mise install` planned
  - `ppm install -n user/ssh`: `replace` for `~/.ssh/authorized_keys` (into `macos/` today),
    `keep` for `99-defaults.conf`. Its tracker also lists `10-local.conf` and `90-mac.conf`,
    dangling links into `macos/` since those files left the repository (user `28d29a2`); a real
    install's `meta_cleanup_stale` removes them, `-n` does not show them. Note it in the log.
    `user/host`, `stack/tailscale` and `stack/rpi-imager` are not installed here: `-n` plans
    `link` for all of their files (the directory as one entry for the two libraries)
  - every other installed package plans only `keep` (as plan 01's log: 40 layers), and the copy is
    unchanged afterwards (tree listing with link texts and mtimes, before and after)
  - `wsm ls` on a copy of the 7 sources' `spaces/` and their links: unchanged rows; `anfs-dev`'s
    `sources` link `ok` (absolute, left)
- [ ] `grep -rn PPM_GROUP_ID` in core and stack finds only `pcm/legacy/README.md`
- Deferred to the end of the roadmap (in the log, not run): `roundtrip debian` (op's `op.toml`
  linked and `mise install op` run, its macOS file absent; `system.list` a regular file),
  `roundtrip-macos`; a real `install.sh` (the download path of `source_libs` fetching the two new
  libraries works only once they are on main)

## After merge, on this Mac (by hand, for the person)

ppm sources `manifest.sh` from `~/.local/lib/anfs`, which is not linked yet, and the fallback is
gone, so the library directory comes first:

1. `find ~/.local/lib/anfs ! -type l ! -type d` prints nothing (only links there), then
   `rm -r ~/.local/lib/anfs` and
   `ln -s ../share/anfs/sources/core/packages/anfs/home/.local/lib/anfs ~/.local/lib/anfs`.
2. `ppm install -n core/anfs core/pcm core/pmm`, then `ppm install -f` of each: `-f` moves the
   per-file directories of `pcm` and `pmm` (links only) to ppm's backup; delete those backups
   afterwards (`~/.local/state/ppm/backup/files/.local/lib/{pcm,pmm}`, and their manifest lines),
   or `rm -r ~/.local/lib/{pcm,pmm}` first instead.
3. `ppm install stack/op` (after `-n`; no `-f` needed, its old link is a `replace`), and
   `ppm install user/ssh` once the user branch is merged (it also drops the two dangling
   `config.d` links its tracker still lists). `stack/tailscale`, `stack/rpi-imager` and
   `user/host` are not installed here: nothing to do.
4. Remove `PPM_GROUP_ID=macos` from `~/.config/anfs/anfs.local.conf`.
5. Remove `.config/anfs/system.list` from `~/.local/state/ppm/installed/protected.yml` (it is
   there): the file stays as it is, now a copy ppm leaves alone.
6. Replace the absolute links: `rm ~/spaces/anfs-dev/sources && wsm install anfs-dev`;
   `rm ~/spaces/anfs-dev/apex && ovm link apply` in its vault. `find ~/spaces -maxdepth 4 -type l
   -lname '/*'` lists any others.

## Not in this plan

- anfs-dev: `move.sh` still loops over `${PPM_GROUP_ID:-}` (broken since plan 01; plan 03's).
  A package.yml check in anfs-dev's pre-commit hook (refuse an invalid manifest at commit) would
  be cheap; a follow-up.
- `-n` beyond `ppm install -n` and `wsm install -n` (plan 04).
- The `file` group, `anfs owner|mv|status` (plan 03). `anfs_link_owner` of a path *inside* a
  directory link (not itself a link) is plan 03's question.
- `setup:` for "a line in your file" (`~/.ssh/config`'s Include).

## Open questions (the plan proceeds on the default given)

1. **`platform()` for wsm.** Default: wsm sources ppm's `platform.sh`. Cleaner: `platform()` and
   `os()` move into `lib/anfs` (they are in ppm's bootstrap set already), which touches every
   caller of ppm's copy. Say if that move belongs here.
2. **`system.list` once it is the user's.** Default: `anfs src ssh` may rewrite it; `src remove`
   still edits `user.list` only, with new wording. Also: a default source added to core's
   `system.list` later never reaches an existing machine (the copy is not updated; `ppm install -n
   core/anfs` and `ppm show` say `differs`). Accepted, or should `anfs src` say so?
3. **`src:` in `package.yml`** — kept only if it falls out of the shared code (about 40 lines in
   ppm: plan, tracker key, remove). Otherwise refused by validation and noted in the log.
4. **Space-level `platforms:`** (the whole space, as a package's) is not in this plan, though the
   vocabulary is meant to be one. Add it here, or later?


## Update 2026-10-21 (after review)

- `src:` entries are **allowed** in `package.yml` (not refused): `ovm/03-vaults-home` needs
  `links: [{src: ~/Library/.../apex, dest: vaults/apex, platforms: [macos]}]` in a user layer of
  `obsidian`. Relative, as every link; `missing src` is a state, not an error.
- **Per-tool opt-in (2026-10-21).** `manifest.sh` knows every entry kind (per-file `path`,
  directory `dir/`, `src:`/`dest:`, `platforms:`, `files:`); each tool declares the kinds it accepts,
  as each declares its command groups, and validation refuses the rest with a message. ppm accepts
  all of them, `src:` included (decided: option (a), for `ovm/03`); wsm accepts all of them.
- **`platform()` and `os()` move to `lib/anfs/platform.sh`** (from ppm's `platform.sh` and
  `core.sh`), unchanged in behaviour; ppm and wsm source them from there. Merging the two into one
  function is deferred to the end of the roadmap (`ROADMAP.md`, "Last").
- **`system.list` stays a link.** It was protected on this Mac only so that `anfs src ssh` would
  not edit core's checkout, and `src ssh` already switches a system source's clone `origin` to ssh
  without touching the list (the list's URL is used only for the first clone). Do not add system
  sources to `user.list` to override their URL: the kept entry takes the user list's position, which
  reorders layer priority. In this plan: the "protect it" hint and `gitsrc_hook_system_writable`
  go; `src ssh`'s note for a system-list https entry says the origin was switched and the list's URL
  is only used for the first clone. `files:` ships with its tests only (no package uses it yet).
  By hand after merge: `ppm file unprotect ~/.config/anfs/system.list`, delete the copy,
  `ppm install core/anfs`, `anfs src list` (core and stack origins still ssh).
