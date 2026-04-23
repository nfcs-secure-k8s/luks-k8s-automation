## OIDC Integration

Federated access with OpenID Connect (OIDC) integration with Vault simplifies the authentication flow by allowing users to use their existing institution's ID for secure access.

In this setup, Vault acts as the Relying Party (client) and MyAccessID as the identity Provider (IdP).

The flow diagram below shows how external identities, such as user logins, are translated into specific internal permissions within Vault using Group aliases. Vault Group's alias is useful because permission is managed at the group level rather than for individual users. This method allows new users who join, e.g. the "physics dept to automatically inherit the correct luks-policy without being added by the admin.

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
