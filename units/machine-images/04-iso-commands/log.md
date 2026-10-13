---
status: complete
started_at: "2026-10-13T08:50:00-04:00"
completed_at: "2026-10-13T09:50:00-04:00"
deviations: Core gets isos/isos.yml now (plan 02 was to add it) so image ISOs are catalogue entries. iso remove asks (-y) for a file with no url. iso prune refuses an unreadable or empty catalogue. A new list state, invalid. ventoy copy's stick is -t/--target, no longer an argument. The completion bound is tested in iso.bats, not registry.bats. Full list below.
summary: One iso group in lib/anfs/iso.sh, adopted by anfs, pmm and ventoy, on branch machine-images of core and stack (not pushed, not installed). bats anfs 142, pmm 80, ppm 28, pcm 67, stack ventoy 18, all passing; the real Debian e2e passes on macOS/hvf; the plan's Verification walked under an isolated HOME. Not verified - a download from a publisher, the real store, a real Ventoy stick, Linux, the round trips.
---

# Execution Log

`started_at` is the end of plan 01's verification, not a recorded start. The verification pass
("Verification" below) changed no code and made no commit.

## What Was Done

- **The group** (`lib/anfs/iso.sh`, `anfs_iso_cli`): `iso list [--all]`, `show`, `pull` (alias
  `get`), `add <url|file>`, `remove [--entry] [-y]`, `path`, `prune [-y]`, with `complete_iso`.
  `anfs`, `pmm` and `ventoy` each adopt it with one call. Core commits 766c9d8 and 616bc88; stack
  f174c6d and 4ffc4a3.
- **What the tools gave up**: `ventoy isos` and `ventoy get` (64 now); `pmm image prune --isos`
  and `iso_names`.
- **Core's `isos/isos.yml`**: `debian-13-{arm64,amd64}` and `fedora-44-{arm64,amd64}`, the same
  urls and sums as the images' `image.yml`. `packages/pmm/tests/iso.bats` holds the two files
  together until plan 02 makes an image name the id.
- **Docs**: CLAUDE.md (Lib Structure, the key table, the pmm Media bullet, Repository Layout),
  `packages/anfs/README.md`, ventoy's README.

## After Review

A review found these and they are fixed in 616bc88, each with a test:

- **`iso prune -y` emptied the store when a catalogue could not be read.** yq's failure was
  discarded, so an `isos.yml` with a typo (or shaped as a map) named nothing and every file was a
  stray. `prune` now refuses while any catalogue is not a list of entries with an id
  (`anfs_iso_bad_catalogs`), and when there are no entries at all. `iso list` names the file and
  exits 1, and leaves `unlisted` rows out of `--all` while that is so.
- **Image ISOs were strays.** An image names its ISO by URL, so `prune` offered it and
  `pmm iso pull` could not fetch it. Core's catalogue (above) names them.
- **`iso remove` followed `filename: ../..` out of the store.** A file name with a `/`, or none,
  is refused by `anfs_iso_file` and `anfs_iso_fetch`; `list` shows the entry as `invalid`.
- `iso remove` asks before deleting a file whose entry has no url.
- `iso prune --json` answers `{"deleted": N}` (the listing goes to stderr); `size` is bytes in JSON.
- Completion reads an indented list and drops a trailing comment.

## Commits

Core, `/Users/roberto/anfs-dev/machine-images` (12 files, +882 -52 since plan 01's eda9787):

```
616bc88 iso: prune refuses an unreadable or empty catalogue; a filename is a name in the store
766c9d8 iso: one command group, written once in lib/anfs/iso.sh, in anfs and pmm
```

Stack, `/Users/roberto/anfs-dev/machine-images-stack` (7 files, +238 -123 since 7d6e8a7):

```
4ffc4a3 ventoy: README says when iso remove asks and when iso prune refuses
f174c6d ventoy: the iso group is anfs's; isos and get are gone, copy <iso> pulls and selects
```

Neither is pushed.

## Verification

Run on the branch as committed, macOS 26 / Apple silicon, every bats run with `</dev/null`.

| Suite | Result |
| --- | --- |
| `bats packages/anfs/tests` | 142 of 142 |
| `bats packages/pmm/tests` | 80 of 80 |
| `bats packages/ppm/tests` | 28 of 28 |
| `bats packages/pcm/tests` | 67 of 67 |
| stack `bats packages/ventoy/tests` (`ANFS_TEST_LIB` = the core worktree's lib) | 18 of 18, none skipped |
| `iso.bats -f same` with `VENTOY=` stack's ventoy (the three-tool comparison) | 1 of 1 |
| `packages/pmm/tests/e2e debian` | PASS, 31 steps, about 7 minutes (build --test 186s) |

The e2e read `debian-13.7.0-arm64-netinst.iso` from the real store and wrote nothing to it. The
real `~/.cache/anfs/isos` (and each file in it), `/tmp/pmm-501`, `~/.local/{share,state}/pmm` and
`~/.config/anfs` have the same mtimes after all of the above as before. The tart machine
(pid 40541) was left running.

The plan's Verification list, by hand, in a temp HOME whose `~/.local/{bin,lib}` link to the two
worktrees, with both worktrees as the sources `stack` and `core` and `ANFS_ISO_HOME` inside it:

- [x] `anfs iso list`, `pmm iso list`, `ventoy iso list`: byte-identical, as a table, with
  `--all` and with `--json` (14 entries: stack's 10, core's 4).
- [x] `pmm iso pull debian-13-arm64`, **against a copy, not the network**: the real store's Debian
  ISO was cloned (`cp -c`) into the temp store with no `.sha256` beside it. `list` said
  `unverified`; `pull` hashed it against the sum pinned in core's `isos/isos.yml`, wrote the
  sidecar and made no `curl` call (a logging stub was first on PATH); `list` then said `present`.
  So core's catalogue sum is the sum of the file Debian published.
- [x] `anfs iso add <64K file> --id some`: copied, hashed, recorded with `checksum: sha256:`. Again:
  73. `iso add <url> --id other`: recorded, not fetched, `missing`.
- [x] A blank machine, `media: [some, debian-13-arm64]`, validates; `media: [nosuch]` is an error
  naming `pmm iso list`. `pmm up` started it (hvf) with both store files attached as CDs at
  bootindex 1 and 2 after the empty disk; `down` and `remove` cleaned up. What the guest showed
  was not looked at (see Not Verified).
- [x] `ventoy isos` and `ventoy get debian`: `unknown command`, 64. `pmm image prune --isos`: 64.
- [x] `iso prune` with a stray file: lists it and exits 1 with no terminal and no `-y`; `-y`
  deletes the stray only; `--json` answers `{"deleted":"0"}` on a second run.
- [x] `iso remove some` (no url) without `-y` and no terminal: 1, file kept. `remove --entry
  alpine` (stack's entry): 1, file kept. `remove alpine`: file deleted, entry stays.
  `remove --entry other`: entry dropped from `~/.config/anfs/isos.yml`.
- [x] `ventoy copy -t <dir> debian-13-arm64`: a link in `VENTOY_ISO_DIR` into the store, the file
  copied to the directory.

Seen on the way, none of it fixed:

- `iso pull` of a file that is there, for an entry with only a `checksum_url`, takes it as it is:
  no check, and `list` keeps saying `unverified` (alpine did). That is `anfs_iso_fetch`'s rule
  from before this plan ("taken as it is unless a pinned sum says otherwise"); `pull -f` is the
  way to get it verified. Worth a line in the help, or a decision.
- `ventoy help`: the `VENTOY_HOME` line of the Environment block comes after the paragraph about
  the catalogue. It did before this plan too.
- "Deleted 1 files".
- pmm's bats helper leaves an empty `/tmp/pmm-bats-501` behind.

## Deviations

- Core's `isos/isos.yml` is plan 02's, brought forward. Plan 02 still has to switch `image.yml`
  to ids, drop the inline url and sha256, and delete `packages/pmm/tests/iso.bats`.
- `iso remove` has `-y` and `iso list` has the state `invalid`: neither is in the plan.
- `prune` exits 1 with no catalogue entries, where the plan says only that it asks.
- `ventoy copy <target>` is `ventoy copy -t <target>`: the arguments are ISO ids now.
- `ventoy get --all` has no successor (`iso pull` takes ids).
- "The same rows in anfs, pmm and ventoy" is two tests: core compares anfs with pmm (and ventoy
  when `VENTOY=` names a checkout), stack compares ventoy with anfs.
- The completion bound for `iso pull <TAB>` is in `iso.bats`.

## Left Open

- `iso pull a b --json` prints one object per id.
- A file named only by a lower-priority entry that a higher one shadows is a stray.
- An ISO an image in another source names by URL is a stray until plan 02.
- `complete_iso` reads every clone under the sources home, listed or not, and misses an entry
  whose `id` is not its first key.
- A link in `VENTOY_ISO_DIR` dangles after `iso remove` or `iso prune` deletes its file.

## Not Verified

- **A download from a publisher.** No `iso pull` reached the network: bats stubs `curl`, and the
  hand run used a copy of the file already downloaded. Resume, mirrors and a publisher's
  `checksum_url` file are covered only by the stubbed tests. The sums of core's `debian-13-amd64`
  and both `fedora-44-*` entries were compared with their `image.yml` (`pmm/tests/iso.bats`),
  never with a real file.
- **Anything against the real store**, `iso prune` above all. It holds
  `alpine-standard-3.24.1-x86_64.iso` and `debian-13.7.0-arm64-netinst.iso`; both are named by a
  catalogue entry once this branch and stack's are installed, so a first prune should find nothing.
- **A machine installing from an added ISO.** qemu was started with the ISOs attached; nobody
  watched the installer come up, and the added file was 64K of noise, not a bootable image.
- **A real Ventoy stick**, a block device or a disk image: `ventoy copy` was run against a
  directory only. `prepare`, `update`, `status` and `sync` to a device are Linux and were not run.
- **Linux**: no bats suite and no e2e ran there (kvm, `sha256sum` in place of `shasum`, GNU `ls`
  and `stat` behind `size`).
- `packages/pmm/tests/e2e fedora`, the psm and wsm suites, stack's other suites, `roundtrip` and
  `roundtrip-macos`.
- zsh completion in a real shell: `__complete iso pull ""` was called directly and answers the ids.
