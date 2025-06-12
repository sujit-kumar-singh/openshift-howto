## Quay geo replication

Main Ref: https://docs.redhat.com/en/documentation/red_hat_quay/3.14/html/manage_red_hat_quay/georepl-intro#arch-georepl-prereqs

### Deploy a Postgres

A RHEL9 VM is used to run the PostgreSQL
```bash
yum -y install postgresql postgresql-server postgresql-contrib
/usr/bin/postgresql-setup --initdb
systemctl enable --now postgresql.service
```

Changing postgres configruations as needed.

Edit the file # /var/lib/pgsql/data/postgresql.conf to have

```bash
password_encryption = scram-sha-256
```

Edit the file /var/lib/pgsql/data/pg_hba.conf to have
```bash
host    all             all             10.10.11.0/24            scram-sha-256
```

Create DBs 

```bash
systemctl restart postgresql

sudo su - postgres
postgres=# CREATE USER mydbuser WITH PASSWORD 'DB_PASSWD' CREATEROLE CREATEDB LOGIN SUPERUSER;

CREATE DATABASE quay;
\c quay;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE DATABASE clair;
\c clair;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

```

### Deploy Redis

This has been installed on a RHEL9 server Ref: https://access.redhat.com/solutions/7006905

If your cloud instance has a Redis, you can use that. Redis instance is required if you are leveraging builders. You can deploy Redis on VMs.
Here redis is deployed on the same server as the DB. This is just for PoC.

Refer to HA guide for more Robust HA implementation. Ref: https://docs.redhat.com/en/documentation/red_hat_quay/3.14/html/deploy_red_hat_quay_-_high_availability/index

```bash
sudo dnf install -y podman
podman run -d --name redis -p 6379:6379 redis
yum -y install redis
```

Edit the file /etc/redis/redis.conf to put these values

```bash
bind 10.10.11.94
protected-mode no
```

Restart Redis
```bash
systemctl restart redis
```

To test connect to redis server install redis-cli using yum -y install Redis
```bash
redis-cli -h 10.10.11.94
```

Note the Redis installation here is not having any user password for user to connect to.

### Create Object Bucket Claims

Create two object buckets, one for each cluster. The buckets being closer to the OpenShift clusters.
Create the project to hold the OBC. This has to be done on each cluster.


```bash
oc new-project quay-geo-rep
```

#### OBC on the first cluster

```bash
cat > ocp30storage-obc.yaml << EOF
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: ocp30storage
  namespace: quay-geo-rep
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  bucketName: ocp30storage
  generateBucketName: 'false'
  storageClassName: openshift-storage.noobaa.io
EOF
```


```bash
ocp30 && oc create -f ocp30storage-obc.yaml
```

Collect the ConfigMap and secret information related to the bucket.

```bash
oc -n quay-geo-rep extract cm/ocp30storage --to=-
# BUCKET_REGION

# BUCKET_SUBREGION

# BUCKET_HOST
s3.openshift-storage.svc
# BUCKET_NAME
ocp30storage
# BUCKET_PORT
443

ubuntu@ubuntu-test:~/quay/geo-rep$ oc -n quay-geo-rep extract secret/ocp30storage --to=-
# AWS_ACCESS_KEY_ID
wpKDCNhCXnluYc6uShSQ
# AWS_SECRET_ACCESS_KEY
ojXT+VYN5Hfhm9veRnBdySoJvdqDHY7ht5R6+bC9
``

#### OBC on the second cluster

```bash
cat > ocp50storage-obc.yaml << EOF
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: ocp50storage
  namespace: quay-geo-rep
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  bucketName: ocp50storage
  generateBucketName: 'false'
  storageClassName: openshift-storage.noobaa.io
EOF
```

Login to the second cluster and create the second OBC
```bash
# ocp50 && oc create -f ocp50storage-obc.yaml
```

Collect the ConfigMap and secret information related to the bucket.

```bash
oc -n quay-geo-rep extract cm/ocp50storage --to=-
# BUCKET_NAME
ocp50storage
# BUCKET_PORT
443
# BUCKET_REGION

# BUCKET_SUBREGION

# BUCKET_HOST
s3.openshift-storage.svc

oc -n quay-geo-rep extract secret/ocp50storage --to=-
# AWS_ACCESS_KEY_ID
AcMFWn1GQDX842WKvqkM
# AWS_SECRET_ACCESS_KEY
GGeXFZHS3QJ/Gd6EGhQV4C9f5rVecdmxhYIzk/Hs
ubuntu@ubuntu-test:~/quay/geo-rep$
```

### Configure HAProxy Load balancer

This HAPROXY VM works as single entry-point for the Registry Traffic. In this haproxy configuration for the backend, there is one worker node from each of the OpenShift clusters where openshift router pods are running.

```bash
[root@bastion haproxy]# nslookup 10.10.11.92
92.11.10.10.in-addr.arpa        name = quaygeorep.ucmcswg.com.

[root@bastion haproxy]#

[root@bastion haproxy]# nslookup quaygeorep.ucmcswg.com
Server:         10.10.11.5
Address:        10.10.11.5#53

Name:   quaygeorep.ucmcswg.com
Address: 10.10.11.92


yum -y install haproxy
```

Configuration for HAProxy

```bash
cat > /etc/haproxy/haproxy.cfg << EOF
global
  log         127.0.0.1 local2
  pidfile     /var/run/haproxy.pid
  maxconn     4000
  daemon
defaults
  mode                    http
  log                     global
  option                  dontlognull
  option http-server-close
  option                  redispatch
  retries                 3
  timeout http-request    10s
  timeout queue           1m
  timeout connect         10s
  timeout client          1m
  timeout server          1m
  timeout http-keep-alive 10s
  timeout check           10s
  maxconn                 3000

listen ingress-router-443
  bind *:443
  mode tcp
  balance source
  server ocp30-first 10.10.11.151:443 check inter 1s
  server ocp50-first 10.10.11.133:443 check inter 1s

listen ingress-router-80
  bind *:80
  mode tcp
  balance source
  server ocp30-first 10.10.11.151:80 check inter 1s
  server ocp50-first 10.10.11.133:80 check inter 1s


frontend stats
    mode http
    bind *:9000
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST

EOF

haproxy -c -f /etc/haproxy/haproxy.cfg

systemctl enable --now haproxy
systemctl status haproxy
```

### Server certificate for the Replicated Quay registry

For the geo-replication endpoint of the Quay Registry, a public certificate is used. Let'sEncrypt is used to generate the certficate.

This certificate is used for the PassThrough termination of the TLS certificate on the Quay registry pods on the OpenShift Cluster.

Please refer to the below for understanding the TLS termination for the scenarios where either router or the tls is or both of them are managed.

Ref: https://docs.redhat.com/en/documentation/red_hat_quay/3.14/html-single/deploying_the_red_hat_quay_operator_on_openshift_container_platform/index#operator-preconfig-tls-routes

```bash
sudo certbot -d quaygeorep.ucmcswg.com --manual --preferred-challenges dns certonly --register-unsafely-without-email
```

Ensure that SERVER_HOSTNAME matches the route and must match the hostname of the global load balancer

### Red Hat Quay Operator install on both the clusters.

Ensure that you have Red Hat Quay Operator installed on each of the clusters.

### Red Hat ODF and NooBaa installation

This buckets to hold the registry container images for Red Hat Quay are from NooBaa. This example has ODF installed and a StorageSystem defined.

### QuayRegistry Configurations

New project to hold the QuayRegistry instance

```bash
oc new-project quay-enterprise
```

Create Config file for the first cluster

```bash
cat > ocp30-config.yaml << EOF
SERVER_HOSTNAME: quaygeorep.ucmcswg.com
DB_CONNECTION_ARGS:
  autorollback: true
  threadlocals: true
DB_URI: postgresql://mydbuser:DB_PASSWD@10.10.11.94:5432/quay
BUILDLOGS_REDIS:
  host: 10.10.11.94
  port: 6379
USER_EVENTS_REDIS:
  host: 10.10.11.94
  port: 6379
DATABASE_SECRET_KEY: 0ce4f796-c295-415b-bf9d-b315114704b8
DISTRIBUTED_STORAGE_CONFIG:
  ocp30storage:
    - RHOCSStorage
    - access_key: wpKDCNhCXnluYc6uShSQ
      bucket_name: ocp30storage
      bucket_name: ocp30storage
      hostname: s3-openshift-storage.apps.dbs-ocp30.ucmcswg.com
      secret_key: ojXT+VYN5Hfhm9veRnBdySoJvdqDHY7ht5R6+bC9
      is_secure: true
      port: 443
      storage_path: /quaygeorep
  ocp50storage:
    - RHOCSStorage
    - access_key: AcMFWn1GQDX842WKvqkM
      bucket_name: ocp50storage
      hostname: s3-openshift-storage.apps.dbs-ocp50.ucmcswg.com
      secret_key: GGeXFZHS3QJ/Gd6EGhQV4C9f5rVecdmxhYIzk/Hs
      is_secure: true
      port: 443
      storage_path: /quaygeorep
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS:
  - ocp30storage
  - ocp50storage
DISTRIBUTED_STORAGE_PREFERENCE:
  - ocp30storage
  - ocp50storage
FEATURE_STORAGE_REPLICATION: true
EOF
```

Create config file for the second cluster.

```bash
cat > ocp50-config.yaml << EOF
SERVER_HOSTNAME: quaygeorep.ucmcswg.com
DB_CONNECTION_ARGS:
  autorollback: true
  threadlocals: true
DB_URI: postgresql://mydbuser:DB_PASSWD@10.10.11.94:5432/quay
BUILDLOGS_REDIS:
  host: 10.10.11.94
  port: 6379
USER_EVENTS_REDIS:
  host: 10.10.11.94
  port: 6379
DATABASE_SECRET_KEY: 0ce4f796-c295-415b-bf9d-b315114704b8
DISTRIBUTED_STORAGE_CONFIG:
  ocp30storage:
    - RHOCSStorage
    - access_key: wpKDCNhCXnluYc6uShSQ
      bucket_name: ocp30storage
      hostname: s3-openshift-storage.apps.dbs-ocp30.ucmcswg.com
      secret_key: ojXT+VYN5Hfhm9veRnBdySoJvdqDHY7ht5R6+bC9
      is_secure: true
      port: 443
      storage_path: /quaygeorep
  ocp50storage:
    - RHOCSStorage
    - access_key: AcMFWn1GQDX842WKvqkM
      bucket_name: ocp50storage
      hostname: s3-openshift-storage.apps.dbs-ocp50.ucmcswg.com
      secret_key: GGeXFZHS3QJ/Gd6EGhQV4C9f5rVecdmxhYIzk/Hs
      is_secure: true
      port: 443
      storage_path: /quaygeorep
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS:
  - ocp50storage
  - ocp30storage
DISTRIBUTED_STORAGE_PREFERENCE:
  - ocp50storage
  - ocp30storage
FEATURE_STORAGE_REPLICATION: true
EOF
```

The clair database is different from the quay database. is also hosted on the same DB server 

Ref: https://docs.redhat.com/en/documentation/red_hat_quay/3/html-single/vulnerability_reporting_with_clair_on_red_hat_quay/index#configuring-custom-clair-database-managed

Create the configuration file for the clair service

```bash
cat > clair-config.yaml << EOF
indexer:
    connstring: host=10.10.11.94 port=5432 dbname=clair user=mydbuser password=DB_PASSWD sslmode=disable
    layer_scan_concurrency: 6
    migrations: true
    scanlock_retry: 11
log_level: debug
matcher:
    connstring: host=10.10.11.94 port=5432 dbname=clair user=mydbuser password=DB_PASSWD sslmode=disable
    migrations: true
metrics:
    name: prometheus
notifier:
    connstring: host=10.10.11.94 port=5432 dbname=clair user=mydbuser password=DB_PASSWD sslmode=disable
    migrations: true

EOF
```

### Quay registry custom resource for the first cluster

QuayRegistry configuration for the first cluster.

```bash
cat > ocp30-quayregistry.yaml << EOF
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: quayregistry
  namespace: quay-enterprise
spec:
  configBundleSecret: georep-config-bundle
  components:
    - kind: objectstorage
      managed: false
    - kind: route
      managed: true
    - kind: tls
      managed: false
    - kind: postgres
      managed: false
    - kind: clairpostgres
      managed: false
    - kind: redis
      managed: false
    - kind: quay
      managed: true
      overrides:
        env:
        - name: QUAY_DISTRIBUTED_STORAGE_PREFERENCE
          value: ocp30storage
    - kind: mirror
      managed: true
      overrides:
        env:
        - name: QUAY_DISTRIBUTED_STORAGE_PREFERENCE
          value: ocp30storage
EOF
```

Create the secret on the first cluster for Quay Registry deployment.

Ref: https://docs.redhat.com/en/documentation/red_hat_quay/3.14/html-single/deploying_the_red_hat_quay_operator_on_openshift_container_platform/index#operator-preconfig-tls-routes

```bash
ocp30

oc project quay-enterprise

oc create secret generic --from-file config.yaml=./ocp30-config.yaml --from-file clair-config.yaml=./clair-config.yaml --from-file ssl.cert=./fullchain1.pem --from-file ssl.key=./privkey1.pem georep-config-bundle
```
Create the QuayRegistry on the first cluster.

```bash
oc create -f ocp30-quayregistry.yaml
```

Create the Quay registry custom resource for the second cluster


```bash
cat > ocp50-quayregistry.yaml << EOF
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: quayregistry
  namespace: quay-enterprise
spec:
  configBundleSecret: georep-config-bundle
  components:
    - kind: objectstorage
      managed: false
    - kind: route
      managed: true
    - kind: tls
      managed: false
    - kind: postgres
      managed: false
    - kind: clairpostgres
      managed: false
    - kind: redis
      managed: false
    - kind: quay
      managed: true
      overrides:
        env:
        - name: QUAY_DISTRIBUTED_STORAGE_PREFERENCE
          value: ocp50storage
    - kind: mirror
      managed: true
      overrides:
        env:
        - name: QUAY_DISTRIBUTED_STORAGE_PREFERENCE
          value: ocp50storage
EOF
```

Create the secret on the second cluster for Quay Registry deployment.

```bash
ocp50

oc project quay-enterprise

oc create secret generic --from-file config.yaml=./ocp50-config.yaml --from-file clair-config.yaml=./clair-config.yaml --from-file ssl.cert=./fullchain1.pem --from-file ssl.key=./privkey1.pem georep-config-bundle
```

Create the QuayRegistry on the second cluster.

```bash
oc create -f ocp50-quayregistry.yaml
```

### Test the Global replicated registry

Create a container image

```bash
cat > Containerfile << EOF
FROM registry.access.redhat.com/ubi9/ubi:latest
CMD [ "sleep", "4800"]
EOF

podman login -u quayadmin -p quayadmin quaygeorep.ucmcswg.com

podman build -t ubi9-sleep -f Containerfile
```

Push the container image to the global registry

```bash
podman push quaygeorep.ucmcswg.com/quayadmin/ubi9-sleep:v1
```

Crate a deployment using the container image on any of the clusters

```bash

ocp50
oc new-project test-quaygeorep
oc project test-quaygeorep
oc create secret docker-registry quayregcreds --docker-username quayadmin --docker-password quayadmin --docker-server quaygeorep.ucmcswg.com --docker-email quayadmin@example.com

oc secret link default quayregcreds --for pull,mount

oc create deployment test-quay-geo --image quaygeorep.ucmcswg.com/quayadmin/ubi9-sleep:v1

# oc patch deployment test-quay-geo --type json --patch='[{"op": "add", "path": "/spec/template/spec/imagePullSecrets", "value": [name: quayregcreds]}]'
```

### Validate image replication

On a server that can reach both the NooBaa buckets, install minio client and validate the replication

```bash
mc alias set ocp30-quay https://s3-openshift-storage.apps.dbs-ocp30.ucmcswg.com:443 wpKDCNhCXnluYc6uShSQ ojXT+VYN5Hfhm9veRnBdySoJvdqDHY7ht5R6+bC9
mc alias set ocp50-quay https://s3-openshift-storage.apps.dbs-ocp50.ucmcswg.com:443 AcMFWn1GQDX842WKvqkM GGeXFZHS3QJ/Gd6EGhQV4C9f5rVecdmxhYIzk/Hs

mc alias ls

mc ls ocp30-quay

[root@infradbs-02 ~]# mc ls ocp30-quay/ocp30storage/quaygeorep/uploads
[2025-06-12 12:28:38 +04] 2.7KiB STANDARD 2e5f564c-5dd4-4b89-a91d-2b241770a17c
[2025-06-12 12:28:36 +04] 3.0MiB STANDARD 34057272-42a6-46d5-b86f-fc9d3198438a
[2025-06-12 12:28:37 +04] 1.4KiB STANDARD 37ea9851-9d96-45d9-bfbe-929b1af15c60
[2025-06-12 12:28:34 +04]   242B STANDARD 56ad99be-a458-49d2-ab32-cd5eae659ab9
[2025-06-12 12:29:09 +04]  80MiB STANDARD 59ee291b-bbee-46a9-af45-89fd9bcee34f
[2025-06-12 12:28:34 +04] 1.8KiB STANDARD 793da208-82b8-44db-a5c5-b97fa2f1f18b
[2025-06-12 12:28:36 +04] 3.0MiB STANDARD b0d3c1a3-dfd0-4a25-b483-b1ea08fc4855
[2025-06-12 12:28:38 +04] 6.1KiB STANDARD c61c25e6-9428-4c31-8048-59063812394d
[2025-06-12 12:28:57 +04]  78MiB STANDARD e468955f-849f-40b3-8421-95f56954dd93
[2025-06-12 12:28:34 +04]   624B STANDARD fcf13654-6513-4aa5-8b29-fd5396e7cd7f
[2025-06-12 13:08:27 +04] 5.1KiB STANDARD fef9a1c4-de24-4d1b-b4ac-13850efaf101
[root@infradbs-02 ~]#


[root@infradbs-02 ~]# mc ls ocp50-quay/ocp50storage/quaygeorep/uploads
[2025-06-12 12:28:35 +04]    34B STANDARD 09e0cdb3-8c00-4fd7-b59c-5862f4d08437
[2025-06-12 12:28:34 +04]   661B STANDARD 50c9f48c-6a7e-4621-93c7-a6c0facb856c
[2025-06-12 12:28:35 +04] 3.8KiB STANDARD 539a2be0-94fc-4107-acd5-0a402e4ba041
[2025-06-12 12:28:43 +04]  37MiB STANDARD 60bb8336-d705-4548-b728-2e74c948ccc6
[2025-06-12 12:29:16 +04]  19KiB STANDARD 79ebced6-1e1a-481b-92d3-4861a2c87fc3
[2025-06-12 12:28:38 +04] 4.8KiB STANDARD 8ba4c32a-39b0-4caa-96c0-2c04c987db46
[2025-06-12 12:28:09 +04]  76MiB STANDARD bbde143a-2486-4b1e-81ba-7eee139dbbb9
[2025-06-12 12:28:17 +04] 5.1KiB STANDARD cb27896f-70ab-410f-b1ee-79e315a05afe
[2025-06-12 12:28:35 +04]   324B STANDARD cbc3e7f5-2baf-497e-80a3-d851c7424732
[2025-06-12 12:28:56 +04] 170MiB STANDARD d67d64de-e8dd-4eb9-be1b-03b4b4b05721
[2025-06-12 12:28:36 +04]   274B STANDARD dad6c761-a84f-4360-ac36-59f44c8a70bf
[2025-06-12 12:28:34 +04]  24KiB STANDARD ea85c45b-d0d5-468c-86b1-8114bda4fe2d
[root@infradbs-02 ~]#
```
