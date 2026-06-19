## OIDC Integration for Federated Access Control

In a federated TRE, users should not need separate local Vault accounts. Instead, they should be able to authenticate using their existing institutional identity.

OpenID Connect can support this model by allowing Vault to act as the relying party and an institutional identity provider, such as MyAccessID, to act as the identity provider.

In this model, users authenticate through their institution. Vault then maps external identity claims to internal Vault entities and groups. Group aliases can be used to connect an external institutional group, such as a department or project group, to a Vault policy.

This approach is useful because permissions are managed at the group level rather than individually for each user. For example, when a new researcher joins the physics department, membership of the relevant institutional group can automatically grant the correct Vault policy for accessing that department’s LUKS keys.

OIDC integration therefore supports:

- Federated authentication.
- Institution-based access control.
- Reduced manual user administration.
- Group-level Vault policy assignment.
- Better alignment with TRE governance models.

This completes the progression from basic operator-driven encryption, to external key management, to federated identity-based access control.

```{mermaid}
graph TD
    subgraph "1. External Authentication (MyAccessID)"
        A[Researcher] -->|Initiates OIDC Login| B[MyAccessID / IDP]
        B -->|Returns JWT with Claims| C[Vault OIDC Backend]
        note1["<b>Claims:</b><br/>email: user@cam.ac.uk<br/>groups: ['RCS-team']"]
        B -.-> note1
    end

    %% This connection links the top section to the bottom section
    C -->|Check Group Alias| D

    subgraph "Internal Processing"
        direction LR

        subgraph "2. Identity Mapping (Vault Internal)"
            D{Group Alias Lookup} -->|Match: 'RCS-team'| E[Internal Group: university-of-cambridge-group]
            E -->|Attached Policy| F(luks-policy)
        end

        subgraph "3. Authorization & Isolation"
            G{Vault Policy Engine}
            H[Request: /tenants/university-of-cambridge/*] --> G
            G -->|Result| I[✅ ACCESS GRANTED]

            J[Request: /tenants/chemistry-dept/*] --> G
            G -->|Result| K[❌ ACCESS DENIED]
        end

        %% Horizontal link between the side-by-side boxes
        F -->|Evaluates Path Request| G
    end

    style I fill:#d4edda,stroke:#28a745
    style K fill:#f8d7da,stroke:#dc3545
    style E fill:#fff3cd,stroke:#ffc107
```
