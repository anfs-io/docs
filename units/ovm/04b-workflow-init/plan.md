---
---

# Plan 04b — `ovm init` makes a vault workflow ready: pinned plugins, their settings, our templates

Written 2026-10-22 from the decisions with the person that day (`units/ovm.md`, decisions 2 and
3). Runs after `04a-vault-spaces` (it writes into the `vault.yml` world 04a makes) and so after
`03-vaults-home` (`ovm new <name>` in `~/vaults`). Why 04 is two plans: see 04a's header.

## Context — read these files first

The job is `anfs work new <job> stack docs`. Paths are stack's unless they name another repo.

- `units/ovm.md` — the decisions of 2026-10-22, and "Obsidian, checked 2026-10-22" (what is
  verified about plugin files, restricted mode and Templater, with sources); 04a's plan and
  `log.md`
- `packages/obsidian/home/.local/bin/ovm` — `cmd_init`, `cmd_new`, `say`, `anfs_result`, the
  exit codes
- core `packages/anfs/home/.local/lib/anfs/files.sh` — `anfs_files_plan`, `anfs_files_apply`
  (copy-if-missing: `missing`, `present`, `differs`, `in the way`); read only
- `units/linker/02-links-and-files/plan.md` — `files:` semantics (copied when missing, never
  again, the user's once there), and `links: [<dir>/]` (a directory as one link)
- stack `packages/tailscale/` — `package.yml` (`links: [.local/lib/tsm/]`) and how `tsm` finds
  `~/.local/lib/tsm`; stack `packages/rpi-imager/home/.local/bin/rpi-imager` — `fetch_image`
  (download to a cache, `.part`, resume) and `sha256_stdin`
- `units/config/05-xdg-rule/plan.md` — the rule (CACHE is rebuildable; a tool writes a user's
  files only by an explicit command)
- user `packages/pjm/tests/helper.bash` — the curl stub pattern

## Overview

`ovm init` today writes `vault.yml`. After this plan it also makes the vault ready for the
workflow, on every run, without ever overwriting what the person has:

1. **Plugins**: Templater, Shell commands and Meta Bind, at pinned versions, placed as files in
   `.obsidian/plugins/<id>/` (`main.js`, `manifest.json`, `styles.css`), downloaded once into
   `$OVM_CACHE_HOME` and checked against pinned sha256s. A plugin already there is left alone.
   A plugin newly placed is added to `.obsidian/community-plugins.json`.
2. **Settings and templates**: shipped in `stack/obsidian` as a vault skeleton, copied when
   missing and never again (`files:` semantics): each plugin's `data.json`, and our Templater
   templates under `_templates/`.
3. **What a file cannot do**, said once at the end: open the vault in Obsidian and turn on
   community plugins (restricted mode is per device and is not in the vault), and turn on
   Templater's "Trigger Templater on new file creation" (per device too).

`ovm init --update` refreshes the three plugins' code to the pinned versions and touches nothing
else. `ovm new <name>` (from 03) creates `~/vaults/<name>` and runs all of this.

What a user sees: `ovm new notes` prints what it placed and the two clicks; the vault opens in
Obsidian with the three plugins listed, enabled after the one "Turn on community plugins" click.

## Implementation

### 1. What the package ships

- `home/.local/lib/ovm/plugins.yml` — the pins, one per plugin:

  ```yaml
  # The plugins `ovm init` places in a vault, at these versions (ovm help init).
  # Bump: change version, then the sha256s (ovm's tests do not reach the network).
  templater-obsidian:
    repo: SilentVoid13/Templater
    version: 2.26.0
    sha256: {main.js: ..., manifest.json: ..., styles.css: ...}
  obsidian-shellcommands:
    repo: Taitava/obsidian-shellcommands
    version: 0.23.0
    sha256: {...}
  obsidian-meta-bind-plugin:
    repo: mProjectsCode/obsidian-meta-bind-plugin
    version: 1.5.1
    sha256: {...}
  ```

  The ids, repos, tags and asset names were checked on 2026-10-22 (`units/ovm.md`); all three
  releases carry the three files as separate assets. The agent fetches each asset once while
  writing this file, computes the sha256s, and checks the `manifest.json`'s `id` and `version`
  match the key and the pin (record the commands in `log.md`). Templater 2.26.0 is eleven days old
  at writing; if its release notes show a regression, pin 2.25.x and say why. Not Buttons
  (decision 3).
- `home/.local/lib/ovm/vault/` — the skeleton, every file a candidate for copy-if-missing:
  - `.obsidian/plugins/templater-obsidian/data.json`: `"data_version": 2`,
    `"templates_folder": "_templates"`, `"trigger_on_file_creation_mode": "folder"`,
    `"folder_templates": [{"folder": "/", "template": "_templates/note.md"}]`; the other keys at
    2.26.0's defaults (`src/settings/Settings.ts`). Not `trigger_on_file_creation`,
    `enable_folder_templates` or `enable_system_commands`: removed in settings v2 and kept per
    device now.
  - `.obsidian/plugins/obsidian-shellcommands/data.json`: `"settings_version": "0.23.0"` and one
    shell command, ours, and nothing else: id `ovm-validate`, alias "ovm: validate this vault",
    the command `PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH" ovm validate`
    (Obsidian started from the Dock has a short PATH), run in the vault's directory, stdout and
    stderr to a notification. Every other key left to the plugin's defaults (a minimal file is
    accepted and filled in; a `settings_version` newer than the plugin disables it, so it is
    exactly the pin's). The agent confirms the per-command keys (`platform_specific_commands`,
    `shells`, `output_handlers`, the working directory) against 0.23.0's
    `ShellCommandConfiguration.ts` and `SC_MainSettings.ts`.
  - Meta Bind: no `data.json` (its defaults work). Its `excludedFolders` default is
    `["templates"]`, not our `_templates`: ship a `data.json` with
    `"excludedFolders": ["_templates"]` only if 1.5.1 merges a partial settings file with its
    defaults (`Settings.ts` / the plugin's `loadSettings`); if it does not, ship none and note it.
  - `_templates/note.md`: frontmatter `id` (10 characters of `a-z2-7`, minted by Templater JS;
    the alphabet and length of `_new_id`), `type: note`, `title` from the file name; then a body
    heading. `_templates/task-list.md`: the same frontmatter (`type: note`), a `## Tasks` heading
    and one `- [ ] ` line, and a Meta Bind button whose action is
    `{type: command, command: "obsidian-shellcommands:shell-command-ovm-validate"}`, so the three
    plugins are seen working together. Both must pass `ovm validate` once rendered (a test renders
    them by substitution, not by Templater).
- `package.yml`: `links: [.local/lib/ovm/]` (one directory link, as tsm's), `version: 0.5.0`.

### 2. ovm finds its library

`OVM_LIB`: the first of `$HOME/.local/lib/ovm` and the package's own `home/.local/lib/ovm`
beside the script's real path (tsm's lookup, three candidates). Missing: init's plugin and
skeleton steps fail 69 "ovm's files are missing: ppm install -f stack/obsidian"; `vault.yml` is
still written.

### 3. `init`, in order

`cmd_init [dir] [--name NAME] [--update] [--no-plugins]`:

1. `vault.yml` as 04a left it.
2. The skeleton: `anfs_files_apply "$OVM_LIB/vault" "$root" <list>` with the list of every file
   under the skeleton. `present` and `differs` are said in a count ("kept 3 files that are
   yours"); `in the way` stops this step (nothing copied, the reason said) and init carries on.
3. Plugins, for each pin:
   - `.obsidian/plugins/<id>/manifest.json` exists: left alone (say its version, and that
     `--update` would refresh it when it differs from the pin).
   - Not there: fetch (section 4), then copy the three files into the plugin directory (made if
     needed; a `data.json` the skeleton just copied is already there and stays), then add `<id>`
     to `.obsidian/community-plugins.json` (a JSON array; created as `[]` when missing; the id
     appended when absent; nothing else in the file changes; not valid JSON: the step says so and
     does not write it).
   - `.obsidian/` is made when missing. Writes are whole files moved into place.
4. The closing message (and in `--json`, `anfs_result` keys `placed`, `kept`, `updated`,
   `steps_left`): the two per-device clicks, and the Shell commands warning (section 6). Printed
   when any plugin was placed this run, or when `.obsidian/community-plugins.json` lists one of
   ours (ovm cannot see whether the click was made; it says "if you have not yet").

`--no-plugins`: steps 1–2 only (a vault that is plain markdown, or a test). `--update`: section 5
only (with step 1's checks that this is a vault).

Exit: 0 when every step did its work or found it done; 69 when a fetch failed or the library is
missing (what could be done was done, and running init again finishes it); 1 when a step met
something in the way. Each step's outcome is a line.

### 4. Fetch, into the cache

`$OVM_CACHE_HOME/plugins/<id>/<version>/<asset>` (CACHE: rebuildable). Present and its sha256
matches: used. Else `curl -fL --retry 3 -o <asset>.part https://github.com/<repo>/releases/download/<version>/<asset>`,
sha256 checked (`sha256sum`, else `shasum -a 256`), then moved into place. A mismatch deletes the
`.part`, places nothing for that plugin, and is 69 "checksum mismatch for <id> <asset>: the pin or
the release changed". No curl: 69. No network calls anywhere else in ovm.

### 5. `init --update`

For each pin whose plugin directory has a `manifest.json`: its version equal to the pin, nothing;
older (by `sort -V`), the three files replaced from the cache (fetched if needed); newer (the
person updated it in Obsidian), left, with a line saying so. A plugin not there is not placed
(`ovm init` does that) and is named. Never touches `data.json`, `community-plugins.json`,
templates or `vault.yml`. Obsidian rewrites plugin files only on its own update, so say in help:
quit Obsidian or reload it (Command palette, "Reload app without saving") after `--update`.

### 6. Said plainly (help, README, init's closing message)

> Shell commands lets a note run commands on this computer, and Meta Bind buttons can run them
> from inside a note. A vault that syncs (iCloud, git) carries its plugin settings: whoever can
> write to the vault can add a command that runs when you click. ovm ships one command (`ovm
> validate`). Before turning on community plugins in a vault you did not write, read
> `.obsidian/plugins/obsidian-shellcommands/data.json`.

And: Shell commands is desktop only; on a phone Obsidian does not load it.

### 7. README and help

- `README.md`: "A workflow-ready vault": what init places, the pins and how to bump them, the
  cache, `--update`, `--no-plugins`, the two clicks and why they cannot be files, section 6.
- `cli_cmd init` / `new` lines and their options; the help epilogue names `plugins.yml`.

## Test Spec

`packages/obsidian/tests/init.bats` (new). No test reaches the network: `curl` is a stub (pjm's
pattern) that serves files from a fixture directory by URL and logs each URL. The helper links a
**test library** as `~/.local/lib/ovm`: the package's real `vault/` skeleton, and a
`plugins.yml` whose three pins point at fixture assets (small `main.js`, a `manifest.json` with
the right id and a fixture version, a `styles.css`) with their real sha256s.

- `new notes` makes `~/vaults/notes` with `vault.yml`, the skeleton's files, the three plugin
  directories with three files each, and `community-plugins.json` holding the three ids; the
  closing message names restricted mode, Templater's trigger and the Shell commands warning.
- Copy-if-missing: an edited `_templates/note.md` and an edited Templater `data.json` survive a
  second `init` byte for byte; a deleted template comes back; a second `init` downloads nothing
  (the curl log is empty) and places nothing.
- `community-plugins.json` with another plugin's id keeps it and its order; one of ours already
  listed is not added twice; invalid JSON is not written and init says so.
- An existing plugin directory (another version) is left as it is by `init`; `init --update`
  replaces its three files with the pin's and leaves its `data.json`; a newer installed version
  is left, with the line; an absent plugin is named, not placed.
- The cache: the second vault's init uses the cache (no curl call); a corrupted cached file is
  fetched again; a served file with the wrong bytes is a checksum mismatch: 69, nothing of that
  plugin placed, the other two placed.
- No curl on PATH: 69, `vault.yml` and the skeleton still there.
- `--no-plugins`: no `.obsidian/plugins/*/main.js`, no curl call.
- The skeleton with something in the way (a directory where `_templates/note.md` goes): that step
  reports it, nothing of the skeleton copied, plugins still placed, exit 1.
- The rendered templates (Templater tags substituted by the test: an id, a title) pass
  `ovm validate --strict` with the default `vault.yml`; the skeleton's JSON files parse (`yq -p
  json`).
- `--json`: `placed`, `kept`, `updated`, `steps_left`.

A check of the real pins, not part of the suite (network): `packages/obsidian/tests/pins`
(a script, not bats) fetches each pinned asset and checks the sha256s; run by hand when bumping.

## Verification

Every command runs in the job: `anfs work run <job> stack -- <cmd> </dev/null`. Nothing is
installed on this host; no real vault, `~/vaults` or `~/.cache/ovm` is touched.

- [ ] While working: `bats packages/obsidian/tests/init.bats` (`-f <name>` for one)
- [ ] At the end: `bats packages/obsidian/tests`
- [ ] `packages/obsidian/tests/pins` once (it reaches GitHub; the only network step), and its
      output in `log.md`
- [ ] shellcheck on `ovm`
- Not run, for the person after merge (in `log.md`): `ppm install obsidian`; `ovm new scratch`;
  open `~/vaults/scratch` in Obsidian, turn on community plugins (one click), turn on Templater's
  "Trigger Templater on new file creation"; make a note at the root and see the template applied
  with an id; open a `task-list` note and press the button (a notification from `ovm validate`).
  Then for apex: `ovm init ~/vaults/apex` keeps apex's own `_templates/` files and its Shell commands
  `data.json` (so ovm's command is not added there; add it by hand if wanted), places Meta Bind,
  and leaves the `templater-obsidian` directory already there (check its version against the pin;
  `ovm init --update ~/vaults/apex` if older). apex's `bin/apex plugins sync` and `plugins.yml` overlap
  with this; retiring them is the person's call.
