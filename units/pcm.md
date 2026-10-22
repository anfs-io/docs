---
objective: pcm's own commands, one plan at a time, where no larger unit owns them
status: 01-run written 2026-10-22
---

pcm (podman compose services from anfs sources) changes mostly inside larger units (`stacks`,
`config`, `toolkit-api`). This unit holds the pcm changes that stand alone: `01-run`, `pcm run
<service>` (a one-off container from a service's definition, as `docker compose run` is) and the
fix to `pcm show`'s mounts on podman 6. It settles `container-images` plan 01's `pcm run <image>`:
there is none, since `podman run --rm` is the tool for a stock image (the person, 2026-10-22).
