# Automated Encryption of Kubernetes Block Storage for Trusted Research Environment

## Executive Summary

This report evaluates an approach for automating the encryption of Kubernetes block storage in a Trusted Research Environment (TRE). The proposed solution uses a Kubernetes Operator built with the Python Kopf framework to manage LUKS-based encryption of OpenStack Cinder-backed block volumes.
In conventional OpenStack deployments, encryption is commonly handled through Cinder volume encryption integrated with Barbican for key management. While this provides a strong infrastructure-level baseline for encryption at rest, it does not always meet the governance needs of federated research environments. In particular, it can be difficult to provide strong per-project, per-institution, or research-domain-specific key ownership when the infrastructure provider controls the underlying key management service.
The proof of concept developed for this project demonstrates that per-volume encryption can be automated, policy-driven, and enforced through Kubernetes-native resources. The operator watches for custom resources, identifies the associated block volume, applies LUKS encryption, and reports status back through Kubernetes.

Kubernetes Secrets for key storage were used in the initial implementation and have been validated as a functional proof of concept. The next stage integrates HashiCorp Vault to support institution-managed keys, key versioning, rotation, deletion policies, and auditability. OIDC integration is also considered to support federated user authentication and group-based Vault access.

## Background and Context

Trusted Research Environments are used to support research involving sensitive data, including healthcare, financial, institutional, and other regulated datasets. In these environments, storage security is a core requirement because researchers may create, attach, detach, and delete persistent volumes during the lifecycle of an experiment.
A central risk is that storage media or backend storage systems may be compromised, copied, mishandled, or disposed of incorrectly. If block storage is not encrypted, an attacker or unauthorised administrator with access to the storage backend may be able to read sensitive data directly. Encrypting storage reduces this risk and ensures that data remains unreadable without the correct decryption key.
Storage encryption also supports compliance with regulatory and organisational requirements such as GDPR and ISO 27001. However, in a TRE, encryption alone is not sufficient. The ownership, lifecycle, access control, and auditability of encryption keys are equally important.
In a federated research environment that requires independent control of keys on their data on collaborative projects or platforms among institutions, these requirements become more complex.
