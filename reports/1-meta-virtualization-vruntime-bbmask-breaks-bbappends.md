# meta-virtualization: vruntime BBMASK breaks other layers' bbappends, undocumented

The vruntime multiconfig masks whole recipe directories, so another layer's bbappend for a
masked recipe fails to parse there. Expected BBMASK behaviour, but not documented. Found
2026-10-08.

## Environment

- meta-virtualization `24c622ba`, openembedded-core `cc848c40`, meta-openembedded
  `ca3cf5fe`, bitbake `37b9c56a`, `nodistro`, `qemux86-64`
- `meta-virt-setup` in `bblayers.conf`; it has
  `recipes-extended/images/container-yocto-builder.bbappend`

## Reproduce

- Enable vcontainer as in `1-meta-virtualization-vruntime-duplicate-defaultsetup.md`, add
  `meta-virt-setup` to `bblayers.conf`, run `bitbake -p`:

```
ERROR: No recipes in vruntime-x86-64 available for:
  .../meta-virt-setup/recipes-extended/images/container-yocto-builder.bbappend
```

## Cause

- `conf/distro/include/vruntime-bbmask.inc` masks e.g.
  `meta-virtualization/recipes-extended/images/`.

## Fix

- Document it in `recipes-containers/vcontainer/README.md`, with the workaround.

## Workaround

In the other layer's `conf/layer.conf`:

```
BBMASK += "${@'meta-virt-setup/recipes-extended/images/' if d.getVar('DISTRO') == 'vruntime' else ''}"
```

## TODO

- [ ] Confirm the Environment: vcontainer at meta-virtualization `24c622ba` needs meta-poky
      (see "vcontainer SDK (vpdmn)" in `setup/container-yocto-builder.md`), which this one
      lacks; `meta-virt-setup` is used in `bitbake-builds/meta-virt-master` (oe-core
      `aba08734`, meta-yocto). Reproduce doesn't match it either: it points to
      `1-meta-virtualization-vruntime-duplicate-defaultsetup.md` (`86703e65`).
- [ ] Confirm the workaround also needs `BBFILE_PATTERN_IGNORE_EMPTY_meta-virt-setup = "1"`
      (`meta-virt-setup/conf/layer.conf` has it), or the masked layer warns as in
      `1-meta-virtualization-vruntime-empty-layer-warnings.md`.
