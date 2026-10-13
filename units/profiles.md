---
objective: One repo sets up every machine a person has — a profile per host, built from roles, says what each tool installs there
status: pending
---

A user's repo configures one machine well. With a second machine the choices are to install
everything everywhere or to remember what goes where. The only per-machine mechanism is a
package's `macos/` directory and a hand-written `Brewfile` run from a hook (the user's `host`
package does exactly that).

After this unit a source can hold profiles: a file per host and a file per role, each a list of
what to install, by tool. `anfs profile apply` reads one and runs the tools.

Depends on: nothing. Better after `linker` plan 02 (per-platform files), which removes the other
per-machine mechanism, and after `machine-images` plan 05 (a tool that is not installed).

## The shape

```
profiles/                      anfs's directory in a source, as isos/ is
  hosts/<hostname>.yml
  roles/<role>.yml
```

```yaml
# profiles/roles/dev.yml
description: What I want on any machine I write code on
roles: [base]                       # other roles, applied first
packages: [git, zsh, "@ai", stack/rails]     # what `ppm install` takes
containers: [postgres]              # pcm install
machines: [debian-13]               # pmm install
skills: [stack/]                    # psm install
spaces: [lgat/main/tech]            # wsm install
brew: [ripgrep, fd]                 # the same forms package.yml takes:
cask:                               #   a list, or a map by platform
  macos: [ghostty]
system:
  debian: [nfs-common]
```

```yaml
# profiles/hosts/mbp.yml
description: The laptop
roles: [dev, desktop]
cask:
  macos: [utm]
sources:                            # registered first, as `anfs src add` would
  - {url: git@github.com:lgat-io/anfs, alias: lgat}
```

- A host and a role have the same keys. A host is the entry point for a machine; a role is what
  hosts share.
- An id is `host/<name>` or `role/<name>`, and qualified `<source>/host/<name>`. Profiles are
  found across the sources like every other resource, so an org's source can ship
  `role/developer` and a user's host can list it. The highest-priority source's file is the
  whole profile (no merging of files of the same name).

## Decisions

- **`profiles/` is one top-level directory, anfs's**, holding `hosts/` and `roles/`. A tool owns
  one directory of a source; two more top-level names would break that for no gain.
- **The group is `anfs profile`**, singular, like `anfs src` and `pmm image`: `anfs profile
  list`, not `anfs profiles list`.
- **Each key is handed to its tool unchanged.** A profile adds no way to name a package or a
  service; whatever `ppm install` accepts, `packages:` accepts.
- **Software a profile declares uses `package.yml`'s keys and ppm's code**, not a Brewfile: one
  way to declare software, with the per-platform map, the tap rule and the Homebrew-owner rule
  already there. ppm records it against the profile, as it records a package's, so it can be
  removed.
- **Apply adds; it removes only when asked.** `apply` installs what is missing and is safe to
  run again. `--prune` removes what an earlier apply of this profile installed and the profile
  no longer lists.
- **A tool that is not installed is installed** when the profile has entries for it
  (`ppm install core/pmm` before `machines:`).
- **Settings are not in a profile** (yet). `anfs.local.conf` is per machine already; a
  `conf:` key can come when there is a second machine that needs it.

## Open Questions

1. **Does `hosts/<name>` match the short hostname, or a name the machine is given?** macOS
   hostnames change with the network. Recommended: `ANFS_HOST` in `anfs.local.conf` when set,
   else `hostname -s`; `anfs profile apply` with no argument uses it.
2. **Does `install.sh --repo <url>` apply the host's profile when one matches?** Recommended:
   it offers, through `anfs onboard` (a `profile` step), and applies only with `--profile`.
3. **The user's `host` package** (`Brewfile`, `Brewfile.desktop`, a hook that runs `brew
   bundle`) becomes `roles/desktop.yml` and a host file. Converting it is the user's; the plan's
   log gives the mapping.

## Plans

1. `01-profiles` — the files, `anfs profile list|show|apply|validate|path`, software through ppm,
   `--prune`

## Completion Criteria

- On a second Mac: `install.sh --repo <url>`, then `anfs profile apply`, and the machine has
  what its host file and roles list, and nothing a different host's file lists.
- `anfs profile apply` twice in a row: the second run changes nothing and says so.
- Removing a formula from a role and applying with `--prune` uninstalls it, unless a package or
  another applied profile still declares it.
- `anfs profile show host/mbp` prints the resolved lists, by tool, with the role each entry
  came from.
