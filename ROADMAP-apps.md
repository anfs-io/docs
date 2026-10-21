# Roadmap: apps, infrastructure, components

Started 2026-10-19. The second roadmap: what comes after the toolkit's foundations in
`ROADMAP.md`. Update it as units are written and plans finish.

The direction in one paragraph: the primitives of a build-to-deploy platform, on your own
laptop, with agents assisting at every step. Not a design platform. A person (or the dashboard
for them) starts an app from a template, runs it locally as a container, puts it on the internet
or a private network, and deploys it to a cloud. Every step is a toolkit command first, so a
terminal, an agent and the dashboard all do it the same way.

## The layers

| Layer | What it is |
| --- | --- |
| ANFS | The primitives: packages, containers, machines, stacks, skills, spaces; **apps** (`pxm`) and **infrastructure** (`pim`) from this roadmap |
| The dashboard | `core/ui`: one surface over the toolkit API, for people who do not live in a terminal. It knows every core kind; it is not pluggable (below) |
| CNFS | Rails engines (ui, tenanted, iam, cognito, telemetry, chatter, support, ...) offered as app templates and as a menu of components |
| Use cases | Templates and wizards: "a multi-tenant app with IAM and telemetry, run here, online" |

**Extension is through data, not UI code.** The dashboard stays closed: it knows the core kinds
because core owns them. A source extends the platform by shipping resources (app templates,
infrastructure definitions, stacks); the dashboard lists them from every source, as
`pmm image list` does. UI plugins only if something cannot be said as a tool plus data.

## Before this roadmap starts

From `ROADMAP.md`: `stacks/04-boxes`, `05-roundtrips`, the linker, wsm without levels (an app
lives in a space). The `deploy` unit moves here as section 2.

Cross-cutting, settled in the first section that needs them:

- **Approval.** Exit 77 ("needs a person at a terminal") grows into "needs approval": anything
  that costs money or exposes something to the internet stops for a person, whoever asked (an
  agent, a wizard, a script). At a terminal it is a confirm; in the dashboard an Approve button.
  Settled before `pim` applies anything.
- **The agent runner.** A toolkit command that runs an agent (`claude -p` or another) in a
  container on a mounted workspace, under a grant, logged like any other command. What makes
  "agents assisting" more than a person with a terminal.
- **Credentials.** Cloud tokens through `.env.schema` and varlock, as containers' are.

## The checklist

### 1. Apps: `pxm` and the `apps/` resource (`units/apps.md`, to write)

- [ ] `units/apps.md`: two kinds, as `pmm` has images and machines
  - **Templates** — `apps/<name>/` in a source: a manifest and a generator; any source adds them
  - **Apps** — what `pxm new` makes: a repository in a space with its own `app.yml`,
    Containerfile and AGENTS.md. Once made it belongs to itself; pxm tracks it and runs its
    lifecycle, and the app's own agents take over the work
- [ ] `pxm new <template> <name>`: the generator runs in a container, the repository lands in a
      space through `wsm`, the app is registered
- [ ] `pxm list` (`APP STATUS TEMPLATE WHERE`), `show`, `remove` (unregisters; the repository
      stays)
- [ ] `pxm up|down` through `pcm`, from the app's Containerfile
- [ ] `pxm expose`: a tunnel (pcm's `cloudflare-tunnel` edge, tailscale), behind approval
- [ ] A plain ANFS template, and **one thin CNFS template**, so the template contract is shaped
      by a real second source before section 3
- [ ] `anfs app` in the seam; an Apps panel and a "New app" form in the dashboard
- [ ] The slice end to end: new app, `pxm up`, open it in a browser, `pxm expose`

### 2. Infrastructure: `pim` (the `deploy` unit, renamed)

`units/deploy.md` already holds this design under the placeholder `pdm`: targets, providers as
executables on a verb contract, deployments over OpenTofu, publishing. Rework it rather than
start again.

- [ ] Rename `pdm` → `pim` in `units/deploy.md` and its plans; the old pim (the image manager)
      is gone, its name appears only in history
- [ ] The first providers: Proxmox was first; DigitalOcean droplets, Cloudflare Workers and
      AWS EC2 now come first. Decide the order
- [ ] Approval before `plan` becomes `apply`; where tofu state lives
- [ ] `pxm deploy <app> -t <target>`: through pim, and `pcm image` and a registry target for
      the app's image (`container-images/02-pcm-image` comes first)
- [ ] An Infrastructure panel in the dashboard

### 3. CNFS: templates and the component menu

- [ ] `units/cnfs-apps.md`: the cnfs source's `apps/` templates; the engines as components a
      template declares (`engines: [tenanted, iam, telemetry]`)
- [ ] The component menu: choose engines when making an app, at the terminal (`pxm new
      cnfs/rails --with tenanted,iam`) and in the dashboard's "New app" form
- [ ] Adding a component to an existing app: what the app's own agents are given to do it
- [ ] A use case end to end: a multi-tenant app with IAM and telemetry, made, run, exposed,
      deployed

### Then

- [ ] The tutorial in `ROADMAP.md` is this story: its todo app is a `pxm` app. Revise its chapters
      once section 1 is built

## Open decisions

| Decision | Where | Before |
| --- | --- | --- |
| Template contract: manifest keys, how a generator is run, what it may touch | `units/apps.md` | section 1 |
| Where pxm's registry of apps lives, and an app whose repository moved or was deleted | `units/apps.md` | section 1 |
| The agent runner: tool or `anfs` command; which agents; grant | its own unit | an agent is in a wizard |
| Approval: the exit status and API answer; who may approve | `units/toolkit-api.md` | section 2 |
| pim's first providers and their order | `units/deploy.md` | section 2 |
