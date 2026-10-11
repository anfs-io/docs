---
status: complete
started_at: "2026-09-15T10:15:00+08:00"
completed_at: "2026-09-15T12:55:00+08:00"
deviations: >-
  mise install is scoped to the tools named in the resolved packages' .config/mise/**/*.toml files and runs from $HOME (a bare `mise install` could pick up a project's config or install unrelated tools);
  a package records a declared formula/cask when this run installed it OR any tracker already records it as ppm-installed (shared ownership), so it is removed with its last package;
  reinstall (-r) runs the remover with keep_tracker, which skips dependency removal and keeps the tracker
summary: package.yml platforms/brew/cask/system (list or platform map) with batch installs before hooks, owner-only brew/cask, tracked ppm-installed dependencies and shared-aware removal, scoped mise install; verified by unit tests on macOS and 12 container scenarios on Debian 13
---

# Execution Log

## What Was Done

### `lib/packages.sh`
- `meta_platforms`, `meta_supported` (absent = everywhere; `linux` matches any distro)
- `meta_deps <dir> <brew|cask|system>` — list applies everywhere; map resolves exact platform → `linux` on Linux; a `system` map naming other Linux distros but not this one (and no `linux`) returns 1
- `_list_add` — ordered, de-duplicated newline lists (bash 3.2)
- `meta_installed_deps <repo> <pkg> <brew|cask>`; `meta_mark_installed` takes optional brew/cask names and merges them with what the previous tracker recorded (`installed_deps` written only when non-empty)
- `expand_packages` (`repo/`) skips packages unsupported on this platform with a note
- `show` prints `Platforms`, `Brew`, `Cask`, `System (<platform>)` and `Installed by ppm (brew|cask)`

### `lib/platform.sh`
- `system_pkg_missing` / `system_pkg_install` (debian: `dpkg-query`, `apt-get update` + `install` with `DEBIAN_FRONTEND=noninteractive`), `_system_sudo` (one prompt; failure names what an admin must install)
- `brew_missing` (`<prefix>/opt/<name>`, tap names by last segment), `cask_missing` (`<prefix>/Caskroom/<name>`, macOS only)

### `lib/installer.sh`
- `install()`: fails before any work when a resolved package is unsupported; `_install_declared_deps` before the per-package loop (skipped with `-c`): system → brew (`brew_require_owner`) → cask, missing only, one call each; casks on Linux produce a note
- `install_single_package`: records declared formulas/casks installed now or already recorded by ppm (`_dep_recorded_by_ppm`)
- `_mise_install_declared` after the loop: `mise install <tools>` from the packages' toml `[tools]` keys, run from `$HOME`
- `remover`: sets `PPM_CURRENT_PACKAGE`; unless `keep_tracker`, `_remove_installed_deps` then the tracker. Uninstalls recorded formulas/casks not recorded or declared by another installed package (`_dep_needed_elsewhere`); owner only; system packages are reported, never removed
- `install -r` calls `force=true keep_tracker=true remover`

### Other
- `lib/core.sh`: `install_dep` logs a deprecation note under `--debug`
- `CLAUDE.md`: schema and behavior documented in the package.yml section

## Test Results

### Unit tests (macOS, bash 3.2, temporary packages/trackers, platform faked as debian where noted)

All pass: list/map/exact/`linux` fallback/no macos key on debian; `system` map without the distro → 1 on debian, no error on macOS; `meta_supported` for `[macos]`, `[linux]`, absent; `cask_missing` empty on Linux; `_list_add` order/dedupe/empties; tracker merge across installs, valid YAML, no empty `installed_deps`, cask records; `_dep_needed_elsewhere` (self ignored, declared elsewhere, recorded elsewhere); mise tool names from toml (`node`, `npm:prpm`).

Host regressions: `ppm deps git zsh nvim` order unchanged, `ppm list` 61 packages, `ppm show stack/zsh`, `ppm install -r ppm/ppm` writes a correct tracker.

Tool checks: `brew install --yes` accepted by Homebrew 7.0.1 (not listed in `--help`; a dry run proceeds while an invalid flag prints usage) on macOS and Linux; yq 4.53.6 reads TOML (`-p toml -oy`).

### Container (`ppm-debian` from the `debian-brew` snapshot, working tree at `/src`, scratch repos `t` and `u` inside the container)

| # | Scenario | Result |
|---|---|---|
| 1 | `install t/cow` (brew: cowsay) | cowsay installed; tracker `installed_deps.brew: [cowsay]` |
| 2 | install again | no brew install; record kept |
| 3 | `install -r t/cow` | no uninstall; record kept |
| 4 | `t/cow2` also declares cowsay (already present) | cow2 records cowsay; removing cow keeps it; removing cow2 uninstalls it |
| 5 | `brew install hello` by hand, then install/remove `t/hello` | not recorded; hello still installed after remove |
| 6 | `system.debian: [sl]` | 1 sudo prompt, installed; rerun 0 prompts, nothing to install; remove leaves sl with a message |
| 7 | `brew.macos: [jq]` + `system.linux: [jq]` | jq from apt, not brew; `show` prints `System (debian): jq` and no Brew line |
| 8 | `system.fedora` only | exit 1: `declares system packages, but none for debian` |
| 9 | `platforms: [macos]` | `install t/maconly` refused; `install -f u/` skips it and installs `u/plain` |
| 10 | stowed `conf.d` toml with `jq = "1.7.1"` | `mise install jq`; `mise where jq@1.7.1` succeeds |
| 11 | `install -c t/figlet` | no formula installed |
| 12 | other user, `install t/figlet` | exit 1: `Homebrew is owned by owner; ask them to run: brew install figlet`; figlet not installed; prefix still owned by owner |

Test harness issues (not ppm): tracker checks for 1–4 ran `yq` without brew on PATH and read empty values, and case 12's repo setup ran as the wrong user. Both scenarios were re-run by hand with the fixes and pass as listed.

## Findings for Later Plans

- Plan 04 can now declare dependencies; `install_dep` still works (deprecated) until migration finishes
- Sudo prompts are now one per batch for declared system packages; remaining per-hook prompts come from packages still calling `install_dep`/`sudo` (plan 04)

## Context Updates

- The container keeps the scratch `t`/`u` repos and their trackers; `ppm container reset debian brew` returns to a clean state
- Correction after the unit was first written: Homebrew installs casks on Linux too (`brew install --dry-run --cask claude-code` and `1password-cli` both succeed in the Debian container). `cask_missing` no longer returns early on Linux and the "casks are macOS-only" note is gone; a GUI cask is declared under `cask: {macos: [...]}` instead
