# User and Stakeholder Requirements

The design of a storage encryption solution for a TRE must reflect the needs of several stakeholder groups.

| Stakeholder       | Core Requirement                                                            | Architectural Implication                                                       |
| ----------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Researcher        | Storage encryption must be seamless and require no cryptographic expertise. | Encryption must be enforced at the platform level                               |
| TRE Operator      | Encryption is enabled by default                                            | Controls must be declarative and centrally enforced and not optional            |
| Security          | Key usage and access must be auditable and compliant                        | External key management and clear separation between data and keys are required |
| Platform Engineer | Solutions must be automatable                                               | Configuration must be expressed as a Kubernetes-native resource                 |

From these stakeholder needs, the following functional requirements are derived:

- Block storage should be encrypted by default.
- Dynamic PVC provisioning should be supported.
- Users should not manually handle static encryption keys.
- Encryption policy should be enforced automatically.
- External key management systems should be supported.
- Key ownership should be separated from infrastructure administration.
- Key rotation and deletion should be auditable and controlled.

These requirements provide the basis for evaluating whether native OpenStack encryption is sufficient or whether a Kubernetes-native approach is needed.
