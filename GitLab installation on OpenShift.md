### GitLab installation on OpenShift

GitLab comes with it's own implementation of Certificate Management and Ingress Cotroller.
Here we use Nginx provided Nginx Operator based Ingress Controller and not the nginx ingress controller that GitLab provides.
Also for the certificate management, we leverage the Default Wild Card Certificate for the Nginx Ingress Controller.


### GitLab operator

Install the GitLab operator. A namspace with the name of gitlab-system gets created.
The GitLab instance will be deployed to this namespace.

### Create the wildcard TLS secret

```bash
oc create secret tls wildcard-tls --cert fullchain2.pem --key privkey2.pem
```

### Create an instance of GitLab in the gitlab-system namespace

An instance is created as per the YAML below.

spec.chart.values.global.hosts.domain is set to the **apps.dbs-ocp07.ucmcswg.com** - This will be the wildcard domain of the OCP cluster.

spec.chart.values.global.ingress.tls.secretName is set to the **wildcard-tls** - This is the tls secret with the tls key and certificate that has the wildcard CN and SAN for the cluster wildcard DNS

```yaml
apiVersion: apps.gitlab.com/v1beta1
kind: GitLab
metadata:
  name: gitlab
  namespace: gitlab-system
spec:
  chart:
    values:
      gitlab:
        gitlab-shell:
          maxReplicas: 2
          minReplicas: 1
        sidekiq:
          maxReplicas: 2
          minReplicas: 1
          resources:
            requests:
              cpu: 500m
              memory: 1000M
        webservice:
          maxReplicas: 2
          minReplicas: 1
          resources:
            requests:
              cpu: 500m
              memory: 1500M
      global:
        hosts:
          domain: apps.dbs-ocp07.ucmcswg.com
          hostSuffix: null
        ingress:
          class: nginx
          configureCertmanager: false
          provider: nginx
          tls:
            enabled: true
            secretName: wildcard-tls
      installCertmanager: false
      minio:
        resources:
          requests:
            cpu: 100m
      nginx-ingress:
        enabled: false
      postgresql:
        primary:
          extendedConfiguration: max_connections = 200
      redis:
        resources:
          requests:
            cpu: 100m
    version: 9.5.2
```

### Access the GitLab instance

#### Get the root password

```bash
oc extract secret/gitlab-gitlab-initial-root-password --to=-
```
#### Get the URL to access GitLab over a browser using HTTPs

```bash
oc get ingress -n gitlab-system gitlab-webservice-default -o jsonpath='https://{.spec.rules[0].host}{"\n"}'
```
You might get something like this - https://gitlab.apps.dbs-ocp07.ucmcswg.com


