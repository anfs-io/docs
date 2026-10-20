---
objective: Onboarding is not a mechanism of its own — any manifest (a stack, a space) lists ordinary toolkit commands to run, every command can ask for what it needs with -i, and the same list run backwards is offboarding
status: plan 01 written
---

Replaces `anfs onboard` (`lib/anfs/onboard.sh`, `onboard:` in `package.yml`, `ppm __onboard`,
`lib/ppm/onboard.sh`). Decided with the person, 2026-10-19.

## Why

Onboarding today is a one-off: it runs once, on a fresh machine, through code no other command
uses. But the same questions come back at other levels. A new space may be a department, or prod
beside test, and needs its own `opm add` (a 1Password account or vault), its own key, its own
identity. A person onboards to anfs, then to each space, and later to whatever else. So the steps
are ordinary commands, and a manifest at each level says which to run.

## Decisions

- **`-i, --interactive`**, on every command (cli.sh, like `-y`): ask for each declared value option
  and each positional argument not given, with the option's help as the question and its kind's
  completion (`complete_<kind>`) as the choices. Without `-i` nothing asks: a missing value is a
  usage error, as now. `-i` with no terminal exits 77; `-i` with `-y` is a usage error. The
  conventional pair (`rm -i`, `git rebase -i`; `apt -y`): `-y` asks nothing, `-i` asks for
  what is missing. pikvm's `-i --identity` keeps only `--identity`; pmm keeps `-p --publish`.
- **`setup:`** in a manifest: a list of toolkit commands, each an argument list. Run in order
  by one runner in `lib/anfs`. `anfs install system -i` runs system's; `wsm install <space> -i`
  a space's (open question 1).
- **Only the toolkit's commands.** The first word must be one of anfs's tools (the toolkit API's
  rule: resolved in `ANFS_BIN_DIR`, never `PATH`; an argument list, never a shell). A `space.yml`
  comes from someone's repository; cloning an org's space must not mean running its code.
- **An `add` already done does nothing and exits 0.** That keeps onboarding's best property: a
  rerun does only what is missing. `--check` on the runner runs each entry's detect alone.
- **No facts file.** A command reads what an earlier one left on the machine (`git config
  user.name`, the key on disk), never a value handed along by the runner.
- **Offboarding** is the record of what a run added, removed last first, each through its tool's
  `remove`; as `anfs remove <stack>` does from a stack's record.
- **The dashboard's forms** are a command's declarations: one field per option, its help as the
  label, its kind's completion as the choices. No wizard code (plan 05).
- `anfs onboard` and everything under "Replaces" go; no alias.

## Open questions

1. **When does a space's `setup:` run?** Proposed: every `wsm install` of it, with `-y`
   semantics (what can run without asking does); `-i` asks for the rest.
2. **Whose is a space's setup: the space's, or a stack the space names?** Proposed: a space names a
   stack (`stack: lgat/prod-tools` in `space.yml`), so one mechanism carries records and removal,
   and offboarding from a space is `anfs remove` of its stack.

## Plans

1. `01-interactive` — `-i` in cli.sh; pikvm's `-i` goes
2. `02-setup` — the `setup:` key in `stack.yml`, the runner, the record and its reverse;
   `anfs install system -i`
3. `03-steps` — the onboard steps become `add` commands (stack's git, ssh, op; anfs's `repo` and
   `packages`); `anfs onboard` and the onboard contract go; `install.sh` ends with
   `anfs install system -i` when it has a terminal
4. `04-spaces` — after the open questions: a space's setup
5. `05-forms` — `<tool> __describe <command> --json` (the declarations); the dashboard renders a
   command as a form, and a stack's setup as a wizard
