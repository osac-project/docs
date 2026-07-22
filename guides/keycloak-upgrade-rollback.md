# Keycloak Upgrade and Rollback Runbook

This guide covers how to upgrade the OSAC Keycloak deployment (operator, server, and
database) and how to roll back if something goes wrong. For general Keycloak
configuration, see [keycloak-configuration.md](keycloak-configuration.md).

## Table of Contents

- [Overview](#overview)
- [Pre-Upgrade Checklist](#pre-upgrade-checklist)
- [Database Backup](#database-backup)
- [Upgrade Procedure](#upgrade-procedure)
  - [Operator Upgrade (OLM)](#operator-upgrade-olm)
  - [PostgreSQL Upgrade](#postgresql-upgrade)
- [Post-Upgrade Verification](#post-upgrade-verification)
- [Rollback Procedure](#rollback-procedure)
  - [When to Roll Back](#when-to-roll-back)
  - [Full Rollback (Operator + Database)](#full-rollback-operator--database)
  - [PostgreSQL-Only Rollback](#postgresql-only-rollback)
- [Version Compatibility Matrix](#version-compatibility-matrix)

---

## Overview

OSAC's Keycloak deployment has three independently versioned layers:

| Layer | What changes | How it's controlled | Rollback path |
|-------|-------------|--------------------|--------------------|
| **Keycloak Operator** | Operator pod image | OLM `Subscription` with `installPlanApproval: Manual` | Uninstall + reinstall with previous `startingCSV` |
| **Keycloak Server** | Keycloak pod image (StatefulSet) | Managed automatically by the operator — the operator version determines the server version | Restore DB from backup + reinstall old operator |
| **PostgreSQL** | Database container image | Manual update in `prerequisites/keycloak/database/statefulset.yaml` | Revert the image tag (data on PVC is unchanged) |

The operator and server versions move in lockstep — each operator release pins its
own Keycloak server image. You cannot independently upgrade the server without
upgrading the operator.

**Key constraint**: Keycloak applies forward-only database migrations (via Liquibase)
every time the server starts after an upgrade. These schema changes cannot be
automatically undone. A database backup taken *before* the upgrade is the only way to
roll back safely.

---

## Pre-Upgrade Checklist

Run these checks before starting any upgrade.

### 1. Record current versions

```bash
# Operator version (CSV)
oc get csv -n keycloak -o custom-columns=NAME:.metadata.name,PHASE:.status.phase

# Keycloak server image
oc get pod -l app=keycloak -n keycloak \
    -o jsonpath='{.items[0].spec.containers[0].image}'

# PostgreSQL image
oc get statefulset keycloak-database -n keycloak \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Save the output — you will need these values if you need to roll back.

### 2. Verify Keycloak is healthy

```bash
# Keycloak CR should be Ready
oc get keycloak osac-keycloak -n keycloak \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

# Realm import should be Done
oc get keycloakrealmimport osac-realm-import -n keycloak \
    -o jsonpath='{.status.conditions[?(@.type=="Done")].status}'
# Expected: True
```

### 3. Verify fulfillment service is healthy

```bash
# All fulfillment pods should be Running
oc get pods -n osac -l app.kubernetes.io/part-of=fulfillment-service --no-headers

# Controller should have no recent auth errors
oc logs deploy/fulfillment-controller -n osac --tail=20 | grep -i "error"
```

### 4. Check available storage

Keycloak migrations may temporarily increase database size:

```bash
KC_DB_PASS=$(oc get secret keycloak-db-credentials -n keycloak \
    -o jsonpath='{.data.password}' | base64 -d)

oc exec keycloak-database-0 -n keycloak -- \
    bash -c "PGPASSWORD='${KC_DB_PASS}' psql -h localhost -U keycloak \
    -c \"SELECT pg_size_pretty(pg_database_size('keycloak'));\""
```

---

## Database Backup

Always back up the Keycloak database before upgrading the operator or server.

### Create the backup

The Keycloak database uses mTLS for TCP connections (`pg_hba.conf` requires
client certificate auth). Use the `postgres` superuser via local unix socket
instead of authenticating as `keycloak` over TCP:

```bash
BACKUP_FILE="keycloak-backup-$(date +%Y%m%d-%H%M%S).dump"

oc exec keycloak-database-0 -n keycloak -c server -- \
    pg_dump -U postgres -Fc keycloak \
    > "${BACKUP_FILE}"

echo "Backup saved to ${BACKUP_FILE} ($(du -h "${BACKUP_FILE}" | cut -f1))"
```

The `-Fc` flag creates a custom-format dump that supports selective restore and
compression. The `-c server` flag selects the database container (the
StatefulSet also has a `password-generator` init container).

> **Note:** The `PGPASSWORD` + `-h localhost` approach documented in some
> guides does not work when `pg_hba.conf` is configured for `hostssl cert`
> authentication. The unix socket approach bypasses TCP authentication entirely
> using the `postgres` superuser's `local peer` entry.

### Verify the backup

Verification must run inside the postgres container (the host may not have a
compatible `pg_restore`):

```bash
oc cp "${BACKUP_FILE}" keycloak/keycloak-database-0:/tmp/backup-verify.dump -c server
oc exec keycloak-database-0 -n keycloak -c server -- \
    pg_restore --list /tmp/backup-verify.dump | head -20
```

You should see a table of contents listing Keycloak tables (e.g., `realm`,
`client`, `user_entity`, `credential`). If this command fails, the backup is
corrupt — do not proceed with the upgrade.

### Backup retention

Keep at least the last two backups. Store them off-cluster if possible (e.g.,
on the bastion host or an object store).

---

## Upgrade Procedure

### Operator Upgrade (OLM)

The Keycloak operator is installed via OLM with `installPlanApproval: Manual`. OLM
creates an `InstallPlan` when a new version is available in the channel, but does not
apply it until you approve.

#### Step 1: Check for pending upgrades

```bash
oc get installplan -n keycloak
```

A pending upgrade shows `APPROVED=false`:

```
NAME            CSV                           APPROVED
install-abc12   keycloak-operator.v26.7.0     false
```

If no pending InstallPlan exists and you want to upgrade to a specific version,
update the `startingCSV` in `prerequisites/keycloak/operator.yaml`:

```yaml
spec:
  startingCSV: keycloak-operator.v26.7.0   # new target version
```

Then re-apply:

```bash
oc apply -f prerequisites/keycloak/operator.yaml
```

#### Step 2: Back up the database

See [Database Backup](#database-backup) above. Do not skip this step.

#### Step 3: Approve the InstallPlan

```bash
PLAN_NAME=$(oc get installplan -n keycloak \
    -o jsonpath='{.items[?(@.spec.approved==false)].metadata.name}')

oc patch installplan "${PLAN_NAME}" -n keycloak \
    --type=merge -p '{"spec":{"approved":true}}'
```

#### Step 4: Wait for the new operator to be ready

```bash
# Wait for the new CSV to reach Succeeded
oc wait csv -n keycloak -l operators.coreos.com/keycloak-operator.keycloak= \
    --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s

# The operator automatically updates the Keycloak StatefulSet image.
# Wait for the Keycloak pod to restart with the new version.
oc rollout status statefulset/osac-keycloak -n keycloak --timeout=600s
```

#### Step 5: Wait for database migrations

The new Keycloak server runs Liquibase migrations on startup. This can take a few
minutes depending on the version gap. Watch the Keycloak pod logs:

```bash
oc logs -f statefulset/osac-keycloak -n keycloak | grep -i "liquibase\|migrat"
```

Wait until you see `Keycloak ... started` in the logs before proceeding.

#### Step 6: Verify

See [Post-Upgrade Verification](#post-upgrade-verification).

### PostgreSQL Upgrade

PostgreSQL upgrades are independent of the Keycloak operator version. Only upgrade
PostgreSQL when the current version falls out of the
[compatibility matrix](#version-compatibility-matrix) or when a security patch is
needed.

#### Minor version updates (e.g., 18.1 to 18.2)

Minor updates are safe — they do not change the on-disk data format:

```bash
# Update the image tag in the StatefulSet manifest
# Edit: prerequisites/keycloak/database/statefulset.yaml
#   image: quay.io/sclorg/postgresql-18-c10s:latest

oc apply -k prerequisites/keycloak/database/ -n keycloak
oc rollout status statefulset/keycloak-database -n keycloak --timeout=300s
```

#### Major version updates (e.g., 15 to 18)

Major PostgreSQL updates require a dump-and-restore because the on-disk format
changes between major versions:

1. Back up the database (see [Database Backup](#database-backup))
2. Scale down Keycloak: `oc patch keycloak osac-keycloak -n keycloak --type=merge -p '{"spec":{"instances":0}}'`
3. Delete the old StatefulSet and PVC
4. Update the image in `statefulset.yaml` to the new major version
5. Re-apply the database manifests: `oc apply -k prerequisites/keycloak/database/ -n keycloak`
6. Restore the backup into the new database:
   ```bash
   KC_DB_PASS=$(oc get secret keycloak-db-credentials -n keycloak \
       -o jsonpath='{.data.password}' | base64 -d)
   oc exec -i keycloak-database-0 -n keycloak -- \
       bash -c "PGPASSWORD='${KC_DB_PASS}' pg_restore -h localhost -U keycloak \
       -d keycloak --clean --if-exists" < keycloak-backup-YYYYMMDD.dump
   ```
7. Scale Keycloak back up: `oc patch keycloak osac-keycloak -n keycloak --type=merge -p '{"spec":{"instances":1}}'`
8. Verify (see [Post-Upgrade Verification](#post-upgrade-verification))

---

## Post-Upgrade Verification

Run these checks after every upgrade.

### 1. Keycloak CR health

```bash
oc get keycloak osac-keycloak -n keycloak \
    -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

Expected output:

```
Ready=True
HasErrors=False
RollingUpdate=False
```

### 2. Token acquisition test

```bash
KC_SVC="https://keycloak.keycloak.svc.cluster.local"

# From inside the cluster (e.g., oc debug or oc run):
oc run token-test --rm -i --restart=Never --image=curlimages/curl -- \
    -sk -X POST "${KC_SVC}/realms/osac/protocol/openid-connect/token" \
    -d 'grant_type=password&client_id=admin-cli&username=tenant1_admin&password=foobar'
```

A successful response contains `"access_token":"eyJ..."`.

### 3. Fulfillment controller connectivity

```bash
# Should show no auth-related errors in recent logs
oc logs deploy/fulfillment-controller -n osac --tail=10 --since=5m \
    | grep -i "error\|fail\|denied"
```

### 4. Realm integrity

```bash
KC_ROUTE="https://$(oc get route keycloak -n keycloak -o jsonpath='{.spec.host}')"

# Get admin token (from inside cluster or with route access)
ADMIN_TOKEN=$(curl -sk -X POST "${KC_SVC}/realms/master/protocol/openid-connect/token" \
    -d 'grant_type=password&client_id=admin-cli' \
    -d "username=$(oc get secret osac-keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d)" \
    -d "password=$(oc get secret osac-keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d)" \
    | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

# Check realm clients
curl -sk -H "Authorization: Bearer ${ADMIN_TOKEN}" \
    "${KC_SVC}/admin/realms/osac/clients" | python3 -m json.tool | grep clientId

# Check realm users
curl -sk -H "Authorization: Bearer ${ADMIN_TOKEN}" \
    "${KC_SVC}/admin/realms/osac/users?max=50" | python3 -m json.tool | grep username
```

Verify that the expected clients (`osac-controller`, `admin-cli`, etc.) and users
(`tenant1_admin`, `tenant1_user`, etc.) are present.

---

## Rollback Procedure

### When to Roll Back

Roll back if any of the following occur after an upgrade:

- Keycloak CR is stuck in a non-Ready state for more than 10 minutes
- Token acquisition fails (OIDC endpoints return errors)
- Fulfillment controller logs show persistent authentication failures
- Realm data is missing (clients, users, or groups disappeared)
- Keycloak pod is in `CrashLoopBackOff` after the Liquibase migration

### Full Rollback (Operator + Database)

This procedure restores both the operator and the database to the pre-upgrade state.
You need the backup file created during [Database Backup](#database-backup) and the
previous operator version (the `startingCSV` value you recorded in the pre-upgrade
checklist).

#### Step 1: Scale down Keycloak

Stop the Keycloak server to prevent further database changes:

```bash
oc patch keycloak osac-keycloak -n keycloak \
    --type=merge -p '{"spec":{"instances":0}}'

# Verify the pod is gone
oc wait --for=delete pod/osac-keycloak-0 -n keycloak --timeout=120s
```

#### Step 2: Restore the database

Keycloak's Liquibase migrations are forward-only — they add tables and foreign
key constraints that `pg_restore --clean` cannot remove in dependency order. The
only reliable restore method is to drop the entire database and recreate it:

```bash
# Copy backup into the container
oc cp keycloak-backup-YYYYMMDD.dump \
    keycloak/keycloak-database-0:/tmp/restore.dump -c server

# Drop and recreate the database
oc exec keycloak-database-0 -n keycloak -c server -- \
    psql -U postgres -c "DROP DATABASE keycloak;"
oc exec keycloak-database-0 -n keycloak -c server -- \
    psql -U postgres -c "CREATE DATABASE keycloak OWNER keycloak;"

# Restore from backup
oc exec keycloak-database-0 -n keycloak -c server -- \
    pg_restore -U postgres -d keycloak /tmp/restore.dump
```

Replace `YYYYMMDD` with the actual backup date.

> **Why not `pg_restore --clean --if-exists`?** Keycloak upgrades add new
> tables with foreign key constraints pointing to existing tables (e.g.,
> `user_ver_credential` → `user_entity`). The `--clean` flag tries to drop
> tables in dump order, but cannot drop `user_entity` because the new
> (post-upgrade) FK constraints still reference it. This results in partial
> restores with data integrity issues. A full `DROP DATABASE` avoids this
> entirely.

Verify the restore succeeded:

```bash
oc exec keycloak-database-0 -n keycloak -c server -- \
    psql -U postgres -d keycloak -c "SELECT count(*) FROM realm;"
```

#### Step 3: Uninstall the current operator

```bash
# Get the current CSV name
CURRENT_CSV=$(oc get csv -n keycloak -o jsonpath='{.items[0].metadata.name}')

# Delete the OLM resources
oc delete subscription keycloak-operator -n keycloak
oc delete csv "${CURRENT_CSV}" -n keycloak
oc delete installplan --all -n keycloak
oc delete operatorgroup keycloak-operator -n keycloak
```

#### Step 4: Reinstall the previous version

Edit `prerequisites/keycloak/operator.yaml` to use the previous `startingCSV`:

```yaml
spec:
  startingCSV: keycloak-operator.v26.6.4   # previous version
```

Apply it:

```bash
oc apply -f prerequisites/keycloak/operator.yaml
```

#### Step 5: Approve the InstallPlan

```bash
# Wait for OLM to create the InstallPlan
sleep 30
PLAN_NAME=$(oc get installplan -n keycloak \
    -o jsonpath='{.items[?(@.spec.approved==false)].metadata.name}')

oc patch installplan "${PLAN_NAME}" -n keycloak \
    --type=merge -p '{"spec":{"approved":true}}'

# Wait for the CSV to be ready
oc wait csv -n keycloak -l operators.coreos.com/keycloak-operator.keycloak= \
    --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s
```

#### Step 6: Scale Keycloak back up

```bash
oc patch keycloak osac-keycloak -n keycloak \
    --type=merge -p '{"spec":{"instances":1}}'

oc rollout status statefulset/osac-keycloak -n keycloak --timeout=600s
```

#### Step 7: Verify

Run the full [Post-Upgrade Verification](#post-upgrade-verification) procedure.

### PostgreSQL-Only Rollback

If only the PostgreSQL image was changed (no operator/server upgrade), the rollback
is simpler because no schema migrations are involved:

```bash
# Revert the image in statefulset.yaml to the previous version, then:
oc apply -k prerequisites/keycloak/database/ -n keycloak
oc rollout status statefulset/keycloak-database -n keycloak --timeout=300s
```

The data on the PVC is untouched — PostgreSQL minor versions use the same on-disk
format.

---

## Version Compatibility Matrix

The Keycloak operator pins the server version it manages. Each operator release
includes a specific Keycloak server image. The table below shows PostgreSQL
compatibility by Keycloak version:

| Keycloak Version | Operator CSV | Supported PostgreSQL | Notes |
|-----------------|-------------|---------------------|-------|
| 26.7.x | `keycloak-operator.v26.7.0` | 14.x, 15.x, 16.x, 17.x, 18.x | Upgrade + rollback verified on edge-16 |
| 26.6.x | `keycloak-operator.v26.6.4` | 14.x, 15.x, 16.x, 17.x, 18.x | Current OSAC version |
| 26.5.x | `keycloak-operator.v26.5.x` | 14.x, 15.x, 16.x, 17.x, 18.x | PostgreSQL 13 dropped |
| 26.4.x | `keycloak-operator.v26.4.x` | 13.x, 14.x, 15.x, 16.x, 17.x | Last version with PG 13 |

**Important notes:**

- PostgreSQL 13 reached end-of-life in November 2025. Keycloak 26.5+ dropped support
  for it. If you are on PostgreSQL 13, upgrade PostgreSQL before upgrading Keycloak
  past 26.4.
- The OSAC deployment currently uses `quay.io/sclorg/postgresql-18-c10s:latest`
  (PostgreSQL 18), which is compatible with all Keycloak 26.x versions.
- Always check the [Keycloak release notes](https://www.keycloak.org/docs/latest/upgrading/)
  for breaking changes before upgrading. Pay particular attention to the
  "Migrating to ..." sections that document database schema changes.
- The operator and server versions are coupled — you cannot run a 26.6 operator with a
  26.4 server or vice versa.

### Checking the upstream upgrade path

Before upgrading, verify the upgrade path is valid:

```bash
# List available versions in the OLM channel
oc get packagemanifest keycloak-operator -o jsonpath='{.status.channels[?(@.name=="fast")].entries[*].name}'
```

OLM enforces an upgrade graph — it will only offer valid upgrade paths. If a version
is not listed as available from your current version, you may need to upgrade through
intermediate versions.
