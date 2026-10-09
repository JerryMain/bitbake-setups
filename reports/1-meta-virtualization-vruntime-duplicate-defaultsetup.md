# meta-virtualization: vruntime distro includes `defaultsetup.conf` twice

With vcontainer enabled, every parse shows 19 "Duplicate inclusion" warnings from the
`vruntime-x86-64` multiconfig. Found 2026-10-09.

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
WARNING: Duplicate inclusion for .../openembedded-core/meta/conf/distro/defaultsetup.conf in .../openembedded-core/meta/conf/bitbake.conf
WARNING: Duplicate inclusion for .../openembedded-core/meta/conf/distro/include/default-providers.inc in .../openembedded-core/meta/conf/distro/defaultsetup.conf
WARNING: Duplicate inclusion for .../openembedded-core/meta/conf/distro/include/tcmode-default.inc in .../openembedded-core/meta/conf/distro/defaultsetup.conf
...
WARNING: Duplicate inclusion for .../meta-virtualization/conf/distro/include/maintainers.inc in .../openembedded-core/meta/conf/distro/defaultsetup.conf
```

- One for `defaultsetup.conf`, one for each file it includes.

## Cause

- `conf/distro/include/vruntime-base.inc:18`: `require conf/distro/defaultsetup.conf`.
- oe-core `conf/bitbake.conf:833` already includes it, after the distro config.
- Introduced by `86703e65` ("vruntime-base: base on oe-core defaultsetup instead of
  poky.conf").

## Possible fix

- Drop the `require`; first check that nothing in `vruntime-base.inc` needs a
  `defaultsetup.conf` value set before it.
