# 01-interactive — log

2026-10-19. core 12796ee, stack 6cc52ba; merged into main the same day.

- cli.sh: `-i, --interactive` supplied by `_cli_opt` like `-y` (CLI_OPT[interactive]).
  `_cli_parse`, after the options: `-i -y` → 64; no terminal → 77; then `_cli_ask_missing`.
- Asks positional arguments first, then value options ("what" before "how"; the plan's
  Implementation said options first, its Test Spec arguments first; arguments first kept).
- Prompts on stderr: options `<help> (<KIND>):`, arguments as the usage line writes them
  (`<thing>:`, `[snapshot]:`, `<thing>...:`). Candidates from complete_<kind> listed (≤20, then
  "..."); a lone one is the default; dir/file list none.
- Enter: optional left out (and the arguments after it); required asked again; EOF → exit 1.
- CLI_OPT[interactive] stays visible, so a command forwards -i through cli_call explicitly.
- Tests: `ANFS_TEST_TERMINAL=1` makes stdin count as a terminal for -i. 5 new tests in
  registry.bats.
- pikvm: `--identity` only; `-i` on copy-id is now --interactive.
- AGENTS.md: -i paragraph after -y's; passthrough bullet; exit 77.
- tests/all 391/0 (pmm, pcm off); pikvm.bats 17/17 against the worktree lib.
- Not done: by-hand checks at a terminal (wsm repo install -i, pcm validate -i, anfs src add -i).
