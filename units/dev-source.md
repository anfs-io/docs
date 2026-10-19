---
objective: core holds the toolkit and nothing else — what is only for developing and testing it lives in its own source, `anfs-dev`
status: pending
---

core is the first thing every user clones, and its resources are every user's: `pcm list` shows
two test boxes, `pmm list` a macOS test machine, `pmm image list` two test images. None of that
is the toolkit. After this unit core's only container is the one it ships (anfs-ui, `toolkit-api`),
its images are the base images, and the dev tooling is a source a contributor adds.

The source is `anfs-dev` (`git@github.com:anfs-io/dev-anfs`), listed in `user.list`, not in
`system.list`. It exists and is empty.

Depends on: the `machine-images` branch merged to main in core and stack. That branch moves the
images under `machines/` and rewrites the tests; splitting before it lands means conflicts.

## What moves, what stays

| Moves to `anfs-dev` | Stays in core |
| --- | --- |
| `packages/dev`: `ppm hooks`, `ppm container`, `ppm vm`, `ppm move`, `ppm user`, the `pre-commit` hook | Each tool's bats suite, in `packages/<tool>/tests/` |
| `containers/anfs-test-debian`, `containers/anfs-test-fedora` | `packages/pmm/tests/e2e` (and any tool's own e2e) |
| `machines/anfs-test-macos` | The base images `machines/images/debian-13`, `fedora-44` |
| `machines/images/anfs-test-macos`, `machines/images/anfs-dev` | `isos/`, `skills/`, `install.sh` |
| `packages/anfs/tests/{harness,roundtrip,roundtrip-macos}` and `fixtures/acme` | |
| `spaces/space.yml`'s `anfs-io/docs` (the plans and logs: the history of core and stack) | |

## Decisions

- **Unit tests stay beside the code.** They are not stowed, so they never reach a user's `$HOME`;
  they run from a checkout against the libraries in that same checkout; and a change to a tool
  and to its tests is one commit. In another repo they would need to be told which core to test
  (the installed one is the wrong one on a branch), every change would be two branches kept in
  step, and `packages/pmm/` in the dev source would be an empty layer of `core/pmm` to ppm.
- **A tool's own e2e stays with it** for the same reason: it changes with the tool's commands.
- **The round trips move.** They already span sources (`install.sh`, stack, the `acme` fixture)
  and they need the test boxes, which move.
- **The dev source is not a default.** A contributor runs `anfs src add
  git@github.com:anfs-io/dev-anfs anfs-dev` and `ppm install anfs-dev/dev`.
- **Nothing in core may need the dev source**: not `install.sh`, not `core/anfs`'s `depends:`,
  not a bats suite. The dependency is one way.
- **`dev-anfs` is public**: it is what a contributor or a fork pulls to run the tests and read
  the plans. The docs stay their own repo (`anfs-io/docs`); the dev source's `spaces/space.yml`
  is what clones it, so wsm only clones and never links a file of it. core has no `spaces/`.
  `anfs-io/infra` and `anfs-io/website`, which core's `space.yml` also lists, go to the private
  `admin-anfs` source's.
- **No history is carried over.** The first commit in `anfs-dev` names the core commit the files
  came from; `git log` in core still has the rest.

## Worktrees (plan 02)

Parallel jobs each need their own checkout of every repo they touch, on one branch name:

```
~/work/<job>/            one job, one branch name in every repo it touches
  core/  stack/  anfs-dev/           git worktrees of the clones under sources/, named by alias
  docs/                              a worktree of the docs clone (a repository of a space, not a source)
```

- The shape is what the round trips already take as `ROUNDTRIP_SRC_ROOT`: a directory of
  checkouts named by alias. A source the job does not touch falls through to the installed one.
  `~/anfs-dev` (one agent's leftover) is replaced by this, and so is the default.
- `anfs work new|list|path|remove <job> [alias...]` in the `dev` package makes and removes them;
  `remove` refuses a branch that is not merged. A repository that is not a source (docs) is
  found through the `space.yml` that clones it.
- **Jobs running at once share the host.** Each needs its own socket directory and ssh port
  range (`PMM_SOCK_DIR`, `PMM_SSH_PORT_BASE`), derived from the job; the ISO store is shared and
  read-only to tests; tart allows two macOS guests in all.

## Plans

1. `01-move` (the package and the docs space moved by hand 2026-10-17; the rest runs second, as
   a job under `anfs work`) — the move, every path that named the old places, the docs space, the docs
2. `02-work` (runs first, in `anfs-dev` and core) — commands from packages (`anfs <name>` runs `anfs-<name>`; anfs only), `anfs hooks`,
   the tools as `anfs <resource>`, `~/work/<job>/`, `anfs work`, and what lets two jobs test at once
3. `03-boxes` — `anfs work install|shell|reset` replace `ppm container`, `ppm vm` and `ppm user`
   (after `container-images/01`)

## Completion Criteria

- On a machine with only the default sources, `pcm list`, `pmm list` and `pmm image list` show
  nothing whose name starts with `anfs-test` or `anfs-dev`, and `ppm list` has no `dev`.
- `grep -rn 'anfs-test\|core/dev' .` in core finds nothing outside history notes.
- With `anfs-dev` added: `ppm install anfs-dev/dev`, `ppm container up debian`, and the Debian
  round trip pass, testing the working trees of core, stack and `anfs-dev`.
- Every bats suite in core passes on a checkout with no `anfs-dev` beside it.
