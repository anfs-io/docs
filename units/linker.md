---
objective: The toolkit links files itself — one library that ppm and wsm share, built for layers, platforms and trackers, with GNU stow gone from the bootstrap
status: pending
---

ppm and wsm put files in place with GNU stow. They use five of its options (`-d`, `-t`, `-D`,
`--no-folding`, `--ignore`) and a good part of both tools is code that works around the rest.
This unit replaces stow with `lib/anfs/link.sh`, then uses what that makes possible: files per
platform declared in `package.yml`, and one answer to "who owns this file, and how do I move it".

Depends on: nothing. It touches ppm, wsm, `install.sh` and `ppm move` (in the `dev` package), none
of which the `machine-images` branch changes, so it can run as its own job beside it.

## 1. Where stow is used today, and what improves

Every call, from the code (`main`, 2026-10-13). 47 packages and 224 links on this Mac.

| # | Where | What stow does there | What is built around it today | With our own linker |
| --- | --- | --- | --- | --- |
| 1 | ppm install: `stow_subdir`, `installer.sh:561` | Links `home/` and the `$PPM_GROUP_ID` directory (`macos`, `linux`) into `$HOME`, file by file | Layering is a growing list of `--ignore=^path$` regexes, one per file every higher layer linked (`PPM_IGNORE_ARGS`). Only `.` is escaped (`_stow_ignore_arg`), so a file name with `+`, `(` or `[` is a wrong regex | A set of exact paths to skip. No regex, no argument list that grows with the install |
| 2 | Before every one of those: `remove_dangling_links`, `force_remove_conflicts`, `meta_cleanup_stale` | Nothing: stow aborts the whole operation at the first conflict, and calls a dangling link into another package "not owned by stow" | Three pre-passes over the package's files that clear what would make stow abort | One plan pass that classifies every target (ours and current, ours and stale, another layer's, the user's) and reports **all** conflicts at once; `-f` is a rule in that pass |
| 3 | ppm remove: `unstow_package`, `installer.sh:518` | `stow -D`: scans the target for links into the package | The tracker already lists the files | Unlink the tracker's list, checking each still points into the package; walk the package tree as the fallback when a tracker is missing |
| 4 | Bootstrap: `install.sh:346` | Links `core/anfs` and `core/ppm` before ppm exists | Homebrew must install `stow` (and its Perl) first; a 20-line pre-pass clears links that would abort it | No `stow` formula in the bootstrap. The library is part of the bash 3.2 set |
| 5 | `ppm file add`, `claim`: `file.sh:200`; `reset`: `file.sh:259` | Re-stows a whole package to link one file; `reset` rebuilds an ignore list of every other file first | — | Link exactly that file |
| 6 | `ppm move`: `move.sh:212` | Re-stows from the new source | The complement of the tracker as an ignore list, so a higher layer's files are not taken | Link the tracker's list |
| 7 | wsm install: `wsm:558` | Links one level of `spaces/` into the tree | stow runs with `HOME` pointed at a fake directory, because its built-in ignore list drops `README.*`, `LICENSE.*` and `.gitignore`; `--ignore` per sub-level directory | wsm already computes the level's files (`_unit_files`); it links that list. No fake HOME, no built-in list |
| 8 | wsm remove: `wsm:436` | `stow -D`, output discarded | Then its own loop removes each recorded link anyway | The loop is the whole of it |
| 9 | `wsm add`: `wsm:1454`, `1469` | Nothing: wsm writes the link itself, "as stow would" | Its own `_relpath`, because stow treats an absolute link as foreign (`wsm:540` still cleans up absolute links from wsm 0.3) | The same function the installs use |

Also fixed, from CLAUDE.md's own notes: a `README.md` under a package's `home/` is not stowed
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
  functions. `ppm move` (a package between sources, today in the `dev` package) goes the same way:
  `anfs mv <source/package> --to <source>`, with the work in ppm (plan 03).

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
- **Links are per-file and relative**, as `stow --no-folding` writes them. No migration.
- **The library knows nothing about packages or spaces.** It links a list of files from one
  directory into another, skipping a set, and answers who a link points into. Layers, trackers,
  backups and platforms stay in the tools.
- **A plan, then the changes.** Nothing is touched until every target is classified; a conflict
  stops the run with all of them listed.
- **Per-platform files are declared in `package.yml`** (plan 02); the `macos/` and `linux/`
  directories and `PPM_GROUP_ID` go.

## Plans

1. `01-link-library` — `lib/anfs/link.sh`; ppm, wsm, `install.sh` and `ppm move` on it; stow out
   of the bootstrap
2. `02-platform-files` — `stow:` in `package.yml`; the four packages that have a `macos/` or
   `linux/` directory converted; `PPM_GROUP_ID` removed
3. `03-owned-files` — `anfs owner`, `anfs mv`, `anfs status`; the opt-in shell `mv`

## Completion Criteria

- `grep -rn 'stow ' packages install.sh` in core finds only prose; `install.sh` does not install
  stow, and a round trip on a box with no stow passes.
- An install made by the old ppm is taken over by the new one with no link rewritten.
- A conflict report names every file in the way, not the first.
- `README.md` under a package's `home/` is linked.
- No package has a `macos/` or `linux/` directory; `anfs.local.conf` has no `PPM_GROUP_ID`.
- `anfs owner ~/.zshrc` names its package; `anfs mv` of a linked file leaves the source, the
  link and the tracker agreeing, and `anfs status` is then empty.
