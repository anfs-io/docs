---
---

# Plan 02 — The proxmox provider

## Context — read these files first

- Plan 01's contract (`lib/pdm/providers/null` is the reference implementation)
- `~/spaces.orig/cnfs-io/gems/pim/lib/pim/services/deployers/proxmox_deployer.rb` and
  `lib/pim/models/targets/proxmox.rb` — the Ruby pim's deployer; this is a port of it
- `lib/pmm/defaults/finalize.sh` and `machines/images/debian-13/provision/10-base.sh` — what makes an image
  bootable as a new machine elsewhere (cloud-init, host keys, machine-id)
- `qm` and `pvesh` man pages on the PVE host

## Overview

`providers/proxmox`: a build becomes a VM template on a PVE node. Over ssh to the node, as the
Ruby deployer did: `scp` the disk, `qm create`, `qm importdisk`, `qm set`, `qm template`.
ssh is chosen over the API because `importdisk` needs the file on the node anyway.

The provider ships in a package of its own, `core/pdm-proxmox`, stowing
`~/.local/lib/pdm/providers/proxmox`: the first proof that a provider is not part of pdm.

## Implementation

### 1. Target keys

```yaml
kind: proxmox
ssh: root@pve1.lab        # where qm runs
node: pve1
storage: local-lvm        # where template disks go
bridge: vmbr0
vmid_range: 9000-9099     # templates pdm creates get ids from here
endpoint: https://pve1.lab:8006   # for tofu (plan 03); not used by publish
```

`.env.schema`: `PVE_API_TOKEN` (plan 03 only). `publish` uses the ssh agent; no key is stored.

### 2. Verbs

- `check`: required keys; with `PDM_OFFLINE` unset, `ssh -o BatchMode=yes <ssh> pvesh get
  /version`, the storage exists and takes images, the bridge exists.
- `accepts`: `machine-image`.
- `publish machine-image <name> <arch> <key> <file>`:
  1. Refuse `arch` other than `amd64`.
  2. Lowest free id in `vmid_range`.
  3. `scp` to `/var/tmp/pdm-<name>-<key>.qcow2`.
  4. `qm create <id> --name <name>-<key> --description "pdm: image=<name> arch=<arch> key=<key>"
     --net0 virtio,bridge=<bridge> --scsihw virtio-scsi-pci --agent 1 --serial0 socket --vga serial0
     --ostype l26`.
  5. `qm importdisk`, attach as `scsi0`, `--boot order=scsi0`, add a cloud-init drive
     (`--ide2 <storage>:cloudinit`).
  6. `qm template <id>`, delete the upload. Print `<node>/<id>`.
  A failure after step 4 destroys the VM it created before exiting.
- `published`: `qm list` plus each config's description; lines whose description starts `pdm:`.
- `unpublish <node>/<id>`: `qm destroy`; refuses an id whose description is not pdm's.
- `tfvars`: `endpoint`, `node`, `storage`, `bridge`.

### 3. What the image needs

An image published to PVE must come up as a new machine there. Already true of the shipped
images: cloud-init with the NoCloud and ConfigDrive datasources, the guest agent, host keys and
machine-id regenerated at first boot. Two things to verify on real hardware and fix in the image
if they fail, not in the provider:

- The root filesystem grows to a larger disk (`cloud-guest-utils` is installed; growpart must run).
- amd64 images boot SeaBIOS under pmm; the template is created with PVE's default (SeaBIOS).
  If an image is ever UEFI on amd64, the provider needs `--bios ovmf` and an EFI disk.

### 4. The architecture question

PVE is amd64 and the Mac is arm64, so `pmm image build --arch amd64 debian-13` there is TCG.
Measure it once (the audit only measured arm64/hvf: 151 s). Then one of:

- **Build where it is native.** The image is built on a Linux amd64 host (the backplane, or the
  old laptop) and `pdm publish` runs there. Nothing new to write; documented as how it is done.
- **`pdm publish --build-on <host>`**: ssh to a host with pmm installed, build and export there,
  publish from there. Only if the first proves awkward in practice.

Record the measurement and the choice in `log.md`. Do not build the second option speculatively.

## Test Spec

bats, with `ssh` and `scp` stubbed to record their arguments and replay canned `qm` output:

- `check`: each missing key; an unreachable host; a storage that does not exist.
- `publish`: the sequence of `qm` calls; the id is the lowest free in range; a full range fails
  before any upload; arm64 is refused before any upload; a failing `importdisk` destroys the VM.
- `published` ignores VMs pdm did not create; `unpublish` refuses one.
- Idempotence is pdm's (plan 01); here: `published` reports the key so pdm can skip.

On the real host, by hand, recorded in `log.md`:

- Publish `debian-13`, clone the template in the PVE UI with a cloud-init user and key, boot it,
  log in, confirm a fresh machine-id and host keys, and the grown filesystem.

## Verification

- [ ] `bats packages/pdm-proxmox/tests </dev/null`
- [ ] `pdm target install pve-lab` reports ready against the real node
- [ ] `pdm publish debian-13 --to pve-lab` twice: one template, the second run a no-op
- [ ] The clone boots and passes the image's `test.sh` run over ssh
- [ ] `log.md` has the amd64 build time and the decision from step 4
