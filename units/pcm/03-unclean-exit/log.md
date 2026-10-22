---
unit: pcm/03-unclean-exit
job: pcm-status
date: 2026-10-22
report: r-20261022-1
---

What: `pcm show` prints `exited (unclean)` for an exited container whose finish time podman never set.
Why: podman builds its Status from the zero FinishedAt, so pcm passed through "Exited (0) 292 years ago".
How: show's containers rows carry State and ExitedAt and are rewritten in pcm; everything else is podman's Status.

# Log: pcm/03-unclean-exit

## The real shape

Read-only `podman ps -a --format json` on this Mac (podman 6.1.1): the ps equivalent of
`.State.FinishedAt` is `ExitedAt`, Unix seconds; the zero time is `-62135596800`. Running
containers have that same value (`ui_ui_1`: running, `-62135596800`), so the zero time alone is not
enough: the rule is `State == exited` and `ExitedAt <= 0`. A created container is unchanged too.

## What was built

core, pcm 0.2.1:

- `inspect.sh`: `container_rows`, an awk filter from `name state exited-at status ports` to
  `name status ports`; `cmd_show`'s containers section asks podman for those five fields and pipes
  them through it before `column`.
- `tests/run.bats`: the stub prints `SHOWN_ROWS` for the show format; one test with a zero-time
  exited container (`exited (unclean)`, no "292 years"), a normal exited one (`Exited (143) 4 weeks
  ago` and its ports, unchanged) and a running one with the zero ExitedAt (`Up 2 hours`, unchanged).

## Where pcm prints status, and what changed

- `pcm show`: the containers rows. Fixed. Its `status` line is pcm's own (`running`, `stopped`,
  `not created`), never podman's string: unchanged.
- `pcm list`: STATUS is pcm's `running`/`stopped`/`shadowed`, from State: unchanged.
- `pcm ps`: `exec podman ps` with pcm's label filter; pcm does not format its rows, so podman's
  output is podman's and still says "292 years ago". Left alone.
- `pcm_containers` still carries podman's Status as its fifth field; no caller prints it.

## Tests

- pcm's bats, each file with `ANFS_TESTS_PMM_PCM=1`: all pass but manifest.bats "machine resize:
  running containers and no terminal" and toolkit.bats "macOS: compose runs with the grant's token",
  which fail the same with the change stashed (not this unit's).
- `packages/anfs/tests/all`: 662 passed, 0 failed (pmm and pcm off).
- `container_rows` on the real authentik rows (read-only `podman ps`): both `exited (unclean)`.

## By hand, after merge

Reinstall core/pcm, then:

    pcm show authentik

The containers section shows `authentik_server_1  exited (unclean)  0.0.0.0:9000->9000/tcp, ...`
and `authentik_worker_1  exited (unclean)`. Close r-20261022-1 in BACKLOG.md (it is not on this
job's docs branch, which predates the report).
