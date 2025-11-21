## OpenShift user Authentication using Microsoft AD

Ref:

https://rhthsa.github.io/openshift-demo/infrastructure-authentication-providers.html#syncing-ldap-groups-to-openshift-groups

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/authentication_and_authorization/index#ldap-auto-syncing_ldap-syncing-groups

Go to Administration -> Cluster Settings -> Global Configuration -> OAuth -> Add -> LDAP

| Option	| Value |
| -------- | ------ |
| Name	| Active Directory |
| URL	| ldaps://domaincontroller/DC=demo,DC=openshift,DC=pub?sAMAccountName?sub| 
| Bind DN |	service-account |
|Bind Password	 | ********* |
|Attributes	| **** |
|ID	| sAMAccountName |
|Preferred Username	 | sAMAccountName |
|Name	cn|
|Email	| mail|
-----------------------


## LDAP sync config

```bash
cat <<EOF > groupsync.yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://ad1.dcloud.demo.com:389
insecure: true
bindDN: cn=ldapuser,cn=Users,dc=dcloud,dc=demo,dc=com
bindPassword: b1ndP^ssword
rfc2307:
  groupsQuery:
    baseDN: cn=Users,dc=dcloud,dc=demo,dc=com
    derefAliases: never
    filter: (cn=ocp-*)
    scope: sub
    pageSize: 0
  groupUIDAttribute: distinguishedName
  groupNameAttributes:
  - cn
  groupMembershipAttributes:
  - member
  usersQuery:
    baseDN: cn=Users,dc=dcloud,dc=demo,dc=com
    derefAliases: never
    filter: (objectclass=user)
    scope: sub
    pageSize: 0
  userUIDAttribute: distinguishedName
  userNameAttributes:
  - sAMAccountName
EOF
```
## sync the groups test manual sync

```bash
oc adm groups sync --type openshift --sync-config=config.yaml --confirm
```

### Whitelist.txt

CN=Basis Server Admins,OU=Groups,DC=demo,DC=openshift,DC=pub
CN=OCP-Users,OU=Groups,DC=demo,DC=openshift,DC=pub

### ca.crt

-----BEGIN CERTIFICATE-----
.....
-----END CERTIFICATE-----

### Create secret with the LDAP Sync config files

```bash
oc create secret generic ldap-sync \
    --from-file=ldap-sync.yaml=ldap-sync.yaml \
    --from-file=whitelist.txt=whitelist.txt \
    --from-file=ca.crt=ca.crt
```

### Create the cluster role

```bash
oc create clusterrole ldap-group-sync \
    --verb=create,update,patch,delete,get,list \
    --resource=groups.user.openshift.io
```

### Create the project, serviceaccount and cluster-role-binding

```bash
oc new-project ldap-sync
oc create sa ldap-sync
oc adm policy add-cluster-role-to-user ldap-group-sync \
    -z ldap-sync \
    -n ldap-sync
```

### Create the cronjob


```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: ldap-group-sync
spec:
  # Format: https://en.wikipedia.org/wiki/Cron
  schedule: '@hourly'
  suspend: false
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ldap-sync
          restartPolicy: Never
          containers:
            - name: oc-cli
              command:
                - /bin/oc
                - adm
                - groups
                - sync
                - --whitelist=/ldap-sync/whitelist.txt
                - --sync-config=/ldap-sync/ldap-sync.yaml
                - --confirm
              image: registry.redhat.io/openshift4/ose-cli
              imagePullPolicy: Always
              volumeMounts:
              - mountPath: /ldap-sync/
                name: config
                readOnly: true
          volumes:
          - name: config
            secret:
              defaultMode: 420
              secretName: ldap-sync
```
