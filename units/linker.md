---
objective: The toolkit links files itself — one library that ppm and wsm share, built for layers, platforms and trackers, with GNU stow gone from the bootstrap
status: pending
---

ppm and wsm put files in place with GNU stow. They use five of its options (`-d`, `-t`, `-D`,
`--no-folding`, `--ignore`) and a good part of both tools is code that works around the rest.
This unit replaces stow with `lib/anfs/link.sh`, then uses what that makes possible: files per
platform, directories linked whole and files copied instead of linked, declared the same way in
`package.yml` and `space.yml` (plan 02), and one answer to "who owns this file, and how do I move
it" (plan 03).

Depends on: nothing. It touches ppm, wsm, `install.sh` and `ppm move` (anfs-dev's `dev` package;
plan 05 moves it into core), none of which the `machine-images` branch changes, so it can run as its own job beside it.

## 1. Where stow is used today, and what improves

Every call, from the code (`main`, refreshed 2026-10-21). 47 packages and 228 tracked files on
this Mac.

| # | Where | What stow does there | What is built around it today | With our own linker |
| --- | --- | --- | --- | --- |
| 1 | ppm install: `stow_subdir`, `installer.sh:563` | Links `home/` and the `$PPM_GROUP_ID` directory (`macos`, `linux`) into `$HOME`, file by file | Layering is a growing list of `--ignore=^path$` regexes, one per file every higher layer linked (`PPM_IGNORE_ARGS`). Only `.` is escaped (`_stow_ignore_arg`), so a file name with `+`, `(` or `[` is a wrong regex | A set of exact paths to skip. No regex, no argument list that grows with the install. **Plan 02:** `home/` is the only directory linked; what is linked how comes from `lib/anfs/manifest.sh` (`links:` for a directory as one link or a path per platform, `files:` for copies, `platforms:`); `PPM_GROUP_ID` and the `macos/`/`linux/` directories go |
| 2 | Before every one of those: `remove_dangling_links`, `force_remove_conflicts` (`installer.sh:560–561`), `meta_cleanup_stale` (after, `installer.sh:123`) | Nothing: stow aborts the whole operation at the first conflict, and calls a dangling link into another package "not owned by stow" | Three pre-passes over the package's files that clear what would make stow abort | One plan pass that classifies every target (ours and current, ours and stale, another layer's, the user's) and reports **all** conflicts at once; `-f` is a rule in that pass |
| 3 | ppm remove: `unstow_package`, `installer.sh:520` | `stow -D`: scans the target for links into the package | The tracker already lists the files | Unlink the tracker's list, checking each still points into the package; walk the package tree as the fallback when a tracker is missing. **Plan 02:** the fallback is the manifest's list (a directory link is one entry; walking the tree would miss it); copies (`files:`) are left |
| 4 | Bootstrap: `stow_base`, `install.sh:347` (ppm's `stow_package`) | Links `core/anfs` and `core/ppm` before ppm exists | Homebrew must install `stow` (and its Perl) first (`base_formulas`, `install.sh:176`; checks at `:324` and `:499`); a 20-line pre-pass (`:325–345`) clears links that would abort it | No `stow` formula in the bootstrap. The library is part of the bash 3.2 set. **Plan 02:** `manifest.sh` and `files.sh` join it (core/anfs's `lib/anfs` is a directory link, and `system.list` a copy, from the first install) |
| 5 | `ppm file add`, `claim`: `_file_claim`, `file.sh:200`; `reset`: `_file_reset`, `file.sh:259` | Re-stows a whole package to link one file; `reset` rebuilds an ignore list of every other file first | — | Link exactly that file |
| 6 | `ppm move`: `_move_restow`, anfs-dev `packages/dev/home/.local/lib/ppm/move.sh:212` | Re-stows from the new source | The complement of the tracker as an ignore list, so a higher layer's files are not taken; it also calls core's `unstow_package` | Link the tracker's list. Handled in plan 05 (`move` in core, with the package's history), not plan 01 |
| 7 | wsm install: `_link_unit`, `wsm:563` | Links one level of `spaces/` into the tree | stow runs with `HOME` pointed at a fake directory, because its built-in ignore list drops `README.*`, `LICENSE.*` and `.gitignore`; `--ignore` per sub-level directory | wsm already computes the level's files (`_unit_files`); it links that list. No fake HOME, no built-in list. **Plan 02:** a space's `links:` and `files:` read through `manifest.sh`, as a package's; its `links: {src}` entries (absolute today) are made relative by `anfs_link_one` |
| 8 | wsm remove: `_unlink_unit`, `wsm:441` | `stow -D`, output discarded | Then its own loop removes each recorded link anyway | The loop is the whole of it |
| 9 | `wsm add`: `_add_one`, `wsm:1521`, `1536` | Nothing: wsm writes the link itself, "as stow would" | Its own `_relpath`, because stow treats an absolute link as foreign (`wsm:545` still cleans up absolute links from wsm 0.3) | The same function the installs use. **Plan 02:** `anfs_link_one` itself, not `ln -s` with `anfs_relpath` |

Not calls, but there because of stow: `ppm implode` names it as kept (`implode.sh:35`), `anfs
implode --all` removes it as a recorded bootstrap formula, ppm's `backup.bats` and anfs-dev's
`work.bats` skip without it, and anfs-dev's round trip bakes it into its bootstrap snapshot
(`roundtrip:68`).

Not stow either, but links the toolkit makes with its own `ln -s`, absolute (plan 02 moves both
onto `anfs_link_one`, relative): wsm's `links:` (`_space_links`, `wsm:~1200`) and ovm's vault
links (`_link_make`, stack `packages/obsidian/home/.local/bin/ovm:~841`).

Also fixed, from core `AGENTS.md`'s own notes ("wsm: workspaces"): a `README.md` under a package's `home/` is not stowed
today (stow's built-in list), and ppm and wsm disagree about that list.

## 2. What we lose

- **A second implementation that many people have run for years.** Stow's handling of odd targets
  (a directory in the path that is itself a link, a stow directory inside the target, names with
  spaces or newlines) becomes ours to test. Plan 01's test spec lists them.
- **Unstow with no state.** `stow -D` needs nothing but the package directory. Kept: unlinking
  walks the package tree when there is no tracker.
- **Features we never used**: tree folding (always switched off), `--adopt`, `--dotfiles`,
  `--defer`, `--override`, `.stow-local-ignore` (no package in any source on this machine has
  one; checked).
- **Running stow by hand on a package.** Kept in practice: the links stay per-file and relative,
  exactly what `stow --no-folding` writes, so an existing install needs no migration and
  `stow -D` still undoes one. A test holds that while stow is still installed anywhere.
- **Nothing else.** No other tool on the system depends on our use of stow.

What it costs: about 200 lines of bash 3.2 and their tests, against roughly that many lines of
workaround removed from ppm, wsm and `install.sh`.

## 3. Moving files

Today there are five commands, by tool and by case: `ppm file add`, `claim`, `reset`, `protect`,
`wsm add`, plus `ppm move` for a whole package. Which one applies depends on who owns the file,
which the user has to know.

**3.1 One front door: `anfs mv`, and `anfs owner`.** Not `ppm mv` and `wsm mv` as well.

- The owner of a path can be read off the path: a link into `sources/<s>/packages/<p>/` is
  ppm's, into `sources/<s>/spaces/...` is wsm's, anything else is the user's. The library
  answers that (`anfs_link_owner`), so a command that spans tools can dispatch. Commands that
  span the tools are anfs's.
- `anfs owner <path>...` prints who owns each: tool, source, package or space, the file in the
  source.
- `anfs mv <from> <to>` does what the two paths imply:

  | From | To | What happens |
  | --- | --- | --- |
  | owned | another path under `$HOME` | The file moves inside the same package (or space) and is linked at the new place; the tracker follows |
  | not owned | a path, with `--to <source/package>` or inside an installed space | Adopted: today's `ppm file add` and `wsm add` |
  | owned | `--to <source/package>` | Today's `ppm file claim` |

- `ppm file ...` and `wsm add` stay as the tool's own words for its own kind, calling the same
  functions. `ppm move` (a package between sources, today in the `dev` package) becomes core's
  `<tool> move` (plan 05: ppm and wsm, on `lib/anfs/move.sh`); `anfs mv <source/name> --to
  <source>` only runs the owning tool's `move` (plan 03).

**3.2 Wrapping `mv` in the shell: an opt-in second step, not the mechanism.**

What goes wrong today with a plain `mv` of a linked file: the link is relative, so moving it to
a directory at another depth breaks it; the tracker still names the old path; the next install
puts the old link back. An editor that saves by rename does something similar: it replaces the
link with a real file. So the problem is real, and wider than `mv`.

A wrapper alone does not solve it:

- A shell function exists only in interactive shells. Scripts, editors, Finder and an agent's
  non-interactive shell all call the real `mv`, so the same command would do two different
  things depending on where it runs.
- `mv` then edits a git repository (the source) without saying so, and a source with
  uncommitted changes is skipped by auto-update.
- `mv`'s own surface is large (`-i`, `-n`, `-t`, several sources, directories). `rm` is worse.

So:

1. **Detect and repair, wherever the change came from.** `anfs status` lists drift: an owned
   link that moved, one replaced by a real file, a dangling one, a tracked file that is gone.
   Each line names the fix (`anfs mv`, `anfs adopt`, reinstall). This covers editors and scripts.
2. **The wrapper is a thin, opt-in layer over `anfs mv`** (`ANFS_SHELL_MV=true` in `anfs.conf`):
   a function in `sh/anfs.sh` that hands over only when an argument is an owned link (one
   `readlink`, no process), says in one line what it did in the source, and otherwise runs
   `command mv "$@"` untouched. Never `rm`.

## Decisions

- **The library is `lib/anfs/link.sh`, in the bash 3.2 bootstrap set.** `install.sh` and ppm's
  `installer.sh` use it before Homebrew's bash is the one running.
- **Links are per-file and relative**, as `stow --no-folding` writes them. No migration. A
  package says when a directory is one link instead (`links:`, plan 02).
- **The library knows nothing about packages or spaces.** It links a list of files from one
  directory into another, skipping a set, and answers who a link points into. Layers, trackers,
  backups and platforms stay in the tools (plan 02: reading `links:`, `files:` and `platforms:` is
  `lib/anfs/manifest.sh`'s, shared by ppm and wsm).
- **A plan, then the changes.** Nothing is touched until every target is classified; a conflict
  stops the run with all of them listed.
- **Per-platform files are declared in `package.yml`** (plan 02); the `macos/` and `linux/`
  directories and `PPM_GROUP_ID` go.

### Decided with the person, 2026-10-21 (the review of plan 02)

1. **One vocabulary in `package.yml` and `space.yml`.** The default is unchanged: a package's
   `home/` (a space's own files) is linked file by file, everywhere. `home/` is the only directory
   of a package that is linked: `macos/`, `linux/` and `PPM_GROUP_ID` go, and a leftover platform
   directory gets no warning (it is just never linked).
   - **`links:`** changes how things are linked. An entry is a string or a map:
     - `.local/lib/anfs/` (a trailing `/`): the directory as one link (this replaces the earlier
       `link_dirs:` idea);
     - `{path: <in the tree>, platforms: [macos]}`: only on those platforms;
     - `{src: <outside the tree>, dest: <optional>}`: space.yml's existing `links:`, now also in
       `package.yml` where it falls out of the shared reader.
   - **`files:`** copies instead of linking: copied when missing, never overwritten, the user's
     afterwards; remove leaves it. Entries take `platforms:` too. First use:
     `~/.config/anfs/system.list` (a real file today only because ppm's protected list holds it).
   - **`platforms:`** means the same at every level: a package's (exists: filters the whole
     package), a `links:` or `files:` entry's (new). user/host becomes `platforms: [macos]` with
     its files in `home/` (no rename); user/ssh's `authorized_keys` and stack/op's two files become
     `links:` entries with platforms.
   - A line in the user's file (`~/.ssh/config`'s `Include`, stack/ssh's `add_to_file`) is a third
     kind; it stays in `install.sh`. A candidate for `setup:` later.
2. **Relative links everywhere**, through `link.sh`, computed on canonical paths. The one
   exception: a link whose only common ancestor with its target is `/` (`/Volumes/x`) is absolute.
   wsm's `links:` and ovm's `ln -s` (both absolute today) move onto `anfs_link_one`; ovm sources
   `link.sh`. The absolute links on this Mac are replaced by hand after merge (no migration code).
3. **Three core libraries, by domain.** `link.sh`: symlinks only, no yq, bash 3.2 bootstrap set
   (its role unchanged). **`files.sh`** (new): real files in a target; for now copy-if-missing with
   the states `missing`, `present`, `differs` (information only), `in the way`; built to hold plan
   03's file operations. **`manifest.sh`** (new, yq): reads `links:`, `files:` and `platforms:`
   from a `package.yml` or `space.yml` into one plan input (per-file links, directory links,
   external sources, copies, without what this platform filters out). ppm and wsm both read
   through it, plan over link and copy targets together (every conflict listed, nothing touched on
   one), then apply.
4. **Gaps closed in the same plan.** ppm's `mise install` reads the package's resolved link list
   (a Linux-only `op.toml` is never installed on macOS; no `install_linux` hacks). A platform entry
   inside a directory link is refused. ppm's and wsm's fallbacks to the core clone's `link.sh` go
   (`lib/anfs` is a directory link). Directory links: core `anfs` (`.local/lib/anfs`), `pcm`
   (`.local/lib/pcm`), `pmm` (`.local/lib/pmm`), stack `tailscale` (`.local/lib/tsm`), stack
   `rpi-imager` (`.local/lib/rpi-imager`), each with one owner (checked 2026-10-21);
   `.local/lib/ppm` stays per file (core/ppm, anfs-dev/dev and stack/ruby write there).
   `PPM_CORE_KEYS` gains `links` and `files`. stack/bash's "letting stow abort" comment is
   reworded. No backward compatibility: the person's by-hand steps after merge are in plan 02.
5. **Plan 03 becomes "file operations, shared"**: a `file` command group built like config's
   (`lib/anfs/files.sh` plus a `cli.sh`-declared group), with `anfs owner|mv|status` over it.
   Its notes are at the top of plan 03; the rest of that plan is refreshed after they are settled.

### Decided with the person, 2026-10-22 (move)

1. **`ppm move` leaves anfs-dev.** `packages/dev/home/.local/lib/ppm/move.sh` is deleted with its
   declaration and its mentions (anfs-dev's `AGENTS.md`, `packages/dev/README.md`); its `stow` calls
   and `PPM_GROUP_ID` loop, the last in anfs-dev's code, go with it. Moving is core's (plan 05).
2. **The tool orchestrates; the library supplies building blocks**, with no hooks or callbacks. A
   tool that has a move writes its own `cmd_move`, which reads top to bottom with its own functions
   (ppm: check, unlink, tree, tracker, relink; pcm, later: check, stop if running, tree, start again
   if it was, its own rollback). A tool without a `cmd_move` has no move.
3. **`lib/anfs/move.sh`**:
   - `anfs_move_check <dir> <source/name> <to>`: both repositories clean (`-f` overrides, as
     today), the path free in the destination, the destination a source, and the dependents rule
     over `resolve.sh` (moving *up* in priority hides the resource from lower sources' dependents);
     the same for packages, spaces and containers.
   - `anfs_move_tree <dir> <source/name> <to>`: the directory moves **with its history** in one
     step: `git -C <from> format-patch --root -o <tmp> -- <dir>/<name>`, then `git -C <to> am
     --committer-date-is-author-date` (the same relative path in both), then `git rm -r` and one
     commit in the origin naming the destination. When the replay fails (merged parallel branches,
     a message the mailbox format cannot carry, a conflict): `am` aborted cleanly, the files copied,
     one commit in the destination naming the origin repository and commit, and the method said.
   - `anfs_move_cli`: one helper, so every tool declares `move <source/name> <to-source>` with the
     same help, `-f` and `-n`. `-n` prints the checks, the commits that would be replayed, and the
     tool's own steps (each tool prints its own).
   - Safe: never pushes; refuses when the destination has the path (`-f` or not); leaves both
     repositories clean, or as they were on failure.
   - **A move commits, by design** (history is commits). This reverses plan 03's "it never
     commits" for `move` only; `anfs mv` of a file still does not commit.
4. **In plan 05: ppm and wsm.** ppm keeps today's semantics (unlink without remove hooks, the tree,
   relink without install hooks, the tracker and the claims renamed). wsm moves a space's installed
   directory too (`~/spaces/<from>/<path>` to `~/spaces/<to>/<path>`, the clones inside untouched,
   relinked); ids that name the source (`depends:`) are not rewritten. pcm, pmm and psm only if
   small: found not to fit this plan (their suites are off; they resolve through `catalog.sh`), so
   they are a follow-on plan with what was found recorded in plan 05.
5. **`anfs mv <source/name> --to <source>`** (plan 03) only runs the owning tool's `move`. Plan 05
   does not depend on plan 03.
6. No backward compatibility, no migration code.

## Plans

1. `01-link-library` — `lib/anfs/link.sh`; ppm, wsm and `install.sh` on it; stow out of the
   bootstrap
2. `02-links-and-files` — `lib/anfs/manifest.sh` and `files.sh`; `links:`, `files:` and
   `platforms:` in `package.yml` and `space.yml` (directory links, per-platform paths, copies);
   relative links everywhere (wsm's `links:`, ovm); the packages with platform directories or a
   library directory converted; `PPM_GROUP_ID` removed
3. `03-owned-files` — file operations, shared: the `file` group (`ppm file`, `wsm file`) on
   `lib/anfs/files.sh`, `anfs owner`, `anfs mv`, `anfs status`; the opt-in shell `mv`. Notes from
   the 2026-10-21 review at its top, to settle before it is refreshed
4. `04-dry-run` — `-n` on every command that installs, adds, removes or purges, across the
   tools (`ppm install -n` itself is plan 01). Plan to write
5. `05-move` — `lib/anfs/move.sh` (`anfs_move_check`, `anfs_move_tree`, `anfs_move_cli`): a
   resource moves to another source with its history (format-patch/am, a one-commit copy when the
   replay fails); `ppm move` and `wsm move` on it; `ppm move` leaves anfs-dev. Independent of 03 and
   04. Plan written 2026-10-22
6. `06-move-containers-machines-skills` — `pcm move`, `pmm move` (machines), psm: after pmm's and
   pcm's suites are back on. Plan to write (plan 05's "Not in this plan" has what was found)

## Completion Criteria

- `grep -rn 'stow ' packages install.sh` in core finds only prose; `install.sh` does not install
  stow, and a round trip on a box with no stow passes.
- An install made by the old ppm is taken over by the new one with no link rewritten.
- A conflict report names every file in the way, not the first.
- `README.md` under a package's `home/` is linked.
- No package has a `macos/` or `linux/` directory; `anfs.local.conf` has no `PPM_GROUP_ID`.
- `~/.local/lib/anfs` is one link, and a file added to it upstream is there after a pull with no
  reinstall. Every link ppm, wsm and ovm make is relative (but one whose only common ancestor with
  its target is `/`).
- `~/.config/anfs/system.list` is a copy the user owns; reinstalling `core/anfs` never touches it.
- `anfs owner ~/.zshrc` names its package; `anfs mv` of a linked file leaves the source, the
  link and the tracker agreeing, and `anfs status` is then empty.
