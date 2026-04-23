~~# Implemented Proof-of-Concept: Kubernetes Operator driven LUKS encryption~~

# Kubernetes Operator driven LUKS encryption using Kubernetes Secrets

## Purpose and Scope

This section describes the evaluation and implementation of automating Cinder block volume encryption attached to a Kubernetes workload at volume creation time driven by declarative intent expressed using Custom Resource.

## Scope and Objectives of the PoC

The main objectives of this PoC is to:

1. Validate automation of Kubernetes to enforce block-level encryption without manual intervention.
2. Tests using a custom resource and operator are suitable for expressing encryption intent in a Trusted Research Environment (TRE).
3. Show that LUKS can be used programmatically to dynamically provision encrypted volumes.
4. Demonstrate later the integration of an external key management system, e.g., Vault.

The PoC scope is limited to functional correctness and feasibility rather than production-level attainment and performance optimization of full lifecycle management.

## Architecture Overview

The PoC consists of the following components:

- Kubernetes Operator: A custom Kubernetes operator using the Python-based Kopf framework to watch for lifecycle events, e.g., creating a volume, and then enforces encryption on the volume.
- Kubernetes Custom Resource: Declares the requirements for an encrypted volume and shows the user or platform intent that storage needs to be encrypted.
- Containerized Operator Deployment: The operator is packaged as a Docker image using Python 3.9 or newer and deployed onto a cluster as a Kubernetes deployment with appropriate RBAC permissions and Kubernetes secrets.
- LUKS: The operator uses standard Linux tooling such as LUKS to encrypt the block storage.
- Block storage: OpenStack Cinder block storage is used as a backend and attached to Kubernetes pods via an existing CSI driver.

## Operator Activities and Control Flow

When a user or platform creates and submits a custom resource, this creates an encrypted volume resource. The operator performs the following high-level activities:

1. Event Detection: The operator watches for the encryption lifecycle and detects, e.g., the creation of a new custom resource via Kopf’s control loop.
2. Volume Identification: The operator identifies the underlying block device associated with the declared volume, in this case a Cinder-backed device attached to the node.
3. Encryption Enforcement: Using LUKS tooling, the Operator:

- Initializes an init-container and formats the device as an encrypted device using the command cryptsetup luksFormat if not already formatted.
- It then opens it using cryptsetup luksOpen. This requires a passphrase, which is passed as a Kubernetes secret, and creates a virtual device in the /dev/mapper directory. (Red Hat, 2026)
- A filesystem, e.g., ext4, is then created on the virtual device mapper for use.

4. Completion and Status Reporting: After successful encryption, the operator records the encryption status via logs, which allows for verification that encryption has been enforced.
   This process is fully automated and does not require manual intervention.

## Validation and Results

Encryption is validated to be enforced by direct inspection and functional testing:

- The creation of a custom resource triggers the operator’s event handler. Based on the desired state declared in the resource, the operator reconciles the state by performing the required actions to configure and manage the Cinder storage encryption.
- Confirmation that the underlying block-storage device is encrypted by inspecting the /dev/mapper/cryptdisk is created, and the specified volume size is shown in OpenStack.
- The approach used for encryption is dynamically provisioning the block device rather than using static device.
- Encryption happens predictably and consistently which demonstrates that the operator model is suitable for enforcing policy-driven encryption.

These validation steps show that it is technically feasible to enforce encryption on a Cinder block storage using Kubernetes Operation in a TRE environment.

## Current Limitations

The current implementation deliberately excludes certain critical aspects required for production readiness, such as key vault integration, which is outside the initial feasibility scope.

- Key Management: Encryption keys are currently managed locally using Kubernetes secrets. Using an externally managed key (e.g., HashiCorp Vault) has been identified as the next step.
- The PoC currently focuses on the creation step in the encryption lifecycle, and the handling of updating, deletion, detachment, reattachment, and node failure scenarios is not yet implemented.

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
