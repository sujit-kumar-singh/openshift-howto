## NFS Storage from ODF for internal and external mounts and RWX use


## References

https://access.redhat.com/solutions/7014596

https://docs.openshift.com/container-platform/4.12/networking/ingress-node-firewall-operator.html

https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html/networking/load-balancing-with-metallb#olm-updating-metallb-operatorgroup_metallb-upgrading-operator

https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.17/html-single/managing_and_allocating_storage_resources/index#consuming-nfs-exports-externally-from-the-openshift-cluster_rhodf


## Enable ODF NFS

```bash
oc --namespace openshift-storage patch storageclusters.ocs.openshift.io ocs-storagecluster --type merge --patch '{"spec": {"nfs":{"enable": true}}}'
```
### Ensure nfs pods are running

```bash
oc -n openshift-storage describe cephnfs ocs-storagecluster-cephnfs
oc -n openshift-storage get pod | grep csi-nfsplugin
```

### Create pvc and Note RWX as NFS allows that

```yaml

# nfs-pvc-1.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-1
  namespace: msyql-new
spec:
  accessModes:
   - ReadWriteMany
  resources:
    requests:
      storage: 9Gi
  storageClassName: ocs-storagecluster-ceph-nfs
```

```bash
# oc create -f  nfs-pvc-1.yaml
```

## Expose the NFS server outside the OCP cluster

This example makes use of MetalLB operator that has a defined addresspool.

For this the default instllation of MetalLB operator was performed, followed by creation of an instance of the MetalLB. An "ipaddresspool" was defined that looked like below for this.

Metal LB Instance

```yaml

apiVersion: v1
items:
- apiVersion: metallb.io/v1beta1
  kind: MetalLB
  metadata:
    name: metallb
    namespace: metallb-system

```


IPAddressPool definition

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-addresspool-sample1
  namespace: metallb-system
spec:
  addresses:
  - 10.10.11.218-10.10.11.218
  - 10.10.11.219-10.10.11.219
  - 10.10.11.226-10.10.11.226
  - 10.10.11.229-10.10.11.229
  - 10.10.11.230-10.10.11.230
  autoAssign: true
  avoidBuggyIPs: false
```

L2Advertisement Definition

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  creationTimestamp: "2024-11-26T17:12:26Z"
  generation: 1
  name: l2-adv-sample1
  namespace: metallb-system
  resourceVersion: "112944263"
  uid: 28aac232-8f4b-47d2-8b7b-33190ddb2c1b
spec:
  ipAddressPools:
  - ip-addresspool-sample1
```


## Create a Load Balancer type of service

### MetalLB gives an IP IP has to be in the same range as the OCP node IPs

```yaml
cat > lb-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-nfs-ocs-storagecluster-cephnfs-load-balancer
  namespace: openshift-storage
spec:
  ports:
    - name: nfs
      port: 2049
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: rook-ceph-nfs
    ceph_nfs: ocs-storagecluster-cephnfs
    instance: a
EOF
```

```bash
# oc create -f lb-service.yaml
```

### Ensure that the loadbalancer type service gets an IP from the MetalLB pool

```bash
[root@infradbs-02 ~]# oc get service -n openshift-storage | grep -i LoadBalancer
rook-ceph-nfs-ocs-storagecluster-cephnfs-load-balancer   LoadBalancer   172.32.35.53     10.10.11.229   2049:32607/TCP                                             35m
s3                                                       LoadBalancer   172.32.193.194   10.10.11.218   80:32305/TCP,443:31584/TCP,8444:31608/TCP,7004:30799/TCP   82d
sts                                                      LoadBalancer   172.32.143.10    10.10.11.219   443:32156/TCP                                              82d
[root@infradbs-02 ~]#
                                                                                         101m
```

### Get PV name from PVC

```bash
[root@infradbs-02 ~]# oc -n mysql-new get pvc nfs-pvc-1 -o jsonpath='{.spec.volumeName}{"\n"}'
pvc-4eeb0c8d-9465-4112-a941-08a34f8ba1bb
[root@infradbs-02 ~]#

```

### Get Share name from PV
```bash

[root@infradbs-02 ~]# oc get pv pvc-4eeb0c8d-9465-4112-a941-08a34f8ba1bb -o jsonpath='{.spec.csi.volumeAttributes.share}{"\n"}'
/0001-0011-openshift-storage-0000000000000001-9b58969f-accf-4774-90c9-817ec5ada3c2
[root@infradbs-02 ~]#

```

### Get the Ingress to reach NFS

```bash
[root@infradbs-02 ~]# oc -n openshift-storage get service rook-ceph-nfs-ocs-storagecluster-cephnfs-load-balancer --output jsonpath='{.status.loadBalancer.ingress}{"\n"}'
[{"ip":"10.10.11.229"}]
[root@infradbs-02 ~]#

```

## Mount on the client

##NOTE: This might need iptables to be opened on the OCP worker/storage nodes In additon to the outside firewall rules

```bash
mkdir -p /tmp/mount

mount -t nfs4 -o proto=tcp 10.10.11.229:/0001-0011-openshift-storage-0000000000000001-9b58969f-accf-4774-90c9-817ec5ada3c2 /tmp/mount/
```

### Unmount the NFS filesystem if it does not remain to be mounted
```bash
cd && umount /export/mount/path
```

### Mounting the same inside a deployment as RWX that has mulitple pods

This is how a deployment with multiple Pods looks like to mount the PVC effectively on RWX on all the pods


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-mount
  namespace: mysql-new
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nfs-mount
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nfs-mount
    spec:
      containers:
      - image: image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest
        imagePullPolicy: Always
        name: container
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /tmp/mount
          name: nfs-pvc-1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: nfs-pvc-1
        persistentVolumeClaim:
          claimName: nfs-pvc-1
```

### Validate the pods can write to the NFS Share

```bash
[root@infradbs-02 ~]# oc get pods -n mysql-new -l app=nfs-mount
NAME                         READY   STATUS    RESTARTS   AGE
nfs-mount-7f8fcffccc-2h275   1/1     Running   0          36m
nfs-mount-7f8fcffccc-hcpqp   1/1     Running   0          36m
nfs-mount-7f8fcffccc-mlxwr   1/1     Running   0          37m
[root@infradbs-02 ~]#

```

### Test by Creating the files

```bash
for pod in $(oc get pods -n mysql-new -l app=nfs-mount -o name| sed 's|pod/||g'); do oc exec $pod -- touch /tmp/mount/$pod; done
```

### Test by reading the files

```bash
[root@infradbs-02 ~]# for pod in $(oc get pods -n mysql-new -l app=nfs-mount -o name| sed 's|pod/||g'); do oc exec $pod -- ls -la /tmp/mount/$pod; done
-rw-r--r--. 1 1000900000 1000900000 0 Nov 26 18:15 /tmp/mount/nfs-mount-7f8fcffccc-2h275
-rw-r--r--. 1 1000900000 1000900000 0 Nov 26 18:15 /tmp/mount/nfs-mount-7f8fcffccc-hcpqp
-rw-r--r--. 1 1000900000 1000900000 0 Nov 26 18:15 /tmp/mount/nfs-mount-7f8fcffccc-mlxwr
[root@infradbs-02 ~]#
```
