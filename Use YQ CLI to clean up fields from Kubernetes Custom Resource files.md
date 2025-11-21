## Use of YQ CLI to clean up fields from the kubernetes YAML objects

### Install the latest yq command

```bash
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq &&    chmod +x /usr/bin/yq
```

### On the source cluster in the source project

### Backup all secrets in a single file

```bash
for secret in $(oc get secrets | grep 'tars-es' | awk '{print $1}'); do oc get secret $secret -o yaml > secret.yaml; done
```

### Get all the secrets in their YAML formatted files

```bash
for secret in $(oc get secret | grep -i 'tars-es'| awk '{print $1}'); do oc get secret $secret -o yaml > secret-$secret.yaml; done
````

### Clean up fields from the YAML for creationTimestamp, ownerReferences etc.

```bash
for file in *.yaml
do
yq 'del(.metadata.ownerReferences,.metadata.creationTimestamp,.metadata.resourceVersion,.metadata.uid, .metadata.generation)' $file > cleaned_$file
done
```

### On the target cluster in the target project

### Create the secret objects using the cleaned up files

```bash
for file in cleaned*tars-es*.yaml
do
oc create -f $file
done
```
