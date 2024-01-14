---
title: How to find volume paths for running apps on TrueNAS Scale
---
If you want to edit files within default (not host path) volumes for a TrueNAS scale app, you need to know where to
find the volume mounted on disk.

This process will only work if a pod currently exists (the app is not stopped/scaled to `0` replicas) since the volume
is only mounted by `kubelet` then.  
If that is not the case (you want to edit files offline), you can just mount the underlying 
ZFS volume yourself to make the changes. See [the TrueCharts documentation](https://truecharts.org/manual/SCALE/guides/pvc-access) for that.

### Get PVCs in namespace
```
# k3s kubectl -n ix-appdaemon get pvc
```

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                 AGE
appdaemon-conf   Bound    pvc-e726219d-e2d5-4e77-9e20-02408203587d   256Gi      RWO            ix-storage-class-appdaemon   125d
```

You can see that `appdaemon-conf` is the underlying PV: `pvc-e726219d-e2d5-4e77-9e20-02408203587`.

### Get PVC ZFS location

If we then `describe` the PV:
```
# k3s kubectl describe pv pvc-e726219d-e2d5-4e77-9e20-02408203587d
```

```
Name:              pvc-e726219d-e2d5-4e77-9e20-02408203587d
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: zfs.csi.openebs.io
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      ix-storage-class-appdaemon
Status:            Bound
Claim:             ix-appdaemon/appdaemon-conf
Reclaim Policy:    Retain
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          256Gi
Node Affinity:
  Required Terms:
    Term 0:        openebs.io/nodeid in [ix-truenas]
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            zfs.csi.openebs.io
    FSType:            zfs
    VolumeHandle:      pvc-e726219d-e2d5-4e77-9e20-02408203587d
    ReadOnly:          false
    VolumeAttributes:      openebs.io/cas-type=localpv-zfs
                           openebs.io/poolname=Vol1/ix-applications/releases/appdaemon/volumes
                           storage.kubernetes.io/csiProvisionerIdentity=1693657662253-8081-zfs.csi.openebs.io
Events:                <none>
```

We can see that it's a ZFS dataset located in `Vol1/ix-applications/releases/appdaemon/volumes`.

### Find mount point
If we check `zfs list` for our PV:
```
# zfs list | grep pvc-e726219d-e2d5-4e77-9e20-02408203587d
```

```
Vol1/ix-applications/releases/appdaemon/volumes/pvc-e726219d-e2d5-4e77-9e20-02408203587d         1007K   256G   240K  legacy
```

We can see the ZFS volume path, and that it is configured with [`mountpoint` set to `legacy`](https://docs.oracle.com/cd/E19253-01/819-5461/gbaln/index.html).

So we need to check the `mount` command:
```
# mount | grep pvc-e726219d-e2d5-4e77-9e20-02408203587d
```

```
/var/lib/kubelet/pods/e45037b1-08c8-4e74-85d0-fc00b63c7ba5/volumes/kubernetes.io~csi/pvc-e726219d-e2d5-4e77-9e20-02408203587d/mount
```

**And there we have the final path where the volume is currently mounted and can `cd` to it.**

## TL;DR

The above in short:
```
# k3s kubectl -n ix-<appname> get pvc
```

```
NAME             STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS                 AGE
<claimname>      Bound    <pvname>   256Gi      RWO            ix-storage-class-<appname>   125d
```

```
# mount | grep <pvname>
```