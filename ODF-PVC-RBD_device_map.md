
## Mapping PVC to the CEPH block device in ODF

## Note CEPH Toolbox is recommended to be installed. You can run CEPH commands using the toolbox

### Ensure CEPH Toolbox is installed, if not, install using the below command

```bash
## ODF v4.15 and above To enable the toolbox pod, patch/edit the StorageCluster CR like below:

# oc patch storagecluster ocs-storagecluster -n openshift-storage --type json --patch '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
```

### Get the PVC

```bash
# oc get pvc

NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  VOLUMEATTRIBUTESCLASS   AGE
mysql       Bound    pvc-d4628c61-06a8-43f5-9ae3-61f894b946ef   1Gi        RWO            ocs-storagecluster-ceph-rbd   <unset>                 9d
```

### Get the PV name from the PVC

```bash
# oc get pvc mysql -o jsonpath='{.spec.volumeName}{"\n"}'
pvc-d4628c61-06a8-43f5-9ae3-61f894b946ef
```

### Get the RBD image name from the PV 

```bash
# oc get pv pvc-d4628c61-06a8-43f5-9ae3-61f894b946ef -o jsonpath='{.spec.csi.volumeAttributes.imageName}{"\n"}'
csi-vol-5019287e-6736-40b0-bf9d-1973df748992
```



### Login to the CEPH Toolbox pod

```bash

# oc get pods -n openshift-storage | grep -i rook-ceph-tools
rook-ceph-tools-6f854c4bfc-8z7c9                                  1/1     Running   0               22m

# oc rsh rook-ceph-tools-6f854c4bfc-8z7c9

```

### Get all the CEPH pools

```bash

sh-5.1$ ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
ssd    900 GiB  656 GiB  244 GiB   244 GiB      27.08
TOTAL  900 GiB  656 GiB  244 GiB   244 GiB      27.08

--- POOLS ---
POOL                                                   ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
.mgr                                                    1    1  1.2 MiB        2  3.6 MiB      0    174 GiB
ocs-storagecluster-cephblockpool                        2   32   84 GiB   31.20k  239 GiB  31.42    174 GiB
.rgw.root                                               3    8  5.7 KiB       17  192 KiB      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.log              4    8  777 KiB      341  4.1 MiB      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.otp              5    8      0 B        0      0 B      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.buckets.index    6    8   26 KiB       11   79 KiB      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.meta             7    8  4.9 KiB       17  150 KiB      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.control          8    8      0 B        8      0 B      0    174 GiB
ocs-storagecluster-cephobjectstore.rgw.buckets.non-ec   9    8      0 B        0      0 B      0    174 GiB
ocs-storagecluster-cephfilesystem-metadata             10   16   43 MiB       36  130 MiB   0.02    174 GiB
ocs-storagecluster-cephobjectstore.rgw.buckets.data    11   32  244 KiB       59  1.3 MiB      0    174 GiB
ocs-storagecluster-cephfilesystem-data0                12   32   14 KiB    1.01k   12 MiB      0    174 GiB
.nfs                                                   13   32  4.7 KiB        5   58 KiB      0    174 GiB

```

### Block devices are in the default Block pool that is created at the time of installing ODF "ocs-storagecluster-cephblockpool"

List all the devices in the given block pool and grep the RBD image name for the VPC collected earlier.

```bash
sh-5.1$ rbd ls -p ocs-storagecluster-cephblockpool | grep csi-vol-5019287e-6736-40b0-bf9d-1973df748992
csi-vol-5019287e-6736-40b0-bf9d-1973df748992


sh-5.1$ rbd info -p ocs-storagecluster-cephblockpool csi-vol-5019287e-6736-40b0-bf9d-1973df748992
rbd image 'csi-vol-5019287e-6736-40b0-bf9d-1973df748992':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 2cce2947c02850
        block_name_prefix: rbd_data.2cce2947c02850
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features:
        flags:
        create_timestamp: Tue Nov 26 15:35:26 2024
        access_timestamp: Tue Nov 26 15:35:26 2024
        modify_timestamp: Tue Nov 26 15:35:26 2024
sh-5.1$
```


References:

https://docs.redhat.com/en/documentation/red_hat_ceph_storage/4/html/block_device_guide/ceph-block-device-commands#displaying-the-command-help_block

https://access.redhat.com/articles/4628891
