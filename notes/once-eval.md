---
objective: Decide what anfs/pcm takes from 37signals ONCE; design pluggable pcm ingress (proxy + edge) and plan step 1
status: in progress (steps 1–2 done, step 3 built)
---

# ONCE Evaluation and pcm Ingress Design

Evaluated 2026-10-04 against `curl https://get.once.com | sh`, `basecamp/once` at da130f0
(2026-09-16), `basecamp/kamal-proxy` at HEAD, once.com, and the 37signals dev post
"The ONCE app server".

**Verdict.** Don't adopt ONCE. Port four of its ideas into pcm. pcm is already ahead on
everything declarative: definitions in git, layered sources, typed env and secrets, service
dependencies, and validation. ONCE is ahead on the runtime: ingress with TLS, zero-downtime
redeploys, automatic updates, and scheduled backups. Start with **ingress** (section 4): a pluggable proxy (Traefik first) and pluggable edges for internet exposure (Cloudflare Tunnel first, section 5). Section 6 is the step-1 implementation plan.

---

## 1. What get.once.com does

The script is 191 lines of POSIX sh and only bootstraps. The real product is the `once` Go binary.

1. Detects the OS (linux/darwin) and CPU architecture (amd64/arm64).
2. Makes sure Docker is present:
   - Linux: runs `curl get.docker.com | sudo sh` silently (output goes to `>/dev/null`), then `usermod -aG docker $USER`.
   - macOS: tells you to install Docker Desktop.
3. Reads the latest release from the GitHub API, parsing the JSON with awk so jq is not needed. `GITHUB_TOKEN` is honoured.
4. Runs `sudo install` to put the binary in `/usr/local/bin`, then `once background install` (systemd or launchd).
5. Launches the TUI. With `ONCE_INTERACTIVE=false` it stops before launching.

Security notes: there is no checksum or signature check on the binary, Docker install failures are silent, and joining the docker group is equivalent to root.

## 2. What the once binary does

- **Ingress.** One `basecamp/kamal-proxy` container owns ports 80/443. It routes by host, gets Let's Encrypt certificates automatically, and redeploys with zero downtime: the new container boots, passes `/up`, then traffic switches over and the old container drains.
- **App contract.** Serves HTTP on port 80, answers `/up`, and keeps its data in `/storage` (also mounted at `/rails/storage`).
  - Optional hooks: `/hooks/pre-backup` and `/hooks/post-restore`.
  - Injected env: `SECRET_KEY_BASE`, `BASE_URL`, `DISABLE_SSL`, `VAPID_*`, `SMTP_*`, `NUM_CPUS`.
- **Background runner** (checks every 5 minutes):
  - Pulls images every 24h and redeploys with zero downtime.
  - Backs up daily: a tar.gz of the volume plus settings, kept for 30 days. Without a pre-backup hook it pauses the container during the copy.
  - Updates itself.
- **State** is a JSON `once` label on each container and volume. There are no config files.
- **TUI.** Charts of request and error rates (from proxy Prometheus metrics), unique visitors (HyperLogLog over client IPs), CPU, memory and disk, logs, and settings forms.
- **CLI:** `deploy list start stop remove update exec backup restore keys background self-update teardown`.
- **Apps:** Campfire, Writebook and Fizzy, open-sourced in the March 2026 "ONCE Again" pivot.
- **Roadmap:** there isn't a formal one. Built-in **offsite backups** are floated as a next step. There is deliberately no auth layer, and it is aimed at SQLite apps that keep their data in `/storage`.

## 3. Comparison and decision

| Capability | ONCE | pcm | Action |
|---|---|---|---|
| Declarative defs in git, layered sources | ✗ | ✓ | keep |
| Typed env and secrets (varlock/fnox) | ✗ | ✓ | keep |
| Service dependencies, provision outputs | ✗ | ✓ | keep |
| Validation (mounts, ports, privileges) | ✗ | ✓ | keep |
| Proxy, per-service hostnames, automatic TLS | ✓ | ✗ | **port: sections 4–5** |
| Zero-downtime redeploy | ✓ | ✗ | later (section 4.7) |
| Scheduled app-aware backups, retention | ✓ | partial (`snapshot`/`reset`) | port: `backup` hook, `pcm backup/restore` |
| Automatic image updates, boot persistence | ✓ | ✗ | port: Quadlet plus `podman auto-update` (revives `legacy/quadlet.sh`) |
| Generated `SECRET_KEY_BASE`, `BASE_URL` | ✓ | ✗ | port: `BASE_URL` falls out of ingress |
| TUI, metrics, visitor counts | ✓ | ✗ | skip (`core/dockge` exists) |
| State in container labels | ✓ | ✗ | skip: against anfs's "reproducible from git" model |

**Why not run ONCE alongside pcm:**
- It would add a second runtime (Docker next to podman).
- Its proxy would claim ports 80/443 outside pcm's control.
- It can't express multi-service, postgres-backed apps.
- Its state can't be rebuilt from a source.

If we want Campfire, Writebook or Fizzy, each is a small pcm definition: port 80, healthcheck `/up`, and a bind mount at `/storage`.

---

## 4. Design: pluggable ingress (proxy)

### 4.1 Goal and layers
`pcm up twenty` makes Twenty reachable by hostname. On a laptop that is `http://twenty.localhost:8088`, needing no DNS. Once exposed, it is `https://crm.roteoh.com`. No service needs its own host port, and each service gets its public URL injected.

Two plugin types, each chosen by configuration:

| Layer | Job | First implementation | Later |
|---|---|---|---|
| **Proxy** (this section) | route a hostname to a container on `dev-net` | Traefik | Caddy, kamal-proxy |
| **Edge** (section 5) | get internet traffic to the proxy; DNS for a host | `local`, `cloudflare-tunnel` | `tailnet`, `direct` |

pcm core never names a product. It knows the roles `proxy` and `edge`, plus a small hook contract for each.

### 4.2 Services declare ingress, not a proxy
```yaml
x-pcm:
  ingress:
    service: server          # compose service to route to (default: the only one)
    port: 3000               # container port
    healthcheck: /healthz    # default /up
    hosts: ${TWENTY_HOSTS}   # comma-separated; default <name>.$PCM_INGRESS_DOMAIN
```
- `hosts` goes through compose-style variable expansion, the same `expand_vars` used for mounts. `pcm expose` (section 5) only edits `<NAME>_HOSTS` in `$PCM_ENV_HOME/<name>.env`, so definitions in git never change when a service is exposed.
- `PCM_INGRESS_DOMAIN` defaults to `localhost`, so every ingress service gets `<name>.localhost` with no setup.
- A service with `ingress` gets an **implicit dependency on the active proxy**. Today's dependency machinery then does the rest:
  - `start_deps` starts the proxy and waits for it to be healthy.
  - `compose_flags` joins the service to `dev-net`.
  - `pcm down traefik` refuses while ingress services run.
  - `check_deps` validates the dependency.

### 4.3 A proxy is a definition with a role
```yaml
# core/containers/traefik/compose.yml
x-pcm:
  provides: proxy
```
- **Selecting the proxy.** `PCM_PROXY=<service>` picks it. Like `PCM_COMPOSE` and `PCM_ATTACHED_SERVICES`, it is a plain `anfs.conf` setting (the environment still wins), so there is no new config file. When unset, pcm uses the only effective definition that `provides: proxy`. If there are none, or more than one, any ingress service fails validation with a clear message.
- **Swapping proxies.** Set `PCM_PROXY`, then `pcm down` and `pcm up` the ingress services. No definition changes.

### 4.4 Proxy hook contract
Hooks are executables in the proxy's definition directory. They run like today's `provision`, through `run_hook`/`service_run`: under varlock with the proxy's env. They get the ingress as `key=value` args (`name= service= port= healthcheck= hosts= network=`) plus `PCM_SERVICE` and `PCM_DEPENDENT`.

| Hook | When | Prints | Needed by |
|---|---|---|---|
| `labels` | while `compose_flags` writes the dependent's override | `label=value` lines; pcm writes them as labels on `ingress.service` | Traefik, caddy-docker-proxy |
| `provision` (existing) | on `up`, proxy healthy, before the dependent starts | `URL=<scheme>://<host>[:port]` → exported as **`PCM_INGRESS_URL`** | all |
| `attach` / `detach` | after `up` / before `down` | — | kamal-proxy, Caddy API. **Not built until a second proxy needs them** |

Notes:
- **Hosts go through a hook, not into core.** Traefik's routing rule needs the actual host list (``Host(`a`) || Host(`b`)``). pcm expands the hosts and passes them, so hooks never parse YAML or env files. Hooks print flat `key=value` lines, the same format `provision` uses, and pcm writes the YAML.
- **A role's outputs use the role's prefix.** Output from a role provider's `provision` is exported under the role prefix (`PCM_INGRESS_*`) rather than `PCM_<DEPNAME>_*`. Without that, a service would read `PCM_TRAEFIK_URL`, which breaks swapping proxies. Definitions use `${PCM_INGRESS_URL}` for their base URL (Twenty: `SERVER_URL`).
- **Who decides the scheme.** In step 1 the proxy decides scheme and port: `http` and `PROXY_HTTP_PORT`, with the port omitted when it is 80 or 443. When edges land (section 5), the edge that owns a host decides its scheme, because behind Cloudflare the proxy serves HTTP while the public URL is `https`.

### 4.5 `core/containers/traefik/`
**`compose.yml`:**
- Image `docker.io/library/traefik:${TRAEFIK_TAG}` (pinned v3.x), `x-pcm.provides: proxy`.
- **Static config through command-line flags:**
  - `--providers.docker=true`
  - `--providers.docker.exposedByDefault=false`
  - `--providers.docker.network=dev-net`
  - `--providers.docker.constraints=Label(\`io.pcm.ingress\`,\`true\`)`
  - `--entryPoints.web.address=:80`
  - `--entryPoints.websecure.address=:443`
  - `--api.insecure=true`: the dashboard and API on container port 8080. That port is not published, so it is reachable only on `dev-net`
  - `--ping=true`
- **Ports:** `${PROXY_HTTP_PORT}:80` and `${PROXY_HTTPS_PORT}:443`. The defaults are **8088/8443** (8080 is telemetry's Grafana) because rootless podman can't publish ports below 1024 without `sysctl net.ipv4.ip_unprivileged_port_start=80`. The README documents 80/443 for servers.
- **Mounts:** `${PCM_PODMAN_SOCKET}:/var/run/docker.sock:ro`, listed in `x-pcm.allow_mounts`. pcm already resolves the socket, including inside the podman machine on macOS. Also `${PCM_VOLUMES_HOME}/traefik/acme:/acme` for later certificate resolvers.
- **Healthcheck:** `traefik healthcheck --ping`.
- **SELinux:** `security_opt: [label=disable]`, declared under `x-pcm.privileges`. SELinux is enforcing in the podman machine and denies containers the podman socket (`user_tmp_t`); without it the docker provider gets `permission denied`. Found during step 2. `dockge` mounts the socket the same way and likely needs the same fix.
- **Dashboard:** routed through Traefik itself. The definition declares `x-pcm.ingress: {service: traefik, port: 8080, healthcheck: /ping}` and is excluded from its own implicit dependency, which gives `traefik.localhost:8088`.

**`.env.schema`:** `TRAEFIK_TAG`, `PROXY_HTTP_PORT=8088`, `PROXY_HTTPS_PORT=8443`, `PCM_PODMAN_SOCKET`, `PCM_VOLUMES_HOME`.

**`labels`** (about 15 lines of bash). With `r=pcm-$name`, it prints:
```
io.pcm.ingress=true
traefik.enable=true
traefik.http.routers.$r.rule=Host(`a`) || Host(`b`)
traefik.http.routers.$r.entrypoints=web
traefik.http.services.$r.loadbalancer.server.port=$port
traefik.http.services.$r.loadbalancer.healthcheck.path=$healthcheck
```
TLS router and certificate-resolver labels are added when edges need them (section 5).

**`provision`:** prints `URL=http://<first host>[:$PROXY_HTTP_PORT]`.

**Traefik makes this simple.** Routes live on the containers as labels, so a route appears when its container starts and disappears when it stops, with no state in pcm. That is why `attach`/`detach` aren't needed for v1.

### 4.6 First adopter: Twenty
- Add `x-pcm.ingress: {service: server, port: 3000, healthcheck: /healthz}`.
- Add `TWENTY_HOSTS` to the schema, optional with an empty default.
- Change `TWENTY_SERVER_URL` to default to `${PCM_INGRESS_URL}`.
- Keep `ports:` for direct access.

### 4.7 Not in v1
- **Zero-downtime redeploys.** Traefik's health-checked load balancer helps, but `compose up` still recreates the container under the same name. Revisit together with automatic updates (Quadlet plus `podman auto-update`).
- **`attach`/`detach` hooks.** They wait for the first imperative proxy.
- **More proxies.** Caddy and kamal-proxy are each one definition plus hooks, with no core change.

---

## 5. Design: pluggable edges (internet exposure)

### 5.1 Options considered

| Mechanism | Own domains | Inbound ports | TLS | Role |
|---|---|---|---|---|
| Cloudflare Tunnel | ✓ (5 zones already there) | none (outbound, works behind CGNAT) | Cloudflare edge | **default public** |
| Tailscale serve, or own-domain A record to `100.x` | ✓ via Cloudflare DNS | none | DNS-01 wildcard via Traefik's Cloudflare resolver | **default private** |
| Direct public IP | ✓ | 80/443 forwarded on the gateway | Traefik Let's Encrypt (HTTP-01) | fallback (large uploads, video, non-HTTP) |
| Tailscale Funnel | ✗ (`*.ts.net` only) | none | Tailscale | demo links only |
| Pangolin / frp / rathole on a VPS | ✓ | none at home | on the VPS | if leaving Cloudflare |
| Vercel `vc dev --tunnel`, portless | ✗ (dev sessions) | — | — | not for services; steal portless's naming idea |

### 5.2 Configuration: `$PCM_CONFIG_HOME/edges.yml`
This is a map from domain to provider, so it lives in a file. The longest suffix wins.
```yaml
roteoh.com:     { provider: cloudflare-tunnel }
int.roteoh.com: { provider: tailnet }
localhost:      { provider: local }            # implicit default
example.org:    { provider: direct }
```

### 5.3 Edge contract
An edge provider is an executable in `lib/pcm/edges/<provider>` (packages can contribute more). It is run under fnox, so the Cloudflare API token comes from 1Password.

| Verb | Does |
|---|---|
| `add <host>` | creates DNS and, optionally, access policy. Idempotent |
| `remove <host>` | undoes `add`. Idempotent |
| `scheme <host>` | prints `https` or `http`, so pcm builds `PCM_INGRESS_URL` from the edge, not the proxy |
| `labels <host>` (optional) | extra proxy labels, e.g. `certresolver=cf-dns` for tailnet or direct TLS |
| `check` | prints problems (missing token, tunnel not running) for `pcm validate` |

### 5.4 `cloudflare-tunnel`
- **One tunnel per machine:** a `core/containers/cloudflared` definition (`provides: edge-cloudflare`) running a remotely managed tunnel from its token. It sits on `dev-net`, with a single catch-all ingress rule, `* → http://traefik:80`. Adding a host never changes the tunnel.
- **`add crm.roteoh.com`:**
  1. Find the zone by matching the host's suffix against the account's zones.
  2. Upsert a proxied CNAME `crm → <tunnel-id>.cfargotunnel.com`.
  3. With `--access <email>`, also create a Cloudflare Access app. That gives login for free, which ONCE deliberately lacks.
- **`scheme`:** `https`. The proxy router stays on `web` (HTTP), because Cloudflare terminates TLS.

### 5.5 CLI
```
pcm expose <service> <host> [--access <email>]   # edge add, then <NAME>_HOSTS += host, then pcm up
pcm unexpose <service> <host>                    # hosts -= host, pcm up, then edge remove
pcm exposures                                    # SERVICE HOST EDGE URL, for every ingress host
```
`expose` changes only `$PCM_ENV_HOME/<name>.env` and outside DNS. The definitions in git stay declarative.

---

## 6. Step 1 plan: pcm core ingress and proxy role

Step 1 covers core only. It is fully testable with bats against a fake proxy definition, so neither Traefik nor podman is needed.

### 6.1 New `lib/pcm/ingress.sh`
It is sourced automatically: the dispatcher loads every `*.sh`.
- `service_ingress <id>`: prints `service|port|healthcheck|hosts` from `.["x-pcm"].ingress` via yq. Fills in defaults: the only compose service, `/up`, and `<name>.$PCM_INGRESS_DOMAIN`. Prints nothing when there is no ingress.
- `ingress_hosts <id>`: the hosts expanded with `load_env` and `expand_vars` from `validate.sh`, comma-separated, trimmed.
- `role_providers <role>`: each id from `effective_ids` whose `.["x-pcm"].provides` is that role.
- `active_proxy`: `canon "$PCM_PROXY"` when that is set, and it must provide `proxy`. Otherwise the single provider. Otherwise it fails and the message lists the candidates.
- `ingress_labels <id> <compose-svc>`: when that compose service is the ingress target, runs the active proxy's `labels` hook through the same path as `run_hook`, with ingress args. Prints indented `key: "value"` YAML lines.

### 6.2 Changes to existing libs
- **`deps.sh`**
  - `service_deps`: also prints the active proxy's name when the service has ingress and isn't the proxy itself. This one change makes start order, `dev-net` attachment, down refusal and `service_dependents` work unchanged. If `active_proxy` fails, it prints nothing; validation reports the problem.
  - `dep_settings`: for the implicit proxy dependency, prints the ingress args instead of reading a depends_on map.
  - `provision`: when the dependency `provides` a role, uses the prefix `PCM_<ROLE>_`, so `proxy` becomes `PCM_INGRESS_` (one role-to-prefix map: `proxy→INGRESS`). Otherwise it keeps `PCM_<NAME>_`.
- **`runtime.sh` (`compose_flags`):** the label `printf` per compose service becomes a block that also adds `ingress_labels "$service" "$key"`. Add `io.pcm.ingress: "true"` there too, so `ps`/`show` can find ingress containers without knowing the proxy.
- **`validate.sh`:** a new `check_ingress`, called from `validate_services`.
  - **Errors:**
    - `ingress.port` is not an integer.
    - `ingress.service` is not in compose services, or is omitted when there are several.
    - No proxy, or an ambiguous proxy, when a service has ingress.
    - `PCM_PROXY` names a service that doesn't `provides: proxy`.
    - The active proxy lacks an executable `labels`.
    - `provides` is not a known role (`PCM_ROLES="proxy"`).
    - A host is not a valid hostname.
  - **Warning:** two services claim the same host, mirroring `check_ports`.
- **`inspect.sh` (`pcm show`):** an `Ingress:` line with hosts → `compose-svc:port`, and the active proxy.
- **`home/.local/bin/pcm`:**
  - `: "${PCM_INGRESS_DOMAIN:=localhost}"`. Like `PCM_PROXY`, it can be set in `anfs.conf`.
  - Help epilogue: an "Ingress" paragraph, plus the new validation errors.

### 6.3 Tests: a new `tests/ingress.bats`
Uses the `helper.bash` setup. The fixture `define`s a fake `proxy` with `x-pcm.provides: proxy` and a `labels` script that echoes its args back as labels.
1. A service with ingress depends on the single proxy. `service_deps` includes it, and the proxy doesn't depend on itself.
2. `PCM_PROXY` selects between two providers. With two providers and no `PCM_PROXY`, validation fails and names both.
3. An ingress service fails validation when there is no proxy.
4. Validation errors for a bad port, an unknown compose service, an ambiguous service, an unknown role, and a proxy without `labels`.
5. `compose_flags` writes the hook's labels plus `io.pcm.ingress` onto the ingress compose service only.
6. Host defaults: `<name>.localhost` with no hosts set. `PCM_INGRESS_DOMAIN` and `<NAME>_HOSTS` from `$PCM_ENV_HOME` override it (needs the existing varlock or schema-less paths).
7. `provision` output from a role provider lands as `PCM_INGRESS_URL`. A plain dependency still lands as `PCM_<NAME>_*`. This is a regression check for postgres and valkey.
8. Existing suites still pass: `bats packages/pcm/tests/`.

### 6.4 Steps after step 1
2. The `core/containers/traefik` definition, then convert Twenty (section 4.6).
3. Edges: `edges.yml`, then `edges/local` and `edges/cloudflare-tunnel`, then `core/containers/cloudflared`.
4. `pcm expose`, `unexpose` and `exposures`.

Verify end to end after step 2:
1. `pcm validate traefik twenty`
2. `pcm up twenty`. This starts postgres, valkey and traefik.
3. `curl -fsS http://twenty.localhost:8088/healthz`
4. `podman inspect` shows the labels on `twenty-server-1`.
5. `pcm down twenty`. The route is gone, and Traefik returns 404.
6. `pcm down traefik` refuses while twenty runs.

Repeat on macOS through the podman machine.

Step 3 (edges) is built and unit-tested; the live Cloudflare run is pending credentials:
- **`lib/pcm/edges.sh`:** `edges.yml` resolution (longest domain wins), the provider contract (`deps`, `scheme`, `check`, `add`, `remove`), publishing on `up` and unpublishing on `remove`, and validation. The design changed in two places:
  - Providers have a `deps` verb. The pcm service an edge runs (`cloudflared`) becomes an implicit dependency, like the proxy.
  - `add` runs **before** dependencies start, so the tunnel token exists when `cloudflared` starts.
- **`lib/pcm/edges/local` and `lib/pcm/edges/cloudflare-tunnel`:**
  - **Tunnel:** one per machine (`pcm-<hostname>`), with a single catch-all ingress rule pointing at `http://traefik:80`.
  - **Token:** written to `$PCM_ENV_HOME/cloudflared.env` (mode 600).
  - **DNS:** a proxied CNAME per host, never replacing a record that isn't this tunnel's.
  - **Credentials:** `CLOUDFLARE_API_TOKEN` comes through fnox.
- **`core/containers/cloudflared`:** provides `edge`, and its health check is `cloudflared tunnel ready`.
- **Network alias:** a single-service role provider is aliased by its name on `dev-net`. Verified live: another container reaches `http://traefik:80` and is routed by Host.
- **Hosts:** `ingress_hosts` now reads `$PCM_ENV_HOME/<name>.env` itself, so `<NAME>_HOSTS` is seen wherever hosts are needed.
- **Tests:** `tests/edges.bats` adds 13, against a fake Cloudflare API. All 65 pcm tests pass.

### 6.5 Decisions (2026-10-05)
1. `PCM_INGRESS_DOMAIN` defaults to `localhost`, so every ingress service gets `<name>.localhost` automatically.
2. Role-prefixed exports use the fixed map `proxy→INGRESS` (`PCM_INGRESS_URL`) for now.
3. The Traefik dashboard is local-only by default (`traefik.localhost`). Never `pcm expose` it without `--access`.

### 6.6 Status
Step 1 is implemented: `lib/pcm/ingress.sh`, plus changes to `deps.sh`, `runtime.sh`, `validate.sh`, `inspect.sh`, and the `pcm` help. `tests/ingress.bats` adds 15 tests, and all 45 pcm tests pass. `pcm validate` on the real definitions is unchanged: the only errors are the 4 privilege errors in `stack/dnsmasq` and `stack/netboot`, which were already there. 
Step 2 is built and partly verified:
- **Built:** `core/containers/traefik` (compose.yml, .env.schema, `labels`, `provision`, README), and Twenty converted (`x-pcm.ingress`, `TWENTY_HOSTS`, `TWENTY_SERVER_URL` defaulting to `PCM_INGRESS_URL`).
- **Verified live:**
  - Traefik is healthy, and its dashboard is routed through itself at `traefik.localhost:8088`.
  - A temporary whoami service was routed on two hosts.
  - `pcm down traefik` is refused while a dependent runs.
  - After `pcm down <service>`, its route returns 404.
  - Twenty got the right labels and `SERVER_URL=http://twenty.localhost:8088`.
- **Twenty, verified 2026-10-05 after resizing the machine:** a 2 GiB podman machine OOM-killed Twenty's server (server plus worker need about 1.3 GB). The new `PCM_MACHINE_MEMORY` setting (anfs.conf, MiB) makes `pcm up` resize a machine of another size, restarting it and naming the containers that stops. With it set to 4096 in `anfs.local.conf`, Twenty runs stably and `http://twenty.localhost:8088` serves the app through Traefik (router `pcm-twenty` enabled, server UP).
