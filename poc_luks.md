# Part A: Kubernetes Operator-Based Approach

## Proposed Kubernetes Operator-Based Approach

To deal with the limitations of infrastructure-level encryption, this project explores a Kubernetes-native approach using a custom Operator built with Kopf. The operator automates LUKS encryption for Cinder-backed block volumes and allows encryption intent to be declared through a Kubernetes custom resource.
The purpose of this approach is not to replace all cloud-provider encryption controls, but to provide additional control at the workload and tenant level. By moving the enforcement logic into Kubernetes, encryption can be aligned more closely with namespaces, research projects, institutional ownership, and platform policy.

This approach provides several benefits:

- Encryption can be enforced declaratively.
- Per-volume encryption can be automated.
- The platform can apply encryption consistently at creation time.
- Encryption status can be reported through Kubernetes.
- Key ownership can later be delegated to an external key management system such as HashiCorp Vault.
- Institutions can manage their own keys independently of the infrastructure provider.

## Proof of Concept: LUKS Encryption Using Kubernetes Secrets

The preliminary proof of concept validates whether a Kubernetes Operator can automate the encryption of Cinder-backed block storage using LUKS.
The scope of the proof of concept is limited to feasibility and functional correctness. It does not yet aim to provide production-level lifecycle management, performance optimisation, or full failure recovery.

The proof of concept has four main objectives:

1. Validate that Kubernetes can enforce block-level encryption without manual intervention.
2. Demonstrate that a custom resource can express encryption intent.
3. Show that LUKS can be used programmatically to encrypt dynamically provisioned block volumes.
4. Provide a foundation for future integration with HashiCorp Vault.

## Architecture Overview

The proof of concept consists of the following components:

| Component           |                               Purpose                               |
| ------------------- | :-----------------------------------------------------------------: |
| Kubernetes Operator | Watches custom resources and enforces encryption lifecycle actions. |
| Kopf Framework      |           Provides the Python-based operator control loop           |
| Custom Resource     |     Declares the user or platform intent for encrypted storage.     |
| LUKS / cryptsetup   |           Performs block-level encryption on the device.            |
| Kubernetes Secrets  |     Stores the initial proof-of-concept encryption passphrase.      |
| OpenStack Cinder    |                 Provides the backend block storage.                 |
| CSI Driver          |          Attaches Cinder volumes to Kubernetes workloads.           |

## Operator Control Flow

When a user or platform creates an encrypted volume custom resource, the operator begins a reconciliation process.

First, the operator detects the creation event through Kopf. It then identifies the underlying block device associated with the declared volume. Once the correct device has been identified, the operator applies encryption using LUKS tooling.

The process includes the following steps:

1. Detect the creation of the encrypted volume custom resource.
2. Identify the Cinder-backed block device attached to the node.
3. Initialise the device using cryptsetup luksFormat, if it has not already been formatted.
4. Open the encrypted device using cryptsetup luksOpen.
5. Use a passphrase stored in a Kubernetes Secret.
6. Create a mapped device under /dev/mapper.
7. Create a filesystem, such as ext4, on the mapped device.
8. Report completion through logs and resource status.

This demonstrates that encryption can be performed automatically and consistently without requiring researchers to execute cryptographic commands manually.

## Validation and Results

The proof of concept was validated through direct inspection and functional testing.
Creating a custom resource successfully triggered the operator event handler. The operator reconciled the declared desired state and performed the required actions to configure encryption on the Cinder-backed storage volume.

Encryption was confirmed by inspecting the creation of the mapped encrypted device under /dev/mapper/cryptdisk. The expected volume size was also displayed in OpenStack, confirming that the encrypted device corresponded to the dynamically provisioned Cinder volume.

The proof of concept demonstrates that:

- Kubernetes custom resources can express encryption intent.
- A Kopf-based operator can detect lifecycle events.
- LUKS encryption can be applied programmatically.
- Dynamic block volumes can be encrypted without manual user intervention.
- Encryption can be enforced as a platform policy.

These results show that the operator model is technically feasible for enforcing policy-driven encryption in a TRE environment.

## Current Limitations of the Proof of Concept

Although the proof of concept validates the core encryption workflow, several limitations remain.

The most important limitation is key management. The current implementation uses Kubernetes Secrets to store encryption material. This is acceptable for early functional testing, but it is not sufficient for a production TRE because Kubernetes Secrets do not provide the required level of external ownership, auditability, versioning, and institutional separation.

The proof of concept also focuses mainly on the creation stage of the encryption lifecycle. It does not yet fully handle update, deletion, detachment, reattachment, node failure, or recovery scenarios.

The current limitations are therefore:

- Keys are stored locally using Kubernetes Secrets.
- External key management is not yet implemented in the initial proof of concept.
- Key rotation is not fully handled in the basic implementation.
- Deletion and cleanup workflows require further development.
- Node failure and reattachment scenarios are not yet covered.
- Production hardening and performance evaluation remain future work.

These limitations lead directly to the need for Vault integration.

```{mermaid}
graph TD
    %% Define Subgraphs
    subgraph UserSpace[Research User]
        Browser[fa:fa-desktop Browser]
        YAML[fa:fa-file-code EncryptedVolume CRD]
    end

    subgraph AuthPlane[1. Identity Plane]
        Google[fa:fa-google Google / MyAccessID]
    end

    subgraph VaultRaft[2. Key Management]
        Vault[fa:fa-key HashiCorp Vault]
        Raft[fa:fa-database Raft Storage]
    end

    subgraph K8sControl[3. Control Plane]
        Operator[fa:fa-cogs Kopf LUKS Operator]
        RekeyJob[fa:fa-hourglass-start Rekey Job]
    end

    subgraph DataPlane[4. Data Plane - Worker Node]
        subgraph AppPod[User Workload]
            InitContainer[fa:fa-shield-alt luks-setup Init]
            Jupyter[fa:fa-book-reader Jupyter Notebook]
        end
        Mapper[fa:fa-folder-open /dev/mapper/luks-pvc]
    end

    subgraph StorageLayer[5. Physical Storage]
        Cinder[fa:fa-hdd OpenStack Cinder]
    end

    %% Vertical Flow
    Browser -->|1. OIDC Login| Google
    Google -->|2. Identity Token| Vault
    Vault -->|3. Map to Policy| Vault

    YAML -->|4. Create Request| Operator
    Operator -->|5. Provision Key| Vault

    Vault -.->|6. Fetch Key| InitContainer
    InitContainer -->|7. luksOpen| Mapper
    Mapper -->|8. Mount FS| Jupyter

    Operator -->|9. Trigger Rotation| RekeyJob
    RekeyJob -->|10. Fetch Keys| Vault
    RekeyJob -->|11. Update Header| Cinder
    Cinder --- Mapper

    %% Styling
    classDef control fill:#d1e7f5,stroke:#2e74b5,stroke-width:2px;
    classDef data fill:#d4edda,stroke:#155724,stroke-width:2px;
    classDef vault fill:#e2d9f3,stroke:#593196,stroke-width:2px;
    classDef auth fill:#f8d7da,stroke:#721c24,stroke-width:2px;
    classDef job fill:#fff3cd,stroke:#856404,stroke-width:2px,stroke-dasharray: 5 5;

    class Operator,YAML control;
    class Jupyter,DataPlane,Cinder,Mapper,InitContainer data;
    class Vault,Raft vault;
    class Google,Browser auth;
    class RekeyJob job;
```
