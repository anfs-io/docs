---
objective: It works on more Linux — ppm bootstraps and installs packages on Fedora, then Arch, with the cross-platform model unchanged
status: pending
---

Distros tier. Adds Linux distributions beyond Debian by extending the platform adapters, not by adding new install paths. Because brew and mise carry most software, each distro needs only: detection, brew's prerequisites, a `system` package adapter, `system.<distro>` entries in packages whose names differ from the `linux` fallback, and a VM test path.

Depends on: `cross-platform` unit complete.

## Order

1. **Fedora** — Red Hat family (most familiar), upstream of RHEL so Rocky/Alma follow via `ID_LIKE`; first-class podman; recent glibc; PIM gains kickstart alongside the Debian preseed.
2. **Arch — on hold.** Fedora (plus RHEL derivatives) is enough to prove multi-family Linux support. Arch also has no official arm64 container image, so it can't be container-tested on Apple silicon without emulation; automated installs rely on `archinstall` JSON configs; and pacman + AUR overlap most of what brew provides.

## Out of Scope

- PIM kickstart and archinstall implementation (tracked in PIM; plans here only require a working VM image to test with)
- RHEL/Rocky/Alma as tested targets (expected to work via `ID_LIKE`, verified opportunistically)

## Plans

1. `01-fedora`
2. `02-arch` — on hold; not part of this unit's completion criteria

## Completion Criteria

- `install.sh` completes on fresh Fedora (current release) containers and VMs as admin and as a second non-admin user, matching the cross-platform unit's behavior (one sudo prompt, consumer mode, idempotent re-run)
- `platform()` returns `fedora`, with `ID_LIKE` fallback for RHEL derivatives
- `system_pkg_missing` / `system_pkg_install` support dnf
- Every package with a distro-keyed `system` map has an entry for `fedora`, a `linux` fallback, or `platforms` excluding Fedora with a documented reason
- README lists supported platforms and their prerequisites
