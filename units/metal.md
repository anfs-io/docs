---
objective: An image definition installs onto hardware over the network as it builds into a qcow2 — one definition of a host, rehearsed in VMs, proven on a real machine
status: pending
---

Outline only. Write the plans after `machine-images` is complete and `deploy` has its proxmox
provider, since both decide what this unit can lean on.

Depends on: `machine-images` unit complete; `deploy` plans 01 and 02 complete.

## The idea

pmm builds a root image by booting the installer's kernel with an initrd that has the rendered
answer file appended as one more cpio archive. A machine that PXE-boots that same kernel and
initrd installs from the same definition. So "a PVE host" is one image definition (answer file,
provision steps) with two deliveries: built as a qcow2, or installed over the network.

This replaces how pcs was conceived (PXE to netboot.xyz, a Debian install, then an Ansible run
that upgrades the host to Proxmox). The upgrade becomes provision steps on the definition; an
ansible provisioner is a file under the `machine-images` contract if the existing playbook is
worth keeping.

## Decisions

- **pcs is written last and is small.** Most of what it was to do now has a home: the backplane
  is a machine running pcm services (`stack/containers/netboot`, `dnsmasq`); calls to the cluster
  are pdm's `proxmox` provider; ISOs are the shared store. What is left for pcs is an inventory
  of physical hosts and the workflow that takes one from powered off to registered as a target.
- **Rehearse in VMs before hardware.** A blank pmm machine on an isolated pmm network, served by
  the backplane's containers, is the same flow with nothing to carry.
- **Debian plus Proxmox's packages first.** It reuses the Debian answer file and is Proxmox's
  documented install-on-Debian route. Proxmox's own answer-file installer is the alternative;
  check how it network-boots before choosing it.

## Open Questions

- **Provision steps on hardware.** In a build, pmm boots the installed disk and runs the steps
  over ssh. On a network install something has to do the same after the first boot: the
  backplane reaching the new host with the image's key, or the steps baked into the install as a
  first-boot unit. The first keeps one code path.
- **pikvm's role.** Power, boot order and console for a host that is not answering: the provider
  for "make this host PXE boot now". Hosts without one are power-cycled by hand.
- **Does pcs deploy through pdm?** After install, a host is a target; configuring it further
  (storage, cluster join) could be a deployment.
- **pmm networks are Linux-only**, so the rehearsal runs on a Linux host, not the Mac.

## Plans (outline)

1. `01-netboot-artifacts`: `pmm image netboot <image>` writes the kernel, the initrd with the
   injected answer file, and the kernel command line into a directory. This is the part of
   `_build_qemu` before qemu starts, made reachable.
2. `02-pmm-network-boot`: a blank machine can boot from its first interface (`boot: network`).
   Needed only for the rehearsal.
3. `03-backplane-serving`: the `netboot` and `dnsmasq` services serve step 1's directory; one
   host entry (MAC to image) decides what a machine gets.
4. `04-rehearsal`: on a Linux host, a blank machine on an isolated network installs `debian-13`
   from the backplane's containers, comes up, and passes the image's `test.sh`.
5. `05-hardware`: the same on the old laptop (2014, spinning disk, currently trixie from the
   ventoy stick). The first real machine installed by the toolkit over the network.
6. `06-pve-host-image`: an image definition for a PVE host (`from:` nothing; Debian's answer
   file plus the Proxmox steps), installed by 04's path.
7. `07-pcs`: the host inventory (`hosts/<name>/host.yml`: MAC, address, image, pikvm), the
   bring-up workflow, and writing a `targets/<host>/target.yml` when a PVE host answers.

## Completion Criteria

- The laptop, powered on and set to network boot, installs an image defined in a source with no
  keyboard input, and `pmm image test`'s script passes against it over ssh.
- The same definition builds as a qcow2 with `pmm image build`.
- A host installed as a PVE node appears in `pdm target list` as `ready` without a hand-written
  target.
