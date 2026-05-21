# Automated Encryption of Kubernetes Block Storage for Trusted Research Environment

## Executive Summary

This report evaluates the automation of per-volume Kubernetes block Cinder storage in a Trusted Research Environment (TRE) environment using LUKS (Linux Unified Key setup), Kubernetes custom operator with Kopf. The proposed solution uses Kubernetes Operator using Python Kopf to manage LUKS-based encryption of OpenStack Cinder-backed block volumes.

- Watch for custom resources when submitted by the user or the system.

- Triggers the encryption of the volume based on the life cycle state.

- Enforces the encryption of the volume at the creation time.

This approach demonstrates that per volume encryption can be policy-driven, enforced and automated rather than using manual operations.

### Current Maturity:

- Functional proof-of-concept implemented and validated.

- Using local Key Management System (Kubernetes secrets). Integration of HashiCorp vault still pending.
