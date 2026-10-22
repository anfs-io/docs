---
objective: A bug in an anfs tool becomes a report in one command, filed where this machine says, so an agent that hits one never has to know where reports go
status: plan 01 written and built 2026-10-22 (job anfs-report)
---

An agent that hits a bug in core or stack today writes it up by hand, guesses at the repo, and
leaves out what a fixer needs (versions, which commit each source is on, what the log said).
`anfs report` (core) builds the issue in markdown with the diagnostics it gathers itself, and
files it where `ANFS_REPORT_TO` in `anfs.env` says: `github` (the shipped default: the repo that
owns the package, after a search for duplicates) or a markdown file (appended to, never
rewritten). `anfs report check` says whether this machine is set up to file one. A core skill,
`anfs-bug-report`, tells an agent to reproduce the bug minimally and run `anfs report`, and
nothing else: it never picks the destination and never calls `gh` itself. Plan 01 is all of it.
