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
  namespace: test-nfs-pvc
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

```yaml

[root@infradbs-02 testcerts]# oc get metallbs.metallb.io metallb -o yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  creationTimestamp: "2024-11-18T08:56:19Z"
  generation: 1
  name: metallb
  namespace: metallb-system
  resourceVersion: "102210640"
  uid: 7e369490-1752-470b-bb6e-6e584b6fff01
spec: {}
status:
  conditions:
  - lastTransitionTime: "2024-11-19T03:12:46Z"
    message: ""
    reason: Available
    status: "True"
    type: Available
  - lastTransitionTime: "2024-11-19T03:12:46Z"
    message: ""
    reason: Upgradeable
    status: "True"
    type: Upgradeable
  - lastTransitionTime: "2024-11-19T03:12:46Z"
    message: ""
    reason: Progressing
    status: "False"
    type: Progressing
  - lastTransitionTime: "2024-11-19T03:12:46Z"
    message: ""
    reason: Degraded
    status: "False"
    type: Degraded
```


```yaml
# oc get ipaddresspools.metallb.io ip-addresspool-sample1 -o yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-addresspool-sample1
  namespace: metallb-system
spec:
  addresses:
  - 10.10.11.218-10.10.11.220
  autoAssign: true
  avoidBuggyIPs: false
```


## Create a Load Balancer type of service

## MetalLB gives an IP IP has to be in the same range as the OCP node IPs

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
# oc get service -A | grep -i loadbalancer | grep -i nfs
openshift-storage                                  rook-ceph-nfs-ocs-storagecluster-cephnfs-load-balancer            LoadBalancer   172.32.194.250   10.10.11.220                                                                        2049:31920/TCP                                                                                             101m
```

### get PV name from PVC

```bash
# oc -n test-nfs-pvc  get pvc nfs-pvc-1 -o jsonpath='{.spec.volumeName}'
pvc-43ff87a8-7f89-4af3-92bb-38a670ff70d3
```

### Get Share name from PV
```bash

# oc get pv pvc-43ff87a8-7f89-4af3-92bb-38a670ff70d3 --output jsonpath='{.spec.csi.volumeAttributes.share}'
/0001-0011-openshift-storage-0000000000000001-82b5af97-708a-4038-bc7a-74c5188cc340
```

### Get the Ingress to reach NFS

```bash
# oc -n openshift-storage get service rook-ceph-nfs-ocs-storagecluster-cephnfs-load-balancer --output jsonpath='{.status.loadBalancer.ingress}'
[{"ip":"10.10.11.220"}]
```

## Mount on the client

##NOTE: THis might need iptables to be opened on the OCP worker/storage nodes In additon to the outside firewall rules

```bash
mkdir -p /export/mount/path

mount -t nfs4 -o proto=tcp 10.10.11.220:/0001-0011-openshift-storage-0000000000000001-ba9426ab-d61b-11ec-9ffd-0a580a800215 /export/mount/path
```

### Unmount the NFS filesystem if it does not remain to be mounted
```bash
cd && umount /export/mount/path
```

### Mounting the same inside a deployment as RWX that has mulitple pods

This is how a deployment with multiple Pods looks like to mount the PVC effectively on RWX on all the pods


```yaml
# Deployment example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  name: example
  namespace: test-nfs-pvc
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: example
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: example
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
        - mountPath: /testcephfs
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
# oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
example-6df4594c8b-9698h   1/1     Running   0          46m
example-6df4594c8b-cwgz6   1/1     Running   0          46m
example-6df4594c8b-skvd7   1/1     Running   0          46
```

### Test by Creating the files

```bash
for pod in example-6df4594c8b-9698h example-6df4594c8b-cwgz6 example-6df4594c8b-skvd7 ; do oc exec $pod -- touch /testcephfs/$pod; done
```

### Test by reading the files

```bash
# for pod in example-6df4594c8b-9698h example-6df4594c8b-cwgz6 example-6df4594c8b-skvd7 ; do oc exec $pod -- ls -la /testcephfs/$pod; done
-rw-r--r--. 1 1001020000 1001020000 0 Nov 18 10:17 /testcephfs/example-6df4594c8b-9698h
-rw-r--r--. 1 1001020000 1001020000 0 Nov 18 10:17 /testcephfs/example-6df4594c8b-cwgz6
-rw-r--r--. 1 1001020000 1001020000 0 Nov 18 10:17 /testcephfs/example-6df4594c8b-skvd7
#

```
