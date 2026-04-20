# Console Access

Console access gives end users direct serial console connectivity to their
running compute instances. This enables interactive troubleshooting, initial
setup, and recovery operations — particularly useful when network-based access
(SSH) is unavailable due to misconfigured networking, firewall rules, or
boot-time issues.

## Overview

OSAC provides serial console access to compute instances through a gRPC
streaming API. The console feature bridges the gap between the Fulfillment
Service API (where users manage their resources) and the KubeVirt serial
console subresource (where the actual VM runs on the Management Cluster).

The design follows a key architectural principle: **the user never interacts
directly with the Management Cluster**. All access is mediated through the
Fulfillment Service, which enforces authentication, authorization, and
multi-tenancy boundaries. The user identifies their resource by its fulfillment
ID — the Fulfillment Service resolves that to the correct Management Cluster,
namespace, and VM, then proxies the console connection transparently.

## Deployment and Configuration

Console access is built into the Fulfillment Service — there are no additional
services to deploy. However, several configuration items must be in place for
it to work correctly.

### Envoy Ingress Proxy

The Envoy ingress proxy that sits in front of the Fulfillment Service's gRPC
server applies a default request timeout to all routes. Console sessions are
long-lived streaming connections that would be terminated by this timeout.

A dedicated route must be configured for the `Console/Connect` RPC with
`timeout: 0s` to disable Envoy's request timeout for console streams. This
route is included in the default OSAC installer manifests
(`fulfillment-service/manifests/base/ingress-proxy/configmap.yaml`).

If you are using a custom Envoy configuration, ensure the console route is
present **before** the default gRPC catch-all route:

```yaml
- name: console
  match:
    safe_regex:
      regex: ^/osac\.public\.v1\.Console/Connect$
  route:
    cluster: grpc-server
    timeout: 0s
```

### Session Timeout

The server enforces a maximum session duration (default 30 minutes). This can
be configured via the `OSAC_CONSOLE_SESSION_TIMEOUT` environment variable on
the Fulfillment Service deployment (e.g., `1h`, `45m`).

The timeout is a hard deadline — sessions are terminated regardless of
activity. Users can also set a shorter client-side timeout using the CLI's
`--timeout` flag, but cannot exceed the server limit.

### Network Requirements

Console connections traverse the following network path:

```
Client → OpenShift Route (TLS passthrough) → Envoy → gRPC Server → Hub K8s API (TLS) → KubeVirt WebSocket (TLS)
```

For console access to function, the following network connectivity must exist:

| From | To | Protocol | Purpose |
|------|----|----------|---------|
| Client | Fulfillment API route | HTTPS (gRPC) | Console stream |
| Fulfillment Service pod | Hub cluster K8s API | HTTPS | ComputeInstance CR lookup + WebSocket |

No additional ports need to be opened beyond what is already required for
normal Fulfillment Service operation. The Fulfillment Service already has
network access to hub cluster APIs for CR management — console access reuses
the same path.

### KubeVirt Prerequisites

KubeVirt exposes serial console access by default for all VirtualMachineInstances.
No additional KubeVirt configuration is required on the Management Cluster.

The Fulfillment Service connects to the KubeVirt console subresource API
(`/apis/subresources.kubevirt.io/v1/namespaces/<ns>/virtualmachineinstances/<vmi>/console`)
using the hub's kubeconfig credentials. The service account associated with
the hub kubeconfig must have permission to access this subresource.

The OSAC installer includes the required RBAC (`base/hub-access/rbac.yaml`).
If you are using a custom deployment, ensure the hub-access service account
has a **ClusterRole** (not a namespace-scoped Role) granting `get` access
to `virtualmachineinstances/console` — VMs may span multiple namespaces
depending on tenant and subnet configuration:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: hub-access-console
rules:
- apiGroups:
  - subresources.kubevirt.io
  resources:
  - virtualmachineinstances/console
  verbs:
  - get
```

Bind it to the hub-access service account with a ClusterRoleBinding.

## Template Configuration

For serial console to be useful, the guest OS inside the VM must be configured
to present a login prompt on the virtio serial port. Without this, the console
connection succeeds at the infrastructure level but the user sees no output.

**CSPs should include serial console configuration in their default VM
templates** so that console access works out of the box for all tenants,
without requiring end users to configure it themselves.

The recommended approach is to include cloud-init configuration in the
template defaults that enables the `serial-getty@ttyS0` systemd service
(login prompt on serial port):

```yaml
#cloud-config
runcmd:
  - systemctl enable serial-getty@ttyS0.service
  - systemctl start serial-getty@ttyS0.service
```

This ensures the serial console presents a login prompt. Users are
responsible for setting a console password in their own cloud-init
configuration — templates should not include default passwords. Users can
add password configuration when creating their compute instance:

```yaml
#cloud-config
password: <their-password>
chpasswd:
  expire: false
```

See the `ocp_virt_vm` template for a reference implementation.

**Platform considerations:**

- **Fedora / CentOS / RHEL**: `serial-getty@ttyS0` is available by default
  but not enabled. Cloud-init runcmd is the simplest way to enable it.
- **Ubuntu**: Same approach works. The `ttyS0` serial port is standard for
  KubeVirt VMs.
- **Windows**: Serial console access is not supported through this mechanism.
  Windows VMs would require VNC console support (see Future Considerations).

## Monitoring and Troubleshooting

### Log Messages

The console server emits structured log messages at key points. Operators
should monitor for the following:

| Log Level | Message | Meaning |
|-----------|---------|---------|
| INFO | `Console connect request` | A user initiated a console connection |
| INFO | `Opening console session` | Session established, includes user and timeout |
| INFO | `Closing console session` | Session ended normally |
| INFO | `Console session timed out` | Server-side timeout expired |
| WARN | `Running compute instance not found on hub` | VM is running in DB but the CR is missing on the Management Cluster — indicates a sync issue |
| WARN | `Running compute instance has no VM reference on hub` | CR exists but KubeVirt VM reference is not yet populated — may indicate provisioning is still in progress or the OSAC Controller is not reconciling |
| ERROR | `Failed to open console backend connection` | WebSocket to KubeVirt failed — typically missing RBAC or hub connectivity. Includes hub, namespace, VM name, and error details |
| INFO | `Console proxy ended with error` | Backend connection dropped unexpectedly |

### Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Console unavailable, "not running" | VM not yet in Running state | Wait for provisioning to complete |
| Console unavailable, "not found on hub" | ComputeInstance CR missing from hub | Check OSAC Controller logs on the Management Cluster; verify the CR exists |
| Console unavailable, "no VM reference" | CR exists but `status.virtualMachineReference` is empty | Check OSAC Controller reconciliation; the KubeVirt VM may not have been created yet |
| Console connects but no output | Guest OS serial console not configured | Update the VM template to include serial-getty cloud-init configuration |
| Console connects but login fails | No password set in guest OS | Update cloud-init to set a password |
| Session terminates after 30 minutes | Server-side timeout (default) | Increase `OSAC_CONSOLE_SESSION_TIMEOUT` if longer sessions are needed |
| "active console session" error | Another user is connected | Only one session per VM is allowed; the first session must disconnect |
| CLI reconnect loop, ERROR log with 403 | Hub-access service account missing KubeVirt RBAC | Add the `hub-access-console` ClusterRole and ClusterRoleBinding (see KubeVirt Prerequisites) |
| Connection drops immediately | Envoy timeout killing the stream | Verify the console route has `timeout: 0s` in the Envoy config |

### Capacity Considerations

Each active console session consumes:
- One gRPC stream on the Fulfillment Service
- One WebSocket connection to the hub cluster's KubeVirt API
- Two goroutines for bidirectional proxying
- Minimal memory (4 KB read buffer per session)

The resource overhead per session is small. For most deployments, console
sessions will not be a bottleneck. Note that console sessions are stateful
(held in memory on the replica that accepted the connection). Horizontal
scaling of the Fulfillment Service with concurrent console sessions has not
been validated — session affinity or shared session state would need to be
considered if this becomes a requirement.

## Web UI Integration

CSPs that operate their own web portal can integrate console access using the
Fulfillment Service API. The console is accessible through the same
gRPC/REST API that powers the CLI, so no additional backend services are
needed.

**High-level integration approach:**

1. **Check availability** using the `Console/GetAccess` RPC before showing a
   "Connect" button in the UI. This returns whether the console is available
   and which console types are supported (currently serial only).

2. **Open a bidirectional stream** using the `Console/Connect` RPC. The first
   message must specify the resource type, resource ID, and console type.

3. **Embed a terminal emulator** in the browser (e.g.,
   [xterm.js](https://xtermjs.org/)) and wire it to the stream:
   - Server → Client: raw bytes from the VM serial console, rendered by the
     terminal emulator.
   - Client → Server: keystrokes from the terminal emulator, forwarded to
     the VM.

4. **Handle lifecycle events** using status messages from the server:
   `CONNECTING`, `CONNECTED`, `DISCONNECTED` (with reason).

**Protocol options:** The Fulfillment Service exposes both gRPC and REST
(via gRPC-Gateway). For bidirectional streaming from a browser, the gRPC-Web
protocol or WebSocket-based gRPC transport are both viable. The OSAC UI
(if used) can serve as a reference implementation.

**Authentication:** The web UI must forward the user's bearer token in the
gRPC metadata. The same OAuth2/OIDC tokens used for other API calls work for
console access.

## Architecture

### Request Flow

```
 User (CLI / Web UI)
       │
       │  1. Console/Connect (gRPC bidi stream)
       ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                    Fulfillment Service                       │
 │                                                             │
 │  Envoy Ingress Proxy                                        │
 │       │                                                     │
 │       │  (route: Console/Connect, timeout: 0s)              │
 │       ▼                                                     │
 │  Console gRPC Server                                        │
 │       │                                                     │
 │       │  2. Get compute instance from DB (private API)      │
 │       │     → Verify state = Running                        │
 │       │     → Read hub ID from status                       │
 │       │                                                     │
 │       │  3. Get hub kubeconfig (private Hubs API)           │
 │       │                                                     │
 │       │  4. Query hub K8s API for ComputeInstance CR         │
 │       │     → List by label: computeinstance-uuid=<id>      │
 │       │     → Read status.virtualMachineReference           │
 │       │        (namespace + kubeVirtVirtualMachineName)      │
 │       │                                                     │
 │       │  5. Open WebSocket to KubeVirt serial console       │
 │       │     → wss://<hub>/apis/subresources.kubevirt.io/    │
 │       │       v1/namespaces/<ns>/                           │
 │       │       virtualmachineinstances/<vmi>/console          │
 │       │                                                     │
 │       │  6. Proxy: gRPC stream ↔ WebSocket (bidirectional)  │
 │       │                                                     │
 └───────┼─────────────────────────────────────────────────────┘
         │
         │  WebSocket (binary frames)
         ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                    Management Cluster (Hub)                  │
 │                                                             │
 │  KubeVirt VirtualMachineInstance                             │
 │       └─ serial console subresource                         │
 │          (virtio serial port / ttyS0)                        │
 └─────────────────────────────────────────────────────────────┘
```

### VM Reference Resolution

The console server must resolve a fulfillment UUID to a specific namespace and
VM name on a specific Management Cluster. It does this by querying the
Management Cluster directly:

1. The Fulfillment Service database stores the **hub ID** for each compute
   instance (assigned during scheduling). This identifies which Management
   Cluster the VM runs on.

2. The OSAC Controller on the Management Cluster labels each ComputeInstance
   CR with `osac.openshift.io/computeinstance-uuid=<fulfillment-id>`. This
   label is set at CR creation time and is immutable.

3. The console server queries the hub's Kubernetes API for ComputeInstance CRs
   matching that label, then reads `status.virtualMachineReference` which
   contains the namespace and KubeVirt VM name.

This approach avoids the need to synchronize VM reference data back to the
Fulfillment Service database — the Management Cluster is the source of truth
for where the VM actually runs.

### Session Management

- **Single session per resource**: Only one console session can be active for a
  given compute instance at a time. Concurrent attempts receive an error with
  details about the existing session.
- **Server-side timeout**: Fixed deadline (default 30 minutes, configurable).
  Sessions are terminated regardless of activity.
- **Automatic cleanup**: Sessions are cleaned up on client disconnect, backend
  connection drop, or timeout expiry.

### Authentication and Multi-Tenancy

Console access is governed by the same authentication and authorization model
as all other Fulfillment Service APIs:

- Users must present a valid bearer token (OAuth2/OIDC via Keycloak, or a
  service account token).
- Compute instance lookups are tenant-scoped — users can only access consoles
  for instances belonging to their tenant.
- Users never interact with the Management Cluster's Kubernetes API. The
  Fulfillment Service mediates all access using its own service credentials.

## End-User Guide

### CLI Usage

```
fulfillment-cli console computeinstance <name-or-id>
```

- **Name or ID**: Users can reference compute instances by display name or UUID.
- **Disconnect**: `Ctrl+]` (immediate) or `Enter ~.` (SSH-style escape).
- **Auto-reconnect**: On unexpected drops, the CLI reconnects automatically
  with exponential backoff. Permanent errors (auth failure, VM not found)
  cause immediate exit.
- **Timeout**: `--timeout` flag sets a client-side session limit
  (e.g., `--timeout 1h`).

<details><summary><b>Creating a VM with Console Access</b></summary>

Save the following as `cloud-init.yaml`:

```yaml
#cloud-config
password: console123
chpasswd:
  expire: false
ssh_pwauth: true
runcmd:
  - systemctl enable serial-getty@ttyS0.service
  - systemctl start serial-getty@ttyS0.service
```

Create the compute instance:

```bash
fulfillment-cli create computeinstance \
  --template osac.templates.ocp_virt_vm \
  --name my-vm \
  --cores 2 \
  --memory-gib 4 \
  --boot-disk-size 20 \
  --image quay.io/containerdisks/fedora:latest \
  --run-strategy Always \
  -p cloud_init_config="$(base64 -w0 < cloud-init.yaml)"
```

Once the VM reaches `RUNNING` state, connect:

```bash
fulfillment-cli console computeinstance my-vm
```

</details>

## Future Considerations

- **Bare metal console**: The backend interface supports additional resource
  types. A bare metal console backend (e.g., IPMI Serial over LAN) could
  provide console access to physical servers through the same API and CLI.
  The API already includes a `HOST` resource type for this purpose.
- **VNC console**: The API includes a `ConsoleType` enum and resize messages,
  designed to support graphical VNC console access. This would enable console
  access for Windows VMs and graphical Linux desktops.
- **Console proxy extraction**: If console sessions become a scalability
  bottleneck, the console package is designed to be self-contained and could
  be extracted into a standalone microservice with independent scaling.
- **Activity-based timeout refresh**: The fixed session timeout could be
  enhanced to reset on user activity, preventing active sessions from being
  terminated while the user is still working.
- **Session limits**: Per-user or per-tenant concurrent session caps could be
  introduced if console resource usage becomes a concern.
- **Session recording**: Record console I/O for compliance auditing and
  security forensics. All bytes proxied through the server could be logged
  to a secure store with timestamps and user identity.
- **Audit logging**: Log console session start/end events with user identity
  for compliance and security auditing.
- **Hub client caching**: The current implementation creates a new Kubernetes
  client for each console connection. A hub client cache could improve
  performance for repeated connections.
