# vcontainer-vpdmn: VM size fixed at 2 vCPUs and 2 GB

The vpdmn VM always gets `-smp 2 -m 2048`; there's no override like vxn's
`VXN_VCPUS`/`VXN_MEM`. Found 2026-10-08.

## Environment

- "vcontainer SDK (vpdmn)" in `setup/container-yocto-builder.md`.

## Reproduce

From the build directory:

```sh
grep -n -- '-smp' vcontainer-sdk/vrunner-backend-qemu.sh
```

```
177:    HV_OPTS="$HV_MACHINE -nographic -smp 2 -m 2048 -no-reboot"
```

## Cause

- `recipes-containers/vcontainer/files/vrunner-backend-qemu.sh:177`: `-smp 2 -m 2048`.

## Possible fix

- Read `VPDMN_SMP`/`VPDMN_MEM` (or `vconfig` settings), defaulting to 2 and 2048.

## Workaround

From the build directory; `vrunner.sh` `source`s the backend, so the variables reach it:

```sh
sed -i 's/-smp 2 -m 2048/-smp ${VPDMN_SMP:-2} -m ${VPDMN_MEM:-2048}/' vcontainer-sdk/vrunner-backend-qemu.sh
VPDMN_SMP=8 VPDMN_MEM=8192 vpdmn --no-kvm memres restart
```
