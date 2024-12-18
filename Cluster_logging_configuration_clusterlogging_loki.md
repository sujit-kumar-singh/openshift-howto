## Cluster Logging Configuration


## Note:

Logging can put heavy CPU and IO load on your cluster and storage. You must do proper sizing and investigation before hand.

Details:

- OCP version 4.15
- OpenShift cluster logging operator cluster-logging.v5.8.15
- Loki Operator - loki-operator.v5.8.15
- Storage ODF - RBD and S3 (NooBaa OBC)

## Install cluster logging operators and Logging operator

```yaml

apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  resourceVersion: '251377197'
  name: logging-loki-s3
  namespace: openshift-logging
  finalizers:
    - objectbucket.io/finalizer
  labels:
    app: noobaa
    bucket-provisioner: openshift-storage.noobaa.io-obc
    noobaa-domain: openshift-storage.noobaa.io
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  bucketName: logging-loki-s3-64014c1c-8506-480c-9d0b-4f25ec2920e6
  generateBucketName: logging-loki-s3
  objectBucketName: obc-openshift-logging-logging-loki-s3
  storageClassName: openshift-storage.noobaa.io
status:
  phase: Bound
```

## Get the Bucket information from the OBC configmap

```bash
# oc get cm logging-loki-s3 -o jsonpath='{.data}' | jq .
{
  "BUCKET_HOST": "s3.openshift-storage.svc",
  "BUCKET_NAME": "logging-loki-s3-64014c1c-8506-480c-9d0b-4f25ec2920e6",
  "BUCKET_PORT": "443",
  "BUCKET_REGION": "",
  "BUCKET_SUBREGION": ""
}
#

```

##  Get the Access key and secret access key from the OBC secret

```bash
oc extract secret/logging-loki-s3 -n openshift-logging --to=-
# AWS_ACCESS_KEY_ID
FjRqpxm7vFOg7ctqPxOT
# AWS_SECRET_ACCESS_KEY
3q1DIkKL9JQl9bdnnoqSZNyc+A7Emtgi8qfWU7iU

```


## Create the secret to be used for the Loki s3 storage

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: logging-loki-s3-noobaa
  namespace: openshift-logging
stringData:
  access_key_id: FjRqpxm7vFOg7ctqPxOT
  access_key_secret: 3q1DIkKL9JQl9bdnnoqSZNyc+A7Emtgi8qfWU7iU
  bucketnames: logging-loki-s3-64014c1c-8506-480c-9d0b-4f25ec2920e6
  endpoint: https://s3.openshift-storage.svc
  region: us-west-1
  
```
  
## Test if it has been created correctly

```bash
# oc extract secret/logging-loki-s3-noobaa --to=-
```

## Create the lokistack instance

```yaml

apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki 
  namespace: openshift-logging 
spec:
  size: 1x.small 
  storage:
    schemas:
    - version: v12
      effectiveDate: "2022-06-01"
    secret:
      name: logging-loki-s3-noobaa
      type: s3 
      credentialMode: static
  storageClassName: ocs-storagecluster-ceph-rbd
  tenants:
    mode: openshift-logging 
    
...
```


## Create the cluster logging instance

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance 
  namespace: openshift-logging 
spec:
  collection:
    type: vector
    retentionPolicy: 
      application:
        maxAge: 1d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
  logStore:
    lokistack:
      name: logging-loki
    type: lokistack
  visualization:
    type: ocp-console
    ocpConsole:
      logsLimit: 15
  managementState: Managed
  
...
```

### Accessing the logs

From OCP GUI 


### Ref: 

https://docs.openshift.com/container-platform/4.16/observability/logging/cluster-logging-deploying.html

https://docs.openshift.com/container-platform/4.16/observability/logging/log_storage/installing-log-storage.html

https://docs.openshift.com/container-platform/4.16/observability/logging/log_storage/cluster-logging-loki.html

https://docs.openshift.com/container-platform/4.16/observability/logging/log_visualization/log-visualization-ocp-console.html
