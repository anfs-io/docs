# 00-ui-in-core — log

2026-10-19, job `ui`, core branch `ui` (not pushed).

## What moved

The anfs-ui spike (`~/spaces/user/dev/apps/anfs-ui`, copied with rsync, left untouched) is now
core's `containers/ui/`: app.rb, event_stream.rb, config.ru, lib/, views/, public/, spec/,
config/, Gemfile(.lock), .rspec, .rubocop.yml, .gitignore, README.md. First commit is the copy as
it was (plus `container.yml`, which the pre-commit hook requires).

## The container files

- `Containerfile`: `dev` target FROM `docker.io/library/ruby:4.0.7-slim` (host Ruby is 4.0.7;
  the tag exists on Docker Hub), build-essential for the native gems, `COPY Gemfile Gemfile.lock`,
  `bundle install` with `BUNDLE_FROZEN=true`, no source; `release` FROM dev, `COPY . /app`,
  `RACK_ENV=production`. Workdir /app, puma `--pidfile /tmp/puma.pid`, gems in /usr/local/bundle.
  `.containerignore` drops .rspec_status and tmp/.
- `Gemfile.lock`: `aarch64-linux` and `x86_64-linux` added to PLATFORMS, so the read-only mounted
  lockfile needs no rewrite in the container.
- `compose.yml`: service `ui`, `build: {context: ., target: dev}`, image `localhost/anfs-ui:dev`,
  `127.0.0.1:9292:9292`, `.:/app:ro`, `RACK_ENV=development`, `HOST=0.0.0.0`. No variables, so no
  `.env.schema`.
- `container.yml`: version 0.1.0, description, upstream (the ruby image); no url, no memory.
- Reloading: `rack-unreloader` (development group), wired in `config.ru` only when
  `RACK_ENV=development`, tracking `views/` only. `app.rb` and `lib/` create the store, the hub
  and the poller threads at load, which a reload would orphan: for those, `pcm restart ui`.
  Checked on the host (copy in a scratch dir, puma in development): an edit to
  `views/overview.rb` showed on the next request with no restart.

## The no-tools state

`Anfs::CLI#run` checks the tool is executable before running it; when it is not, the result is
`"<tool>: the tools are not reachable from here (no toolkit API yet)"`, which every panel and the
overview show as a failure. The log follower already tolerated a missing `anfs.log` (retries every
5s). Checked by hand: puma with an empty `ANFS_BIN` and a fresh `XDG_STATE_HOME` served `/`,
`/packages`, `/containers`, `/machines`, `/jobs`, `/up` with 200. New spec: every page serves 200
and shows the message with an empty bin dir; the cli spec for a missing tool updated.

## Stack and docs

- `stacks/system/stack.yml`: `containers: [core/ui]`, no `status:`. system.bats holds (8/8),
  extended: `anfs show core/system` lists core/ui, implode runs `pcm remove -y core/ui`.
- **Note:** `pcm install` is `pcm up` (`cmd_install` in `lib/pcm/sources.sh`), and a stack with no
  `status:` installs with the tool's default. So `anfs install system -y`, the last step of
  `install.sh`, will build the image and start ui; "installed, not started" is not what pcm does
  for a container. Decide before pushing: keep it, or give pcm/stacks a way to install without up.
- sources.bats "core has no containers/" became "core's containers/ is ui only".
- AGENTS.md: layout (containers/ui), the system stack, a pcm line, the stack example's
  `core/anfs-ui` renamed `core/ui`. README: Run (`pcm up ui`, or `bundle exec puma` on the host),
  the two targets, the reload, the tools not reachable until the toolkit API; points to
  `units/toolkit-api.md`.

## pcm validate / show (isolated HOME, worktree core as source `core`, podman stubbed)

```
$ pcm validate ui
pcm: valid: core/ui
exit 0

$ pcm show ui
core/ui  /Users/roberto/work/ui/core/containers/ui
  version      0.1.0
  description  anfs-ui, the toolkit's dashboard (Roda, Phlex, Puma)
  upstream     docker.io/library/ruby:4.0.7-slim
  image        built from ./Containerfile
  status       not created
  depends on   —
  used by      —
  data         <tmp>/data/volumes/ui  (none yet)

declared (not created)
  ui  port   127.0.0.1:9292:9292
  ui  mount  . -> /app (ro)
exit 0
```

## Tests

- App suite on the host: rspec 40 examples, 0 failures; rubocop 24 files, no offenses.
- `anfs work run ui core -- packages/anfs/tests/all`: 338 passed, 0 failed (pmm and pcm off).

## Not run

No podman build, no `pcm up`, no VMs; pcm's and pmm's bats suites off.

## What the person checks

1. `pcm up ui` (builds the dev target), open http://localhost:9292: pages load, each panel says
   the tools are not reachable.
2. Edit a view on the host (say a heading in `views/overview.rb`), reload: the change shows.
3. Optionally `podman build --target release containers/ui`.
