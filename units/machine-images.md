---
objective: Machine images are pmm's, as container images are pcm's — one tool, an image directory that holds everything that defines it, and provisioning open to any provisioner
status: pending
---

Brings VM images into line with the rest of the toolkit. Today `pim` builds and `pmm` runs, the
scripts and answer file an image is built with live in pim's library, and provisioning is shell
over ssh and nothing else. After this unit:

- `pmm` is the one tool for machines; `pmm image ...` manages what they run from, as `pcm image`
  does for containers. pim is gone.
- An image directory is complete: answer file, provision steps, files, test, README. Nothing a
  build runs is hidden in a library except the finalize step.
- A provision step is run by a provisioner, an executable on a small contract. `sh` and `anfs`
  ship; anything else (ansible) is a file another package stows.

There is no backwards compatibility: no migration of pim's builds, no forwarding stub. The
installs that exist are this Mac and a few Debian hosts that can be rebuilt.

## Decisions

- **`pmm image`, not a separate tool.** The rule across the toolkit: one tool per runtime, its
  main object is the running thing, `image` is the group for what it runs from. The split cost
  three things that disappear inside one tool: pmm sourcing four pim libraries under a "no `cmd_`
  function" rule, holds as a cross-tool protocol, and `pmm create` shelling out to `pim build`.
- **`pmm image` is the reference API**, not a port of pim's. `pcm image` follows it when it is
  written. One list instead of `list` and `builds`; `test` instead of `verify`, which read like
  `validate`; `run <image>` for a machine from an image and `up <machine>` for a defined one.
- **Settings for images are `PMM_IMAGE_*`**; what machines read too stays `PMM_*`.
- **The model does not change.** An image is built once and any number of machines run from it,
  each on the build it was created from (the AMI/instance model).
- **One tool, one directory in a source.** Images and networks move under `machines/`
  (`machines/images/`, `machines/networks/`). It frees the name `images` at the top level, which
  containers need as much as machines do, and pcm gets the same layout (`containers/images/`).
- **`export`, not `publish`.** Writing a build as one standalone qcow2 is `pmm image export`.
  "Publish" is the deploy tool's word for putting an image on a target (`deploy` unit).
- **Packer is not used.** Evaluated and rejected for local builds: its qemu builder types a boot
  command over VNC, has no overlay chain or content-hash cache, and adds HCL as a second
  definition language. Packer builds per platform natively; it is not a build-once tool.
- **Template variables stay in `image.yml`.** With the defaults moved into each shipped image's
  `vars:`, nothing is hidden, and no schema file is needed. `.env.schema` is reserved for the day
  an image needs a secret.
- **A provisioner is chosen by the step's extension**, and a step nothing provides fails
  `validate`. Same idea as a `package.yml` key with no handler: declaring something and silently
  doing nothing is the outcome to rule out.

## Out of Scope

- Putting an image anywhere but this machine (`deploy` unit).
- Installing an image definition onto hardware over the network (`metal` unit).
- Root images that start from a distro's cloud qcow2 instead of an ISO install. The installer is
  75% of a build (111 s of 151 s for Debian on Apple silicon); worth a plan of its own once this
  unit lands, since it is a new root type.
- Converting the qemu and tart backends to executables on a contract. Do it when a third backend
  arrives.
- Reproducible package sets (snapshot.debian.org). The build key says "same inputs", not "same
  bytes"; noted, not addressed.

## Plans

1. `01-fold-pim-into-pmm` — commands, libraries, data, tests; pim's package removed
2. `02-image-definition` — the image directory is complete; ISO ids; `show` prints the plan; `new`
3. `03-provisioners` — the contract; `sh` and `anfs` behind it
4. `04-iso-commands` — one `iso` command group, defined once, in every tool that uses ISOs

Each plan leaves the bats suites and `packages/pmm/tests/e2e` passing. 02 and 03 both change the
image directory; 02 does the layout, 03 the dispatch.

## Completion Criteria

- No `pim` command, package, library directory or state directory exists after
  `ppm install core/anfs` on a fresh machine; `grep -rw pim` over core and stack finds only history.
- `pmm image build debian-13 && pmm run debian-13` works on macOS (hvf) and Linux (kvm).
- `machines/images/debian-13/` and `machines/images/fedora-44/` contain every file their builds read except
  `finalize.sh`; `pmm image show debian-13` lists each with its path.
- A provisioner stowed by a test fixture package runs a step in a build, and changing a file its
  `fingerprint` names changes the build key.
- `pmm image list --json` is documented in CLAUDE.md's key table, and `anfs iso`, `pmm iso` and
  `ventoy iso` are the same commands.
- `packages/anfs/tests/roundtrip` and `roundtrip-macos` pass.
