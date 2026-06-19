## Native Cloud Provider Encryption Using OpenStack Cinder

OpenStack Cinder supports encryption at the block storage layer, usually through integration with Barbican as the key management service. This allows volumes to be encrypted at rest, with encryption handled transparently by the infrastructure layer.

This model has several strengths. It is mature, widely used, and operationally simple. Because encryption is handled below the workload layer, application code does not need to change. Users can consume encrypted volumes in much the same way as standard block devices. In addition, using Barbican provides a standard OpenStack-native approach to key management.

However, while native Cinder encryption is suitable for many general-purpose cloud environments, it has limitations in the context of a federated TRE.

The key limitation is ownership and control. In a TRE, different institutions may need exclusive control over the keys used to protect their own research data. If encryption keys are managed primarily at the infrastructure level, the hosting administrator may retain privileged control over key management. This weakens the separation of responsibilities between the infrastructure provider and the data owner.

A second limitation is policy alignment. Kubernetes workloads are usually organised around namespaces, tenants, projects, and research groups. Native Cinder encryption does not naturally expose Kubernetes-native policy controls for per-project or per-institution encryption behaviour. As a result, it can be difficult to express research-specific encryption requirements directly through Kubernetes.

For these reasons, native OpenStack Cinder encryption provides an important baseline but does not fully satisfy the requirements of a federated TRE. This motivates the exploration of a Kubernetes Operator-driven encryption model.
