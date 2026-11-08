# rivalcfg for Void Linux

rivalcfg packaged for Void Linux: the command line tool and Python library for
configuring SteelSeries gaming mice. This package installs the udev rules in
`/usr/lib/udev/rules.d` with `MODE="0660"`, `GROUP="plugdev"` and
`TAG+="uaccess"`, so the CLI accepts system-wide rules and does not need a
`0666` file in `/etc`.

## Why this exists

At the time of writing, the official Void repositories do not carry `rivalcfg`.
Upstream writes its rules to
`/etc/udev/rules.d/99-steelseries-rival.rules` and makes the device
world-writable. Void packages system rules in `/usr/lib/udev/rules.d`, so the
CLI reported missing rules even when the package had installed them.

This package closes both gaps. The rules go to the system directory, access is
granted to the `plugdev` group, and `rivalcfg --print-debug` checks the same
path the package uses.

It is a personal project, not an official Void package, and there is no support
promise behind it. Bug reports and patches are welcome anyway.

## Install

### Signed XBPS repository

```sh
# add the repository
sudo sh -c 'echo "repository=https://khame.github.io/void-packages/" > /etc/xbps.d/10-khame.conf'

# sync the index, answer Y when xbps asks to import the key
sudo xbps-install -S

sudo xbps-install rivalcfg
```

The key sits inside the signed index, so xbps imports it without an extra
step. It ends up as
`/var/db/xbps/keys/35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66.plist`.
The same key is published as
[Khame-void-repo.pub](https://khame.github.io/void-packages/Khame-void-repo.pub),
and you can compute the fingerprint from it:

```sh
curl -fsSO https://khame.github.io/void-packages/Khame-void-repo.pub
ssh-keygen -E md5 -lf <(ssh-keygen -i -m PKCS8 -f Khame-void-repo.pub)
```

The result should match the fingerprint above. It shows the key and the
fingerprint belong together, though both come from the same site.

Updates come with `sudo xbps-install -Su`. Only x86_64 is built.

### Release asset

Import the key first, then:

```sh
sudo xbps-install --binary-package-url https://github.com/Khame/rivalcfg-void-linux/releases/download/v4.17.0/rivalcfg-4.17.0_6.x86_64.xbps
```

This is for one install. Updates are manual. The release also has the `.sig2`
file and the public key, fingerprint
`35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66`.

### From source

The package lives in
[Khame/void-packages](https://github.com/Khame/void-packages), not here.

```sh
git clone --depth=1 https://github.com/void-linux/void-packages.git
cd void-packages
./xbps-src binary-bootstrap

git clone --depth=1 https://github.com/Khame/void-packages.git /tmp/khame-void-packages
cp -a /tmp/khame-void-packages/srcpkgs/rivalcfg srcpkgs/

./xbps-src pkg rivalcfg
sudo xbps-install --repository=hostdir/binpkgs rivalcfg
```

Do this when you want to change the template or build another revision.

## Device access

The package installs `/usr/lib/udev/rules.d/99-steelseries-rival.rules`. Reload
the rules and add your user to `plugdev`:

```sh
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo usermod -aG plugdev "$USER"
```

Log out and back in after changing the group. Check the result with:

```sh
rivalcfg --print-debug | grep -i udev
ls -l /dev/hidraw*
```

The device node must belong to `plugdev` with mode `0660`. No reboot is needed.

## What gets installed

- `/usr/bin/rivalcfg`
- the `rivalcfg` Python package under
  `/usr/lib/python3.*/site-packages/rivalcfg/`
- `/usr/lib/udev/rules.d/99-steelseries-rival.rules`
- the upstream license text in the package metadata

The rules file is generated from rivalcfg 4.17.0 and covers the mice upstream
supports, including the Sensei 310 (`1038:1722`).

## Patches

Two patches against rivalcfg 4.17.0:

1. `relax-flit-core-version.patch` drops the `flit_core <4` upper bound from
   `pyproject.toml`, so the package builds with the flit_core version Void
   ships.
2. `udev-system-rules.patch` adds `RULES_FILE_PATHS` with `/etc` first and
   `/usr/lib` second, writes `MODE="0660"`, `GROUP="plugdev"` and
   `TAG+="uaccess"`, and teaches `rivalcfg/__main__.py` and
   `rivalcfg/debug.py` to use the first rules file that exists.

Full reasoning, verification steps and drop conditions:
[PATCHES.md](https://github.com/Khame/void-packages/blob/main/srcpkgs/rivalcfg/PATCHES.md).
Nothing has been sent upstream.

## Requirements

Void Linux x86_64 with glibc, `python3` and `python3-hidapi`. musl is untested.
The CLI works without a desktop session; only device permissions matter.

## Tested on

ASUS ROG Strix G513QM, Void x86_64 with glibc, kernel 7.3.1, XBPS 0.60.7 and
runit. The mouse is a SteelSeries Sensei 310 (`1038:1722`). `rivalcfg
--print-debug` reports the installed system rules as current, and the device
node belongs to `plugdev` with mode `0660`.

## Repository layout

- `README.md`: this file
- `VOID_PACKAGING.md`: how the template is put together, how a release is built
- `LICENSE`: MIT for what I wrote here

The build files are not copied here. The single copy is
[`srcpkgs/rivalcfg/`](https://github.com/Khame/void-packages/tree/main/srcpkgs/rivalcfg).

## Upstream

Project: [flozz/rivalcfg](https://github.com/flozz/rivalcfg),
[homepage](https://rivalcfg.flozz.org/)

Packaged version: `4.17.0_6` (upstream release tarball from PyPI)

License: WTFPL

## Author

Khame ([@Khame](https://github.com/Khame)),
[khame.dev@protonmail.com](mailto:khame.dev@protonmail.com)
