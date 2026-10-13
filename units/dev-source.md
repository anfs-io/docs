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
- **No history is carried over.** The first commit in `anfs-dev` names the core commit the files
  came from; `git log` in core still has the rest.

## Plans

1. `01-move` — the move, every path that named the old places, the docs

## Completion Criteria

- On a machine with only the default sources, `pcm list`, `pmm list` and `pmm image list` show
  nothing whose name starts with `anfs-test` or `anfs-dev`, and `ppm list` has no `dev`.
- `grep -rn 'anfs-test\|core/dev' .` in core finds nothing outside history notes.
- With `anfs-dev` added: `ppm install anfs-dev/dev`, `ppm container up debian`, and the Debian
  round trip pass, testing the working trees of core, stack and `anfs-dev`.
- Every bats suite in core passes on a checkout with no `anfs-dev` beside it.
