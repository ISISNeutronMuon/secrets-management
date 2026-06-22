# secrets-management

# Create cluster

These instructions will use the files in the capi directory in this repo. So navigate to this directory before following CLI commands at STFC-Cloud documentation for a management cluster. The cluster we will be making is self-managed.

Use the capi directory in this repository as your folder that contains the clouds.yaml, and values.yaml files for the helm charts that handle upgrades. The clouds.yaml file will need to be provided by the developer and is in the .gitignore purposefully.

Follow these instructions [here](https://stfc.atlassian.net/wiki/spaces/CLOUDKB/pages/211878034/Cluster+API+Setup) for setting up the cluster on the cloud, currently we make the cluster "manually" not using the bootstrap as it seems to be a little problematic under some circumstances, the only caveat is that there is not intended to be any management cluster and the cluster should self manage (the management cluster just does everything i.e. there is no prod/staging/dev).

# How to deploy vault to the cluster
## 1. DB secret creation 🔒
After the cluster has been setup you will need to create a secret manually that connects vault to the database.

Create a namespace "vault" and run `kubectl create secret generic vault-db-creds --from-literal=db_url=postgresql//:<username>:<password>@dbspgha-prod01.fds.rl.ac.uk:6432/<db_name> -n vault` replacing <username>, <password> and <db_name> with the actual values.

This should create a secret that is required by vault

## 2. Deploy argocd 🤖
The next step is to deploy argocd and the app-of-apps. To do that simple run `kubectl apply -k gitops/components/argocd` This should take care of deploying argocd. However, you will also need to deploy the app-of-apps. To do that apply app-of-apps app like so `kubectl apply -f gitops/apps/app-of-apps/app.yml`

Give it a few minutes and the entire cluster should be up and running

## 3. Unlocking vault 🔓
Whenever the pods start/restart the vault starts in a sealed state and needs to be unlocked with unseal keys.
Check the current pod status with `kubectl get pods -n vault` - each pod will show `0/1` in the `READY` column if they are sealed.

### Generating root and unseal keys (only after first deployment)

If this is the first run of the deployment, the root and unseal keys need to be generated:

```sh
> kubectl exec vault-0 -n vault --stdin --tty -- /bin/sh
(inside pod) > vault operator init
```
This should output 5 unseal keys as well as a root token.

> [!IMPORTANT]
> Save those keys and the root token somewhere safe like Keeper

### Unsealing the vault pods

Now we have the root and unseal keys we can unseal each pod. For each pod named `vault-` do the following

```sh
> kubectl exec vault-0 -n vault --stdin --tty -- /bin/sh
> vault operator unseal
<paste key 1>
> vault operator unseal
<paste key 2>
> vault operator unseal
<paste key 3>
> exit
```

Check `kubectl get pods -n vault` and the `vault-{i}` pod should show `1/1` in the `READY` column. Ensure this process is completed for each pod separately.

At this point you should have configured vault and it should be reachable on https://secrets.isis.rl.ac.uk for vault and for argoCD use https://argo.secrets.isis.rl.ac.uk
