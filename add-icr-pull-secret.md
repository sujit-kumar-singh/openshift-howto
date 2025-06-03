### IBM Container Registry Entitlement Key and IBM Operator Catalog

https://www.ibm.com/docs/en/fusion-software/2.10.0?topic=prerequisites-obtaining-entitlement-key

Assuming this is your entitlelment key

eyJ0eXaSI6I.......E3hQXf3nfr2xpN5wdvsU9SvRrFHXD6o



```bash
export ibm_entitlement_key='eyJ0eXaSI6I.......E3hQXf3nfr2xpN5wdvsU9SvRrFHXD6o'

# Create the json file

cat > authority.json << EOF
{
  "auth": "$(echo -n "cp:${ibm_entitlement_key}" | base64 -w0)"
}
EOF


# Append the IBM Entitlement Key to the OpenShift Global pull secret.

oc get secret/pull-secret -n openshift-config -ojson | jq -r '.data[".dockerconfigjson"]' | base64 -d  | jq '.[]."cp.icr.io" += input' - authority.json > temp_config.json

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=temp_config.json

# Validate the key is added to Global pull secret

oc get secret/pull-secret -n openshift-config -ojson | jq -r '.data[".dockerconfigjson"]' | base64 -d

# Create and Apply IBM Catalog source YAML

cat > ibm-operator-catalog-source.yaml << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: IBM Operator Catalog
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-operator-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

oc apply -f ibm-operator-catalog-source.yaml

# Ensure you see pods related to IBM operator catalog

oc get pods -n openshift-marketplace | grep -i '^ibm'

```

## References:

### Add IBM Container Registry pull secret to OpenShift global pull secret
https://www.ibm.com/docs/en/fusion-software/2.10.0?topic=prerequisites-creating-image-pull-secret

### Add the IBM Operator Catalog
http://ibm.com/docs/en/cloud-paks/1.0?topic=clusters-adding-operator-catalog
