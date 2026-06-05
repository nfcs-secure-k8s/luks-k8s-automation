# CSI Driver Approach: Transparent LUKS Encryption via StorageClass

## Overview

The CSI driver implementation provides LUKS encryption as a standard Kubernetes
`StorageClass`. A user creates a PVC with `storageClassName: luks-encrypted` and
receives an encrypted, mounted filesystem — no custom resources, no helper pods, and
no per-workload operator configuration required.

This is the second implementation approach evaluated in this project. The first
(Kopf operator + custom `EncryptedVolume` CRD) is described in the
[Operator PoC](poc_luks.md) and [Vault Integration](vault.md) sections. The two
approaches are compared at the end of this page.

---

## Architecture

The driver is split into a **Controller plugin** (Deployment) and a **Node plugin**
(DaemonSet). Each component is a Python gRPC server implementing the
Container Storage Interface spec.

```{mermaid}
flowchart TB
    subgraph cluster["Kubernetes Cluster"]
        subgraph userNS["User Namespace"]
            pod(["User Pod"])
            upvc["User PVC"]
        end

        subgraph ctrlPod["LUKS Controller Pod  ·  kube-system"]
            ep["external-provisioner"]
            ctrl["controller.py"]
            ctrlK8s["k8s.py"]
            ctrlVault["vault.py"]
            ep --> ctrl --> ctrlK8s & ctrlVault
        end

        subgraph nodePod["LUKS Node DaemonSet  ·  per node"]
            registrar["node-driver-registrar"]
            nodeSvc["node.py"]

            subgraph devPy["device.py  ·  RESOLVERS registry"]
                direction TB
                r1["1 · LocalResolver  ·  /dev/path from spec"]
                r2["2 · LonghornResolver  ·  /dev/longhorn/handle"]
                r3["3 · CephRBDResolver  ·  /dev/rbd/pool/image"]
                r4["4 · ByIdResolver  ·  scan /dev/disk/by-id/  fallback"]
                r1 --- r2 --- r3 --- r4
            end

            luksPy["luks.py"]
            nodeVault["vault.py"]
            nodeSvc --> devPy & luksPy & nodeVault
        end

        kapi[("Kubernetes API")]
        kubelet(["kubelet"])

        subgraph backDrv["Backing CSI Driver"]
            backCtrl["Controller + external-attacher"]
            backNode["Node Plugin"]
            backCtrl --> backNode
        end

        storage[("Backing Storage  ·  Longhorn · Ceph · Cinder · EBS · local-path")]
    end

    vault[("HashiCorp Vault  ·  KV v2")]

    pod --> upvc
    upvc -- "unbound PVC" --> ep
    ctrlK8s <-- "PVC · PV · Events" --> kapi
    ctrlVault <-- "ensure / read / delete key" --> vault

    registrar -. "register CSI socket" .-> kubelet
    kubelet -- "NodeStageVolume etc." --> nodeSvc
    devPy -- "create / delete VolumeAttachment" --> kapi
    kapi -- "external-attacher watches" --> backCtrl
    backNode --> storage
    nodeVault <-- "read key · rotate" --> vault
```

**Controller plugin** runs as a single Deployment alongside the
`external-provisioner` sidecar. When a PVC is created it calls `CreateVolume`,
which creates a backing PVC against the configured `backingStorageClass`, then
auto-generates a LUKS key in Vault (idempotent — no-op if the key already exists).

**Node plugin** runs as a DaemonSet on every node. kubelet calls it at mount time
(`NodeStageVolume`) to attach the backing block device, fetch the LUKS key from
Vault, format and open the LUKS device, and mount it to a staging path. kubelet
then bind-mounts that staging path into the pod. On teardown, the node plugin
closes the LUKS device and triggers the backing driver to detach the block device.

---

## Volume Lifecycle

```{mermaid}
sequenceDiagram
    autonumber
    actor User
    participant K8s as Kubernetes API
    participant EP as external-provisioner
    participant Ctrl as controller.py
    participant Vault as HashiCorp Vault
    participant kbl as kubelet
    participant Node as node.py
    participant Dev as device.py
    participant BDrv as Backing CSI Driver
    participant LUKS as luks.py

    rect rgb(225, 240, 255)
        Note over User,Vault: Phase 1 — Volume Provisioning

        User->>K8s: create PVC with luks-encrypted StorageClass
        K8s->>EP: unbound PVC event
        EP->>Ctrl: CreateVolume(name, params)
        Ctrl->>K8s: create backing PVC using backingStorageClass
        K8s-->>Ctrl: PVC Bound, pv_name returned
        Ctrl->>Vault: ensure_secret(institution, volume_name)
        Note right of Vault: Idempotent — skips if key already exists
        Vault-->>Ctrl: key version
        Ctrl-->>EP: Volume with volume_context backingPvName, vaultPath, luksType
        EP->>K8s: create PV, bind to User PVC
        K8s-->>User: PVC Bound
    end

    rect rgb(255, 243, 224)
        Note over kbl,LUKS: Phase 2 — Volume Staging

        kbl->>Node: NodeStageVolume(volume_id, staging_path, volume_context)
        Node->>Dev: attach_and_resolve(volume_id, pv_name, node_name)

        Dev->>K8s: create VolumeAttachment(attacher, nodeName, pvName)
        K8s->>BDrv: external-attacher sees new VolumeAttachment
        BDrv->>BDrv: ControllerPublishVolume
        BDrv-->>K8s: VolumeAttachment.status.attached = true
        K8s-->>Dev: attached

        Note over Dev: Try RESOLVERS in order until a path is returned
        Dev->>Dev: resolve_device_path(pv) via RESOLVERS registry

        loop wait_for_device — polls every 3s, up to 120s
            Dev->>Dev: os.path.exists(device_path)?
        end

        Dev-->>Node: block device path

        Node->>Vault: read_secret(institution, volume_name)
        Vault-->>Node: LUKS key + current version

        alt First use — device not yet LUKS-formatted
            Node->>LUKS: luks_format(device, key, luks_type)
            Node->>LUKS: luks_open(device, mapper, key)
            Node->>LUKS: make_filesystem(mapper, filesystem)
        else Device already LUKS-formatted
            Node->>LUKS: luks_open(device, mapper, current_key)
            opt Vault version advanced — key rotation required
                Node->>Vault: read_secret(institution, volume_name, version=prev)
                Vault-->>Node: previous key
                Node->>LUKS: luks_add_key(device, new_key, old_key)
                Node->>LUKS: luks_remove_key(device, old_key)
                Node->>LUKS: luks_open(device, mapper, new_key)
            end
        end

        Node->>Node: mount /dev/mapper/luks-name to staging_path
        Node-->>kbl: NodeStageVolumeResponse
    end

    rect rgb(232, 245, 233)
        Note over kbl,Node: Phase 3 — Volume Publish

        kbl->>Node: NodePublishVolume(staging_path, target_path, readonly)
        Node->>Node: bind-mount staging_path to pod target_path
        Node-->>kbl: NodePublishVolumeResponse
        kbl-->>User: Pod running, volume mounted
    end

    rect rgb(252, 228, 236)
        Note over kbl,BDrv: Phase 4 — Teardown

        kbl->>Node: NodeUnpublishVolume(target_path)
        Node->>Node: umount target_path
        Node-->>kbl: NodeUnpublishVolumeResponse

        kbl->>Node: NodeUnstageVolume(volume_id, staging_path)
        Node->>Node: umount staging_path, lazy fallback if busy
        Node->>LUKS: luks_close_robust(mapper)
        Node->>Dev: release_attachment(volume_id, node_name)
        Dev->>K8s: delete VolumeAttachment
        K8s->>BDrv: external-attacher sees deletion
        BDrv->>BDrv: ControllerUnpublishVolume, block device detached
        Node-->>kbl: NodeUnstageVolumeResponse
    end
```

---

## Key Management

LUKS keys are stored in HashiCorp Vault at:

```
secret/tenants/<institution>/luks-keys/<volume-name>
```

This path is shared with the operator implementation so that both approaches can
coexist against the same Vault instance.

**Key generation** is automatic and happens at `CreateVolume` time in the
controller. The call is idempotent — if a key already exists for that volume name
it is reused. Users do not create or manage key material directly.

**Key fetch at mount time** is done by the node plugin, which authenticates to
Vault using the pod's Kubernetes service account JWT (`luks-csi-node` service
account). No Vault Agent sidecar is required.

**Key rotation** is triggered by advancing the KV v2 version in Vault. A background
sync thread in the node plugin checks for version changes every 30 seconds and
annotates the PV. On the next `NodeStageVolume` call the driver detects a version
mismatch and performs an inline rekey: `luksAddKey` with the new key, then
`luksRemoveKey` to remove the old one.

### Vault prerequisites

The CSI driver uses **two** Kubernetes service accounts:

- `luks-csi-controller` — used by the controller pod to generate and delete keys
- `luks-csi-node` — used by the node DaemonSet to read keys at mount time

Both must be included in the Vault Kubernetes auth role:

```bash
vault write auth/kubernetes/role/luks-operator-role \
    bound_service_account_names="luks-csi-controller,luks-csi-node" \
    bound_service_account_namespaces="kube-system" \
    policies="luks-policy" \
    ttl="24h"
```

The Vault policy is identical to the operator's:

```bash
vault policy write luks-policy - <<EOF
path "secret/data/tenants/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/metadata/tenants/*" {
  capabilities = ["read", "list", "delete"]
}
EOF
```

---

## Supported Backing Storage

The driver wraps any block-mode `StorageClass`. At `NodeStageVolume` time it creates
a `VolumeAttachment` to trigger the backing driver's attach flow, then resolves the
on-node device path via an ordered RESOLVERS registry:

| Resolver | Backing driver | Device path |
|---|---|---|
| `LocalResolver` | local / hostPath static PVs | path from PV spec |
| `LonghornResolver` | `driver.longhorn.io` | `/dev/longhorn/<volumeHandle>` |
| `CephRBDResolver` | `rbd.csi.ceph.com` / rook-ceph | `/dev/rbd/<pool>/<imageName>` |
| `ByIdResolver` *(fallback)* | Cinder, EBS, GCE PD, Azure Disk | scan `/dev/disk/by-id/` |

`local` and `hostPath` PVs skip the VolumeAttachment step entirely.

---

## Provisioning a Volume

No key Secret is needed. Create a PVC and the controller handles everything:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-encrypted-pvc
spec:
  storageClassName: luks-encrypted
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
```

Confirm the Vault key was auto-provisioned:

```bash
kubectl describe pvc my-encrypted-pvc
# Normal  LuksKeyProvisioned  ...  LUKS key auto-generated in Vault at
#   "secret/tenants/default/luks-keys/<volume-id>" (v1)
```

---

## Comparison with the Operator Approach

| Aspect | Kopf Operator | CSI Driver |
|---|---|---|
| **User interface** | Custom Resource (`EncryptedVolume`) | Standard PVC (`storageClassName: luks-encrypted`) |
| **Key generation** | Auto-generated in Vault at CR creation | Auto-generated in Vault at `CreateVolume` |
| **Key fetch at mount** | Vault Agent sidecar injects key via annotations | Node plugin fetches directly using service account JWT |
| **Key rotation trigger** | 30s timer detects Vault version bump, patches annotation | 30s sync thread annotates PV; rotation applied in `NodeStageVolume` |
| **AppArmor deployment** | SSH script run manually on each worker node | Loader DaemonSet — automatic, no SSH required |
| **Device path resolution** | Hard-coded `/dev/vdc` in init container script | RESOLVERS registry resolves path at runtime for any backing driver |
| **Privileged pods** | Every user workload pod requires `privileged: true` | Only the node DaemonSet is privileged; user pods are unprivileged |
| **Unmount lifecycle** | No delete handler; LUKS device left open on pod exit | `NodeUnstageVolume` closes the LUKS device cleanly |
| **Backend portability** | Tied to OpenStack Cinder | Works with any block StorageClass |
| **Kubernetes integration** | Operator process must be running for volumes to work | Standard CSI; volumes work independently of the operator process |

### Key limitations of the operator approach

The operator approach was the initial feasibility PoC and has several limitations
that motivated the CSI driver:

1. **Hard-coded device path** — `/dev/vdc` is assumed by the init container. Different
   clouds assign different device names, causing silent failures on non-Cinder storage.

2. **Privileged workload pods** — Every pod using an encrypted volume must run with
   `privileged: true`, which is typically blocked by Pod Security Admission in
   production clusters.

3. **No unmount lifecycle** — There is no delete handler. When a pod or CR is deleted,
   the LUKS mapper stays open on the node until manually closed or the node reboots.

4. **Encryption runs inside the user container** — `cryptsetup` is installed via `apk`
   inside an init container on every pod startup, which is slow and requires internet
   access from within the pod.

### When to use each approach

| Scenario | Recommendation |
|---|---|
| Production cluster with a standard block StorageClass | **CSI driver** |
| Cluster where CSI sidecars are unavailable | Operator |
| Quick PoC on a known single-device-path environment | Operator is simpler to deploy |
| Need unprivileged user workload pods | **CSI driver** |
