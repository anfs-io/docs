---
objective: What the toolkit builds can be put somewhere other than this machine — a target is a place, a deployment is what runs there, and a new target type is one file
status: pending
---

The "deployment" end of "from dotfiles to deployment". pmm and pcm build artifacts and run them
on this host. Nothing puts them anywhere else: `pmm image export` writes a local file and that
is where the toolkit stops. This unit adds the tool between the artifacts and the places.

Working name: **`pdm`**. Decide the name before plan 01 starts; nothing else depends on it.

Depends on: `machine-images` unit complete (`pmm image list --json`, `pmm image export`).

## The model

| Noun | Is | Lives |
| --- | --- | --- |
| Target | A place, its type and how to authenticate. Data only | `targets/<name>/target.yml` in a source |
| Provider | What knows how to talk to one type of target | an executable, `~/.local/lib/pdm/providers/<kind>` |
| Publication | An image's build, present on a target under a remote id | pdm's state |
| Deployment | What runs on a target: OpenTofu code plus what it needs from the toolkit | `deployments/<name>/` in a source |

```
targets/pve-lab/
  target.yml        kind: proxmox, endpoint, node, storage, bridge
  .env.schema       PVE_API_TOKEN
deployments/web/
  deployment.yml    target: pve-lab, images: {base: debian-13}, depends: [lab-network]
  main.tf           ordinary OpenTofu; takes var.target and var.images
  .env.schema
```

## Decisions

- **A tool of its own, not a pmm command.** Building is local and cached by content; deploying
  needs credentials, remote state and a different failure model. Container images need the same
  thing (a registry is a target), so it cannot belong to pmm or to pcm.
- **Targets and deployments are two kinds.** A place and what you put there change at different
  rates and have different owners: pcs will write targets for the hosts it stands up; a person
  writes the deployment. A Cloudflare target with DNS deployments and no image at all is the
  case that shows they are not the same thing.
- **OpenTofu for deployments, wrapped as pcm wraps compose.** The `.tf` files are valid OpenTofu
  that expect variables pdm supplies, which makes them part of the ecosystem the way a
  `compose.yml` with `x-pcm` is. pdm does not reimplement what tofu does.
- **Publishing an image is not tofu's.** Uploading and converting a multi-gigabyte disk is
  imperative and slow; it is the provider's `publish` verb, and tofu only ever sees the id.
- **Providers are executables on a verb contract**, as pcm's edges are. A new target type is a
  file a package stows. This is the extension seam for AWS, GCP and the rest; only `proxmox`
  ships in this unit.
- **pdm reads artifacts through the owning tool's `--json`**, never its files:
  `pmm image list --json`, later `pcm image list --json`.
- **Local stays pmm's and pcm's.** They are conceptually the local target, but overlays,
  snapshots and 9p shares have no remote equivalent. pdm does not wrap them.

## Open Questions

- **Architecture.** Proxmox is amd64. An amd64 image built on Apple silicon is TCG emulation,
  which is slow enough to matter. Either images for PVE are built on a Linux amd64 host (the
  backplane), or pdm can ask a remote pmm to build. Settle in plan 02; it also decides how the
  `metal` unit builds.
- **Where tofu state lives by default.** Plan 03 proposes `$PDM_STATE_HOME/tofu/<deployment>/`
  with a backend block in the deployment winning. Local state on one laptop is a weak default
  for anything shared.
- **Does pcs deploy through pdm?** Configuring a PVE host is also "put something on a place".
  Left to the `metal` unit.
- **pcm's `cloudflare-tunnel` edge** reaches Cloudflare outside tofu. Keep in view; no change now.

## Out of Scope

- Providers other than `proxmox`.
- Container images and registries (after pcm's image spec lands).
- Running tofu for anything not declared as a deployment; a general IaC front end.
- Network installs and hardware (`metal` unit).

## Plans

1. `01-targets` — the `targets/` kind, the tool's skeleton, the provider contract
2. `02-proxmox-publish` — the `proxmox` provider; `pdm publish debian-13 --to pve-lab`
3. `03-deployments` — the `deployments/` kind; `plan`, `up`, `down` over OpenTofu

## Completion Criteria

- `pdm publish debian-13 --to pve-lab` leaves a template on a real PVE host; running it again
  with an unchanged build does nothing; after a rebuild it publishes the new build.
- `pdm up web` creates a VM from that template on that host through OpenTofu; `pdm down web`
  removes it; `pdm list --json` reports each deployment's state.
- A fixture provider stowed by a test package is found, validated and called, with pdm unchanged.
- `anfs install <source>` installs a source's targets and deployments in dependency order
  (`PDM_INSTALL=validate` for the round trips), and `anfs implode` leaves nothing of pdm's.
- CLAUDE.md documents both kinds, the contract and the `--json` keys.

## Input from consumers

- r-20261022-7 (BACKLOG.md, from cnfs, 2026-10-22, at the person's request): production
  provisioning with the same interface as `pcm provision` (same args, same KEY=VALUE per
  service); one app definition (Dockerfile plus a small declaration) across footprints;
  pcm-style scriptable commands; secrets from the space via fnox (see r-20261022-4 in
  `units/setup.md`); wildcard DNS/TLS for tenant subdomains; per-tenant migration and backup as
  part of a release.
