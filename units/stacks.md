---
objective: anfs manages stacks — manifests of resources applied to a target — so every non-trivial install, the toolkit's own included, is declared, repeatable and reversible
status: plans written
---

ANFS is the Agent Native Full Stack: what anfs manages is stacks of resources. Every other
top-level directory of a source is one tool's resource (`packages/` ppm, `containers/` pcm,
`machines/` pmm, `skills/` psm, `spaces/` wsm, `vaults/` pvm). This unit adds `stacks/`, anfs's
own.

Decided with the person, 2026-10-19.

## What a stack is

A manifest that declares resources to install onto a target. It names no target: the same stack
is applied to this host, a container or a VM.

```yaml
# stacks/rails/stack.yml (in lgat's source)
version: 0.2.0
description: LGAT's rails app and what it runs on
stacks: [lgat/base]                      # other stacks this one depends on, installed first
packages: [stack/ruby, stack/rails, stack/solana]
skills: [lgat/rails-conventions]
spaces:
  - lgat/: { status: installed }
containers:
  - core/anfs-ui: { status: running }
```

- **A directory, `stacks/<name>/stack.yml`**, as packages, containers, machines and images are,
  so a stack can carry a README or a hook beside it.
- **Keys are the resource directories, plural.** Each is installed by its tool. A key's value is
  a list of `source/name` (the same strings the tools' lists print); an entry may be a map with
  `status:`.
- **`status:` is a string, per kind**, the same word the kind's list shows in its STATUS column:
  `running|stopped` for containers and machines, `installed|cloned` for spaces. Absent means the
  tool's default after install.
- **The order is anfs's, not the manifest's**: stacks it depends on, then packages, skills,
  vaults, spaces, containers, machines. Remove is the reverse.
- **A stack may depend on stacks**: `stacks:` lists them, resolved with the same layering rule as
  package dependencies (same or lower-priority source). A cycle is an error.
- **Use one for anything non-trivial**: an install that is repeated, or that spans more than one
  resource. The first is `system`, the toolkit itself.

## Commands

The stack is anfs's main resource, so it is at anfs's top level (the shape every tool has since
`cli/01-one-shape`):

```
anfs list                        STACK STATUS VERSION (source/name, installed|partial|-)
anfs show <stack>                its resources, resolved through its stacks, each declared vs actual
anfs install <stack>... [-t T]   apply; -t/--target: host (default), container:<name>, machine:<name>
anfs remove <stack>... [-t T]    reverse it: what it installed and nothing else holds
anfs path <stack>                its directory
anfs src install <source>...     what `anfs install <source>` does today
```

- `--build` installs only the kinds that belong in an image (packages, skills); the rest (spaces,
  vaults, containers, machines, statuses) is for when the box starts. A vault is never installed
  under `--build`: anything written during an image build stays in a layer.

## Reversible

ppm's trackers record what a package put down, not why it was installed. So anfs keeps its own
record: `$ANFS_STATE_HOME/stacks/<source>/<name>.yml`, every resource the install touched, and
for each whether it was already there. `anfs remove` removes what this stack installed and that
nothing else holds: not another installed stack's resource, and not one that was there before.
`anfs remove system` is today's `anfs implode` (plus the bootstrap with `--all`).

## Targets

- `host`: this machine.
- `container:<name>`, `machine:<name>`: through `pcm exec` / `pmm exec`. anfs must be on the target
  first; when it is not, `install` bootstraps it there (`install.sh`, with the host's sources
  copied or mounted), as the round trips do by hand today.
- In a Containerfile a stack is applied the native way: `RUN anfs install <stack> --build`, with
  the rest at start (the entrypoint or a compose command). A pmm image's `anfs` provision step names
  a stack. Nothing in pcm or pmm re-declares resources; a stack is the one place.

## Plans

| # | Plan | What |
| --- | --- | --- |
| 01 | `01-stacks` | `stacks/`, the manifest, `anfs list\|show\|install\|remove\|path` on the host, the record, `--build`, `anfs src install` |
| 02 | `02-system` | `core/stacks/system`; `install.sh` ends with `anfs install system`; `core/anfs` depends on nothing but what anfs itself runs; implode is `remove system` |
| 03 | `03-container-manifest` | `container.yml` per container (version, description, upstream, memory); the hook bumps any resource's `<kind>.yml`; pcm's memory check |
| 04 | `04-boxes` | A box is a Containerfile + compose + a stack, or a pmm image whose `anfs` step names one; `-t container:\|machine:`; pcm validate's mount roots; the anfs-dev box replaces `ppm container` and `ppm vm` |
| 05 | `05-roundtrips` | The round trips become "a stack onto a target" |

Superseded by this unit: `machine-images/05-pmm-optional` (pmm is not in `system`),
`profiles/01-profiles` (a host's role is a stack that depends on stacks), `dev-source/03-boxes`
(plan 04).

Later: the `stack` source is renamed, so `stack/...` and `stacks/` do not read alike.
