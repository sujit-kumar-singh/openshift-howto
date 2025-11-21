## Configure ingress for an application in OpenShift
## This example is for the edge termination

## This uses the self-signed TLS certificates that can be created following this

[Example of self-CA and self-CA-signed certs using openssl](openssl-ownca-selfsigned-certs_readme.md)

```bash
oc new-project ingress-poc
oc project ingress-poc

oc create deployment myapplication --image image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest --replicas 3
oc expose deployment myapplication --port 8080 --target-port 8080

oc get service myapplication
oc get endpoints myapplication

oc create secret tls myapplication-tls --cert cert-ca-combined.pem --key apps.myocp.ucmcswg.com.key
```

## The ingress

Get the ingressclass information, this will be used for the Ingress.

```bash
# oc get ingressclasses
NAME                CONTROLLER                      PARAMETERS                                        AGE
openshift-default   openshift.io/ingress-to-route   IngressController.operator.openshift.io/default   99d
```

Create the Ingress by applying this YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapplication
  namespace: ingress-poc
spec:
  ingressClassName: openshift-default
  tls:
  - hosts:
    - myapplication.apps.myocp.ucmcswg.com
    secretName: myapplication-tls
  rules:
  - host: myapplication.apps.myocp.ucmcswg.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapplication
            port:
              number: 8080
```


# This creates the ingress and also provisions a route

```bash
# oc get ingress
NAME            CLASS               HOSTS                                  ADDRESS                                     PORTS     AGE
myapplication   openshift-default   myapplication.apps.myocp.ucmcswg.com   router-default.apps.dbs-ocp07.ucmcswg.com   80, 443   9m51s




# oc get routes
NAME                  HOST/PORT                              PATH   SERVICES        PORT    TERMINATION     WILDCARD
myapplication-glp99   myapplication.apps.myocp.ucmcswg.com   /      myapplication   <all>   edge/Redirect   None
```

NOTE: The deletion of the ingress also deletes the route

## Test

```bash
curl -x "" --cacert MyOwnRootCA.crt https://myapplication.apps.myocp.ucmcswg.com
```


