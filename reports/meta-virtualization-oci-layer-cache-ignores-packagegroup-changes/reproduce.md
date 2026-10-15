# meta-virtualization: OCI layer cache ignores packagegroup changes

With `OCI_LAYER_MODE = "multi"`, a package added to a layer's packagegroup reaches the rootfs
but not the OCI image: the layer is restored from the stale OCI layer cache. Found 2026-10-15.

## Environment

- Host: Debian GNU/Linux 13 (trixie), kernel 6.12.88+deb13-amd64
- bitbake `a17e8093fd4d5ed7792df86af8d2d63803dc42a4`
- openembedded-core `6b2dcfdbe16055edc88f72dabf07e33f8163d8eb`
- meta-openembedded `57918219f88bbc8ea83b0b7ce5f23f4396eb7319` (meta-oe, meta-python,
  meta-networking, meta-filesystems)
- meta-virtualization `86703e6525c38efeff6a09972bcebdc24adfea10` (master)
- `nodistro-oci` from `registry/configurations/meta-virtualization-master.conf.json`,
  `qemux86-64` (`TUNE_FEATURES = "m64 x86-64-v3"`), `package_rpm`

## Reproduce

From the repo root, build the image once:

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive \
    --setup-dir-name meta-virtualization-oci-layer-cache-ignores-packagegroup-changes \
    $PWD/registry/configurations/meta-virtualization-master.conf.json nodistro-oci \
    machine/qemux86-64 meta-virt-setup/package/rpm
B=bitbake-builds/meta-virtualization-oci-layer-cache-ignores-packagegroup-changes
. $B/build/init-build-env
bitbake container-yocto-builder
```

Add a package to the `build-tools` layer's packagegroup and build again:

```sh
cat >> conf/local.conf <<'EOF'
RDEPENDS:packagegroup-yocto-builder-toolchain:append = " libxcrypt-dev"
EOF
bitbake container-yocto-builder
grep 'OCI Cache' tmp/work/qemux86_64-oe-linux/container-yocto-builder/1.0/temp/log.do_image_oci
```

```
NOTE: OCI Cache HIT: Layer 'systemd-base' (f1d6283f6016b9e3)
NOTE: OCI Cache HIT: Layer 'build-tools' (3d617143253a051b)
NOTE: OCI Cache HIT: Layer 'yocto-extras' (9f98cbc165c7a0d0)
```

The rootfs tarball has the package's files, the OCI image doesn't:

```sh
D=tmp/deploy/images/qemux86-64
tar -tjf $D/container-yocto-builder-qemux86-64.rootfs.tar.bz2 ./usr/include/crypt.h
python3 -I - $D/container-yocto-builder-latest-oci <<'EOF'
import json, os, sys, tarfile
d = sys.argv[1]
def blob(dg): return os.path.join(d, "blobs", *dg.split(":"))
for m in json.load(open(os.path.join(d, "index.json")))["manifests"]:
    man = json.load(open(blob(m["digest"])))
    for i, l in enumerate(man["layers"]):
        with tarfile.open(blob(l["digest"])) as t:
            print("layer", i, [n for n in t.getnames() if n.endswith("include/crypt.h")])
EOF
```

```
./usr/include/crypt.h
layer 0 []
layer 1 []
layer 2 []
```

## Workaround

- Disable the cache, `OCI_LAYER_CACHE = "0"` in `conf/local.conf`, or remove
  `$B/build/oci-layer-cache/` before building.
