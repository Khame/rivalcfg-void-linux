# Void Linux packaging: rivalcfg maintainer notes

This repository holds the install docs and release notes. The packaging source
is not here. The single copy is `srcpkgs/rivalcfg/` in
[Khame/void-packages](https://github.com/Khame/void-packages).

Users should read [README.md](README.md) instead.

## Where the files live

| Path (in `Khame/void-packages`) | What it is |
|---|---|
| `srcpkgs/rivalcfg/template` | the `xbps-src` template: version, revision, dependencies, `distfiles`, `checksum`, `post_install` |
| `srcpkgs/rivalcfg/patches/relax-flit-core-version.patch` | drops the `flit_core <4` upper bound |
| `srcpkgs/rivalcfg/patches/udev-system-rules.patch` | accepts `/usr/lib/udev/rules.d` and writes `plugdev` rules |
| `srcpkgs/rivalcfg/files/99-steelseries-rival.rules` | generated udev rules installed by the package |
| `srcpkgs/rivalcfg/INSTALL.msg` | tells the user to reload rules and join `plugdev` |
| `srcpkgs/rivalcfg/PATCHES.md` | patch notes, verification steps and drop conditions |
| `docs/` | signed packages, `x86_64-repodata`, the public key and the landing page |

## About the template

`build_style=python3-pep517`. The build dependencies are `python3-flit_core`
and `python3-installer`. Runtime dependencies are `python3` and
`python3-hidapi`.

`distfiles` points to the upstream PyPI sdist and `checksum` is its SHA-256.
`post_install` installs `99-steelseries-rival.rules` in
`/usr/lib/udev/rules.d` with `MODE="0660"`, `GROUP="plugdev"` and
`TAG+="uaccess"`.

The package installs no systemd units. udev reloads the rules, and `plugdev`
membership grants access.

## Patches

Both patches are documented in `PATCHES.md` in the void-packages tree. Check
them on every upstream version bump.

1. `relax-flit-core-version.patch` drops the `flit_core <4` pin so the package
   builds with the flit_core version Void ships.
2. `udev-system-rules.patch` adds `RULES_FILE_PATHS` with `/etc` first and
   `/usr/lib` second, writes `MODE="0660"`, `GROUP="plugdev"` and
   `TAG+="uaccess"`, and teaches `rivalcfg/__main__.py` and
   `rivalcfg/debug.py` to use the first rules file that exists.

Nothing has been sent upstream.

## Building locally

```sh
git clone --depth=1 https://github.com/void-linux/void-packages.git
cd void-packages
./xbps-src binary-bootstrap

git clone --depth=1 https://github.com/Khame/void-packages.git /tmp/khame-void-packages
cp -a /tmp/khame-void-packages/srcpkgs/rivalcfg srcpkgs/

./xbps-src pkg rivalcfg
./xbps-src check rivalcfg
sudo xbps-install --repository=hostdir/binpkgs rivalcfg
```

The package ends up in `hostdir/binpkgs/`. Run `xbps-src` as a regular user.

## Publishing a release

1. Compare the upstream version with `version=` in the template. If it has not
   changed, stop here.
2. Update `version=`, `distfiles` and `checksum`, then check both patches.
3. Raise `revision=` when you change only the template and the upstream version
   stays the same.
4. Run `xlint srcpkgs/rivalcfg/template` and fix everything it reports.
5. Build and test. Check `rivalcfg --print-debug` and the device node.
6. Sign the package:

   ```sh
   pkg=$(ls -t hostdir/binpkgs/rivalcfg-*.xbps | head -1)
   xbps-rindex -S --privkey="$HOME/.config/Khame-void-repo-private.pem" \
     --signedby="Khame <khame.dev@protonmail.com>" "$pkg"
   ```

7. Delete the old package and its signature from `docs/`, copy the new pair in
   and rebuild the index. Never pass a `rivalcfg-*.xbps` glob to
   `xbps-rindex -a`; it also matches the old package and may register that
   version instead.

   ```sh
   rm -f docs/rivalcfg-*.xbps docs/rivalcfg-*.xbps.sig2
   cp "$pkg" "$pkg.sig2" docs/
   xbps-rindex -a "docs/$(basename "$pkg")"
   xbps-rindex -s --privkey="$HOME/.config/Khame-void-repo-private.pem" \
     --signedby="Khame <khame.dev@protonmail.com>" docs
   ```

8. Commit and push `Khame/void-packages`. GitHub Pages serves `docs/` from
   `main`, so the repository appears at
   `https://khame.github.io/void-packages/`.
9. Attach the same `.xbps` to a release in this repository and update the
   version numbers in `README.md`.

Packages are signed with the Khame repository RSA key. The private key stays
outside both repositories, for example at
`~/.config/Khame-void-repo-private.pem` with mode `600`. The public key is
copied to `docs/Khame-void-repo.pub` and `xbps-rindex -s` embeds it in the
index. Nobody has to put a `.pem` into `/var/db/xbps/keys/`: `xbps` imports the
key from the index on the first `xbps-install -S` and stores it as
`<fingerprint>.plist`, currently
`35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66.plist`. I never support
`allow-unsigned`.

`x86_64-repodata` has no signature of its own. `xbps` checks each package
against the imported key. Only x86_64 is published.

## Notes

- The package installs into `/usr/lib/python3.14/site-packages`, so it must be
  rebuilt when Void moves to a new Python version.
- The udev rules file is generated from rivalcfg 4.17.0. Regenerate it with
  `rivalcfg --update-udev` when the device list changes, then review the diff.
- There is no CI on these repositories. Releases are built by hand.

## Author and license

Khame ([@Khame](https://github.com/Khame)),
[khame.dev@protonmail.com](mailto:khame.dev@protonmail.com)

MIT for the material written here. rivalcfg is WTFPL.
