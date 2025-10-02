## To configure OpenShift authentication using GitHub as the Authentication Provider.

This example requires you belong to an organization in GitHub and have needed privileges in the GitHub organization.

Login to GitHub using your account.
From the user profile icon in top right corner, choose settings -> Developer Settings -> Oauth App -> Create a new APP

## Give the needed information as per your OpenShift cluster.

After putting these information, choose to create a new client secret.
Ensure that you note the Client ID and the Client Secret you create.

Application name: OCP30 Cluster
HomePage URL: https://oauth-openshift.apps.dbs-ocp30.ucmcswg.com
Authorization Call Back URL: https://oauth-openshift.apps.dbs-ocp30.ucmcswg.com/oauth2callback/github/

Here "github" in the last has to match the name of the Authentication Provider for GitHub you define in the OpenShift OAUTH identityProviders.

Assume that these are the values of the Client ID and Client Secret for you.
