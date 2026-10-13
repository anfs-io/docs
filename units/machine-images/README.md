# Images: a first walk

You have installed anfs and `pmm` (`ppm install core/pmm`). This walk takes about fifteen
minutes, most of it one Debian install, and ends with you having built an image, added to it,
run machines from it, changed a shipped image without forking it, and taught pmm a new kind of
build step.

It covers what plans 02 and 03 of this unit built: the image directory and the provisioners.
Every command below was run on 2026-10-13 on an Apple silicon Mac, on the `machine-images`
branch, and the output shown is what it printed. Lists are shown as a terminal prints them;
piped, they have no header.

Until that branch is merged, see "Trying the branch" at the end.

## The idea

An **image** is built once. Any number of **machines** run from the build, each with its own
disk, as instances run from an AMI. An image is one directory, and that directory holds
everything a build reads:

```
machines/images/<name>/
  image.yml                    what it is
  preseed.cfg | kickstart.ks   how the installer is answered (a root image)
  provision/                   steps, run in name order after the install
  files/                       copied into the guest for the steps
  test.sh                      what proves a build is good
  README.md
```

## 1. What is there

```
$ pmm image list
NAME             SOURCE  ARCH  STATE    KEY ...
anfs-dev         core    -     unbuilt  -
anfs-test-macos  core    -     unbuilt  -
debian-13        core    -     unbuilt  -
fedora-44        core    -     unbuilt  -
```

Images come from your sources, like everything else in anfs. `core` ships these four.

```
$ pmm image show debian-13
core/debian-13
  definition  ~/.local/share/anfs/sources/core/machines/images/debian-13
  backend     qemu
  distro      debian
  arch        arm64,amd64
  user        pmm
  answer      .../machines/images/debian-13/preseed.cfg
  vars        HOSTNAME=debian
              TIMEZONE=UTC
              ...
  iso arm64   debian-13-arm64  (~/.cache/anfs/isos/debian-13.7.0-arm64-netinst.iso, present)
  iso amd64   debian-13-amd64  (~/.cache/anfs/isos/debian-13.7.0-amd64-netinst.iso, missing)
  provision   10-base.sh       .../machines/images/debian-13/provision/10-base.sh
              finalize         (pmm) ~/.local/lib/pmm/defaults/finalize.sh
  test        .../machines/images/debian-13/test.sh
  logs        ~/.local/state/pmm/logs/debian-13-arm64/
```

`show` is the plan of a build: every file it will read, in the order it will use them. Read it
before you change an image and again after. `pmm image cd debian-13` takes you to the
directory.

The ISO is named by an id. The ids are in a catalogue:

```
$ pmm iso list
ID               NAME                         ARCH   FILE                             STATE    SIZE
debian-13-arm64  Debian 13.7.0 arm64 netinst  arm64  debian-13.7.0-arm64-netinst.iso  present  701M
debian-13-amd64  Debian 13.7.0 amd64 netinst  amd64  debian-13.7.0-amd64-netinst.iso  missing  -
...
```

A build downloads what is missing. `pmm iso pull debian-13-arm64` does it ahead of time.

## 2. Build it

```
$ pmm image build --test debian-13
== build core/debian-13 (arm64) -> 393470e5bd6a8b69
Installing debian (hvf); log: ~/.local/state/pmm/logs/debian-13-arm64/install.log
  installing, 1m: Configuring linux-image-6.12.107+deb13-arm64 (arm64)
Installed in 116s
Booting; waiting for ssh
== 10-base.sh (14s)
== finalize (1s)
Shutting down
Built core/debian-13 (arm64) in 160s: ~/.local/share/pmm/images/debian-13/arm64/393470e5bd6a8b69.qcow2
== test core/debian-13 (arm64, 393470e5bd6a8b69) with .../debian-13/test.sh
  | ok  built by pmm: image=debian-13 arch=arm64 built=2026-10-13T19:56:46Z
  | ok  ssh is running
  | ok  host keys were generated on this boot
  | ok  machine-id was generated on this boot
  | ok  dhcpcd looks after every interface
  | ok  qemu-guest-agent installed
  | ok  pmm has passwordless sudo
Tested core/debian-13 (arm64)
```

What happened: the installer ran unattended from the ISO with `preseed.cfg`, the disk booted,
`provision/10-base.sh` ran as root in it, pmm's `finalize` step cleaned it, and `test.sh` ran on
a throwaway boot of the result.

`393470e5bd6a8b69` is the **build key**: a hash of everything the build read. Run the same
command again and nothing is built:

```
$ pmm image build debian-13
core/debian-13 (arm64) is up to date (393470e5bd6a8b69)
```

If the installer hangs, `--console` shows it. `-v` shows the steps' output.

## 3. An image of your own

Almost always you want a **child**: another image's build with your steps on top.

```
$ pmm image new web --from debian-13
Created local/web: ~/.config/pmm/machines/images/web
```

That is an `image.yml` (`from: debian-13`), an empty `provision/` and a README, in your `local`
source. `--source user` would write it into your own repo instead.

Give it a file, a step that uses the file, and a test:

```bash
cd "$(pmm image path web)"
mkdir files && echo "hello from web" > files/motd

cat > provision/50-motd.sh <<'EOF'
#!/bin/bash
set -euo pipefail
install -m 644 "$ANFS_IMAGE_FILES/motd" /etc/motd
EOF

cat > test.sh <<'EOF'
#!/bin/bash
set -euo pipefail
grep -q "hello from web" /etc/motd && echo "ok  motd"
EOF
```

A `.sh` step runs as root in the guest. `ANFS_IMAGE_FILES` is where `files/` was copied; the
other `ANFS_IMAGE_*` variables say what is being built (`NAME`, `ARCH`, `DISTRO`, `USER`).

```
$ pmm image validate web
local/web ok
1 image(s): 0 error(s), 0 warning(s)

$ pmm image show web
local/web
  definition  ~/.config/pmm/machines/images/web
  from        core/debian-13
  ...
  provision   10-base.sh       .../core/machines/images/debian-13/provision/10-base.sh
              50-motd.sh       ~/.config/pmm/machines/images/web/provision/50-motd.sh
              finalize         (pmm) ~/.local/lib/pmm/defaults/finalize.sh
  test        ~/.config/pmm/machines/images/web/test.sh
```

`show` lists the parent's steps and yours, in the order a machine got them.

```
$ pmm image build --test web
core/debian-13 (arm64) is up to date (393470e5bd6a8b69)
== build local/web (arm64) -> ab67d1fffde554fe
Booting; waiting for ssh
== 50-motd.sh (0s)
== finalize (1s)
Shutting down
Built local/web (arm64) in 30s: ~/.local/share/pmm/images/web/arm64/ab67d1fffde554fe.qcow2
== test local/web (arm64, ab67d1fffde554fe) with ~/.config/pmm/machines/images/web/test.sh
  | ok  motd
Tested local/web (arm64)
```

Thirty seconds: a child is an overlay on its parent's disk, so there is no install.

## 4. Machines from it

```
$ pmm run web --name web1 --cpus 2 --memory 2G
Created web1 from local/web (ab67d1fffde554fe); pmm up web1
Started web1 (arm64, hvf); waiting for ssh
web1 is up: pmm shell web1  (ssh -p 2222 pmm@127.0.0.1 -i ~/.local/share/pmm/keys/debian-13/id_ed25519)

$ pmm exec web1 -- cat /etc/motd
hello from web
```

`pmm shell web1` logs you in. Save its disk, break it, and go back:

```
$ pmm snapshot create web1 clean
Saved snapshot clean of web1 (pmm reset web1 clean)

$ pmm exec web1 -- sudo rm /etc/motd
$ pmm snapshot list web1
NAME   BUILD
clean  ab67d1fffde554fe

$ pmm reset -y web1 clean
Reset web1 to clean

$ pmm exec web1 -- cat /etc/motd
hello from web
```

A machine holds the build it runs on:

```
$ pmm image remove -y web
pmm: web (arm64) has machines on it: web1 web1@clean; pmm remove them first, or pass -f

$ pmm remove -y web1
Removed web1
```

For a machine you only want for a moment, `--rm` removes it when it stops:

```
$ pmm run debian-13 --rm --name tmp1
$ pmm down tmp1
Stopping tmp1
Removed tmp1 (it was run with --rm)
```

## 5. Change a shipped image without forking it

You want every Debian machine of yours to have something core's `debian-13` lacks. Make an
image of the **same name**, built on that name:

```
$ pmm image new debian-13 --from debian-13
Created local/debian-13: ~/.config/pmm/machines/images/debian-13

$ printf '#!/bin/bash\nset -euo pipefail\necho mine > /etc/mine\n' \
    > "$(pmm image path debian-13)/provision/50-mine.sh"

$ pmm image list
NAME       SOURCE  ARCH   STATE     KEY
debian-13  local   arm64  stale     393470e5bd6a8b69
debian-13  core    -      shadowed  -
web        local   arm64  built     ab67d1fffde554fe
```

`debian-13` now means yours; `from: debian-13` inside it is the next one down, core's. Build
it, then `web`, which is built on that name:

```
$ pmm image build debian-13
core/debian-13 (arm64) is up to date (393470e5bd6a8b69)
== build local/debian-13 (arm64) -> aefbe0c5e2377681
== 50-mine.sh (0s)
Built local/debian-13 (arm64) in 31s

$ pmm image build web
core/debian-13 (arm64) is up to date (393470e5bd6a8b69)
local/debian-13 (arm64) is up to date (aefbe0c5e2377681)
== build local/web (arm64) -> c149ef58c1ae128d
Built local/web (arm64) in 22s

$ pmm run web --name web2
$ pmm exec web2 -- cat /etc/mine
mine
$ pmm remove -y web2
```

Core's build was not redone, and `web` got a new key because its parent's changed. The old
`web` build is now superseded:

```
$ pmm image prune
Deleting ~/.local/share/pmm/images/web/arm64/ab67d1fffde554fe.qcow2
Pruned (1 superseded disks)
```

This adds to an image. To change what its installer does (partitions, mirror, packages), copy
it under a new name and edit the copy: `pmm image new my-debian --copy debian-13`.

## 6. A new kind of step

A step's extension names the **provisioner** that runs it. pmm ships `sh` and `anfs`. A file
nothing can run is refused, never skipped:

```
$ pmm image new site --from debian-13
$ touch "$(pmm image path site)/provision/60-play.yml"
$ pmm image validate site
local/site
  error    provision/60-play.yml: nothing provides .yml steps: install the package that does
$ rm "$(pmm image path site)/provision/60-play.yml"
```

A provisioner is an executable named for its extension, with three verbs. This one makes a
`.tgz` a step: the archive, unpacked over `/` in the guest.

```bash
mkdir -p ~/.config/pmm/provisioners
cat > ~/.config/pmm/provisioners/tgz <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
source "$HOME/.local/lib/anfs/bash.sh"
case "${1:-}" in
  check)       command -v tar >/dev/null || { echo "tar is not installed"; exit 1; } ;;
  fingerprint) echo "$2" ;;
  run)         $ANFS_IMAGE_GUEST_EXEC tar -C / --no-same-owner -xzf - < "$2" ;;
  *)           echo "tgz: unknown verb '${1:-}'" >&2; exit 1 ;;
esac
EOF
chmod +x ~/.config/pmm/provisioners/tgz
```

| Verb | Is asked |
| --- | --- |
| `check` | Can you run on this host? Print what is missing |
| `fingerprint <step>` | Which files does this step's result depend on? They go into the build key |
| `run <step>` | Do it. `$ANFS_IMAGE_GUEST_EXEC <cmd>` runs a command as root in the guest; `$ANFS_IMAGE_GUEST_PUSH <local> <guest path>` copies |

Use it:

```
$ mkdir -p /tmp/t/srv/site && echo "<h1>hi</h1>" > /tmp/t/srv/site/index.html
$ COPYFILE_DISABLE=1 tar -C /tmp/t -czf "$(pmm image path site)/provision/50-site.tgz" srv

$ pmm image show site
  from        local/debian-13 <- core/debian-13
  provision   10-base.sh       .../core/machines/images/debian-13/provision/10-base.sh
              50-mine.sh       ~/.config/pmm/machines/images/debian-13/provision/50-mine.sh
              50-site.tgz      ~/.config/pmm/machines/images/site/provision/50-site.tgz
              finalize         (pmm) ~/.local/lib/pmm/defaults/finalize.sh

$ pmm image build -v site
== build local/site (arm64) -> 9f3341b5c1b26619
== 50-site.tgz
== finalize
Built local/site (arm64) in 30s

$ pmm run site --rm --name site1
$ pmm exec site1 -- cat /srv/site/index.html
<h1>hi</h1>
$ pmm down site1
```

pmm did not change to gain it. A package ships one the same way, at
`home/.local/lib/pmm/provisioners/<ext>`, with what it needs (`brew: [ansible]`) in its
`package.yml`.

## 7. The toolkit in an image

The other shipped provisioner installs anfs in the guest. `anfs-dev` is Debian with your
machine's sources installed, for trying a change in a real VM:

```yaml
# machines/images/anfs-dev/provision/50-toolkit.anfs
sources: host        # this machine's source clones are copied in
                     # pushed (the default): the guest fetches install.sh and clones them
# repo: git@github.com:me/dots.git     install.sh --repo
# packages: [stack/zsh]                installed after the toolkit
```

```
$ pmm image build anfs-dev        # about two minutes on top of debian-13
$ pmm run anfs-dev --rm
$ pmm shell anfs-dev
```

Not run for this walk; plan 03's log has the run. Only `sources: host` has been run in a guest.

## 8. Clean up

```
$ pmm image remove -y site web debian-13
```

`pmm implode -y` removes everything pmm made: machines, builds, and `~/.config/pmm`, which is
where your `local` definitions and provisioners are. Keep what you want in a source first.

## What to try that this walk does not

- `pmm image validate` on an image in the old shape (a `scripts/` directory, top-level
  `memory:`, an inline ISO URL): each is refused with what replaces it.
- `pmm image new my-debian --copy debian-13`, change a `vars:` value, build: a root of your own.
- `pmm image export -c web`: the build as one compressed qcow2.
- Fedora (`fedora-44`), an amd64 build, a Linux host, and the macOS image
  (`anfs-test-macos`): none of these has been run on this branch yet.

## Trying the branch

Before `machine-images` is merged, this gives one shell a `pmm` from the branch with its own
HOME, sockets and ports. Your install, your machines and tart's store are not involved; the ISO
store is shared so the ISO downloads once.

```bash
W=~/anfs-dev/machine-images          # the branch's worktree
T=/tmp/pmm-try-$(id -u)
export ANFS_ISO_HOME="$HOME/.cache/anfs/isos"
export HOME="$T/home"
export XDG_CONFIG_HOME="$HOME/.config" XDG_DATA_HOME="$HOME/.local/share" \
       XDG_STATE_HOME="$HOME/.local/state" XDG_CACHE_HOME="$HOME/.cache"
unset ANFS_CONFIG_HOME ANFS_DATA_HOME ANFS_STATE_HOME ANFS_CACHE_HOME \
      PMM_CONFIG_HOME PMM_DATA_HOME PMM_STATE_HOME PMM_CACHE_HOME PMM_MACHINES_HOME PMM_IMAGES_HOME
mkdir -p "$HOME/.local/lib" "$HOME/.local/bin" "$HOME/.config/anfs" \
         "$HOME/.local/share/anfs/sources" "$T/sock"
ln -sfn "$W/packages/anfs/home/.local/lib/anfs" "$HOME/.local/lib/anfs"
ln -sfn "$W/packages/pmm/home/.local/lib/pmm"   "$HOME/.local/lib/pmm"
ln -sfn "$W/packages/pmm/home/.local/bin/pmm"   "$HOME/.local/bin/pmm"
ln -sfn "$W" "$HOME/.local/share/anfs/sources/core"
printf '%s  core\n' "$W" > "$HOME/.config/anfs/user.list"
export PMM_SOCK_DIR="$T/sock" PMM_SSH_PORT_BASE=2500 TART_HOME="$T/no-tart"
export PATH="$HOME/.local/bin:$PATH"
```

Run it in a `bash` you started for the purpose and `exit` when done. In it, ssh ports start at
2500 where the walk shows 2222, and `pmm implode -y` at the end removes only what is under `$T`.
