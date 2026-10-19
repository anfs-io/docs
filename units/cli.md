---
objective: Every tool's command line reads the same — the same groups, and every list in one shape
status: pending
---

The tools grew one at a time, and their command lines show it. psm puts its main resource in a
`skills` group where every other tool has it at the top. wsm lists repositories with an option
(`list -r`) where psm has a `repo` group. Lists disagree on the first column (pcm's `NAME` and a
separate `SOURCE`, ppm's `PACKAGE` as `source/name`) and on the word for state (`STATUS`,
`STATE`, `NOTE`).

After this unit, a person or an agent who knows one tool's lists can read every tool's, and
`anfs <resource> list` (from `dev-source/02-work`) looks the same for every resource.

Depends on: nothing. Runs as its own job beside `dev-source/01-move`, once `anfs work` exists.

## Decisions

- **No aliases for the old forms.** As with pim, no backwards compatibility.
- **A list's JSON keys change with it.** CLAUDE.md says a renamed key breaks its readers; that is
  the cost, paid once, with every reader changed in the same commit series.

## Plans

1. `01-one-shape` — psm's groups, `wsm repo`, and the list convention in every tool
