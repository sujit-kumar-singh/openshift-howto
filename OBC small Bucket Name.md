## Use an Object Bucket Claim (OBC) in Red Hat OpenShift Data Foundation NooBaa Storage Class with a small bucket name
### Generally for OBC the generated bucket names are large and This is needed to get a small bucket name.


```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  resourceVersion: '14905964'
  name: smallbucket
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  bucketName: smallbucket
  generateBucketName: 'false'
  objectBucketName: obc-obc-claim-project-smallbucket
  storageClassName: openshift-storage.noobaa.io
```
