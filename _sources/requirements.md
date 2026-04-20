# User and Stakeholder Requirements

## Primary User Groups

| Stakeholder       | Core Requirement                                                            | Architectural Implication                                                       |
| ----------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Researcher        | Storage encryption must be seamless and require no cryptographic expertise. | Encryption must be enforced at the platform level                               |
| TRE Operator      | Encryption is enabled by default                                            | Controls must be declarative and centrally enforced and not optional            |
| Security          | Key usage and access must be auditable and compliant                        | External key management and clear separation between data and keys are required |
| Platform Engineer | Solutions must be automatable                                               | Configuration must be expressed as a Kubernetes-native resource                 |

## Functional Requirements

- Block storage encrypted by default
- Uses dynamic PVC provisioning
- Integrates with external Key Management Systems
- No static key usage by users
