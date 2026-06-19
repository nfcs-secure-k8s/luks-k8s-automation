# Vault integration for institution-managed key management

The system is extended to integrate with HashiCorp Vault to manage encryption keys managed by the institution. Vault enables centralised secret storage, versioning, access control, audit logging and managed key life cycle management.

Vault integration is designed to go beyond local Kubernetes Secrets, providing a mechanism that enables each institution to autonomously manage its own encryption keys. This is particularly relevant in the case of a federated TRE, where separate institutions may have distinct governance, regulatory and security requirements.

Three major needs for vault integration:

1. Auditable and centralised key storage.
2. Control of key rotation.
3. Secure erasure of the key when the volume is deleted.

Each encrypted volume employs the Vault KV v2 secrets engine with a tenancy-aware path structure:

/secret/tenants/<institution>/luks-key/<volume-name>
​
When a new encrypted volume is established, the operator will verify if a key already exists for that volume. If no key exists, the operator creates a first 256-bit value using secrets.token_hex(32) and saves it in Vault.

This provides a unique key per volume and decouples the key ownership from the underlying storage backend.

A major design choice is that the operator might create the initial key, but the expectation is that the institution or user will overwrite it with their own key via the Vault interface. It provides a bring-your-own-key paradigm, ensuring that long-term ownership of keys remains with the institution rather than the platform operator.

## Vault Version Synchronisation and Key Rotation

After Vault has been introduced as the key management backend, the operator also needs to detect when a key has changed.

To support this, the operator uses a synchronisation function that checks the Vault key version. A Kopf timer runs periodically, for example every 30 seconds, and reads the current secret metadata from Vault. The operator retrieves the KV version number associated with the stored key and writes that version back into the Kubernetes custom resource.

The version is recorded in two places:

metadata.annotations.vaultversion
status.current_vault_version

This provides a Kubernetes-native way for the operator to track Vault-side key changes.

When the operator detects that the Vault version number has increased, it interprets this as a key rotation event. A second controller function then performs the rekeying workflow.

The rotation process is:

1. Read the latest key from the current Vault version.
2. Read the previous key from the prior Vault version.
3. Identify the node currently associated with the encrypted volume.
4. Dispatch a Kubernetes Job to that node.
5. Add the new key using cryptsetup luksAddKey.
6. Remove the old key using cryptsetup luksRemoveKey.
7. Update the custom resource status to record the processed version.

This ensures that the volume is updated in place and that the same rotation event is not processed repeatedly.

## Deletion of Volume and Key

Vault integration also provides the controlled deletion of encrypted volumes and the keys associated with them.

When an encrypted volume is destroyed, the operator executes a cleanup workflow. This may involve evicting pods that still have active disk handles, executing a janitor task on the corresponding node, closing the LUKS device mapper, erasing the Vault secret and deleting the PVC based on the chosen deletion policy.

The deletion process is:

1. Evict the pods using the encrypted volume.
2. Run a janitor job on the correct node.
3. Close the active LUKS device mapper.
4. If the deletion policy is set to Delete, destroy the Vault secret.
   5.. If the deletion policy is Delete, delete the PVC.

This lifecycle is significant because encrypted storage is only as secure as the key lifecycle around it. If the encrypted drive is erased but the key is still available, data may still be recovered from unmanaged copies or snapshots. On the other hand, if there are no unmanaged copies of the key elsewhere, removing the key makes recovering plaintext data much more difficult.
