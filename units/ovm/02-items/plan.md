---
---

# Plan 02 — `ovm items`, the source `pjm` pulls from (stub)

Stub, 2026-10-21. Not to be run until its open question is answered and the plan is written.

## Why

`user/pjm` (Personal Judge Manager) pulls work items from commands that print JSON; its one
source is `pvm items --json --space <level>` (after `ovm/01`, `ovm items ...`), and its config
relies on an `exclude` key in the vault's marker (`_generated` is excluded so pjm's own report is
not read back). Neither exists in pvm today, so `pjm pull` fails. pjm stays its own tool in the
user source (decided 2026-10-21); this plan gives it the source it expects.

## Open question

- **Lost or never written?** Was `items` (and `exclude`) a feature of pvm while it lived in the
  user source, lost when it moved to core (core 450241b, 2026-10-16), or was it only ever
  planned? If lost: recover it from the user source's history and port it to ovm. If never
  written: design it here.

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
