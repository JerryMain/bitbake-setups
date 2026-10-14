# vcontainer-vpdmn: `cp` from a container to the host always fails

`vpdmn cp <container>:<path> <host path>` fails with `lchown ...: operation not permitted`;
the file reaches the VM's share directory but is not moved to the destination. Found
2026-10-13.

## Environment

- bitbake `a17e8093fd4d5ed7792df86af8d2d63803dc42a4`
- openembedded-core `6b2dcfdbe16055edc88f72dabf07e33f8163d8eb`
- meta-openembedded `9b938c16164e1cdabe3830a35fd09b180b46a317`
- meta-virtualization `86703e6525c38efeff6a09972bcebdc24adfea10`
- `nodistro-vcontainer` from `registry/configurations/meta-virtualization-master.conf.json`,
  `qemux86-64`, vpdmn only (meta-virtualization's `vcontainer-sdk-x86-64.conf` and
  `vcontainer-sdk-vpdmn-x86-64.conf`); `--no-kvm` (QEMU TCG).
- `docker.io/library/alpine:3.22` (image ID `c83674e19990`).

## Reproduce

Build and install the SDK:

```sh
[ -e bitbake/.git ] || git submodule update --init bitbake
./bitbake/bin/bitbake-setup init --non-interactive --setup-dir-name vcontainer-vpdmn-cp-to-host-fails-chown $PWD/registry/configurations/meta-virtualization-master.conf.json nodistro-vcontainer machine/qemux86-64 meta-virt-setup/vcontainer-sdk/no-vdkr meta-virt-setup/vcontainer-sdk/vpdmn-x86-64 meta-virt-setup/vcontainer-sdk/no-vxn meta-virt-setup/vcontainer-sdk/no-aarch64
. bitbake-builds/vcontainer-vpdmn-cp-to-host-fails-chown/build/init-build-env
bitbake vcontainer-tarball
tmp/deploy/sdk/vcontainer-standalone.sh -d $PWD/vcontainer-sdk -y
. vcontainer-sdk/init-env.sh
```

Then:

```sh
vpdmn --no-kvm memres start
vpdmn --no-kvm pull docker.io/library/alpine:3.22
vpdmn --no-kvm run -d --name cpa docker.io/library/alpine:3.22 sleep 100000
vpdmn --no-kvm cp cpa:/etc/hostname /tmp/; echo rc=$?
ls -l /tmp/hostname ~/.vpdmn/x86_64/share/
```

```
Error: 1 error occurred:
	* copying to host: error during bulk transfer for copier.request{Request:"PUT", Root:"/", preservedRoot:"/mnt/share/", rootPrefix:"/mnt/share/", Directory:"/", preservedDirectory:"/mnt/share/", Globs:[]string{}, preservedGlobs:[]string(nil), StatOptions:copier.StatOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), CheckForArchives:false, Excludes:[]string(nil), DisallowWildcard:false, AllowEmptyWildcard:false}, GetOptions:copier.GetOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), Excludes:[]string(nil), ExpandArchives:false, Chmod:"", ChownDirs:(*idtools.IDPair)(nil), ChmodDirs:(*fs.FileMode)(nil), ChownFiles:(*idtools.IDPair)(nil), ChmodFiles:(*fs.FileMode)(nil), Parents:false, StripSetuidBit:false, StripSetgidBit:false, StripStickyBit:false, StripXattrs:false, KeepDirectoryNames:false, Rename:map[string]string(nil), NoDerefSymlinks:false, IgnoreUnreadable:false, NoCrossDevice:false, Timestamp:<nil>, DisallowWildcard:false, AllowEmptyWildcard:false}, PutOptions:copier.PutOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), DefaultDirOwner:(*idtools.IDPair)(nil), DefaultDirMode:(*fs.FileMode)(nil), Chmod:"", ChownDirs:(*idtools.IDPair)(0x157556195860), ChmodDirs:(*fs.FileMode)(nil), ChownFiles:(*idtools.IDPair)(0x1575561958b0), ChmodFiles:(*fs.FileMode)(nil), StripSetuidBit:false, StripSetgidBit:false, StripStickyBit:false, StripXattrs:false, IgnoreXattrErrors:false, IgnoreDevices:true, NoOverwriteDirNonDir:true, NoOverwriteNonDirDir:true, Rename:map[string]string(nil), Timestamp:<nil>, CreateDestPath:0x0}, MkdirOptions:copier.MkdirOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), MakeParents:0x0, ModTimeNew:<nil>, ChownNew:(*idtools.IDPair)(nil), ChmodNew:(*fs.FileMode)(nil)}, RemoveOptions:copier.RemoveOptions{All:false, AllowNotFound:false, AllowWildcard:false}, EnsureOptions:copier.EnsureOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), Paths:[]copier.EnsurePath(nil)}, ConditionalRemoveOptions:copier.ConditionalRemoveOptions{UIDMap:[]idtools.IDMap(nil), GIDMap:[]idtools.IDMap(nil), Paths:[]copier.ConditionalRemovePath(nil)}}: copier: put: error setting ownership of "/hostname" to 0:0: lchown /hostname: operation not permitted
```

```
rc=125
ls: cannot access '/tmp/hostname': No such file or directory
/home/jermain/.vpdmn/x86_64/share/:
total 4
-rw------- 1 jermain jermain 13 Oct 13 12:10 hostname
```

## Workaround

```sh
mv ~/.vpdmn/x86_64/share/hostname /tmp/
```
