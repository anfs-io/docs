---
---

# Plan 01 — `-i, --interactive` on every command

## Context — read these files first

- `units/setup.md` — "Decisions", the `-i` bullet
- core `packages/anfs/home/.local/lib/anfs/cli.sh` — the header ("DECLARING", "RECEIVING",
  "EVERY TOOL ALSO TAKES"), `_cli_opt` (where `-y` is supplied to every command), `_cli_parse`
  (the option loop and the argument count check), `cli_call`, the completion functions
  (`complete_<kind>`, `_cli_complete_names`), `anfs_ask`, `anfs_choose`, `CLI_EX_TERMINAL`
- core `packages/anfs/tests/registry.bats` — the demo tool, the `-y` test
- stack `packages/pikvm/home/.local/bin/pikvm` (`-i --identity`)

## Implementation

1. **`-i` and `--interactive` on every command**, supplied in `_cli_opt` as `-y` is
   (`CLI_OPT[interactive]`). `-i` with `-y`: usage error (64), "-i and -y: one asks, the other
   does not". `-i` with no terminal on stdin: exit 77 (`CLI_EX_TERMINAL`), "-i needs a terminal".
2. **What it asks**, in `_cli_parse` after the options are read and before the argument count is
   checked: each positional argument not given, in order, then each declared **value** option not
   given (flags are not asked), in declaration order. The question is the option's help (or the
   argument's kind), then its kind in brackets: `The 1Password vault for your GitHub token (VAULT):`.
   - When the tool has `complete_<kind>`, its candidates are shown first (up to 20, then "...");
     with one candidate it is the default.
   - Enter on an optional one leaves it out; Enter on a required argument asks again; a value
     option given an empty answer is left out.
   - A `...` argument asks once and splits the answer on spaces.
   - Answers go into `CLI_OPT` and `CLI_POS` exactly as if typed, so the command cannot tell; the
     count check then runs as usual.
   - A command that runs another with `cli_call` passes its own `-i` on only if the caller does.
3. `--passthrough` commands: `-i` is the tool's, never passed on (as `-y`).
4. Help: "Options, before the command" gains `-i, --interactive  Ask for what is not given (after
   any command)`. The header's "EVERY TOOL ALSO TAKES" gains it.
5. pikvm: `--identity` loses its `-i`; its callers and README.
6. Core `AGENTS.md`, next to the `-y` paragraph in "Options": `-i`, what it asks, 77, `-i -y` 64.

## Test Spec

`packages/anfs/tests/registry.bats`, on the demo tool, stdin from a pty or `script` where a
terminal is needed (or a test hook in cli.sh that treats a given fd as a terminal; say which):
- `demo up -i` asks for `thing` then the optional `[snapshot]`, then `--share` and `-p`; answers
  land in the same args and `CLI_OPT` as typing them; Enter leaves optional ones out
- a required argument answered empty is asked again
- candidates from a `complete_<kind>` are shown; one candidate is the default
- `-i` with no terminal: 77; `-i -y`: 64; `-i` given alongside some values asks only the rest
- `--interactive` the same as `-i`; `-i` is not passed on by a `--passthrough` command
- help lists `-i, --interactive`
- stack: pikvm's tests for `--identity`

## Verification

- [ ] `packages/anfs/tests/all`; stack's pikvm bats
- [ ] By hand at a terminal: `wsm repo install -i`, `pcm validate -i`, `anfs src add -i`
