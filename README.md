# secrets-management

# Create cluster

These instructions will use the files in the capi directory in this repo. So navigate to this directory before following CLI commands at STFC-Cloud documentation for a management cluster. The cluster we will be making is self-managed.

Use the capi directory in this repository as your folder that contains the clouds.yaml, and values.yaml files for the helm charts that handle upgrades. The clouds.yaml file will need to be provided by the developer and is in the .gitignore purposefully.

Follow these instructions [here](https://stfc.atlassian.net/wiki/spaces/CLOUDKB/pages/211878034/Cluster+API+Setup) for setting up the cluster on the cloud, currently we make the cluster "manually" not using the bootstrap as it seems to be a little problematic under some circumstances, the only caveat is that there is not intended to be any management cluster and the cluster should self manage (the management cluster just does everything i.e. there is no prod/staging/dev).

# How to deploy vault to the cluster
## 1. DB secret creation 🔒
After the cluster has been setup you will need to create a secret manually that connects vault to the database.

Create a namespace "vault" navigate to the location of the secret and run `kubectl apply -f vault-db-creds-secret.yaml`

This should create a secret that is required by vault

## 2. Deploy argocd 🤖
The next step is to deploy argocd and the app-of-apps. To do that simple run `kubectl apply -k gitops/components/argocd` This should take care of deploying argocd. However, you will also need to deploy the app-of-apps. To do that apply app-of-apps app like so `kubectl apply -f gitops/apps/app-of-apps/app.yml`

Give it a few minutes and the entire cluster should be up and running

## 3. Unlocking vault 🔓
The last bit is to unlock vault, after argocd has been ran, check the pods on the cluster, there should be pods named like `vault-0`, exec into a vault and run the following command: `vault operator init` this should output 5 unseal keys as well as a root token.

> [!IMPORTANT]
> Make sure to save those keys and the root token somewhere safe like Keeper

After that is done run `vault operator unseal` when promted enter the unseal key, repeat this process 3 times (each time using a new key) until the vault is unsealed.

*the commands will look like so*

```
vault operator init

<output of the keys and a root toke>

vault operator unseal
<inputy key 1>

vault operator unseal
<input key 2>

vault operator unseal
<input key 3>

exit
```

After the first pod is unsealed it should come online. Repeat the process for all remaining pods but **DON'T RUN `vault operator init`**

*So the commands for other pods will look like so*
```
vault operator unseal
<inputy key 1>

vault operator unseal
<input key 2>

vault operator unseal
<input key 3>

exit
```

At this point you should have configured vault and it should be reachable on https://secrets.isis.rl.ac.uk for vault and for argoCD use https://argo.secrets.isis.rl.ac.uk