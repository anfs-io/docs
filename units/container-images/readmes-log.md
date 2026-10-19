# readmes: slim container READMEs, pcm show prints the rest

Job `readmes`, branch `readmes` (core 22a3717, stack: one commit "containers: READMEs keep what
the definition can't say"). Not pushed.

## pcm show (core `packages/pcm/home/.local/lib/pcm/inspect.sh`)

Already printed: container.yml fields, status, deps, used by, data dir, env file, provides,
ingress, privileges, mounts sets, snapshots; ports and bind mounts (declared, or from podman when
created). Named volumes need nothing: validate rejects them. Added:

- `image` per compose service, expanded with the resolved env (`service_images`); `build:` gives
  `built from <context>/<dockerfile|Containerfile>`; ` in <service>` when compose has several.
  An env that does not resolve (e.g. `PCM_PODMAN_SOCKET` unset: dockge, traefik) prints the
  declared `${X_IMAGE}:${X_TAG}`.
- `reads env` / `secrets`: the `.env.schema` names, never values (`schema_names`); `@sensitive`
  items (or `@defaultSensitive=true`) are secrets.
- `commands.sh` help line and AGENTS.md pcm section say what show prints.

## READMEs (stack `containers/`)

Every README got one pointer line to `pcm show <name>`. Nothing moved into container.yml: every
url/upstream a README names was already there (traefik's README links the docs site, container.yml
has traefik.io; both kept).

| container | removed | kept |
|---|---|---|
| README.md (top) | — | added what READMEs keep vs what `pcm show` prints |
| authentik | data dirs line; "all tunables — image, ports, secret key — live in .env.schema" | why, no-Redis note, first-run wizard, OIDC walkthrough, secret warning, outposts |
| n8n | "Data lives in `$PCM_VOLUMES_HOME/n8n/data`"; tunables list | adaptation, memory/OOM gotcha, secret, encryption key, upgrading |
| telemetry | Tempo data path; OTLP/Grafana bullets folded into one sentence; tunables list | architecture, connect, dashboards seam, verify ingest, files table |
| traefik | SELinux reasoning (show prints the privilege's reason) | how services use it, ports gotchas (8080, <1024), symptom without the privilege |
| valkey | upstream url (in container.yml); persistence path | dependents, provision, things to know |
| dockge | upstream url (in container.yml); "tunables" wording | socket, stacks path, security |
| glitchtip | tunables list | why, run, first account, DSN, secret |
| cloudflared | — | all (setup and publishing are not derivable) |

Not moved: n8n's memory note (~570 MB, peaks higher) is a gotcha, not a `memory:` limit; setting
one would be a guess.

## Tests

- Written, not run (pcm suites are off; `ANFS_TESTS_PMM_PCM` never set), in
  `packages/pcm/tests/manifest.bats`: show one image; images per compose service with `build:`;
  `.env.schema` names with sensitive ones as secrets and no values; no schema, no env lines.
- By hand: `pcm show` (`cmd_show`, sourced as tests/helper.bash does) on all 11 stack containers
  in a temp HOME (HOME, XDG_*, every *_HOME and PCM_* in it), the stack worktree as a local-path
  source in `ANFS_USER_LIST`, podman stubbed (`ps` empty, everything else fails). varlock is real.
- `anfs work run readmes core -- packages/anfs/tests/all`: 325 passed, 0 failed (pmm and pcm off).
- Not run: pcm/pmm suites, any VM/box, real podman, `pcm up`.

## pcm show n8n (isolated; $T is the temp dir)

```
stack/n8n  /Users/roberto/work/readmes/stack/containers/n8n
  version      0.1.0
  description  Workflow automation
  upstream     docker.io/n8nio/n8n, docker.io/n8nio/runners
  url          https://n8n.io
  image        docker.io/n8nio/n8n:2.38.7 in n8n
  image        docker.io/n8nio/runners:2.38.7 in n8n-runner
  status       not created
  depends on   stack/postgres
  used by      —
  data         $T/data/volumes/n8n  (none yet)
  reads env    N8N_IMAGE N8N_RUNNERS_IMAGE N8N_TAG N8N_PORT N8N_WEBHOOK_URL N8N_TIMEZONE PCM_POSTGRES_HOST PCM_POSTGRES_PORT PCM_POSTGRES_DATABASE PCM_POSTGRES_USER PCM_VOLUMES_HOME
  secrets      N8N_RUNNERS_AUTH_TOKEN PCM_POSTGRES_PASSWORD

declared (not created)
  n8n  port   5678:5678
  n8n  mount  $T/data/volumes/n8n/data -> /home/node/.n8n
```
