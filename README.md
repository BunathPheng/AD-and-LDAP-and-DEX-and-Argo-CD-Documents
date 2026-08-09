# AD + LDAP + DEX + Argo CD Documents

Practical documentation explaining how enterprise authentication works with:

**User → Argo CD → OIDC → DEX → LDAPS → Active Directory → Groups → Argo CD RBAC**

## Repository structure

| Path | Description |
|---|---|
| `docs/full-lesson.md` | Beginner-to-production lesson |
| `assets/` | Architecture and concept diagrams |

## Diagrams

1. `assets/architecture-flow.png` — overall identity architecture
2. `assets/why-dex-is-needed.png` — why DEX sits between Argo CD and AD
3. `assets/authentication-login-flow.png` — production login steps
4. `assets/key-concepts.png` — core terms to remember

## Key concepts

- **AD** = directory service (source of truth for users/groups)
- **LDAP** = protocol used to talk to a directory
- **DEX** = identity broker / OIDC provider
- **OIDC** = authentication protocol used by applications
- **Argo CD** = GitOps app that consumes identity and applies RBAC
- **Authentication** = Who are you?
- **Authorization** = What are you allowed to do?
- **RBAC** = permissions based on roles/groups

## One-line memory

AD stores identity.  
LDAP queries identity.  
DEX translates identity.  
Argo CD consumes identity.
