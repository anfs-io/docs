---
objective: pcm and pmm are one API to learn — the same commands for running things, and `pcm image` for container images as `pmm image` is for machine images
status: pending
---

Draft for discussion. The plans are written once the open questions below are settled.

`pmm image` is the reference (`machine-images` unit, plan 01). This unit brings pcm to it, and
fixes the places where the two tools already differ for no reason.

Depends on: `machine-images` plan 01 (the reference commands exist).

## One command set

| | pcm | pmm |
| --- | --- | --- |
| From a definition | `up <service>` | `up <machine>` |
| Ad hoc, from an image | **`run <image>` (new)** | `run <image>` |
| Stop | `down` | `down` |
| Running / everything | `ps` / `list` | `ps` / `list` |
| Log in / run a command | `shell` / `exec` | `shell` / `exec` (new in pmm) |
| Snapshots | **`snapshot list\|create\|remove` (changed)** | `snapshot list\|create\|remove` (changed) |
| Back to a snapshot or to the definition | `reset` | `reset` |
| Delete | `remove` | `remove` |
| Definitions | `show`, `path`, `cd`, `validate` | the same |
| Images | **`image ...` (new)** | `image ...` |
| The toolkit's | `install`, `implode` | the same |

Where one tool has something the other cannot (pcm's edges and ingress, pmm's networks and
media), it stays that tool's own.

## What pcm has today

- **No image commands.** An image is either pulled because a `compose.yml` names it, or built
  because the service has `build:` and a `Containerfile` in its own directory (`gemstash`, the
  two test boxes). `pcm remove` deletes what compose built; nothing lists, builds or prunes.
- **No `run`.** Everything pcm starts is a service defined in a source.
- **Snapshots are images.** `pcm snapshot` commits each container to
  `localhost/pcm-snapshot/<service>-<key>:<snapshot>`. They are found by label.

## Decisions

- **The layout is the same in both tools.** `machines/images/<name>/` and
  `containers/images/<name>/`: a tool owns one top-level directory of a source, and its images
  are under it (`machine-images` plan 01).
- **`pcm run <image>` is pcm's, not a bare `podman run`.** The container gets pcm's labels and a
  data directory under `$PCM_VOLUMES_HOME`, so `remove` and `implode` find what it made. It joins
  the shared network only when asked.
- **`pcm image list` shows pcm's images**: those a service names, built or snapshotted. `--all`
  is everything podman has.
- **A `Containerfile` lives with the code it builds.** An app's repo builds and publishes its
  image, and a source ships the service that pulls it (anfs-ui, in the `toolkit-api` unit).
  `containers/images/` is for images with no repo of their own; it is reserved, not used yet.
- **No `pcm image test`.** There is no script to run, and a service's healthcheck covers it.

## Placeholders (do not lose)

- **Provision steps in a container build.** `machine-images` plan 03 keeps ssh out of the
  required part of the provisioner contract so that the same step could run in a container build
  (`GUEST_EXEC` as `buildah run`). Nothing here uses it yet. When this unit's `image build` is
  written, decide whether a `containers/images/<name>/provision/` is run by the same
  provisioners, which would need tool-neutral names for the step's variables (`ANFS_IMAGE_*`) and
  the provisioners moved to `lib/anfs/provisioners/`.

## Plans (to be written)

1. `01-run-exec-snapshot` — `pcm run`, the `snapshot` group in pcm (pmm's is in `machine-images`)
2. `02-pcm-image` — the `image` group

## Completion Criteria (draft)

- Every row of the table above is true of both tools, and `pcm help` and `pmm help` list the
  shared commands in the same order with the same words.
- A person who knows `pmm image` can list, build, remove and prune container images without
  reading `pcm help`.
- CLAUDE.md's "The verbs are the same everywhere" paragraph names the shared set once.
