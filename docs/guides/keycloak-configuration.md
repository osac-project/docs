# OSAC Keycloak Configuration

This guide documents the OSAC-specific Keycloak configuration: the "osac" realm,
required clients, custom scopes, group-based multi-tenancy, and how OSAC services
integrate with Keycloak. For general Keycloak installation and administration, see
the [Keycloak documentation](https://www.keycloak.org/documentation).

## Table of Contents

- [Overview](#overview)
- [OSAC Realm](#osac-realm)
  - [Clients](#clients)
  - [Client Scopes](#client-scopes)
  - [Groups and Multi-Tenancy](#groups-and-multi-tenancy)
  - [Realm Roles](#realm-roles)
- [Integration with OSAC Services](#integration-with-osac-services)
  - [Fulfillment Service Authentication](#fulfillment-service-authentication)
  - [Controller Credentials](#controller-credentials)
  - [CLI Authentication](#cli-authentication)
  - [Helm Values](#helm-values)
  - [Verification](#verification)
- [Deployment via OLM](#deployment-via-olm)
  - [Operator Installation](#operator-installation)
  - [Keycloak Instance](#keycloak-instance)
  - [Realm Import](#realm-import)
- [Upgrade Notes](#upgrade-notes)
- [BYO Keycloak](#byo-keycloak)

---

## Overview

OSAC uses Keycloak as its identity provider for OAuth 2.0 / OpenID Connect
authentication. All OSAC services authenticate against a single realm named
`osac`. The realm defines:

- **Clients** for the CLI, controller, and admin tools
- **Client scopes** that add OSAC-specific claims (audience, username, groups) to tokens
- **Groups** that map to OSAC tenants for multi-tenant isolation
- **Realm roles** for authorization (e.g., `tenant-admin`)

The authoritative realm definition lives in the
[fulfillment-service](https://github.com/osac-project/fulfillment-service)
repository at `it/charts/keycloak/files/realm.json`. The
[osac-installer](https://github.com/osac-project/osac-installer) maintains a
deployment-ready copy at `prerequisites/keycloak/files/realm.json`.

---

## OSAC Realm

Realm name: **`osac`**

### Clients

OSAC defines three application-specific clients. Standard Keycloak clients
(`account`, `admin-cli`, `broker`, etc.) are also present with default settings.

| Client ID | Type | Auth Method | Purpose |
|-----------|------|-------------|---------|
| `osac-cli` | Public | Direct access grants (password flow) | CLI tool authentication for end users |
| `osac-controller` | Confidential | Client credentials (service account) | Fulfillment controller → fulfillment API |
| `osac-admin` | Confidential | Client credentials (service account) | Administrative operations |

#### osac-cli

- **Public client** — no client secret required
- **Direct access grants enabled** — supports username/password login from the CLI
- **Redirect URI:** `http://localhost` (for local CLI callback)
- **Default scopes:** `groups`, `basic`, `username`, `organization`, `roles`, `osac-api`
- **Token claims:** username, group membership, `osac-api` audience

#### osac-controller

- **Confidential client** — requires a client secret
- **Service account enabled** — authenticates via client credentials grant
- **Service account username:** `service-account-osac-controller`
- **Client roles (realm-management):** `manage-realm`, `manage-users`, `view-realm`,
  `view-users`, `manage-identity-providers`, `view-identity-providers`
- **Default scopes:** standard Keycloak scopes (web-origins, profile, roles, email)

The controller's client secret is extracted from `realm.json` and stored in a
Kubernetes Secret (`fulfillment-controller-credentials`). The OPA authorization
policy expects the JWT `preferred_username` to be `service-account-osac-controller`.

#### osac-admin

- **Confidential client** — requires a client secret
- **Service account enabled** — authenticates via client credentials grant
- **Default scopes:** standard scopes plus `osac-api`
- **Token claims:** includes `osac-api` audience (required for API access)

### Client Scopes

Three custom client scopes add OSAC-specific claims to access tokens:

| Scope | Mapper Type | Claim | Description |
|-------|------------|-------|-------------|
| `osac-api` | Audience mapper | `aud: "osac-api"` | Adds the `osac-api` audience to access tokens. Required for the fulfillment API to accept the token. |
| `username` | User attribute mapper | `username` | Adds the user's username as a top-level claim in the token. |
| `groups` | Group membership mapper | `groups` | Adds the user's group memberships as an array claim. Used for tenant-scoped authorization. |

All three scopes are assigned as **default scopes** on the `osac-cli` client,
ensuring CLI tokens always contain these claims.

### Groups and Multi-Tenancy

OSAC maps Keycloak groups to tenants. Each group represents a tenant, and users
are assigned to groups to grant access to that tenant's resources.

| Group | Purpose |
|-------|---------|
| `admins` | Platform administrators |
| `tenant1` | Test tenant 1 |
| `tenant2` | Test tenant 2 |
| `my_group` | Test group |
| `your_group` | Test group |

The `groups` claim in the JWT token carries the user's group memberships. The
fulfillment service uses this claim to enforce tenant isolation — users can only
access resources belonging to their group(s).

Users can belong to multiple groups (e.g., the `twotenant` test user belongs to
both `tenant1` and `tenant2`).

### Realm Roles

| Role | Description |
|------|-------------|
| `tenant-admin` | Grants elevated permissions within a tenant (e.g., managing users, creating resources) |
| `default-roles-my realm` | Composite role assigned to all users (includes `offline_access`, `uma_authorization`, `view-profile`, `manage-account`) |

---

## Integration with OSAC Services

### Fulfillment Service Authentication

The fulfillment service validates JWTs issued by Keycloak. It connects to
Keycloak using these configuration values:

| Setting | Value | Description |
|---------|-------|-------------|
| `auth.issuerUrl` | `https://keycloak.keycloak.svc.cluster.local/realms/osac` | OIDC issuer URL for token validation |
| `idp.provider` | `keycloak` | Identity provider type |
| `idp.url` | `https://keycloak.keycloak.svc.cluster.local` | Keycloak base URL for admin operations |

The fulfillment service fetches the OIDC discovery document from
`{issuerUrl}/.well-known/openid-configuration` to discover JWKS endpoints for
token signature verification.

### Controller Credentials

The fulfillment controller authenticates to the fulfillment API using the
`osac-controller` client's credentials:

```bash
# Extract the client secret from realm.json
FC_CLIENT_SECRET=$(jq -r \
  '.clients[] | select(.clientId == "osac-controller") | .secret' \
  prerequisites/keycloak/files/realm.json)

# Create the Kubernetes secret
oc create secret generic fulfillment-controller-credentials \
  --from-literal=client-id=osac-controller \
  --from-literal=client-secret="${FC_CLIENT_SECRET}" \
  -n ${NAMESPACE} --dry-run=client -o yaml | oc apply -f -
```

### CLI Authentication

The `osac` CLI authenticates using the `osac-cli` public client with the
password grant flow:

```bash
# The CLI prompts for username/password and obtains a token from:
# https://keycloak.keycloak.svc.cluster.local/realms/osac/protocol/openid-connect/token
osac login --username tenant1_admin
```

### Helm Values

Configure the fulfillment service Helm chart to point to Keycloak:

```yaml
service:
  auth:
    issuerUrl: https://keycloak.keycloak.svc.cluster.local/realms/osac
    controllerCredentials:
      secretName: fulfillment-controller-credentials
  idp:
    provider: keycloak
    url: https://keycloak.keycloak.svc.cluster.local
```

These values are set in `osac-installer/values/*/values.yaml` and passed
during `helm upgrade --install`.

### Verification

After deploying Keycloak and the fulfillment service, verify the integration:

```bash
# 1. Check the Keycloak realm is accessible
curl -sk https://keycloak.keycloak.svc.cluster.local/realms/osac \
  | jq .realm
# Expected: "osac"

# 2. Obtain a token using the CLI client
TOKEN=$(curl -sk \
  https://keycloak.keycloak.svc.cluster.local/realms/osac/protocol/openid-connect/token \
  -d "client_id=osac-cli" \
  -d "username=tenant1_admin" \
  -d "password=foobar" \
  -d "grant_type=password" | jq -r .access_token)

# 3. Inspect the token claims
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq '{username, groups, aud}'
# Expected: username=tenant1_admin, groups=["/tenant1"], aud includes "osac-api"

# 4. Call the fulfillment API
curl -sk -H "Authorization: Bearer $TOKEN" \
  https://fulfillment-api-${NAMESPACE}.${DOMAIN}/v1/tenants
```

---

## Deployment via OLM

OSAC deploys Keycloak using the upstream Keycloak Operator via OLM (Operator
Lifecycle Manager). This section summarizes the deployment; for full details
see the [osac-installer prerequisites](https://github.com/osac-project/osac-installer/tree/main/prerequisites/keycloak).

### Operator Installation

```bash
# Apply the OLM manifests (Namespace + OperatorGroup + Subscription)
oc apply -f prerequisites/keycloak/operator.yaml

# Wait for the operator CSV to succeed
until KC_CSV=$(oc get csv --no-headers -n keycloak 2>/dev/null \
  | awk '/keycloak/ { print $1 }' | tail -1) && [[ -n "${KC_CSV}" ]]; do
  sleep 10
done
oc wait csv/${KC_CSV} -n keycloak \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s
```

The Subscription pins to a specific operator version via `startingCSV` with
`installPlanApproval: Manual` for controlled upgrades.

### Keycloak Instance

```bash
# Deploy the PostgreSQL database
oc apply -k prerequisites/keycloak/database/

# Deploy the Keycloak CR, TLS certificate, and Route
oc apply -f prerequisites/keycloak/keycloak-instance.yaml

# Wait for the Keycloak CR to be ready
oc wait keycloak/osac-keycloak -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}'=True \
  --timeout=600s
```

The operator creates a Service named `osac-keycloak-service` (derived from the
CR name `osac-keycloak`) on port 8443 (HTTPS).

### Realm Import

The OSAC realm is imported via a `KeycloakRealmImport` CR generated dynamically
from `realm.json`:

```bash
jq -n --arg realm "$(cat prerequisites/keycloak/files/realm.json)" '{
    apiVersion: "k8s.keycloak.org/v2beta1",
    kind: "KeycloakRealmImport",
    metadata: {name: "osac-realm-import", namespace: "keycloak"},
    spec: {keycloakCRName: "osac-keycloak", realm: ($realm | fromjson)}
}' | oc apply -f -

# Wait for the import to complete
oc wait keycloakrealmimport/osac-realm-import -n keycloak \
  --for=jsonpath='{.status.conditions[?(@.type=="Done")].status}'=True \
  --timeout=300s
```

---

## Upgrade Notes

For the complete upgrade and rollback procedure — including database backup,
operator upgrade via OLM, PostgreSQL version changes, and rollback steps — see
the [Keycloak Upgrade and Rollback Runbook](keycloak-upgrade-rollback.md).

Quick summary:

1. **Back up the database** before upgrading (`pg_dump`)
2. **Update the `startingCSV`** in `prerequisites/keycloak/operator.yaml` to the
   new version
3. **Approve the InstallPlan** (manual approval is required)
4. **Wait for the Keycloak CR** to return to Ready state after the operator upgrade
5. **Verify realm access** by checking the realm endpoint responds

For upstream Keycloak upgrade guidance, see the
[Keycloak Operator Upgrade Guide](https://www.keycloak.org/operator/upgrade).

---

## BYO Keycloak

OSAC supports using an existing (Bring Your Own) Keycloak instance. Requirements:

1. **Create an "osac" realm** with the clients, scopes, and groups described above
2. **Configure the Helm values** to point to your Keycloak instance:

```yaml
service:
  auth:
    issuerUrl: https://your-keycloak.example.com/realms/osac
  idp:
    provider: keycloak
    url: https://your-keycloak.example.com
```

3. **Create the controller credentials secret** with the `osac-controller` client
   ID and secret from your realm
4. **Ensure the `osac-api` audience scope** is configured on your clients — the
   fulfillment service validates this audience in tokens

The realm.json in the osac-installer repository can be imported into your
Keycloak instance as a starting point, then customized for your environment.
