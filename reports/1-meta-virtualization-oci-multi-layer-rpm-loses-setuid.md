# meta-virtualization: multi-layer OCI with `package_rpm` loses every setuid/setgid bit

With `OCI_LAYER_MODE = "multi"` and `package_rpm`, the image builds without warnings but has
no setuid/setgid files; `sudo` is `0711` and can't work. Found 2026-10-08.

## Environment

- "Build the image" in `setup/container-yocto-builder.md`, with `package_rpm`.

## Reproduce

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive registry/configurations/meta-virtualization-master.conf.json nodistro-virt machine/qemux86-64
. bitbake-builds/yocto-builder/build/init-build-env
echo 'PACKAGE_CLASSES = "package_rpm"' >> conf/local.conf
bitbake container-yocto-builder
```

Files with `mode & 06000`:

| | setuid/setgid files | `usr/bin/sudo` |
|---|---|---|
| Regular rootfs (`container-yocto-builder-qemux86-64.rootfs.tar.bz2`) | 16 | `---s--x--x 0/0` |
| OCI image, `package_rpm` | 0 | `0711 0:0` |
| OCI image, `package_ipk` + workarounds | 16 | `4711 0:0` |

- The 16: `chage`, `chfn.shadow`, `chsh.shadow`, `gpasswd`, `mount.util-linux`, `newgidmap`,
  `newgrp.shadow`, `newuidmap`, `passwd.shadow`, `ping.inetutils`, `screen-5.0.2`,
  `su.shadow`, `sudo`, `traceroute.inetutils`, `umount.util-linux`,
  `dbus-daemon-launch-helper`.
- Layer rootfs: `-rwx--x--x ${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base/usr/bin/sudo`
- Ownership differs too, but deliberately: every entry in every OCI image is `0:0`; the
  regular rootfs has 4 others (e.g. `/var/lib/dbus` messagebus, `/var/lib/dhcpcd`,
  `/var/spool/mail` group mail). Commit `f7eb4abb` ("image-oci: don't preserve ownership
  ..."); single-layer mode copies with `--no-preserve=ownership` too.

## Cause

Likely, not verified (see TODO):

- Layers are installed outside pseudo's tracked paths, as in
  `1-meta-virtualization-oci-multi-layer-setuid-unreadable.md`.
- `-rwx--x--x` is what pseudo leaves on disk for a tracked file: `u+rw` added, setuid only in
  its database.
- rpm 6.0.2 `fsmSetmeta()` (`lib/fsm.cc`) chowns, then `fchmod(fd, mode & 07777)`. pseudo
  appears to handle this fd-based call even on an untracked path.
- The later path-based rsync reads the real `0711`; umoci packs that.
- opkg sets modes by path, which pseudo passes through, so with ipk the real `4111` stays.

## Possible fix

- Install and copy the layers consistently: all under pseudo, or all without it.

## TODO

- [ ] Verify the cause: trace rpm's `fchmod` under pseudo on an untracked path.
