# Test setup: `container-yocto-builder`

Shared by the `container-yocto-builder`, `meta-virtualization-oci-multi-layer` and
`vcontainer-vpdmn` reports. Not a bug report.

## Build the image

Environment, last seen at:

- Host: Debian GNU/Linux 13 (trixie), kernel 6.12.88+deb13-amd64, no access to `/dev/kvm`
- bitbake `37b9c56ae3b2294ad170859a638c879910c82c59` (BB_VERSION 2.20.0)
- openembedded-core `cc848c403aa3832453f1adfaa45411b8b72b9363`
- meta-openembedded `ca3cf5fe14bed658890bac643cced054932fcb51` (meta-oe, meta-python,
  meta-networking, meta-filesystems)
- meta-virtualization `24c622baf9990d6907b1b325354828df8f314ff2`
- `nodistro`, `qemux86-64` (`TUNE_FEATURES = "m64 x86-64-v3"`), `package_ipk`

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive registry/configurations/meta-virtualization-master.conf.json nodistro-virt machine/qemux86-64
. bitbake-builds/yocto-builder/build/init-build-env
bitbake container-yocto-builder
```

- Fails until the workarounds of `1-meta-virtualization-oci-multi-layer-setuid-unreadable.md`
  and `2-meta-virtualization-oci-multi-layer-opkg-lists-symlinks.md` are in a bbappend.
- "The image" in other reports is this build with both workarounds, combined in one prefunc
  as in `meta-virt-setup/recipes-extended/images/container-yocto-builder.bbappend`.

## vcontainer SDK (vpdmn)

Environment:

- bitbake `37b9c56ae3b2294ad170859a638c879910c82c59`
- openembedded-core `aba08734931428eb3c853bb497d8e21127ad746e` (master, 2026-10-08)
- meta-openembedded `ca3cf5fe14bed658890bac643cced054932fcb51`
- meta-virtualization `24c622baf9990d6907b1b325354828df8f314ff2`
- meta-yocto `49a2b44583968d321b3195bfd0cb2232dff13950`; meta-poky in `bblayers.conf`
  (required before meta-virtualization `86703e65`)
- podman in the VM: 6.2.0-dev, storage driver `vfs`

Add to `conf/local.conf` (or enable the `meta-virt-setup/vcontainer` fragment):

```
DISTRO_FEATURES:append = " vcontainer"
BBMULTICONFIG += "vruntime-x86-64"
VCONTAINER_ARCHITECTURES = "x86_64"
VCONTAINER_INCLUDE_VDKR = "0"
```

From the build directory:

```sh
bitbake vcontainer-tarball
tmp/deploy/sdk/vcontainer-standalone.sh -d $PWD/vcontainer-sdk -y
. vcontainer-sdk/init-env.sh
O=tmp/deploy/images/qemux86-64/container-yocto-builder-latest-oci
```

- vpdmn commands in other reports run in this shell.
- `vrunner.sh` is shared with vdkr; the vpdmn bugs in it probably affect vdkr too (not
  tested).

## Run the image in vpdmn

Import needs the workarounds of `4-vcontainer-vpdmn-state-disk-fixed-2g.md` and
`3-vcontainer-vpdmn-input-disk-sized-from-symlink.md`:

```sh
vpdmn --no-kvm --no-daemon vimport $O/ yocto-builder:latest
vpdmn --no-kvm run --privileged -d --name yb -e BUILDER_UID=1000 -e BUILDER_GID=1000 yocto-builder:latest
```

- The entrypoint creates `builder`, systemd reaches `running` with no failed units,
  `sudo -n id` works for `builder`.
- Without `--privileged`: the container exits with 255.
- With `--systemd=always`, unprivileged: systemd ends in `maintenance` (`/var/volatile` tmpfs
  mount: permission denied; `systemd-resolved` can't keep `CAP_SYS_ADMIN`).
- DNS needs the workaround of `5-container-yocto-builder-ignores-runtime-dns.md`.
- `run -d` and `exec`s after it are hit by `5-vcontainer-vpdmn-daemon-drops-slow-replies.md`.

### Build `core-image-minimal` in the container

`/home/builder/go.sh`:

```sh
#!/bin/sh
set -e
cd /home/builder
git clone https://git.openembedded.org/bitbake
bitbake/bin/bitbake-setup init --non-interactive poky-master poky distro/poky machine/qemux86-64
. bitbake-builds/poky-master/build/init-build-env
bitbake -k core-image-minimal
```

Run it detached (daemon replies time out after 60 s) and read the log back:

```sh
vpdmn --no-kvm exec -u builder yb "sh -c 'nohup /home/builder/go.sh > /home/builder/go.log 2>&1 < /dev/null &'"
vpdmn --no-kvm exec yb tail -5 /home/builder/go.log
```

- Seen working, as `builder`, with `bitbake -e core-image-minimal` in place of the build:
  `bitbake-setup init` about 8 min, `bitbake -e` rc 0 in 9.5 min (8 vCPUs, TCG, see
  `1-vcontainer-vpdmn-vm-size-fixed.md`); `DISTRO="poky"`, `PACKAGE_CLASSES="package_rpm"`.
- `poky-master` resolved to master of bitbake, oe-core and meta-yocto on 2026-10-08.

## TODO

- [ ] Run the full `bitbake -k core-image-minimal` in the container (needs a KVM host).
- [ ] Document how `go.sh` gets into the container (an `exec` here-doc hits
      `5-vcontainer-vpdmn-arguments-lose-quoting.md`).
- [ ] Confirm which build produced the image run in vpdmn. The deployed
      `container-yocto-builder` (2026-10-07) is in `bitbake-builds/meta-virt-master`, which
      also has meta-yocto, `meta-virt-setup` and vcontainer;
      `bitbake-builds/meta-virtualization-multi-layer-oci-bugs` has no deployed images.
      If it's one build, merge "Build the image" and "vcontainer SDK (vpdmn)" into one
      Environment.
