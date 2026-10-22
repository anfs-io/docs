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

## Use cases (the person's, 2026-10-19), and the command each becomes

**`anfs install system -i`**: the toolkit's own setup, in this order:

| Question | Command | Notes |
| --- | --- | --- |
| Your GitHub / GitLab / other account? | `git account add -i` (`--host`, `--user`) | checks `<host>/<user>.keys` for public keys |
| Your public key | `ssh key add -i` | "I don't know" → a new key pair. Keys at the host but no private key here → a new key (or one from the password manager) |
| A password manager? | `secrets add -i` (a provider: 1Password, Bitwarden, KeePassXC, age, the OS keychain) | "I don't know / none" → a no-subscription one: age or KeePassXC (local), Vaultwarden (remote: a pcm container) |
| Your org's space? Its name? | `wsm new -i` | `wsm add` is taken (moves files into a source) |
| AI agents and their API keys | `ppm install -i @ai-agents` | a multi-select of the category; what is installed is preselected; API keys go to the password manager |
| Update anfs automatically? | later | a timer (launchd / systemd) running `anfs src update` |
| Register for updates (email)? | later | sends data off the machine: opt-in, needs a service |

The password manager comes before the key when the key is kept in it (today's op step does that).

**`wsm install <space> -i`**: onboarding to a space. A space has only `space.yml` (its
repositories) today; after its setup it also has the space's `fnox.toml`: the secrets it needs (a
1Password service account token for that space, for one) and where each comes from.

**Rerun.** Every question's default is what the machine has now: the account found, the key
present, the agents installed (preselected). So a second run asks the same things and changes
only what you change.

**Web vs terminal.** The dashboard is the better experience (plan 05); the terminal stays for a
machine with no browser, and is what the dashboard's forms are made from.

## What -i needs beyond plan 01

- **A current value.** `complete_<kind>` may mark a candidate as what the machine has now; `-i`
  makes it the default (a value) or preselects it (a `...` argument).
- **Multi-select** for a `...` argument with candidates: the candidates as a checklist, the
  current ones ticked.
- **A category as the selector**: `ppm install -i @ai-agents` offers the category's members.
  (Proposed instead of a package with `installable: no` and an onboarding script, which would
  bring back the contract this unit removes.)
- Split `ai` into `ai` (skills and tools) and `ai-agents` (the agents).

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

## Input to the space-setup rewrite

- r-20261022-4 (BACKLOG.md, from partitia/cnfs, 2026-10-22): a space's setup gives it a
  space-root `fnox.toml` whose 1Password provider authenticates with the space's
  `OP_SERVICE_ACCOUNT_TOKEN` (one vault per space), under a stable or discoverable provider name;
  app dirs below inherit it (their own `fnox.toml` holds only their keys); fnox's shell
  activation on cd; no unlock prompt when the token is set. The person asked that it be covered.
