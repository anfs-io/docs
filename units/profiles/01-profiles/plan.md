---
---

# Plan 01 — Profiles: hosts and roles

## Context — read these files first

- `units/profiles.md` — the file shape and the decisions
- `packages/anfs/home/.local/bin/anfs` — `cmd_install` (how it runs each tool over a source),
  `ANFS_TOOLS`, the `src` group
- `packages/anfs/home/.local/lib/anfs/catalog.sh` — finding one kind of definition across the
  sources; `iso.sh` for a second example of an anfs-owned directory with a layered catalogue
- `packages/ppm/home/.local/lib/ppm/packages.sh` and `installer.sh` — how `brew`, `cask` and
  `system` are read (list or platform map), installed in one batch per manager, recorded in the
  tracker as `installed_deps`, and removed when nothing else recorded or declares them
- `packages/anfs/home/.local/lib/anfs/onboard.sh` and `lib/ppm/onboard.sh` — the step contract
- `packages/anfs/tests/fixtures/acme` (or its new home in `anfs-dev`)

## Overview

`profiles/hosts/*.yml` and `profiles/roles/*.yml` in any source. anfs resolves a profile to one
list per tool and runs each tool's own `install`; software keys go through ppm. What an apply
installed is recorded, which is what `--prune` and `show` read.

## Implementation

### 1. Finding and resolving

`lib/anfs/profile.sh`:

- Profiles are found with the catalogue (`catalog_load profiles ...` twice, or one load over
  `profiles/` with ids `host/<n>` and `role/<n>`: whichever the catalogue takes without change;
  one catalogue per process is the rule).
- `profile_resolve <id>` walks `roles:` depth first, each role once, a cycle is an error naming
  the path. Output: per key, the entries in order, each with the profile it came from; an entry
  seen twice is kept once.
- `brew`, `cask` and `system` are reduced to this machine's lists by ppm's reader.

### 2. Commands

```
anfs profile list                     ID SOURCE KIND APPLIED DESCRIPTION
anfs profile show <profile>           the resolved lists by tool, with where each entry came from
anfs profile validate [profile...]    keys, roles that exist, entries each tool can resolve
anfs profile apply [-n] [--prune] [-y] [profile]
anfs profile path <profile>
```

- `list` keys: `id`, `source`, `kind` (`host`, `role`), `applied` (the time of the last apply
  on this machine, null otherwise), `description`. The host that matches this machine is marked.
- `apply` with no argument is `host/<this machine>` (`ANFS_HOST`, else `hostname -s`); with no
  such file it says so and exits 1.
- `validate` asks each tool whether it knows an entry (`ppm path`, `pcm path`, and so on),
  without installing.

### 3. Apply

In this order, each step skipped when its list is empty:

1. `sources:` — registered as `anfs src add` does, then cloned.
2. Software: `ppm __profile deps <id>`, plumbing that takes the three resolved lists on stdin and
   installs what is missing through the code a package's declared software uses, recording them
   in `$PPM_STATE_HOME/installed/profiles/<id>.yml` (`installed_deps`, the tracker's own key).
   The Homebrew-owner rule applies unchanged.
3. `packages:` — `ppm install <entries>` in one call, so dependencies resolve together.
4. `containers:`, `machines:`, `skills:`, `spaces:` — `<tool> install <entries>`. A tool that is
   not installed is installed first (`ppm install core/<tool>`).
5. The record: `$ANFS_STATE_HOME/profiles/<id>.yml` with the resolved lists and the time.

A tool that fails is reported and the rest still run; the exit status is 1 and the record is
written with what did succeed. `-n` prints the five steps with their entries and runs nothing.

### 4. Prune

`apply --prune` compares the record of the last apply with the resolved lists now, and for each
entry that left: `<tool> remove <entry>` (after `anfs_confirm`, listing them), and
`ppm __profile deps` removes software no package and no other applied profile's record declares.
System packages are never removed, as for packages. Without `--prune`, `apply` prints one line
when the record has entries the profile no longer lists.

### 5. Onboard

A `profile` step in anfs's own steps, after `repo`: detect is "a host file matches this machine
and its record is current"; apply proposes `anfs profile apply`. `-y` does not take it; an
`--profile` option does.

### 6. Around it

- `anfs implode` removes `$ANFS_STATE_HOME/profiles/`; `ppm implode` the profile trackers
  (and the software they recorded, by the existing rule).
- `anfs src list`'s `provides` shows `profiles` for a source that has the directory.
- The fixture source gets a host and two roles.
- CLAUDE.md: the table of top-level directories, the key table row for `anfs profile list`, a
  "Profiles" section under "Your Customization Repo".

## Test Spec

`packages/anfs/tests/profile.bats` (fixture sources; ppm's managers stubbed as its own tests do):

- Resolve: nested roles in order, a role named twice applied once, a cycle refused, a
  higher-priority source's role of the same name wins whole.
- Platform maps reduce to this machine's lists (stub `platform`).
- `apply -n` prints every step and changes nothing; `apply` calls each tool once with the
  resolved entries; a second `apply` installs nothing.
- A failing tool: the others run, exit 1, the record has what succeeded.
- `--prune`: an entry removed from a role is removed; a formula still declared by an installed
  package stays; without `--prune` the one-line notice.
- No argument: `ANFS_HOST` wins over `hostname -s`; no matching file exits 1.
- `list --json` keys; `validate` names an entry no tool knows.

## Verification

- [ ] `bats packages/{anfs,ppm}/tests </dev/null`
- [ ] `roundtrip debian` with the fixture's host profile applied in the box: its packages, its
      container and its formula are there; `anfs implode` leaves the `$HOME` diff clean
- [ ] On this Mac, isolated HOME: the user's `host` package's `Brewfile` written as
      `roles/desktop.yml`, `anfs profile apply -n` lists the same formulas and casks
