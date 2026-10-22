---
unit: pcm/05-remove-when-down
job: pcm-remove-down
date: 2026-10-22
report: r-20261022-5
---

What: `pcm remove` and `pcm down` skip compose down when the service's project has no containers; pcm removes what is left itself (pod_<name> if it exists, the project's networks, and its volumes for remove or `down -v`), then remove goes on as before (deprovision, data, override, state).
Why: podman-compose 1.6 down stops and removes every container the compose file defines and its pod whether they exist or not, so an already-down service printed `Error: no container with name or ID` per container and `Error: unable to find pod`.
How: `project_objects_rm` in runtime.sh, shared with implode (which now removes the pod too); a failing podman is an error (remove keeps the data), not "nothing there". `reset` was already guarded.

# Log: pcm/05-remove-when-down

## Tests

- tests/run.bats: a compose runner that, like podman-compose, prints podman's errors for down
  with no containers. Remove and down of an already-down service: no compose call, no `Error:`,
  pod/networks/volumes removed only when they exist; a running service still gets compose down;
  `pod exists` failing (125) keeps the data. They fail without the fix.
- tests/helper.bash: the no-engine stub answers `network ls` / `volume ls` with nothing.
- pcm's bats, each with `ANFS_TESTS_PMM_PCM=1`: all pass but the known manifest.bats "machine
  resize" and toolkit.bats "macOS: compose runs with the grant's token" failures.
- `packages/anfs/tests/all`: 662 passed, 0 failed (pmm and pcm off).

## By hand, after merge

Reinstall core/pcm (0.3.2), then with a service that has a deprovision dependency (authentik):

    pcm down authentik && pcm down authentik && pcm remove -y authentik
    podman pod ps; podman volume ls --filter label=com.docker.compose.project=authentik

The second down says "is not running"; neither it nor remove prints an `Error:` line; the postgres
deprovision still runs; no pod_authentik and no authentik volumes are left. Close r-20261022-5 in
BACKLOG.md.
