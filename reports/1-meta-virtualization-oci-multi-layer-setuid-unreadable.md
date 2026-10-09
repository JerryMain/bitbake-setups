# meta-virtualization: multi-layer OCI `do_image_oci` can't read `/usr/bin/sudo`

With `OCI_LAYER_MODE = "multi"` and `package_ipk`, `sudo` lands on disk as `4111` (unreadable)
in the per-layer rootfs, and the rsync in `do_image_oci` fails. Found 2026-10-07.

## Environment

- "Build the image" in `setup/container-yocto-builder.md`, without workarounds.
- Also with `DISTRO_FEATURES:append = " virtualization"`.

## Reproduce

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive registry/configurations/meta-virtualization-master.conf.json nodistro-virt machine/qemux86-64
. bitbake-builds/yocto-builder/build/init-build-env
bitbake container-yocto-builder
```

```
NOTE: OCI: Processing layer 1/3: systemd-base (packages)
NOTE: OCI: Copying pre-installed packages from ${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base
rsync: [sender] send_files failed to open "${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base/usr/bin/sudo": Permission denied (13)
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1412) [sender=3.5.1]
ERROR: container-yocto-builder-1.0-r0 do_image_oci: Execution of '${WORKDIR}/temp/run.do_image_oci.2717200' failed with exit code 23
```

- Layer rootfs: `---s--x--x ${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base/usr/bin/sudo`
- Regular rootfs of the same recipe: `-rwx--x--x ${WORKDIR}/rootfs/usr/bin/sudo`
- The task log has about 15,200 opkg warnings, one per extracted entry:

```
Warning when extracting archive entry '${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base/boot/': Can't set user=0/group=0 for ${WORKDIR}/oci-layer-rootfs/layer-1-systemd-base/boot (errno=22)
```

## Cause

- oe-core `sudo_1.9.17p2.bb:44`: `chmod 4111 ${D}${bindir}/sudo`.
- pseudo tracks only `PSEUDO_INCLUDE_PATHS` (oe-core `meta/conf/bitbake.conf:755`):
  `${WORKDIR}/rootfs` yes, `${WORKDIR}/oci-layer-rootfs` no. In tracked paths pseudo keeps
  files owner-readable on disk.
- `oci_install_layer_packages()` (`classes/image-oci.bbclass:463`) installs into
  `${WORKDIR}/oci-layer-rootfs/layer-N-<name>`, so `4111` is applied for real.
- `classes/image-oci-umoci.inc:586` copies as the build user:
  `rsync -a --checksum --no-times --no-owner --no-group "$oci_preinstall_rootfs/" "$image_bundle_name/rootfs/"`
- Same root cause: `1-meta-virtualization-oci-multi-layer-rpm-loses-setuid.md`.

## Possible fix

- Add `${WORKDIR}/oci-layer-rootfs` to `PSEUDO_INCLUDE_PATHS` and copy with
  `rsync --chmod=u+rw`: the rsync target under `${IMGDEPLOYDIR}` is untracked, and umoci (Go)
  isn't intercepted by pseudo.
- Don't also track `${IMGDEPLOYDIR}`: deploy then fails with
  `tar: .: Cannot change ownership to uid 0, gid 0: Operation not permitted`.
- Or add `u+rw` after each layer install (the workaround); `sudo` then ships as `4711`.

## Workaround

`container-yocto-builder.bbappend`:

```
python oci_fixup_layer_rootfs() {
    import glob, os, stat
    for rootfs in glob.glob(d.expand("${WORKDIR}/oci-layer-rootfs/*")):
        for root, dirs, files in os.walk(rootfs):
            for name in files:
                path = os.path.join(root, name)
                st = os.lstat(path)
                if stat.S_ISREG(st.st_mode) and (st.st_mode & 0o600) != 0o600:
                    os.chmod(path, st.st_mode | 0o600)
}
do_image_oci[prefuncs] += "oci_fixup_layer_rootfs"

# The layer cache is saved before this prefunc runs and would restore broken layers.
OCI_LAYER_CACHE = "0"
```

- The build then fails later, in `do_image_complete`.
- With that failure worked around too, in the same prefunc: 3 layers, `usr/bin/sudo` is
  `4711`, owned `0:0`.
