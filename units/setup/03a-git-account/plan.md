---
---

# Plan 03a — `git account` and `ssh-key`: the first onboarding step as commands

The first slice of plan 03, end to end, before the runner (plan 02). Decided with the person
2026-10-19: start with the git account, agile.

## Context — read these files first

- `units/setup.md` — Decisions, the use-case table (the first two rows)
- stack `packages/git/install.sh` (`onboard_github*`, `_git_gh_authed`, `_git_gh_has_key`,
  `_git_fingerprint`, `_git_gh_env`) and `packages/ssh/install.sh` (`onboard_key*`,
  `_ssh_key_find`, `_ssh_fingerprint`): the logic to move into commands
- stack `packages/pikvm/home/.local/bin/pikvm` — a stack tool on core's `lib/anfs` (bash.sh,
  log.sh, cli.sh, table.sh): copy its head
- core `packages/anfs/home/.local/lib/anfs/cli.sh` — `cli_cmd`, `complete_<kind>`, `-i`, `-y`,
  `anfs_confirm`, `anfs_choose`, `anfs_table`; core AGENTS.md "Lists and --json", "Options"

## Implementation

### 1. `ssh-key` (stack `packages/ssh/home/.local/bin/ssh-key`)

- `ssh-key list` (`ls`): `KEY STATUS FILE` — each key this machine can use: `~/.ssh/*.pub`
  with its private key beside it (`local`), and the agent's (`ssh-add -L`, `agent`). KEY is the
  fingerprint (SHA256:...); `--json` keys `key status file type comment public`.
- `ssh-key add [--type ed25519]`: a key exists already (any row of `list`) → says which, exit 0,
  nothing done. Else creates `~/.ssh/id_ed25519` (mode 700 dir), comment `<user>@<host>`;
  ssh-keygen asks the passphrase at a terminal, none without one (as onboard_key does).
- `ssh-key public [key]`: prints a public key (the only one, or the one named by fingerprint).

### 2. `git account` (stack `packages/git/home/.local/bin/git-account`, so `git account ...`)

- Accounts are kept in git's own config, global: `account.<host>.user` (`git config --global
  account.github.com.user rjayroach`).
- `git account add [--host HOST] [--user USER]`: `-i` asks both; HOST completes `github.com`,
  `gitlab.com` (and hosts already configured). Then:
  1. records the account (already recorded with the same user: nothing to do there);
  2. reads `https://<host>/<user>.keys` (public, no sign-in; curl, a 10 s timeout). An unknown
     user (404) is an error naming the URL;
  3. one of `ssh-key list`'s keys is among them → "your key <fp> is on <host>", exit 0;
  4. no key here → runs `ssh-key add` (the -y/-i it was given passed on), then 5;
  5. a key here that is not on the account → github.com with `gh` signed in (`_git_gh_authed`,
     scope admin:public_key): `anfs_confirm`-free offer, `gh ssh-key add --title <host>`; else
     prints the public key and the settings URL (`https://github.com/settings/ssh/new`,
     `https://gitlab.com/-/user_settings/ssh_keys`) and exits 0 with "add it there, then run
     this again". Never signs in for the person (gh auth login opens a browser: say the command).
- `git account list` (`ls`): `ACCOUNT STATUS KEYS` — ACCOUNT `<host>/<user>`, STATUS `key on
  account` | `no key here` | `key not on account` | `unreachable`; `--json`.
- `git account remove <host>` (`rm`): forgets the account (asks; `-y`). Keys stay.
- Rerun of `add` with nothing new does nothing and exits 0; with `-i` the defaults are the
  recorded ones.

### 3. Packaging

- Both scripts are new files: `package.yml` versions bumped by the hook; say in the report that
  this Mac needs `ppm install stack/git stack/ssh` again (ppm links files one by one).
- git's package depends on ssh's (for `ssh-key`), if it does not already.
- The existing `onboard_*` steps in both install.sh files stay until plan 03 removes onboarding
  wholesale; do not change them.
- Each package's README (create a short one for git): the commands, one line each.

## Test Spec

stack `packages/ssh/tests/ssh-key.bats` and `packages/git/tests/account.bats`, temp HOME,
`ssh-keygen` real, `ssh-add` and `curl` and `gh` stubbed on PATH:
- ssh-key: list is empty, add creates one, add again does nothing, list shows it (`--json` keys),
  an agent key is listed; public prints it
- git account: add records `account.github.com.user`; the `.keys` stub holding our key → "on
  github.com"; not holding it and gh unauthenticated → prints the key and the settings URL; gh
  authenticated → `gh ssh-key add` called with the key; no key → ssh-key add runs first; 404 →
  error with the URL; list's statuses; remove asks (77 without -y) and forgets
- `-i`: `git account add -i` asks host then user (ANFS_TEST_TERMINAL=1, answers piped)

## Verification

- [ ] stack's ssh and git bats, against the job's core lib if needed (`ANFS_TEST_LIB`)
- [ ] core `packages/anfs/tests/all` (cli.sh unchanged, as a check)
- [ ] By hand (the person): `ppm install stack/git stack/ssh`, then `git account add -i`
