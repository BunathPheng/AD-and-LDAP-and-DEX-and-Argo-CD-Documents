# Production Diagrams — AD + LDAP + DEX + Argo CD

Use this page as the reference for:

1. Production-style authentication sequence
2. Final comprehensive production diagram

---

## 1) Production-style sequence

```mermaid
sequenceDiagram
    actor U as User Browser
    participant A as Argo CD
    participant D as DEX
    participant AD as Active Directory (DC)

    U->>A: 1. Open Argo CD UI (HTTPS)
    A->>U: 2. Not authenticated → redirect to DEX
    U->>D: 3. OIDC authorize request
    D->>U: 4. Show login form
    U->>D: 5. Submit username/password
    D->>AD: 6. LDAP bind + user/group search (LDAPS)
    AD-->>D: 7. Auth result + attributes/groups
    D->>D: 8. Build OIDC identity (claims)
    D->>U: 9. Redirect back with authorization code
    U->>A: 10. Browser returns to Argo CD callback
    A->>D: 11. Exchange code for tokens (OIDC)
    D-->>A: 12. ID token (+ access token)
    A->>A: 13. Validate token, extract user/groups
    A->>A: 14. Apply RBAC
    A-->>U: 15. Authorized Argo CD session/UI
```

### Step ownership

| Step | System | What happens |
|---|---|---|
| 1 | User | Opens Argo CD |
| 2 | Argo CD | Detects no session, redirects to DEX |
| 3–5 | DEX + User | OIDC login starts, credentials submitted |
| 6–7 | DEX + AD | LDAPS bind/search, credential verification |
| 8–9 | DEX | Creates OIDC identity, returns auth code |
| 10–12 | Argo CD + DEX | Token exchange over OIDC |
| 13–15 | Argo CD | Validate identity, apply RBAC, grant access |

---

## 2) Final comprehensive production diagram

```mermaid
flowchart TB
    U[User Browser] -->|HTTPS| DNS[DNS<br/>argocd / dex hostnames]
    DNS --> LB[Load Balancer / Ingress + TLS]
    LB -->|HTTPS| A[Argo CD<br/>OIDC client + RBAC]
    A -->|OIDC Authorize / Token / JWKS over HTTPS| D[DEX<br/>OIDC IdP + LDAP connector]
    D -->|LDAPS 636<br/>bind + search| AD[Active Directory<br/>Domain Controllers]
    AD --> USERS[Users]
    AD --> GROUPS[Groups]
    USERS --> D
    GROUPS --> D
    D -->|OIDC ID Token + claims<br/>sub email groups| A
    A -->|RBAC policies| R[Authorized Argo CD resources<br/>apps sync admin read]
```

### Protocol labels on the real path

| Connection | Protocol | Purpose |
|---|---|---|
| User ↔ Ingress / Argo CD | HTTPS | Open UI / API |
| Argo CD ↔ DEX | OIDC over HTTPS | Login redirect, token exchange, JWKS |
| DEX ↔ Active Directory | LDAP / LDAPS | Verify credentials, fetch users/groups |
| Inside Argo CD after login | RBAC | Decide what the user can do |

---

## 3) Compact production flow (quick reference)

```text
User Browser
   ↓ HTTPS
Ingress / Load Balancer
   ↓ HTTPS
Argo CD
   ↓ OIDC
DEX
   ↓ LDAPS
Active Directory
   ↓ user + group information
DEX
   ↓ OIDC ID token / claims
Argo CD
   ↓ RBAC
Authorized Argo CD resources
```

---

## 4) One-line memory

AD stores identity.  
LDAP queries identity.  
DEX translates identity.  
Argo CD consumes identity and enforces RBAC.
