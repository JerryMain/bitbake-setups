# meta-virtualization: multi-layer OCI with `package_rpm` loses every setuid/setgid bit

With `OCI_LAYER_MODE = "multi"` and `package_rpm`, the image builds without warnings but has
no setuid/setgid files; `sudo` is `0711` and fails in the container. Found 2026-10-08.

## Environment

- Host: Debian GNU/Linux 13 (trixie), kernel 6.12.88+deb13-amd64
- bitbake `a17e8093fd4d5ed7792df86af8d2d63803dc42a4`
- openembedded-core `6b2dcfdbe16055edc88f72dabf07e33f8163d8eb`
- meta-openembedded `57918219f88bbc8ea83b0b7ce5f23f4396eb7319` (meta-oe, meta-python,
  meta-networking, meta-filesystems)
- meta-virtualization `86703e6525c38efeff6a09972bcebdc24adfea10` (master)
- `nodistro-oci` from `registry/configurations/meta-virtualization-master.conf.json`,
  `qemux86-64` (`TUNE_FEATURES = "m64 x86-64-v3"`), `package_rpm`
- Container run with docker, `--privileged`.
- Also seen 2026-10-08 at bitbake `37b9c56ae3b2294ad170859a638c879910c82c59`,
  openembedded-core `cc848c403aa3832453f1adfaa45411b8b72b9363`, meta-openembedded
  `ca3cf5fe14bed658890bac643cced054932fcb51`, meta-virtualization
  `24c622baf9990d6907b1b325354828df8f314ff2`.

## Reproduce

From the repo root:

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive \
    --setup-dir-name meta-virtualization-oci-multi-layer-rpm-loses-setuid \
    $PWD/registry/configurations/meta-virtualization-master.conf.json nodistro-oci \
    machine/qemux86-64 meta-virt-setup/package/rpm
. bitbake-builds/meta-virtualization-oci-multi-layer-rpm-loses-setuid/build/init-build-env
bitbake container-yocto-builder
```

Load the image, from the build directory, and look up its ID:

```sh
docker load -i tmp/deploy/images/qemux86-64/container-yocto-builder-latest-oci.tar
docker images
```

Tag it by that ID (the name `docker load` prints, `latest:latest`, can't be tagged) and run it:

```sh
docker tag <IMAGE ID> yb-oci
docker run --privileged -d --name yb --rm yb-oci
docker exec -it -u builder yb bash
```

```
$ sudo
sudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set
$ ls -l /usr/bin/sudo
-rwx--x--x 1 root root 298152 Apr  5  2011 /usr/bin/sudo
```
