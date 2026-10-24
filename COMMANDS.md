# Commands: what to run, and when

A task-oriented guide to the anfs command line. Each section starts from something you want to do, says when to reach for it, gives the few commands that do it, and says what you should see. The [reference](#reference) at the end lists every command, tool by tool.

Every command and option here is from the tools' own `--help`. When in doubt, ask the tool: `<tool> --help`, or `<tool> <command> --help`.

## The tools at a glance

| Tool | Also | What it manages |
| --- | --- | --- |
| `anfs` | | Sources (the git repos every tool reads), stacks, onboarding, events, bug reports, settings, the log |
| `ppm` | `anfs package` | Packages: software and dotfiles |
| `pcm` | `anfs container` | Containers: podman compose services |
| `pmm` | `anfs machine` | Virtual machines, their images and networks |
| `psm` | `anfs skill` | Agent skills, synced to every installed agent |
| `wsm` | `anfs space` | Spaces: where agents and people work; leases and listeners |
| `anfs bus` | | NATS and anfsd, the daemon that ships events and dispatches agents |
| `anfs agents` | | Dispatched agent runs and the rules that start them |
| `anfs inbox` | | What waits on you: approvals and unhandled work |
| `anfs work` | | Jobs: git worktrees on a branch, where agents (and you) work in isolation |

`anfs bus`, `anfs agents` and `anfs inbox` come with the `core/anfsd` package. `anfs work` and `anfs hooks` come with the `dev` package of the `anfs-dev` source (tooling for working on anfs itself). `anfs` lists the ones installed under "From packages".

### Options every tool takes

| Option | Does |
| --- | --- |
| `-y`, `--yes` | Do not ask |
| `-i`, `--interactive` | Ask for what is not given |
| `--json` | Lists as JSON, anywhere on the line |
| `-d`, `--debug` | Debug output (it is always written to the log: `anfs log`) |
| `--trace` | Trace every command the tool runs |
| `--version` | Print the version |
| `-h`, `--help` | Help; after a command, that command's |

Most tools also have `config <list|get|set|unset>` (settings in `~/.config/<tool>/`: `<tool>.local.env` over the shipped `<tool>.env`, the environment over both), `completion [shell]` and `help [command...]`.

### Names: `name` or `source/name`

A package, service, machine, image or skill is named `name` or `source/name`. A plain name resolves to the highest-priority source that has it; `source/name` picks one. `anfs src list` shows the sources in priority order. A space is named by its id (`anfs-dev`, `lgat/tech`); a prefix ending in `/` (`lgat/`) means every space under it.

## Set up a machine

**When:** a new laptop or server, or a machine you want to make reproducible from your own repo.

Install the toolkit (as your normal user, not root), open a new shell, then make it yours:

```bash
curl -fsSL https://raw.githubusercontent.com/anfs-io/system/refs/heads/main/install.sh | bash
anfs onboard              # packages to install, your git identity, an ssh key, GitHub, your own repo
anfs src list             # the sources, in priority order, and what each provides
anfs ls                   # the stacks every source offers, and which are installed
```

On Debian use `wget -qO- <url> | bash`. The installer puts `anfs` and `ppm` on your PATH and installs the `system` stack (`pcm`, `psm`, `wsm` and what they run on).

**Add an organization's source** and install what it holds:

```bash
anfs src add git@github.com:lgat/lgat-anfs.git lgat
anfs src update lgat
anfs src install lgat     # its packages, containers, machines, skills and spaces, in that order
anfs install lgat/rails   # or one stack: the resources its stack.yml declares
```

**Turn on the bus** (events on NATS, agent dispatch, the inbox):

```bash
ppm install core/nats core/anfsd
anfs bus status
```

**What you should see:** `anfs onboard --check` lists each step and exits 0 when nothing is left to do. `anfs src list` shows one row per source: alias, status (`clean`, `dirty`, `unpushed`), URL, list (`user` or `system`) and what it provides (`packages containers machines skills spaces stacks`). `anfs bus status` shows every row `ok`.

`anfs onboard` takes `--repo REPO` (`local`, `new`, or a git URL) for your own repo, the `user` source, which goes to the top of the list so your files override shared ones. A second machine is then one command: pass `--repo <url>` to the installer.

## Install and update packages

**When:** you want a tool and its configuration (`ppm install nvim` brings neovim and its config together), or you want the latest from your sources.

```bash
ppm list                  # every package available, and the installed version
ppm list @                # the categories; ppm list @ai: the packages in one
ppm show nvim             # layers, dependencies, what is installed
ppm install -n nvim       # dry run: what would be linked, kept or in the way
ppm install zsh tmux nvim git
```

**Update:** pull your sources, then install again what you want refreshed.

```bash
anfs src update           # pulls every source; skips one with uncommitted changes
ppm install nvim          # re-applies the package (declared software, hooks, links)
ppm install core/ppm      # how ppm updates itself; anfs install system for the whole toolkit
```

Package files are symlinks into the source checkouts, so most edits arrive with `anfs src update` alone. `ppm install` also pulls sources older than `ANFS_UPDATE_CACHE_DURATION` (24 hours by default).

**Remove:** `ppm remove <package>` takes the package and the software ppm installed for it.

**Own a file:** `ppm file claim <file>` copies a stowed file into your repo and links it from there; `ppm file protect <file>` detaches it so ppm leaves your copy alone.

**What you should see:** `ppm install` names each package and what it linked; something in the way stops it (`-f` replaces conflicting files). `ppm list --installed` shows what is on this machine.

## Run services and machines

**When:** a project needs Postgres, a dashboard, or a disposable Linux VM.

Containers, with `pcm`:

```bash
pcm ls                    # CONTAINER STATUS VERSION
pcm up postgres           # start (detached); dependencies started and provisioned first
pcm logs -f postgres
pcm shell postgres        # a login shell in its container
pcm down postgres
```

`pcm provision postgres --as myorg/shop database=shop` prints connection settings (`URL=...`) for an app that runs on the host, not in pcm. `pcm snapshot` and `pcm reset` save and restore a service with its data. `pcm remove` deletes everything pcm created for it but keeps the definition.

Machines, with `pmm`:

```bash
pmm image build debian-13
pmm run debian-13         # a machine from the image, started
pmm shell debian-13
pmm ls                    # NAME SOURCE IMAGE STATE ADDRESS
pmm down debian-13
```

pmm is not part of the `system` stack: `ppm install core/pmm` first.

**What you should see:** `pcm ls` shows `running` for started services. `pmm ls` shows each machine's state and, when running, its ssh address (`127.0.0.1:2223`).

## Work in a space

**When:** you start work for an organization or project. A space is a directory with a `space.yml`: its files are linked from a source into `~/spaces/<id>/`, its repositories are cloned there, and its skills installed for it. It is where you and agents work on the same things.

```bash
wsm ls                    # SPACE STATUS DIR NOTE
wsm install lgat/tech     # link it, clone its repositories, install its skills (lgat/ for every space under lgat)
wsm cd tech               # go there (an installed id, or the end of one)
wsm repo ls               # its repositories and their git state
wsm id                    # the space's id, which the event log is filed under
```

**Create one:** `wsm new <source>/<path>` makes a space in a source. `wsm add <file>` moves a file you created in the spaces tree into the source and links it back, so it is committed with the space.

**Push your work:** `wsm repo push <spec>` pushes the clones under a space that are ahead (`--all` for every one). Clones are never pulled or deleted by wsm: what is checked out is yours.

**What you should see:** `wsm ls` shows `ok` for a healthy space. `partial` comes with a NOTE that counts what is `missing`, `wrong`, `in the way` or `leftover`, or says `coordinator outdated` (run `wsm install <space>` again). `no id` means the space has no `id:` yet: `wsm id --new <space>` writes one for you to commit.

`wsm cd` and `ppm cd` need the shell function their packages install; `wsm path` prints the directory instead.

## Run agents on a job

**When:** you want an agent (or yourself) to work on a change without touching the live checkouts, or you want to see what anfsd dispatched.

A **job** is `~/work/<job>/`: a git worktree per repo, all on the branch `<job>`.

```bash
anfs work new fix-pcm-logs core            # worktree of core on a new branch fix-pcm-logs
anfs work ls                               # JOB BRANCH REPOS STATUS NUMBER
anfs work run fix-pcm-logs core -- packages/anfs/tests/all   # run with the job's isolation variables
ANFS_WORK=fix-pcm-logs pcm ls              # try the job's copy of a tool on live data
anfs work rm fix-pcm-logs                  # once merged: worktrees, merged branches, the directory
```

`anfs work rm` refuses a dirty worktree or a branch with commits not in main.

**Dispatched runs.** A space's `space.yml` can say which event starts which agent (`agents:`, `dispatch:`). anfsd then starts the agent CLI in a job of its own and supervises it:

```bash
anfs agents rules                          # RULE STATUS SPACE ON AS HARNESS
anfs agents match --type report.filed --space anfs-dev --data package=core/pcm   # what would happen; starts nothing
anfs agents ls                             # RUN STATUS RULE ROLE SPACE HARNESS JOB AGE
anfs agents logs -f <run>                  # its output as text
anfs agents stop <run>
```

**What you should see:** in `anfs agents ls`, a run goes `submitted`, `working`, then `completed`, `failed` or `canceled`. A run that lands its work shows its step, like `working (verify)` or `auth-required (merge)`: it waits on you in the inbox. `anfs agents ls --unhandled` lists events a rule turned away for its queue or rate.

## Report a bug and follow it

**When:** an anfs tool (anfs, ppm, pcm, pmm, psm, wsm, or a stack tool) does something wrong. Not for bugs in your own project. Agents do this through the `anfs-bug-report` skill.

```bash
anfs report check                          # can this machine file one? one ok/fail row per item
anfs report --package core/pcm --title "pcm up hangs without its network" \
  --expected "an error naming the network" --actual "hangs; exit 130" --steps "pcm up x"
anfs report status r-20261024-1            # open, fixed (not here yet) or available
anfs report wait r-20261024-1              # returns when the fix is on this machine
```

The report goes where `ANFS_REPORT_TO` says: a GitHub issue on the owning repo, or a file of your own (a `BACKLOG.md`). It carries diagnostics (versions, each source's commit, the log's failures). `-n` prints it without filing; `anfs report diag` prints the diagnostics alone.

Run `anfs report wait` in the background: it waits on events (`report.closed`, `source.updated`), not a timer. If the fix is merged but not here, `anfs src update` brings it.

Whoever merges the fix closes it: `anfs report close <id> --commit <sha>`. Under a dispatch rule with `land:`, anfsd closes it after you approve the landing.

`anfs report ls [--open]` (coming): list the reports, or only the open ones.

**What you should see:** `anfs report` prints the id (`r-20261024-1`, or `owner/repo#42`). `anfs report status` exits 0 only for `available`. `anfs report show <id>` prints the report.

## Approve what agents did (inbox)

**When:** an agent finished work that needs your yes before it lands (a merge, an install, a push), a space's approval policy changed, or an event was turned away and someone must decide. anfsd pops an OS notification when an approval waits, and Claude Code sessions in a space print an inbox line.

```bash
anfs inbox ls                              # ID STEP SPACE WHAT RUN AGE STATUS
anfs inbox show <id>                       # commits, diffstat, verify: result, the exact commands approve runs
anfs inbox show --diff <id>                # the full diff (--log: the run's log)
anfs inbox approve <id>                    # anfsd performs it
anfs inbox refuse --reason "wrong fix" <id>
```

`approve --through install` also gives the run's later steps now. A policy change, a push whose policy is `person`, and `--through push` are typed back at the terminal, never taken with `-y`.

**Unhandled work** (STEP `dispatch`): `anfs inbox redispatch <id>` runs the event after all; `anfs inbox dismiss <id>` records that nobody will.

**Work you did by hand:** `anfs inbox request --job <job> merge` asks for a merge (or `install`, `push`) of a job you worked in yourself, so it lands the same way.

**What you should see:** `anfs inbox ls` prints nothing when nothing waits; `anfs inbox ls -a` shows the last 30 days, answered and dismissed too. After `approve`, the run moves on: `anfs agents ls` shows the next step, and the report closes once the fix is installed.

## Watch the bus and events

**When:** you want to know what happened, in which space, and by whom, or check the bus is healthy.

Every tool appends events to a log per space (`~/.local/state/anfs/events/`). anfsd ships them to NATS JetStream for programs to consume. The files are the complete record; the commands below read them whether NATS is up or not.

```bash
anfs bus status                            # nats, jetstream, anfsd, shipper, recorder, dispatch, harnesses, podman, notify, inbox
wsm events ls                              # the space you are in
anfs events ls -f                          # every space, as it happens
anfs events ls --type 'package.*' --since 1h
anfs events ls --type 'container.*' -f     # containers, as podman reports them
```

Filters: `--type`, `--subject`, `--since`, `--by` (the actor), `--after <id>`. `-f --once` waits for the first match and exits, which is how a script or agent waits for one event:

```bash
wsm events ls -f --once --type report.closed --subject r-20261024-1
```

For NATS itself: `anfs bus nats -- stream info ANFS`, or `anfs bus nats -- sub 'anfs.>'` to watch every message. `anfs events validate` checks the log against the event spec ([anfs-io/events](https://github.com/anfs-io/events)).

**What you should see:** each event row is `EVENT SUBJECT SPACE BY`: time and type, its subject (a report id, a package, a run), the space, and the actor (`claude:2e9431a0`, `shell:rob`, `anfsd`, `podman`). `anfs bus status` exits 1 when a row is not `ok`; podman `away`, notify `off` and items waiting are not failures.

## Coordinate a space with Claude Code

**When:** you use Claude Code in a space and want one session to be the space's **coordinator**: it is woken when a report is filed, a run ends, or an approval is requested, and tells you what is yours.

Installing `core/wsm` writes the hooks into `~/.claude/settings.json` (merged; your own keys and hooks stay). Every installed space has a coordinator unless its `space.yml` says `coordinator: none`. Then:

```bash
wsm install anfs-dev                       # records the space's coordinator policy
wsm cd anfs-dev && claude                  # the first session in the space claims the coordinator role
wsm lease holder coordinator               # who holds it (exit 0 when live)
wsm lease ls                               # SPACE ROLE HOLDER STATUS SINCE RENEWED
```

The first session prints a short brief; any later session in the space prints one line naming the holder. When the holder ends, the next session to take a prompt takes the role over. Events are delivered from where the last holder left off, so nothing between two holders is missed.

Turning it off:

```bash
ANFS_COORDINATOR=off claude                # a session that never coordinates
ANFS_INBOX_HINT=off claude                 # no inbox line in this session
wsm config set WSM_CLAUDE_STATUSLINE=off   # no status line (wsm config unset ... to turn it back on)
```

For a space, `coordinator: none` in its `space.yml`, then `wsm install <space>` again. The long form `coordinator: {harness: claude, wake: [TYPE...], window: 5s}` changes which events wake it and how long it gathers them.

**Other tools as holders.** The lease is not Claude-specific: `wsm lease claim <role>` prints a claim id, and `wsm listen --claim <id> <role>` waits for the events that wake the role, prints the batch and exits (0 events, 1 the lease was lost, 3 `--max` passed). Dispatched runs never hold a lease.

**What you should see:** the coordinator session's status line shows `coordinator: <space path>` (for example `coordinator: lgat/next`); other sessions show nothing from anfs. A new form, `space: <space> role: coordinator|none`, is coming. `wsm lease ls` shows the holder `listening` while it waits and `working` once woken.

## Troubleshoot

**When:** a command failed, did something unexpected, or a service is not up.

```bash
anfs log -f                                # every tool's log; anfs log pcm for one tool
pcm -d up postgres                         # debug output (--trace: every command the tool runs)
anfs bus status                            # which part of the bus is not ok
anfs bus logs -f anfsd                     # the daemon's log
wsm config ls                              # a tool's settings and where each comes from
```

More places to look:

- **A space is `partial` or `invalid`:** `wsm ls <space> --json` lists each file and link with its state. `wsm install -n` says what install would do; `-f` moves what is in the way aside.
- **A package will not install:** `ppm install -n <package>` lists what would be linked, kept or in the way; `ppm deps <package>` its dependency tree.
- **A container definition is refused:** `pcm validate <service>`; for machines, `pmm validate` and `pmm image validate`.
- **A dispatch rule does not fire:** `anfs agents rules` (status and reason with `--json`), then `anfs agents match` with the event to see why. `anfs bus reload` rereads the rules.
- **Skills missing for an agent:** `psm ls` (`ok`, `partial`, `missing`), then `psm sync`.
- **Listeners between tools:** `anfs callbacks` shows who listens to which tool's events, `ok` or `stale`.
- **It is a bug in the tool:** `anfs report`, above.

## Reference

One line per command. Options are on each command's `--help`.

### anfs

| Command | Purpose |
| --- | --- |
| `anfs src list` | The sources in priority order: status, and what each provides |
| `anfs src add [--top] <url> [alias]` | Add a source to your list, and clone it |
| `anfs src remove <source>` | Take a source off your list |
| `anfs src update [source...]` | Clone missing sources and pull the others |
| `anfs src push [--all] [source...]` | Push each source's current branch to its upstream |
| `anfs src ssh [source]` | Switch GitHub HTTPS sources to SSH |
| `anfs src install <source>...` | Install everything a source holds |
| `anfs list` | Every stack in every source: STACK STATUS VERSION |
| `anfs show <stack>` | A stack's resources, in install order, with their status |
| `anfs install [--build] <stack>...` | Install stacks, their dependencies first |
| `anfs remove <stack>...` | Undo what a stack's install put here |
| `anfs path <stack>` | Print a stack's directory |
| `anfs onboard [--check] [step...]` | Make a fresh install yours, step by step |
| `anfs iso <list\|show\|pull\|add\|remove\|path\|prune>` | Install ISOs: the catalogue, and the shared store |
| `anfs events list` | The event log of every space and the host |
| `anfs events validate` | Check the event log against the event spec |
| `anfs report [new] ...` | File a bug report in an anfs tool, and print its id |
| `anfs report check` | Can this machine file a report |
| `anfs report diag` | Print the diagnostics block only |
| `anfs report status <id>` | open, fixed or available |
| `anfs report show <id>` | Print a report |
| `anfs report wait <id>` | Wait until the fix is available here |
| `anfs report close --commit <sha> <id>` | Record the fixing commit and close |
| `anfs report ls [--open]` | List reports (coming) |
| `anfs config <list\|get\|set\|unset>` | Settings in `~/.config/anfs` |
| `anfs log [-f] [tool]` | Print the log every tool writes |
| `anfs callbacks [tool]` | Who listens to which tool's events |
| `anfs rpc` | The toolkit's API: JSON requests on stdin, answers on stdout |
| `anfs implode [--all]` | Remove everything the toolkit put on this machine |

### anfs bus

| Command | Purpose |
| --- | --- |
| `anfs bus status` | Each part of the bus, and the inbox |
| `anfs bus start [service]` | Start NATS and anfsd, or the one named |
| `anfs bus stop [service]` | Stop them (they start again at the next login) |
| `anfs bus restart [service]` | Write the service again and restart it |
| `anfs bus reload` | anfsd rereads the dispatch rules |
| `anfs bus logs [-f] [service]` | The services' log |
| `anfs bus nats -- <args>` | The nats CLI as the admin user |

### anfs agents

| Command | Purpose |
| --- | --- |
| `anfs agents list [-a] [-u] [spec]` | The runs (`--unhandled`: events a rule turned away) |
| `anfs agents show <run>` | A run: its record, its event, its command line, its job |
| `anfs agents logs [-f] <run>` | A run's output as text |
| `anfs agents stop [run]` | Stop a run |
| `anfs agents rules [spec]` | The installed spaces' dispatch rules |
| `anfs agents match ...` | What each rule would do with an event; starts nothing |
| `anfs agents redispatch [--again] <event> [rule]` | A run for an event after all |

### anfs inbox

| Command | Purpose |
| --- | --- |
| `anfs inbox list [-a] [spec]` | What waits on you |
| `anfs inbox show [--diff] [--log] <id>` | One item, with the commands approve runs |
| `anfs inbox approve [--through STEP] <id>...` | Approve; anfsd performs each step |
| `anfs inbox refuse [--reason TEXT] <id>` | Refuse; the run ends canceled, its job stays |
| `anfs inbox request [--job JOB] [--repo REPO] <step>` | Ask for a merge, install or push of a job worked by hand |
| `anfs inbox redispatch <id>...` | Unhandled work: run it after all |
| `anfs inbox dismiss [--reason TEXT] <id>...` | Unhandled work: nobody will handle it |

### anfs work

| Command | Purpose |
| --- | --- |
| `anfs work new [--from REF] <job> [repo...]` | A job: a worktree of each repo on a new branch |
| `anfs work add <job> <repo>...` | Another repo joins a job |
| `anfs work list` | Every job and its status |
| `anfs work show <job>` | One job: worktrees, commits ahead, dirty or not |
| `anfs work path <job> [repo]` | Print the job's directory, or one worktree |
| `anfs work cd <job> [repo]` | Go there (shell function) |
| `anfs work env <job>` | The job's isolation variables as export lines |
| `anfs work run <job> [repo] -- <cmd>...` | Run a command with the job's variables |
| `anfs work remove <job>` | Remove the worktrees, merged branches and directory |

### anfs hooks

| Command | Purpose |
| --- | --- |
| `anfs hooks status [source...]` | Which repos have the version-bump git hooks |
| `anfs hooks install [source...]` | Point each repo's `core.hooksPath` at them |
| `anfs hooks uninstall [source...]` | Unset it again |

### ppm

| Command | Purpose |
| --- | --- |
| `ppm list [--installed] [filter]` | Packages available, and the installed version |
| `ppm show <package>` | Layers, dependencies, what is installed |
| `ppm deps <package>...` | A package's dependency tree |
| `ppm install [-n] [-f] [-r] <package>...` | Install packages and their dependencies |
| `ppm remove <package>...` | Remove packages and the software ppm installed for them |
| `ppm path <package>` | Print a package's directory |
| `ppm cd [package]` | Go to a package directory (shell function) |
| `ppm file <add\|claim\|reset\|protect\|unprotect>` | Own single files, or detach them from ppm |
| `ppm move <package> <source>` | Move a package to another source, with its history |
| `ppm container ...` | Disposable Linux boxes for testing anfs installs |
| `ppm vm ...` | Disposable macOS VMs for testing anfs installs |
| `ppm user ...` | Throwaway users on this machine, for testing installs |
| `ppm implode` | Remove every package ppm installed, then ppm |

### pcm

| Command | Purpose |
| --- | --- |
| `pcm list` | Services: CONTAINER STATUS VERSION |
| `pcm show <service>` | Definition, images, env names, dependencies, ports, mounts |
| `pcm install <service>...` | Validate and start services |
| `pcm up <service>...` | Start services |
| `pcm down <service>...` | Stop services |
| `pcm remove <service>...` | Delete what pcm created for it; keep the definition |
| `pcm provision [--as NAME] <service> [k=v...]` | Run its provision hook for an app outside pcm |
| `pcm deprovision [--as NAME] <service> [k=v...]` | Undo that |
| `pcm ps [service...]` | pcm's containers |
| `pcm logs [-f] <service>` | A service's logs |
| `pcm exec <service> -- <cmd>...` | Run a command in a running container |
| `pcm run <service> [cmd...]` | A one-off container of a service |
| `pcm shell <service>` | A login shell in its container |
| `pcm snapshot <service> [name]` | Save the service and its data; no name lists them |
| `pcm reset <service> [snapshot]` | Recreate it from a snapshot or its definition |
| `pcm validate [service...]` | Check services against pcm's conventions |
| `pcm path`, `pcm cd` | A service definition's directory |
| `pcm machine resize` | The podman machine (macOS) |
| `pcm implode` | Delete everything pcm created |

### pmm

| Command | Purpose |
| --- | --- |
| `pmm run [options] [image]` | A machine from an image, started |
| `pmm up [machine...]` | Start machines (a defined one is made first) |
| `pmm down <machine>...` | Shut machines down |
| `pmm shell <machine>` | Log in to a running machine |
| `pmm exec <machine> -- <cmd>...` | Run a command in a running machine |
| `pmm ps` | Running machines |
| `pmm list` | Every machine: defined, made or running |
| `pmm show <machine>` | Settings, build, snapshots |
| `pmm reset <machine> [snapshot]` | Recreate from a snapshot or the image's current build |
| `pmm remove <machine>...` | Stop and delete machines and their snapshots |
| `pmm snapshot <list\|create\|remove>` | Saved copies of a machine's disk |
| `pmm image <list\|show\|new\|build\|test\|export\|remove\|prune\|path\|cd\|validate>` | Images: definitions and builds |
| `pmm network <list\|create\|show\|path\|install\|remove>` | Networks machines are on |
| `pmm iso ...` | Install ISOs (as `anfs iso`) |
| `pmm install <machine-spec>...` | Create and start machines a source defines |
| `pmm validate`, `pmm path`, `pmm cd` | Definitions |
| `pmm implode` | Stop every machine, delete everything pmm created |

### psm

| Command | Purpose |
| --- | --- |
| `psm list` | Each declared skill: ok, partial, missing |
| `psm install <skill-spec>...` | Install a source's skills, or one |
| `psm sync [config...]` | Install declared skills that are missing or not linked to every agent |
| `psm update [config...]` | Update remote skills; re-copy local ones |
| `psm remove <skill>...` | Remove skills |
| `psm prune` | Remove installed skills no config declares |
| `psm repo <list\|add\|remove>` | The repos skills come from |
| `psm agents <list\|remove>` | The agents skills are installed for |
| `psm cd` | The config directory (shell function) |
| `psm implode` | Remove every skill and psm's config |

### wsm

| Command | Purpose |
| --- | --- |
| `wsm list [spec]` | The spaces, and the state of each installed one |
| `wsm install [-n] [-f] [spec...]` | Link spaces, clone their repositories, install their skills |
| `wsm remove <space>...` | Unlink spaces; clones and your files stay |
| `wsm new <space>` | Create a space in a source |
| `wsm add <file>...` | Move files from the spaces tree into the source, link them back |
| `wsm cd [query]`, `wsm path [query]` | Go to, or print, an installed space |
| `wsm id [--new] [spec]` | Print a space's id; `--new` writes one where there is none |
| `wsm repo <list\|install\|ssh\|push>` | The repositories spaces declare |
| `wsm skill <list\|install\|remove>` | The skills spaces declare |
| `wsm events list` | The event log of the space around you |
| `wsm lease list [spec]` | Every lease: SPACE ROLE HOLDER STATUS SINCE RENEWED |
| `wsm lease holder <role>` | The role's holder in this space |
| `wsm lease claim <role>` | Take the role; print the claim id |
| `wsm lease renew --claim ID <role>` | Mark the lease alive |
| `wsm lease release --claim ID <role>` | Give the role up |
| `wsm listen [--claim ID] [--type GLOB] <role>` | Wait for the events that wake the role, print them, exit |
| `wsm move <spec> <source>` | Move a space to another source, with its history |
| `wsm config <list\|get\|set\|unset>` | Settings in `~/.config/wsm` |
| `wsm implode` | Unlink every space, delete wsm's state |

### anfsd

The daemon runs as a service (`anfs bus start`); these are for checking it by hand.

| Command | Purpose |
| --- | --- |
| `anfsd rules [space.yml...]` | Every dispatch rule and its status |
| `anfsd match ...` | What each rule would do with an event |
| `anfsd validate FILE...` | Check event lines against the spec |
| `anfsd version` | Print the version |
