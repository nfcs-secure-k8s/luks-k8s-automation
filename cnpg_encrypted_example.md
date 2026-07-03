# Example: LUKS-encrypted CloudNative PostgreSQL

This page walks through deploying a [CloudNative PostgreSQL](https://cloudnative-pg.io/)
(CNPG) cluster backed by a LUKS-encrypted StorageClass.

## Prerequisites

- CNPG operator installed in the cluster (see the [official quickstart](https://cloudnative-pg.io/documentation/1.25/quickstart/))
- LUKS CSI driver chart available locally
- HashiCorp Vault accessible with a Kubernetes auth role configured (see [Vault Integration](vault.md))
- A backing block StorageClass (e.g. `csi-rbd-sc`, `cinder-csi`, `longhorn`)

---

## Configure Vault

Create a Vault policy that scopes LUKS key access to the service account's
namespace:

```bash
bao policy write luks-controller - <<POLICY
path "<vault-kv-mount-path>/metadata/luks-controller/{{ identity.entity.aliases.auth_kubernetes_<accessor-hash>.metadata.service_account_namespace }}/common/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "<vault-kv-mount-path>/data/luks-controller/{{ identity.entity.aliases.auth_kubernetes_<accessor-hash>.metadata.service_account_namespace }}/common/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
POLICY
```

The `<accessor-hash>` is the hex suffix of the Kubernetes auth mount accessor
(retrieve it via `bao auth list -detailed`).

Create a Kubernetes auth role that binds the CSI driver's service accounts to
this policy:

```bash
bao write auth/kubernetes/role/luks-controller \
    bound_service_account_names="luks-controller-controller,luks-controller-node" \
    bound_service_account_namespaces="kube-system" \
    policies="luks-controller"
```

Verify the role was created:

```bash
bao read auth/kubernetes/role/luks-controller
```

Expected output:

```
Key                                         Value
---                                         -----
alias_name_source                           serviceaccount_uid
bound_service_account_names                 [luks-controller-controller luks-controller-node]
bound_service_account_namespace_selector    n/a
bound_service_account_namespaces            [kube-system]
policies                                    [luks-controller]
token_bound_cidrs                           []
token_explicit_max_ttl                      0s
token_max_ttl                               0s
token_no_default_policy                     false
token_num_uses                              0
token_period                                0s
token_policies                              [luks-controller]
token_strictly_bind_ip                      false
token_ttl                                   0s
token_type                                  default
```

---

## Deploy the LUKS CSI Driver

Create a `values-luks-csi.yaml` file with your cluster-specific settings:

```yaml
fullnameOverride: luks-controller

image:
  repository: ghcr.io/nfcs-secure-k8s/k8s-block-storage-luks-csi
  tag: distroless
  pullPolicy: Never

vault:
  address: "https://<vault-address>"
  role: "<vault-role>"
  authMount: "kubernetes"
  mount: "<vault-kv-mount-path>"

storageClass:
  enabled: true
  name: luks-encrypted
  backingStorageClass: <backing-storage-class>
  vaultMount: "<vault-kv-mount-path>"
  vaultPath: "luks-controller/<namespace>/<path>"
  deletionPolicy: "Delete"

apparmor:
  enabled: false
  annotate: false
```

Install the chart:

```bash
helm upgrade --install luks-controller ./luks-csi-driver/ \
  --namespace kube-system \
  --values values-luks-csi.yaml
```

---

## Create a CNPG Cluster

A CNPG Cluster object that uses the `luks-encrypted` StorageClass is created
exactly like any other CNPG cluster — the encryption is entirely transparent:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: example-luks-pg
  namespace: luks-pg-example
spec:
  instances: 1
  storage:
    storageClass: luks-encrypted
    size: 10Gi
  podSecurityContext:
    fsGroup: 26
```

Apply it:

```bash
kubectl apply -f cnpg-cluster.yaml
```

---

## Verify

Check that the cluster initialises and the PVC is backed by the LUKS StorageClass:

```bash
kubectl get cluster -n luks-pg-example
kubectl get pods -n luks-pg-example
kubectl get pvc -n luks-pg-example
```

The PVC should show `luks-encrypted` as its `StorageClass`. Behind the scenes
the CSI driver has provisioned a LUKS-formatted block device and made it
available to the PostgreSQL instance.

---

## Clean Up

```bash
kubectl delete cluster example-luks-pg -n luks-pg-example
helm uninstall luks-controller -n kube-system
```
