# Native Cloud Provider Encryption (OpenStack Cinder)

Cloud Providers like OpenStack provides encryption at the block storage layer by integrating the cinder block storage service with Barbican Key Management Service and designed to encrypt data at rest in the event that the storage is compromised (Red Hat, 2023). It uses encryption volume types and encryption occurs at the infrastructure level which is transparent to the user.

## Strengths

Native cinder encryption is mature and widely adopted. It benefits from:
Operational simplicity: It is handled at the storage layer and does not need additional components e.g. privilege containers, init-containers with Kubernetes or workload layer.
Transparent usage: Workload does not need additional code changes and encryption volumes behaves like any standard block device.
Key management: Integrates with OpenStack Barbican by default (OpenStack, 2025).

## Limitations in a TRE context

While OpenStack native cinder encryption has some benefits, when evaluated against the requirements for Kubernetes-based storage encryption in a Federated TRE platform it presents some limitations as follows:

Key Management: Since research projects often requires explicit control over who owns encryption keys for which dataset the tight coupling of keys to the infrastructure layer which is often outside the control of Kubernetes operator limits the ability to enforce encryption policies that aligns with Kubernetes namespaces, tenants and research projects.

TRE-Specific policies: It is difficult to define per-project key isolations, research-domain-specific controls since OpenStack native cinder encryption applies broadly at the infrastructure level. (Red Hat, 2023)

Although native cloud provider like OpenStack cinder encryption provides a strong baseline for data-at-rest encryption which is fit for general purpose. However, in the context of TRE, federated platform that requires portability, transparency and Kubernetes-native controls, these requirements are key concerns. These limitations hereby motivate exploration of alternative models or implementations.
