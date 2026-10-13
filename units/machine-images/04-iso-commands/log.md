---
status: complete
started_at: "2026-10-13T09:00:00-04:00"
completed_at: "2026-10-13T10:30:00-04:00"
deviations: Core gets isos/isos.yml now (plan 02 was to add it) so image ISOs are catalogue entries. iso remove asks (-y) for a file with no url. iso prune refuses an unreadable or empty catalogue. A new list state, invalid. ventoy copy's stick is -t/--target, no longer an argument. The completion bound is tested in iso.bats, not registry.bats. Full list below.
summary: One iso group in lib/anfs/iso.sh, adopted by anfs, pmm and ventoy, on branch machine-images of core and stack (not pushed, not installed). bats anfs+pmm 222 of 222, stack ventoy 18 of 18. Not verified - anything against the real store or network, a machine booting an added ISO, Linux, the round trips.
---

# Execution Log

`started_at` is the first commit of the work, not a recorded start.

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

- The plan's Verification on this Mac against the real store and network (`anfs iso list`,
  `pmm iso pull debian-13-arm64`, `ventoy iso list`): run only under a temp HOME, `curl` stubbed.
- `anfs iso add <file>` followed by a blank machine booting it from `media:`. No VM was booted.
- `packages/pmm/tests/e2e`, the ppm, pcm, psm and wsm suites, both round trips, anything on Linux.
- `ventoy copy` to a real stick, block device or disk image.
- The first real `iso prune` on this Mac: the store holds `alpine-standard-3.24.1-x86_64.iso`
  and `debian-13.7.0-arm64-netinst.iso`; both are named once this branch and stack's are installed.
