# meta-virtualization: vruntime multiconfig warns about empty meta-python and meta-filesystems

The vruntime BBMASK leaves meta-python and meta-filesystems without recipes in the
`vruntime-x86-64` multiconfig, which warns on every parse. Found 2026-10-08.

## Environment

Last seen at:

- bitbake `37b9c56ae3b2294ad170859a638c879910c82c59`
- openembedded-core `aba08734931428eb3c853bb497d8e21127ad746e`
- meta-openembedded `0a583b35021692dac485c16d85b3f9a9f5601da0`
- meta-virtualization `86703e6525c38efeff6a09972bcebdc24adfea10`
- `nodistro`, `qemux86-64`, no meta-poky

## Reproduce

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive registry/configurations/meta-virtualization-master.conf.json nodistro-virt machine/qemux86-64
. bitbake-builds/yocto-builder/build/init-build-env
cat >> conf/local.conf <<'EOF2'
DISTRO_FEATURES:append = " vcontainer"
BBMULTICONFIG = "vruntime-x86-64"
EOF2
bitbake -p
```

```
WARNING: No bb files in vruntime-x86-64 matched BBFILE_PATTERN_meta-python '^.../meta-openembedded/meta-python/'
WARNING: No bb files in vruntime-x86-64 matched BBFILE_PATTERN_filesystems-layer '^.../meta-openembedded/meta-filesystems/'
```

## Cause

- `conf/distro/include/vruntime-bbmask.inc` masks every recipe these layers provide.
- Builds using `conf/distro/include/meta-virt-host.conf` don't warn; plain
  `DISTRO_FEATURES` + `BBMULTICONFIG` does.

## TODO

- [ ] Find how `meta-virt-host.conf` avoids the warning, to propose a fix.
