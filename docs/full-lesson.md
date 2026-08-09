# AD + LDAP + DEX + Argo CD — Progressive Production Lesson

Mental model:

> AD stores people. LDAP is how you ask AD about people. DEX translates that into modern app login (OIDC). Argo CD consumes that login and decides what you can do (RBAC).

Focus stack only:
**AD + LDAP + DEX + Argo CD**

---

## Level 1 — Basic Concepts

### Active Directory (AD)

Active Directory is Microsoft’s directory service for an organization. It is usually the enterprise source of truth for identity.

AD stores:
- users
- groups
- computers
- OUs
- policies
- hashed credentials and account attributes

Users, passwords, groups, and computers are stored on **Domain Controllers** in the AD database (commonly NTDS).

AD is best described as a **directory service**:
- Service: Active Directory Domain Services (AD DS)
- Server role: Domain Controller
- Database: NTDS
- Protocols used: LDAP/LDAPS, Kerberos, DNS, and more

An **AD Domain** is a security/admin boundary (example: `corp.example.com`).  
A **Domain Controller** authenticates users and answers directory queries.  
An **AD user** is an identity object.  
An **AD group** is a collection used for authorization.

### LDAP

LDAP = Lightweight Directory Access Protocol.

LDAP is a **protocol**, not a server.

Why LDAP is used:
- authenticate (bind)
- search users
- read attributes
- check group membership

How LDAP talks to AD:
- port 389 (LDAP / StartTLS)
- port 636 (LDAPS)

Differences:
- LDAP protocol = wire protocol
- LDAP directory = hierarchical identity data
- LDAP server = software speaking LDAP
- Active Directory = Microsoft directory service that can speak LDAP

Examples:
- OpenLDAP = dedicated LDAP directory server
- Active Directory DC = directory service that also speaks LDAP

### DEX

DEX is an OIDC identity provider / identity broker.

Why DEX exists:
modern apps expect OIDC, while enterprise directories often speak LDAP.

DEX:
- speaks OIDC to Argo CD
- speaks LDAP to AD
- converts successful AD auth into OIDC identity

DEX is not the source-of-truth directory.  
DEX is the translator.

### Argo CD

Argo CD is a GitOps continuous delivery controller for Kubernetes.

It needs authentication because users can view/sync/change delivery state.  
It can use OIDC natively.  
DEX acts as Argo CD’s OIDC Identity Provider.

### Level 1 summary

- AD = where users/groups live
- LDAP = protocol to query/authenticate against AD
- DEX = OIDC broker in front of LDAP/AD
- Argo CD = app that authenticates via OIDC and authorizes via RBAC

---

## Level 2 — Component Relationships

```text
User
 ↓
Argo CD
 ↓ OIDC
DEX
 ↓ LDAP
Active Directory
```

Arrow meanings:

1. **User → Argo CD**  
   HTTPS, open UI/API

2. **Argo CD → DEX**  
   OIDC over HTTPS, delegate authentication, receive tokens/claims

3. **DEX → AD**  
   LDAP/LDAPS, verify credentials and fetch user/group attributes

Boundary:
- Argo CD does not talk LDAP to AD in this design
- AD does not speak OIDC to Argo CD
- DEX is the translator

---

## Level 3 — Authentication Flow

1. User opens Argo CD
2. Argo CD detects unauthenticated request
3. Argo CD redirects browser to DEX
4. DEX starts authentication
5. User submits credentials
6. DEX contacts AD with LDAP/LDAPS
7. AD verifies credentials and returns directory data
8. DEX maps identity into OIDC claims
9. Browser returns to Argo CD callback
10. Argo CD exchanges code and validates tokens
11. Argo CD extracts user/groups
12. Argo CD applies RBAC
13. User gets authorized access

---

## Level 4 — Protocols and Tokens

Why multiple protocols?

- **LDAP** between DEX and AD: directory bind/search
- **OIDC** between Argo CD and DEX: browser app login + tokens

Terms:
- Authentication = Who are you?
- Authorization = What can you do?
- Identity = representation of the user
- Token = signed proof of auth context
- ID Token = identity proof for the app
- Access Token = API access token
- Claims = fields inside tokens (`sub`, `email`, `groups`)
- Groups = authorization anchors from AD to Argo CD
- RBAC = role/group permission mapping in Argo CD

---

## Level 5 — Why DEX Is Needed

Incomplete:

```text
Argo CD → AD/LDAP
```

Problems:
- Argo CD expects OIDC-style login
- LDAP does not provide the same OIDC browser/token model
- Argo CD wants claims-based identity
- identity broker logic should not live inside every app

Production pattern:

```text
Argo CD → DEX → LDAP → AD
```

DEX adds:
- OIDC authorize/token endpoints
- redirect login flow
- ID tokens and claims
- clean integration for Argo CD

DEX does not replace AD.  
DEX adapts AD for modern application login.

---

## Level 6 — Production Architecture

Include:
- multiple users
- AD Domain Controllers
- DEX
- TLS/HTTPS
- LDAPS
- DNS
- certificates
- OIDC
- groups
- Argo CD RBAC
- HA
- secrets
- firewall rules
- logging/monitoring/audit

Typical HA:
- AD DCs: yes
- DEX: often yes
- Argo CD: yes
- Ingress/LB: yes
- DNS: yes

---

## Level 7 — Production Login Workflow

```text
User Browser
 ↓ HTTPS
Ingress / Argo CD
 ↓ OIDC
DEX
 ↓ LDAPS
Active Directory
 ↓ user + group attributes
DEX
 ↓ OIDC token/claims
Argo CD
 ↓ RBAC
Authorized access
```

---

## Level 8 — Kubernetes + Troubleshooting

Inside Kubernetes (typical):
- Argo CD
- DEX pods/service
- ConfigMaps/Secrets
- Ingress/TLS

Outside Kubernetes (typical):
- Active Directory Domain Controllers
- enterprise DNS/PKI

### Scenario 1: user cannot log in

Isolate:
- DNS/TLS
- Argo CD OIDC config
- DEX health/logs
- network DEX → AD
- LDAPS trust
- AD account state
- group/RBAC issues after login

### Scenario 2: DEX cannot connect to AD

Check:
- DC hostname/DNS
- port 636/389
- bind DN/password
- CA trust
- search base/filters
- Kubernetes egress/network policy

### Scenario 3: auth success, no permissions

This is authorization, not authentication.  
Check group claims and Argo CD RBAC mapping.

### Scenario 4: TLS/LDAPS certificate error

Look for x509/unknown authority/hostname mismatch/expired cert.

### Scenario 5: correct AD group not recognized by Argo CD

Breakpoints:
- group search filter
- group name format
- claim inclusion in token
- Argo CD claim/scope config
- RBAC string mismatch

---

## Key Concepts to Remember

- AD = directory service
- LDAP = protocol
- DEX = identity broker / OIDC provider
- OIDC = app authentication protocol
- Argo CD = application using the identity system
- Authentication = Who are you?
- Authorization = What are you allowed to do?
- RBAC = permissions based on roles/groups

Final memory line:

```text
User --HTTPS--> Argo CD --OIDC--> DEX --LDAPS--> Active Directory
                         ^                         |
                         +------- identity/claims -+
                         then Argo CD RBAC decides access
```
