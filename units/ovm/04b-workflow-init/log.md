---
plan: ovm/04b-workflow-init
job: ovm-04b
date: 2026-10-22
---

# Log: ovm/04b-workflow-init

Branch `ovm-04b`, cut from `ovm-04a` (stack `b4739e7`, not merged yet): if 04a changes, this
rebases on it. The shared lines are `cmd_init` (04a's `vault.yml` template is kept byte for byte)
and `README.md`.

## What was built (stack, `packages/obsidian`)

- **`home/.local/lib/ovm/`**, linked as one directory (`package.yml`: `links: [.local/lib/ovm/]`):
  - `plugins.yml`: the three pins (below).
  - `vault/`, the skeleton, every file copied when missing:
    - `.obsidian/plugins/templater-obsidian/data.json`: `data_version: 2`, `templates_folder:
      _templates`, `trigger_on_file_creation_mode: folder`, `folder_templates: [{folder: "/",
      template: _templates/note.md}]`, every other key at 2.26.0's defaults (read from the
      release's `main.js`: `command_timeout 5`, `templates_pairs []`, `intellisense_render "1"`,
      `ignore_folders_on_creation []`, ...). Not `trigger_on_file_creation`,
      `enable_system_commands`, `enable_folder_templates` (2.26.0's migration deletes them).
    - `.obsidian/plugins/obsidian-shellcommands/data.json`: `settings_version: "0.23.0"` and one
      command: `id: ovm-validate`, alias `ovm: validate this vault`,
      `platform_specific_commands.default: PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH" ovm validate`,
      `shells: {}`, stdout and stderr to `notification`. No working-directory key: the plugin's
      global `working_directory` defaults to `""`, which is the vault's directory.
    - `.obsidian/plugins/obsidian-meta-bind-plugin/data.json`: `{"excludedFolders": ["_templates"]}`
      (1.5.1's `loadSettings` is `Object.assign({}, defaults, loaded)`: a partial file is merged).
    - `_templates/note.md`: `id` minted by Templater JS (`crypto.getRandomValues`, 10 of `a-z2-7`,
      `b % 32` on a byte, so uniform), `type: note`, `title: "<% tp.file.title %>"`, a `#` heading.
      `_templates/task-list.md`: the same, a `meta-bind-button` (style primary, `action: {type:
      command, command: obsidian-shellcommands:shell-command-ovm-validate}`), `## Tasks`, `- [ ] `.
- **ovm** (`OVM_VERSION` and `package.yml` 0.5.0, by hand; the hook: "already 0.5.0 since 0.2.0
  was pushed, not bumping"):
  - `_ovm_lib`: `~/.local/lib/ovm`, else `../lib/ovm` beside the script's real path. No `OVM_LIB`
    environment override (it would be an undocumented setting).
  - `_pins_load`: one yq; ids, repos, versions, asset names and sha256s are checked by pattern
    before any URL is built from them; `manifest.json` is placed last, so a plugin is never "there"
    before its code is.
  - `_fetch`: `$OVM_CACHE_HOME/plugins/<id>/<version>/<asset>`; a cached file is used only if its
    sha256 matches (else fetched again); `curl -fsSL --retry 3 -o <asset>.part`, checked, moved;
    a mismatch deletes the `.part`. `sha256sum`, else `shasum -a 256`.
  - `_plugin_put`: every asset fetched before anything is written; each file written beside its
    name and moved into place; `data.json` never touched.
  - `_plugins_enable`: `community-plugins.json` checked as a JSON array of strings (yq), the
    missing ids appended (`yq -o json -I 2`, Obsidian's own layout), not rewritten when nothing is
    missing, moved into place.
  - `_skeleton_apply`: core's `anfs_files_apply` (`files.sh` sourced on first use inside the
    function, so ovm's `source` lines are untouched) with the list of every file under `vault/`.
  - `cmd_init [dir] [--update] [--no-plugins]` in the plan's order; `cmd_new` takes `--no-plugins`.
    `--update --no-plugins` is 64; `--update` on a directory with no `vault.yml` is 3.
  - The closing message and `--json` keys (below); the help epilogue names `plugins.yml`, the
    cache, the two clicks, the warning and the reload after `--update`.
- **README**: "A workflow-ready vault" (what init places, the cache, `--update`, `--no-plugins`,
  exit codes, bumping a pin, the two clicks and why, the Shell commands warning).
- **`tests/pins`**: a script, not bats: fetches each pinned asset, compares the sha256 and each
  `manifest.json`'s id and version; `--print` prints the sha256s for a bump.

### The pins

| id | repo | version | main.js | manifest.json | styles.css |
| --- | --- | --- | --- | --- | --- |
| `templater-obsidian` | SilentVoid13/Templater | 2.26.0 | `0acd45cfdb3a7337e2bef511699bc1f819dc64645ad3480df33579d42d2dcbaa` | `91c9fa8510082fffa0b3d022aac4986e3b392295b1126e847ead7ed7289169b8` | `67a6dd2d1d6dedca1287c334ca884ec4196433f71ca47388a0bd3eac14cf8b5f` |
| `obsidian-shellcommands` | Taitava/obsidian-shellcommands | 0.23.0 | `0c5e247a91c96c0af5f3e333ba43be4285469566c819d59b7de129ad33d14d8a` | `20ae8ccfa8972027d35b4457d21b6ab86aebc8e3b1dee5eba34dad19e46a2462` | `3bd8380e5aa53fc447ea6a4c14ddfb94198f1a6d003e6e5f994b459cf9684c53` |
| `obsidian-meta-bind-plugin` | mProjectsCode/obsidian-meta-bind-plugin | 1.5.1 | `d328bfaa4d1f48259074b28f249a9f911ea34327e7e1123d8a1f2afc3e9784b3` | `3c1dff1e36a17811851f3c7446b875cf901288b73261caea61bc0d440d3c4325` | `97069e183224c55428fe0b983d42435aa2cd4ac14d0f50186138b4751ad55d61` |

How (once, into the job's TMPDIR, 2026-10-22):

```bash
D=/tmp/anfs-work-501-4/pins; mkdir -p $D; cd $D
fetch() { mkdir -p $1; for a in main.js manifest.json styles.css; do
  curl -fsSL --retry 3 -o $1/$a https://github.com/$2/releases/download/$3/$a; done; }
fetch templater-obsidian SilentVoid13/Templater 2.26.0
fetch obsidian-shellcommands Taitava/obsidian-shellcommands 0.23.0
fetch obsidian-meta-bind-plugin mProjectsCode/obsidian-meta-bind-plugin 1.5.1
for id in *; do yq -p json -o tsv '[.id, .version, .minAppVersion]' $id/manifest.json; shasum -a 256 $id/*; done
```

Each `manifest.json`: `templater-obsidian 2.26.0 1.13.0`, `obsidian-shellcommands 0.23.0 1.4.0`,
`obsidian-meta-bind-plugin 1.5.1 1.13.1`: id and version match the key and the pin.

**Templater 2.26.0 kept**: the release (2026-10-11) has no notes; the open issues since then
(GitHub API, 2026-10-22) are one about renaming the templates folder and one feature request,
nothing on folder templates or new-note creation.

`tests/pins`, run through a curl shim that serves the files above by URL (so the assets were
downloaded once, not twice):

```
ok   templater-obsidian 2.26.0 main.js 0acd45cf...
ok   templater-obsidian 2.26.0 manifest.json 91c9fa85...
ok   templater-obsidian 2.26.0 manifest.json: id and version match
ok   templater-obsidian 2.26.0 styles.css 67a6dd2d...
ok   obsidian-shellcommands 0.23.0 main.js 0c5e247a...
ok   obsidian-shellcommands 0.23.0 manifest.json 20ae8ccf...
ok   obsidian-shellcommands 0.23.0 manifest.json: id and version match
ok   obsidian-shellcommands 0.23.0 styles.css 3bd8380e...
ok   obsidian-meta-bind-plugin 1.5.1 main.js d328bfaa...
ok   obsidian-meta-bind-plugin 1.5.1 manifest.json 3c1dff1e...
ok   obsidian-meta-bind-plugin 1.5.1 manifest.json: id and version match
ok   obsidian-meta-bind-plugin 1.5.1 styles.css 97069e18...
```

(full hashes in the table). Also by hand: `ovm new scratch` from the package with the real
`plugins.yml`, a temp `HOME` under the job's TMPDIR and the same shim: the three real plugins
placed, byte-identical to the releases, the cache under `~/.cache/ovm/plugins/<id>/<version>/`.

### What the plugins' code says (read in each release's `main.js`)

- Templater: `Object.assign({}, defaults, loaded)` when `data_version` is present; the root
  folder's path is `/` and folder templates walk up the parents, so `folder: "/"` covers every
  folder; files under `templates_folder` are never templated on creation; the trigger is
  `localSettings.trigger_on_file_creation` (per device).
- Shell commands: `loadSettings` refuses (disables itself) when `settings_version` is newer than
  0.23.0; `EnsureMainFieldsExist` and `EnsureShellCommandsHaveAllFields` fill a minimal file; the
  Obsidian command is `shell-command-<id>`; an empty `working_directory` is the vault's path.
- Meta Bind: a button's `style` is required (`default`, `primary`, `destructive`, `plain`); `action`
  or `actions`; `{type: command, command}`.
- Both Meta Bind and Shell commands rewrite their `data.json` on first load (filled with defaults),
  so a second `ovm init` after Obsidian opened the vault reports them as "kept ... yours". That is
  expected.

## Output and --json

```
initialized vault notes (ir3437myfj) at ~/vaults/notes
copied .obsidian/plugins/obsidian-meta-bind-plugin/data.json
...
copied _templates/task-list.md
placed templater-obsidian 2.26.0
placed obsidian-shellcommands 0.23.0
placed obsidian-meta-bind-plugin 1.5.1
enabled in .obsidian/community-plugins.json: templater-obsidian obsidian-shellcommands obsidian-meta-bind-plugin

If you have not yet, on each device, in Obsidian (no file can do these):
  1. Open this vault and turn on community plugins ...
  2. Settings > Templater: turn on "Trigger Templater on new file creation".

Shell commands lets a note run commands on this computer, ... (section 6 of the plan)
```

A plugin there: `templater-obsidian 2.25.0 is there: left as it is (the pin is 2.26.0; ovm init
--update brings an older one to it)`. A kept Shell commands `data.json` without `ovm-validate`:
"Shell commands' settings in this vault are yours, so ovm's command is not in them. To add it:
Settings > Shell commands > New shell command: `PATH=... ovm validate`" (decided overnight: never
edit the person's file).

`--json` (`anfs_result`): `id`, `name`, `path`, `status` as before, plus `placed`, `updated`,
`kept` (plugin ids left as they were), `copied` (skeleton files), `steps_left`
(`turn-on-community-plugins`, `templater-trigger-on-new-file`, `add-ovm-command`). Lists are
space-separated strings: `anfs_result` takes strings only.

## Commits (branch ovm-04b, not pushed)

stack:
- `e2612d2` obsidian: ovm init makes a vault workflow ready ... (obsidian 0.5.0)

docs:
- this log

## Tests

`ANFS_TEST_LIB=~/.local/lib/anfs anfs work run ovm-04b stack -- bats packages/obsidian/tests
</dev/null`: **77/77** (was 65).

- `init.bats` (new, 12): every case of the plan's spec: `new` (skeleton byte-equal, three
  plugins byte-equal to the served release, `community-plugins.json`, the closing message, no temp
  files left); copy-if-missing (edited template and `data.json` survive, a deleted template comes
  back, "kept 4", no curl call); `community-plugins.json` (another plugin and order kept, ours not
  twice, not rewritten when complete, invalid JSON not written and exit 1, plugins still placed);
  a plugin there left by init, `--update` (older replaced with `data.json` kept, newer left, absent
  named, nothing else touched, at the pin, `--update --no-plugins` 64, `--update` on a non-vault 3);
  the cache (second vault no curl, corrupted cache refetched alone, served wrong bytes: 69, nothing
  of that plugin, no `.part`, the other two placed and enabled); no curl on PATH (69, `vault.yml`
  and the skeleton there, a later init finishes); `--no-plugins`; a directory in the way of
  `_templates/note.md` (nothing of the skeleton, plugins placed, 1); the library missing (69, the
  hint, `vault.yml` written; from the package its own library is found); a person's Shell commands
  `data.json` kept with the hint; the two templates rendered by substitution pass `ovm validate
  --strict` with init's `vault.yml`, and the skeleton's JSON parses; `--json`.
- `helper.bash`: every test links a fixture library as `~/.local/lib/ovm` (the package's real
  `vault/`, a `plugins.yml` with the three real ids at a fixture 1.0.0 and their real sha256s,
  built once per file) and puts a `curl` stub first on PATH that serves those releases by URL,
  logs each URL and fails (22) on anything else: no test can reach the network. `vault()` uses
  `init --no-plugins` (the docs and spaces tests do not need plugins; the suite went from 73 s to
  113 s, it was 131 s with plugins in every `vault()`).
- `vaults.bats`: init's first line is `${lines[0]}` now (more follows); help shows `new
  [--no-plugins] <name>`. `docs.bats`' exclude test: with no `exclude:` the two templates are
  documents too (9 lines, `7 document(s)`).
- **Not run**: shellcheck (not installed on this host). No VM, round trip or core suite (core
  unchanged). `ANFS_WORK` is never set by these tests.

## Decisions made during the work

- **Meta Bind gets a `data.json`** (`excludedFolders: [_templates]`): 1.5.1 merges a partial file.
- **`folder: "/"`** for the root's folder template (Obsidian's root folder path), and no
  `ignore_folders_on_creation` (Templater already skips its templates folder).
- **The id expression** uses `crypto.getRandomValues`, not `Math.random`, and is a plain YAML scalar
  (unrendered it still parses; `_templates` is excluded by init's `vault.yml` anyway).
- **`kept` in --json is plugins**, not skeleton files; `copied` was added for the skeleton.
- **Exit code when both happen**: something in the way (1) wins over a failed download (69): 1
  needs the person, 69 only needs a rerun.
- **`--update` does not run the skeleton or write `vault.yml`**; it needs `vault.yml` there (3
  otherwise), as the plan's "step 1's checks that this is a vault".
- The plan's `cmd_init [... --name NAME ...]` signature: there is no `--name` since 03 (a vault is
  named by its entry); not added.
- No `OVM_LIB` variable (plan section 2 says "three candidates", as tsm's `TSM_LIB` + two): an
  environment override would be a setting `ovm.env` must document; two candidates are enough.
- `say` lines for outcomes, `warn` (stderr) for what did not happen; `--json` silences the former.

## Shared files touched

- `packages/obsidian/home/.local/bin/ovm`: `OVM_VERSION`; the `cli_cmd init`/`new` lines; the help
  epilogue; a new section before `# --- init, new, list`; `cmd_init`, `cmd_new`. The header
  comment, the `source` lines and the `ANFS_LIB` check are untouched (config-03's preamble).
- `packages/obsidian/tests/helper.bash`: `ovm_setup` grows the library and curl stub after the wsm
  stub; `vault()` passes `--no-plugins`. 02-tasks's tests can use `vault()` unchanged.
- `README.md` (a new section, the command block). `package.yml` (`links:`, version).

## By hand, for the person after merge

1. Pull stack, then `ppm install obsidian` (links `~/.local/lib/ovm`; `ovm --version` is 0.5.0;
   `ls ~/.local/lib/ovm` shows `plugins.yml vault`).
2. A scratch vault first:

   ```bash
   ovm new scratch            # placed x3, copied x5, the two steps
   ```

   Open `~/vaults/scratch` in Obsidian ("Open folder as vault"). Then the clicks, once on this Mac:
   - **Turn on community plugins** (the trust dialog's button, or Settings > Community plugins).
     Templater, Shell commands and Meta Bind are then listed and enabled.
   - Settings > Templater: **"Trigger Templater on new file creation"** on.
   Make a note at the root: it gets frontmatter with a 10-character id, `type: note`, its title.
   Make one from `task-list` (Command palette: "Templater: Create new note from template") and
   press **Validate this vault**: a notification with `ovm validate`'s output. If it says `ovm: not
   found`, ovm is not under `~/.local/bin`, `/opt/homebrew/bin` or `/usr/local/bin`: edit the
   command in Settings > Shell commands.
   On each other device (another Mac, the phone): the same two clicks after the vault syncs.
   Shell commands does not load on a phone; the button does nothing there.
3. **apex** (`~/vaults/apex`, after 04a's by-hand steps). Run it; it is safe:

   ```bash
   ovm init ~/vaults/apex
   ```

   What it does there: apex's own `_templates/` files and its Shell commands `data.json` are
   kept (copy-if-missing: a file there is never touched; only `_templates/note.md` and
   `task-list.md` are added if apex has no file of that name, and the Templater and Meta Bind
   `data.json` only if missing). The `templater-obsidian` directory already there is left, with a
   line giving its version against 2.26.0; `ovm init --update ~/vaults/apex` if it is older (then
   reload Obsidian). Meta Bind (and Shell commands, if its directory has no `manifest.json`) is
   placed and enabled. ovm's `ovm-validate` command is not added to apex's Shell commands: init
   prints how to add it by hand, if wanted. Check first what would be added:

   ```bash
   ls ~/vaults/apex/_templates ~/vaults/apex/.obsidian/plugins
   cat ~/vaults/apex/.obsidian/community-plugins.json
   ```

   A deleted template is missing, so the next `ovm init` copies it again. To keep ovm's
   `note.md` or `task-list.md` out of apex for good, keep a file of that name there (even empty):
   it is then yours and never touched. Templater's `data.json`, if apex has none, makes
   `_templates/note.md` the template of every new note: if apex has its own Templater settings
   (it does if the directory exists with a `data.json`), they stay.
4. apex's `bin/apex plugins sync` and its `plugins.yml` overlap with this (and its
   `community-plugin-directory` key in `app.json` does nothing in 1.14.4); retiring them is your
   call.

## Not run / deferred

- No VM, box or round trip: nothing here boots one.
- The real GitHub fetch through ovm (only through the shim, with the assets fetched once).
- Obsidian itself: nothing was tried in a running Obsidian (the clicks, the template, the button).

## Open questions (for the person, before merge)

1. **Templates that come back**: copy-if-missing means a template the person deletes is copied
   again by the next `ovm init` (the plan's test asks for it). For apex, where `_templates/` has
   its own set, `note.md` and `task-list.md` will appear and return if deleted. Acceptable, or
   should init copy the skeleton only into a vault that has no `.obsidian/` yet (a new vault)?
2. **`--json` lists as strings**: `placed`, `kept`, ... are space-separated strings, since
   `anfs_result` takes strings. If a reader wants arrays, core's `anfs_result` could take the
   `key[]` convention `anfs_table` has (a core change, not made here).
3. The unit's open question "`init` on a plain-markdown vault: should the default be
   `--no-plugins` for a vault with no `.obsidian/`?" was decided overnight (plugins by default);
   nothing left, noted for the unit file's open questions list.
