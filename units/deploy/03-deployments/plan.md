---
---

# Plan 03 — Deployments over OpenTofu

Write the details of this plan after 02 is done: the real PVE host will have settled the target's
keys and what a template needs. What is fixed now is the shape.

## Context — read these files first

- Plans 01 and 02, and their logs
- `packages/pcm/home/.local/lib/pcm/{compose,deps,runtime}.sh` — pcm wrapping `podman compose`:
  how it adds to what the user wrote without rewriting it; `x-pcm.depends_on`
- `packages/anfs/home/.local/lib/anfs/resolve.sh` — `anfs_resolve`, ordering by dependency
- OpenTofu: `-chdir`, `TF_VAR_*`, `TF_DATA_DIR`, `-json` plan output, the `bpg/proxmox` provider

## Overview

```
deployments/<name>/
  deployment.yml
  *.tf
  .env.schema
  README.md
```

```yaml
# deployment.yml
target: pve-lab
images:                 # name in tofu: image in the toolkit. Published to the target before apply
  base: debian-13
depends: [lab-network]  # other deployments, applied first
vars:                   # plain inputs
  count: 2
```

```hcl
# main.tf: ordinary OpenTofu. pdm supplies these three.
variable "target" { type = map(string) }     # the provider's tfvars
variable "images" { type = map(string) }     # name => remote id on this target
variable "vars"   { type = map(string) }
```

## Implementation (outline)

1. **The kind.** `catalog_load deployments deployment.yml`; `anfs_tool_dirs pdm` becomes
   `targets deployments`. Deployments do not layer. `depends:` resolves like any dependency:
   same or lower-priority source.
2. **`pdm plan|up|down <deployment>`**, with `list`, `show`, `path`, `cd`, `validate`, `install`,
   `remove`.
   - `up`: dependencies first; publish each image in `images:` that is not on the target (plan
     01's `publish`); export `TF_VAR_target` (the provider's `tfvars`), `TF_VAR_images`,
     `TF_VAR_vars` and the credentials; `tofu -chdir=<dir> init` then `apply`.
   - `plan` does everything `up` does short of `apply`, including saying which images would be
     published. `-n` on `up` is `plan`.
   - `down`: `tofu destroy`, dependents first; refused while a deployment that depends on it is up.
   - `-y` is `-auto-approve`; without it tofu asks, as it does.
3. **State.** `TF_DATA_DIR=$PDM_STATE_HOME/tofu/<deployment>` and local state there, unless the
   deployment's own `.tf` declares a backend, which wins. Nothing of tofu's is written into the
   source checkout (a source may be read-only, and wsm's rule about clones applies here too).
4. **What a deployment is on.** After `apply`, record the image keys it used. `pdm list` keys:
   `name`, `source`, `target`, `state` (`up`, `down`, `drifted`, `defined`, `shadowed`), `images`
   (stale when a newer build exists, as `pmm show` says for a machine). Superseded publications no
   deployment records become removable (`pdm publish --prune`).
5. **OpenTofu itself** is declared by the package (`brew: [opentofu]`), not vendored.
6. **Events.** `anfs_callbacks_run pdm up|down <deployment>` once there is a listener, not before.
7. **A skill**, `pdm-deployments`, on the pattern of `pcm-containers`.

## Decisions to make when this is written

- Whether `vars:` and `images:` arrive as maps (above) or as one variable each. Maps keep the
  `.tf` free of pdm-specific variable names.
- Whether `validate` runs `tofu validate` (needs `init`, which needs the network for providers).
- The first real deployment to prove it with: one VM from the `debian-13` template on `pve-lab`,
  with a cloud-init user and key, reachable by ssh.

## Verification (to be completed)

- [ ] `pdm up web` on the real PVE host; `ssh` to the VM; `pdm down web`
- [ ] A second `up` with nothing changed applies nothing
- [ ] After `pmm image build -f debian-13`, `pdm list` shows the deployment's image as stale and
      `pdm up` publishes the new build and replaces the VM
- [ ] `anfs implode` leaves no pdm state; it does not destroy deployments (it says which are up)
