# coredns

Using this specific [coredns](https://github.com/coredns/coredns) deployment to manage an internal DNS zone for support split-brain DNS for the home network (so that the same host will resolve properly for clients on the internal network as well as the external network).  [This issue](https://github.com/billimek/k8s-gitops/issues/153) explored the problem and landed on this solution.

* [coredns/coredns.yaml](coredns/coredns.yaml)

# descheduler

Leveraging [descheduler](https://github.com/kubernetes-sigs/descheduler) to automatically evict pods that no longer satisfy their NodeAffinity constraints.  This is used to work in concert with `node-feature-discovery` such that when USB devices are moved from one node to a different node, the pods requiring the USB devices will be properly forced to reschedule to the new location

* [descheduler/descheduler.yaml](descheduler/descheduler.yaml)

# Intel GPU Plugin

Leverage Intel-based iGPU via the [gpu plugin](https://github.com/intel/intel-device-plugins-for-kubernetes/tree/master/cmd/gpu_plugin) DaemonSet for serving-up GPU-based workloads (e.g. Plex) via the `gpu.intel.com/i915` node resource

* [intel-gpu_plugin/intel-gpu_plugin.yaml](intel-gpu_plugin/intel-gpu_plugin.yaml)

# kured

![](https://i.imgur.com/wYWTMGI.png)

Automatically drain and reboot nodes when a reboot is required (e.g. a kernel update was applied): https://github.com/weaveworks/kured

* [kured/kured.yaml](kured/kured.yaml)
* [kured/kured-helm-values.yaml](kured/kured-helm-values.yaml)

# metallb

[Run your own on-prem LoadBalancer](https://metallb.universe.tf/)

* [metallb/metallb.yaml](metallb/metallb.yaml)

# nfs-client-provisioner

Using the [nfs-client storage type](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)

* [nfs-client-provisioner/fs-client-provisioner.yaml](nfs-client-provisioner/nfs-client-provisioner.yaml)

# nfs-pv

nfs-based persistent mounts for various pod access (media mount & data mount)

* [nfs-pv/](nfs-pv/)

# nginx

![](https://i.imgur.com/b21MHEE.png)

[ingress-nginx](https://github.com/kubernetes/ingress-nginx) controller leveraging cert-manager as the central cert store for the wildcard certificate

* [nginx/](nginx/)

# node-feature-discovery

Using the USB feature of [node-feature-discovery](https://github.com/kubernetes-sigs/node-feature-discovery) to dynamically label nodes that contain specific USB devices we care about

* [node-feature-discovery](node-feature-discovery/)

# oauth2-proxy

[OAuth2 authenticating proxy](https://github.com/pusher/oauth2_proxy) leveraging Auth0

* [oauth2-proxy/](oauth2-proxy/)

# registry-creds

[registry-creds](https://github.com/alexellis/registry-creds): Automate Kubernetes registry credentials, to extend Docker Hub limits.  This is (sadly) necessary to have cluster-wide imagePulls use an authenticated Docker account so that the cluster doesn't get rate-limited and become unable to schedule workloads. This has already happened once.

* [registry-creds/](registry-creds)

# reloader

[reloader](https://github.com/stakater/Reloader): A Kubernetes controller to watch changes in ConfigMap and Secrets and do rolling upgrades on Pods with their associated Deployment, StatefulSet, DaemonSet and DeploymentConfig 

* [reloader/](reloader/reloader.yaml)

# vault

[vault-helm chart](https://github.com/hashicorp/vault-helm) deployed in HA mode leveraging consul as the storage backend

## Vault HA server

* [vault/vault.yaml](vault/vault.yaml)

See the [vault/vault.yaml](vault/vault.yaml) & [../setup/bootstrap-vault.sh](../setup/bootstrap-vault.sh) files for reference on how these are implemented in this cluster.  The server leverages the Google KMS keystore for automatic unseal as needed.  Further information about Google KMS for unsealing are [Vault GCPKMS Documentation](https://www.vaultproject.io/docs/configuration/seal/gcpckms.html), [Autounseal with GCP KMS](https://learn.hashicorp.com/vault/operations/autounseal-gcp-kms), & [Authenticating to Google Cloud Platform](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform) for passing service account json via a secret.

# vault-secrets-operator

[vault-secrets-operator](https://github.com/ricoberger/vault-secrets-operator)

* [vault-secrets-operator/vault-secrets-operator.yaml](vault-secrets-operator/vault-secrets-operator.yaml)

## Setup

The setup is automatically handled during cluster bootstrapping.  See [bootstrap-vault.sh](../setup/bootstrap-vault.sh) for more detail.

If configuring manually, follow the [vault-secrets-operator guide](https://github.com/ricoberger/vault-secrets-operator/blob/master/README.md) which is mostly the following:

```shell
# if not logged in to vault already:
kubectl -n kube-system port-forward svc/vault 8200:8200 &
export VAULT_ADDR='http://127.0.0.1:8200'
vault login <root token>

# enable kv secrets type
vault secrets enable -path=secrets -version=1 kv

# create read-only policy for kubernetes
cat <<EOF | vault policy write vault-secrets-operator -
path "secrets/*" {
  capabilities = ["read"]
}
EOF

export VAULT_SECRETS_OPERATOR_NAMESPACE=$(kubectl -n kube-system get sa vault-secrets-operator -o jsonpath="{.metadata.namespace}")
export VAULT_SECRET_NAME=$(kubectl -n kube-system get sa vault-secrets-operator -o jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl -n kube-system get secret $VAULT_SECRET_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(kubectl -n kube-system get secret $VAULT_SECRET_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
export K8S_HOST=$(kubectl -n kube-system config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Verify the environment variables
env | grep -E 'VAULT_SECRETS_OPERATOR_NAMESPACE|VAULT_SECRET_NAME|SA_JWT_TOKEN|SA_CA_CRT|K8S_HOST'

vault auth enable kubernetes

# Tell Vault how to communicate with the Kubernetes cluster
vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT"

# Create a role named, 'vault-secrets-operator' to map Kubernetes Service Account to Vault policies and default token TTL
vault write auth/kubernetes/role/vault-secrets-operator \
  bound_service_account_names="vault-secrets-operator" \
  bound_service_account_namespaces="$VAULT_SECRETS_OPERATOR_NAMESPACE" \
  policies=vault-secrets-operator \
  ttl=24h
```
