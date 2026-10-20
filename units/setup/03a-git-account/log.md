# 03a-git-account — log

## 2026-10-19 — plan 03a: git account and ssh-key (stack 93b2572, merged the same day)

- stack/ssh: `ssh-key list|add|public`. list: KEY STATUS FILE (local: ~/.ssh with the private
  key; agent: ssh-add -L, deduped, local wins); --json key status file type comment public.
  add does nothing when any key exists; else ~/.ssh/id_<type>, no passphrase with -y or no tty.
- stack/git: `git-account` (`git account add|list|remove`). Accounts in git's global config,
  account.<host>.user. add: reads https://<host>/<user>.keys (curl, 10 s), records, makes a key
  through ssh-key add when none (-y/-i passed on), then gh ssh-key add (github.com, gh signed
  in as that user, admin:public_key) or prints the key and the settings URL, exit 0.
- Deviation: .keys is read before recording, so a 404 records nothing.
- Host default: the only recorded host, else github.com. -i: the recorded user is the lone
  candidate, so it is the default; the host shows none (needs cli.sh's current-value marking).
- git depends on ssh. zcomp ssh-key / git-account (zsh's _git finds _git-account).
- Found: a set ANFS_TABLE overrides --json in table.sh, so git-account calls ssh-key with
  ANFS_TABLE=. Bash 5.2 expands ~ in ${x/#$HOME/~}: use anfs_display.
- Tests: ssh-key.bats 7, account.bats 14, onboard.bats 14 green; core tests/all 391/391 (pmm,
  pcm off). cli.sh unchanged.
- To do by hand: ppm install stack/git stack/ssh, then git account add -i.
