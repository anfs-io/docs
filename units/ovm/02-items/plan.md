---
---

# Plan 02 — `ovm items`, the source `pjm` pulls from (stub)

Stub, 2026-10-21; its question answered 2026-10-22 (never written). To be designed and written.

## Why

`user/pjm` (Personal Judge Manager) pulls work items from commands that print JSON; its one
source is `pvm items --json --space <level>` (after `ovm/01`, `ovm items ...`), and its config
relies on an `exclude` key in the vault's marker (`_generated` is excluded so pjm's own report is
not read back). Neither exists in pvm today, so `pjm pull` fails. pjm stays its own tool in the
user source (decided 2026-10-21); this plan gives it the source it expects.

## Answered (2026-10-22): never written

`items` and `exclude` never existed in pvm. Checked: the user source's `packages/pvm` history
(`8199ac3` first pass, `4294372` move to core) has neither; core's history has no `cmd_items` and no
`exclude:` for pvm; no repo or space on this machine has an implementation. `pjm` arrived as a
spike in the same commit that moved pvm to core (user `8a37a76`), already calling `pvm items`, so
it was written against a command still to be designed. This plan designs it.

## To settle when the plan is written

- The item shape pjm reads (`tests/helper.bash` and `pjm.bats` fixtures in `user/pjm` are the
  contract today: which keys, `vault` among them).
- What an item is in a vault (a task line? a heading? frontmatter?), and how `--space <level>`
  maps to the vault directories linked into that level (`ovm link list`).
- `exclude:` in `ovm.yml`: paths in the vault that `items` (and `docs`?) skip.

## Context

- `units/ovm.md`; plan 01 and its `log.md`
- user `packages/pjm/` — `home/.local/bin/pjm` (`cmd_pull`), `home/.config/pjm/config.yml`,
  `tests/`
- user source git history for `packages/pvm` before 2026-10-16
