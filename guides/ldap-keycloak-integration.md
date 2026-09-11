# OSAC LDAP + Keycloak Setup Guide

**Last Updated**: 2026-06-23
**Scope**: How to configure LDAP and Keycloak to work together for OSAC authentication
**Audience**: OSAC engineering team, platform operators

---

## Table of Contents

1. [Overview](#overview)
2. [Login Flow](#login-flow)
3. [LDAP Side — Setup & Configuration](#ldap-side--setup--configuration)
4. [Keycloak Side — Setup & Configuration](#keycloak-side--setup--configuration)
5. [Verification](#verification)
6. [User Lifecycle](#user-lifecycle)
7. [User Sync](#user-sync)
8. [Limitations](#limitations)

---

## Overview

OSAC uses Keycloak as its identity provider with LDAP federation to support
enterprise directory authentication. When configured, Keycloak delegates
credential verification to an external LDAP server while managing JWT issuance,
session management, and user attribute storage.

The request lifecycle has two distinct phases:

1. **Authentication** — LDAP → Keycloak → JWT
2. **Authorization** — Envoy → Authorino → OPA → fulfillment-service

Keycloak handles phase 1. The OSAC authorization chain (Envoy, Authorino, OPA)
handles phase 2. The two phases are decoupled — the LDAP server is never
contacted during authorization.

### Component Roles

| Component | Responsibility |
|-----------|---------------|
| **LDAP Server** (389-ds / RHDS) | Source of truth for users and credentials |
| **Keycloak** | Identity broker — federates LDAP, issues JWTs, manages sessions |
| **Envoy** | API gateway — routes gRPC/REST, delegates auth via `ext_authz` |
| **Authorino** | External authorization — validates JWT signature, issuer, expiry |
| **OPA/Rego** | Policy engine — evaluates JWT claims, determines allow/deny |
| **fulfillment-service** | Business logic — receives pre-authorized requests with `x-subject` header |

---

## Login Flow

When an LDAP user logs in through the OSAC CLI, the request passes through
Keycloak (which delegates to LDAP) and then through the authorization chain
on every subsequent API call.

```
 LDAP User             OSAC CLI              Keycloak (osac realm)        LDAP Server
      │                   │                        │                          │
      │  osac login       │                        │                          │
      │  user/password    │                        │                          │
      │ ─────────────────►│                        │                          │
      │                   │  POST /token           │                          │
      │                   │  grant_type=password   │                          │
      │                   │  client_id=            │                          │
      │                   │  fulfillment-cli       │                          │
      │                   │ ──────────────────────►│                          │
      │                   │                        │                          │
      │                   │                        │  1. Check local DB       │
      │                   │                        │     → user not found     │
      │                   │                        │                          │
      │                   │                        │  2. Query federation     │
      │                   │                        │     providers (priority  │
      │                   │                        │     order)               │
      │                   │                        │                          │
      │                   │                        │  3. LDAP Bind            │
      │                   │                        │     dn: uid=<user>,      │
      │                   │                        │         ou=users,...     │
      │                   │                        │     password: <pass>     │
      │                   │                        │ ──────────────────────► │
      │                   │                        │                          │
      │                   │                        │  4. Bind result          │
      │                   │                        │     + user attributes    │
      │                   │                        │     (uid, cn, sn, mail)  │
      │                   │                        │ ◄────────────────────── │
      │                   │                        │                          │
      │                   │                        │  5. Import/update user   │
      │                   │                        │     in Keycloak DB       │
      │                   │                        │                          │
      │                   │                        │  6. Build & sign JWT     │
      │                   │  7. JWT access token   │                          │
      │                   │ ◄──────────────────── │                          │
      │                   │                        │                          │
      │  8. Token stored  │                        │                          │
      │     locally       │                        │                          │
      │ ◄─────────────── │                        │                          │
```

**Key behavior:**

- Keycloak performs an LDAP bind on **every login** — credentials are never cached.
- On successful bind, Keycloak imports/updates user attributes in its local DB.
- The JWT is signed by Keycloak, not the LDAP server. LDAP's only role is
  credential verification and attribute sourcing.

---

## LDAP Side — Setup & Configuration

### Prerequisites

- **Operating System**: RHEL 8/9, CentOS Stream 9, or Fedora
- **LDAP Server Software**: 389 Directory Server (`389-ds-base` package) or Red Hat Directory Server (RHDS)
- **Hardware**: Minimum 2 vCPU, 2 GB RAM, 10 GB disk
- **Network**: Static IP or resolvable hostname accessible from the Keycloak pod network
- **Ports**: 389 (LDAP) and optionally 636 (LDAPS) open in the firewall
- **Root/sudo access** on the LDAP server host
- **Tools installed**: `ldapsearch`, `ldapadd`, `ldapwhoami` (part of `openldap-clients` package)

```bash
# Install LDAP client tools (if not present)
dnf install -y openldap-clients
```

### Step 1: Install 389 Directory Server

```bash
# On CentOS/RHEL
dnf install -y 389-ds-base
```

### Step 2: Create the Directory Instance

```bash
cat > /tmp/ds-setup.inf <<EOF
[general]
full_machine_name = ldap.example.com
start = True

[slapd]
instance_name = osac
root_dn = cn=Directory Manager
root_password = <strong-password>
port = 389
secure_port = 636

[backend-userroot]
suffix = dc=osac,dc=example,dc=com
sample_entries = no
create_suffix_entry = True
EOF

dscreate from-file /tmp/ds-setup.inf
```

### Step 3: Create the Directory Structure

```bash
ldapadd -x -H ldap://localhost:389 \
    -D "cn=Directory Manager" -w <password> <<EOF
dn: ou=users,dc=osac,dc=example,dc=com
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=osac,dc=example,dc=com
objectClass: organizationalUnit
ou: groups
EOF
```

### Step 4: Add Users

Each user must have the `inetOrgPerson` objectClass at minimum:

```bash
ldapadd -x -H ldap://localhost:389 \
    -D "cn=Directory Manager" -w <password> <<EOF
dn: uid=alice,ou=users,dc=osac,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: alice
cn: Alice Admin
sn: Admin
mail: alice@example.com
userPassword: <user-password>
o: acme-corp
EOF
```

**Required attributes:**

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `uid` | Username (must be unique) | `alice` |
| `cn` | Display name | `Alice Admin` |
| `sn` | Surname | `Admin` |
| `mail` | Email address | `alice@example.com` |
| `userPassword` | Password (hashed by LDAP server) | `secretpass` |

**Optional attributes:**

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `o` | Organization (for future org-scoped federation) | `acme-corp` |

### Step 5: Add Groups (Optional)

```bash
ldapadd -x -H ldap://localhost:389 \
    -D "cn=Directory Manager" -w <password> <<EOF
dn: cn=acme-admins,ou=groups,dc=osac,dc=example,dc=com
objectClass: groupOfUniqueNames
cn: acme-admins
description: Admins for acme-corp
uniqueMember: uid=alice,ou=users,dc=osac,dc=example,dc=com
EOF
```

### Step 6: Verify LDAP is Working

```bash
# Search for users
ldapsearch -x -H ldap://localhost:389 \
    -D "cn=Directory Manager" -w <password> \
    -b "ou=users,dc=osac,dc=example,dc=com" \
    "(objectClass=inetOrgPerson)" uid cn mail

# Test authentication (bind as user)
ldapwhoami -x -H ldap://localhost:389 \
    -D "uid=alice,ou=users,dc=osac,dc=example,dc=com" \
    -w <user-password>
```

### Directory Structure Summary

```
dc=osac,dc=example,dc=com              ← Base DN
├── ou=users                            ← Users DN (Keycloak searches here)
│   ├── uid=alice                       ← inetOrgPerson
│   ├── uid=bob
│   └── uid=charlie
├── ou=groups                           ← Groups (optional, for RBAC)
│   ├── cn=acme-admins
│   └── cn=acme-viewers
└── ou=organizations                    ← Org metadata (optional)
    ├── ou=acme-corp
    └── ou=globex
```

### Network Requirements

Keycloak pods must reach the LDAP server on port 389 (or 636 for LDAPS).
In segmented networks, ensure:

- Firewall allows traffic from the Keycloak pod network to the LDAP server
- DNS resolves the LDAP hostname from within the cluster
- If on different subnets, iptables/routing rules are in place

---

## Keycloak Side — Setup & Configuration

### Prerequisites

- Keycloak running with the `osac` realm created
- Admin access to Keycloak (username/password or admin token)
- A Keycloak OAuth client (e.g., `fulfillment-cli`) with **Direct Access Grants** enabled
- Network connectivity from Keycloak to the LDAP server

### Step 1: Obtain Admin Token

```bash
KC_URL="https://keycloak.example.com"
```

> **Note:** The `-k` flag in curl commands below disables TLS certificate
> verification. This is acceptable for lab environments with self-signed
> certificates. In production, configure a trusted CA and remove the `-k` flag.

```bash
KC_TOKEN=$(curl -ks "${KC_URL}/realms/master/protocol/openid-connect/token" \
    -d 'client_id=admin-cli' \
    -d 'grant_type=password' \
    -d 'username=admin' \
    -d 'password=<admin-password>' | jq -r '.access_token')
```

### Step 2: Create LDAP Federation Provider

Register the LDAP server as a User Storage Provider in the `osac` realm:

```bash
curl -ks -X POST "${KC_URL}/admin/realms/osac/components" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "name": "osac-ldap",
  "providerId": "ldap",
  "providerType": "org.keycloak.storage.UserStorageProvider",
  "config": {
    "editMode":              ["READ_ONLY"],
    "vendor":                ["rhds"],
    "connectionUrl":         ["ldap://<ldap-host>:389"],
    "bindDn":                ["cn=Directory Manager"],
    "bindCredential":        ["<bind-password>"],
    "usersDn":               ["ou=users,dc=osac,dc=example,dc=com"],
    "usernameLDAPAttribute": ["uid"],
    "rdnLDAPAttribute":      ["uid"],
    "uuidLDAPAttribute":     ["nsuniqueid"],
    "userObjectClasses":     ["inetOrgPerson"],
    "searchScope":           ["2"],
    "pagination":            ["true"],
    "importEnabled":         ["true"],
    "syncRegistrations":     ["false"],
    "batchSizeForSync":      ["1000"],
    "fullSyncPeriod":        ["-1"],
    "changedSyncPeriod":     ["-1"],
    "trustEmail":            ["true"],
    "enabled":               ["true"],
    "priority":              ["0"]
  }
}'
```

### Step 3: Configuration Explained

| Setting | Value | Why |
|---------|-------|-----|
| `editMode` | `READ_ONLY` | LDAP is the source of truth — Keycloak cannot modify it |
| `vendor` | `rhds` | Optimizes attribute mapping for 389-ds / RHDS |
| `connectionUrl` | `ldap://<host>:389` | LDAP server address |
| `bindDn` | `cn=Directory Manager` | Service account for LDAP queries |
| `bindCredential` | `<password>` | Service account password |
| `usersDn` | `ou=users,<base-dn>` | Where to search for user entries |
| `usernameLDAPAttribute` | `uid` | LDAP attribute → Keycloak username |
| `uuidLDAPAttribute` | `nsuniqueid` | 389-ds UUID (use `entryUUID` for OpenLDAP) |
| `userObjectClasses` | `inetOrgPerson` | Filter: only import entries with this class |
| `searchScope` | `2` (subtree) | Search all entries under `usersDn` |
| `importEnabled` | `true` | Auto-import user on first login |
| `syncRegistrations` | `false` | Don't push new KC users back to LDAP |
| `trustEmail` | `true` | Mark LDAP emails as verified |
| `fullSyncPeriod` | `-1` | No automatic full sync (trigger manually) |
| `changedSyncPeriod` | `-1` | No automatic changed-user sync |
| `priority` | `0` | First provider checked when user not found locally |

### Step 4: Trigger Initial User Sync

Import existing LDAP users into Keycloak:

```bash
# Get the federation provider ID
COMP_ID=$(curl -ks -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/components?type=org.keycloak.storage.UserStorageProvider" \
    | jq -r '.[] | select(.name=="osac-ldap") | .id')

# Trigger full sync
curl -ks -X POST -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/user-storage/${COMP_ID}/sync?action=triggerFullSync"
```

### Step 5: Create OAuth Client (if not exists)

The client is what the OSAC CLI uses to request tokens:

```bash
curl -ks -X POST "${KC_URL}/admin/realms/osac/clients" \
    -H "Authorization: Bearer ${KC_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
  "clientId": "fulfillment-cli",
  "enabled": true,
  "publicClient": true,
  "directAccessGrantsEnabled": true,
  "standardFlowEnabled": false
}'
```

Key settings:
- `publicClient: true` — no client secret needed (CLI tool)
- `directAccessGrantsEnabled: true` — allows username/password login (Resource Owner Password Grant)

### Step 6: Attribute Mappers (Auto-Created)

When you select `vendor: rhds`, Keycloak auto-creates these mappers:

| Mapper | LDAP Attribute | Keycloak Field | Direction |
|--------|---------------|----------------|-----------|
| username | `uid` | username | LDAP → KC |
| first name | `cn` | firstName | LDAP → KC |
| last name | `sn` | lastName | LDAP → KC |
| email | `mail` | email | LDAP → KC |
| modify date | `modifyTimestamp` | (internal) | LDAP → KC |
| creation date | `createTimestamp` | (internal) | LDAP → KC |

---

## Verification

### Test LDAP Connectivity from Keycloak

```bash
# From a pod on the same network as Keycloak
ldapsearch -x -H ldap://<ldap-host>:389 \
    -D "cn=Directory Manager" -w <password> \
    -b "ou=users,dc=osac,dc=example,dc=com" \
    "(uid=alice)" uid cn mail
```

### Test Login via Keycloak

```bash
KC_URL="https://keycloak.example.com"

# Request a token for an LDAP user
curl -ks "${KC_URL}/realms/osac/protocol/openid-connect/token" \
    -d 'client_id=fulfillment-cli' \
    -d 'grant_type=password' \
    -d 'username=alice' \
    -d 'password=<user-password>' | jq .
```

**Expected success:**
```json
{
  "access_token": "eyJhbG...",
  "token_type": "Bearer",
  "expires_in": 300
}
```

**Expected failure (wrong password):**
```json
{
  "error": "invalid_grant",
  "error_description": "Invalid user credentials"
}
```

### Verify User is Federated

```bash
curl -ks -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/users?username=alice&exact=true" \
    | jq '.[0] | {username, email, firstName, lastName, federationLink}'
```

If `federationLink` is set, the user is LDAP-federated.

---

## User Lifecycle

### Adding a New User

1. Add the user to LDAP:
```bash
ldapadd -x -H ldap://<ldap-host>:389 \
    -D "cn=Directory Manager" -w <password> <<EOF
dn: uid=newuser,ou=users,dc=osac,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: newuser
cn: New User
sn: User
mail: newuser@example.com
userPassword: <password>
EOF
```

2. The user can now log in through Keycloak immediately — no sync needed.

### Deleting a User

1. **Login is blocked immediately** when the LDAP entry is removed (LDAP bind fails).
2. **The Keycloak record persists** — full sync does NOT automatically remove orphaned users.

Keycloak's sync process only imports and updates; it does not compare its local
database against LDAP to purge deleted entries.

**Recommended procedure:**

1. Lock the account in LDAP: `nsAccountLock: true` (389-ds)
2. Revoke active sessions via Keycloak admin API
3. Delete from LDAP: `ldapdelete uid=user,...`
4. Remove the orphaned KC record using one of these methods:
   - **Reactive removal** (preferred): Enable "Remove invalid users during searches"
     in the LDAP provider settings. The orphan is deleted the next time it is
     accessed (login attempt, admin search, or API lookup).
   - **Manual API deletion**: Delete the specific user via the Keycloak admin API:
     ```bash
     curl -ks -X DELETE -H "Authorization: Bearer ${KC_TOKEN}" \
         "${KC_URL}/admin/realms/osac/users/<user-id>"
     ```
   - **Nuclear option**: Use "Remove imported" in the Admin Console. **Warning:**
     this removes ALL imported LDAP users from Keycloak, not just the deleted one.
     They will be re-imported on their next login.

### Changing a Password

Passwords are managed in LDAP, not Keycloak (`READ_ONLY` mode):

```bash
ldappasswd -x -H ldap://<ldap-host>:389 \
    -D "cn=Directory Manager" -w <admin-password> \
    -s <new-password> \
    "uid=alice,ou=users,dc=osac,dc=example,dc=com"
```

The change takes effect on the next login — no sync needed.

---

## User Sync

Keycloak can synchronize user records from LDAP. Sync imports/updates
attributes but **never** copies passwords.

### Sync Modes

| Mode | What it does | When to use |
|------|-------------|-------------|
| **On-login import** | Imports user on first successful login | Default behavior, always active |
| **Full sync** | Imports/updates ALL users from LDAP | After initial setup or bulk LDAP changes |
| **Changed-user sync** | Imports only modified users | Periodic catchup without full scan |

### Triggering Sync

```bash
# Full sync (imports/updates all LDAP users — does NOT remove deleted users)
curl -ks -X POST -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/user-storage/${COMP_ID}/sync?action=triggerFullSync"

# Changed users sync (imports modified only)
curl -ks -X POST -H "Authorization: Bearer ${KC_TOKEN}" \
    "${KC_URL}/admin/realms/osac/user-storage/${COMP_ID}/sync?action=triggerChangedUsersSync"
```

### What Gets Synced

| Item | Synced? | Direction |
|------|---------|-----------|
| User attributes (uid, cn, sn, mail) | Yes | LDAP → Keycloak |
| Passwords | **No** | Verified via LDAP bind at login |
| New users in LDAP | Yes | Imported to Keycloak |
| Deleted users in LDAP | **No** | Orphan persists until accessed (with "Remove invalid users" enabled) or manually deleted |
| LDAP groups | Requires Group Mapper | Not configured by default |

---

## Limitations

### 1. Global IdP — LDAP Federation is Realm-Wide

LDAP federation is configured at the **realm** level, not per-organization.
All organizations share the same LDAP provider.

**Impact:** No isolation between tenants at the identity layer.

**Future:** Organization-scoped IdP integration (OSAC-1399) — each org
would bring its own OIDC-compliant IdP (backed by their LDAP).

### 2. Keycloak Organizations — Disabled

The LDAP `o` attribute is imported but has **no effect** on JWT claims.
All users fall through to the "shared" tenant.

**Blocker:** PR #576 + OSAC-204.

### 3. Username Uniqueness — Realm-Wide Constraint

If two LDAP directories both have `alice`, only the first one imported wins.

**Production solution:** KC Organizations + per-org IdPs.

### 4. Delayed User Deprovisioning

Deleted LDAP users: login blocked immediately, but KC record persists
until full sync. Enable periodic full sync to auto-clean.

### 5. No LDAP Group Mapping

LDAP groups have no effect on JWT roles. All users get `is_client`.

**Fix:** Add a Group Mapper to the LDAP federation provider.

### 6. READ_ONLY Mode

Users cannot change passwords through Keycloak. This is intentional —
LDAP is the source of truth.

### 7. No LDAPS (TLS) by Default

Credentials are transmitted in cleartext. Acceptable for lab/dev only.
Production deployments must use LDAPS (port 636) with proper certificates.

### Summary Matrix

| Limitation | Severity | Path to Resolution |
|-----------|----------|-------------------|
| Global IdP (realm-wide) | High | Org-scoped IdP API (OSAC-1399) |
| KC Organizations disabled | High | Enable orgs (PR #576, OSAC-204) |
| Username uniqueness | Medium | Per-org IdP with KC Orgs |
| Delayed deprovisioning | Medium | Enable periodic full sync |
| No LDAP group mapping | Medium | Add Group Mapper |
| READ_ONLY mode | Low | By design |
| No LDAPS | High (prod) | Configure TLS |
