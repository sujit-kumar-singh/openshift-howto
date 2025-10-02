## To configure OpenShift authentication using GitHub as the Authentication Provider.

This example requires you belong to an organization in GitHub and have needed privileges in the GitHub organization.

Login to GitHub using your account.
From the user profile icon in top right corner, choose settings -> Developer Settings -> Oauth App -> Create a new APP

## Give the needed information as per your OpenShift cluster.

After putting these information, choose to create a new client secret.
Ensure that you note the Client ID and the Client Secret you create.

Ensure you put these information as per your OpenShift Cluster

Application name: OCP30 Cluster

HomePage URL: https://oauth-openshift.apps.dbs-ocp30.ucmcswg.com

Authorization Call Back URL: https://oauth-openshift.apps.dbs-ocp30.ucmcswg.com/oauth2callback/github/


NOTE: Here "github" in the last has to match the name of the Authentication Provider for GitHub you define in the OpenShift OAUTH identityProviders.

Assume that these are the values of the Client ID and Client Secret for you.


```bash
ClientID
Ov23liXH1fh7dJDnouYY

Client secret
e740XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Create the secret on OpenShift

```bash
oc create secret generic githuboauthsecret --from-literal=clientSecret=e740XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX -n openshift-config
```

### Edit the OAuth Cluster and add the entry of the new Authentication Provider

```bash
oc edit oauth cluster
```

NOTE: The users belong to the organization "GITHUB_ORG_NAME"

```yaml
... output omitted ....
  identityProviders:
    - name: github
      mappingMethod: claim
      type: GitHub
      github:
        clientID: Ov23liXH1fh7dJDnouYY
        clientSecret:
          name: githuboauthsecret
        organizations:
        - <<GITHUB_ORG_NAME>>
... output omitted ...
```

### Allow the OpenShift authentication pods to restart

```bash
oc get pods -n openshift-authenticaton
```

### Access OpenShift console using the option "github" related to github authentication.

Type in the username and password and if needed confirm on the organization association of the user when prompted.


