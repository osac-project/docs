# VM Fulfillment

VM fulfillment is the process of provisioning individual virtual machines
on-demand for tenants. OSAC uses a template-based approach that enables Cloud
Service Providers (CSPs) to offer standardized VM configurations while
maintaining flexibility to customize the underlying infrastructure and
deployment process.

## Overview

VM fulfillment in OSAC leverages OpenShift Virtualization (based on KubeVirt) to
deploy virtual machines as pods. This approach allows VMs to benefit from
Kubernetes' orchestration capabilities while providing traditional VM
functionality to end users. VMs are isolated per tenant, with each tenant
receiving a dedicated namespace for their compute resources.

The VM fulfillment workflow follows the general pattern described in the [main
architecture document](README.md#workflow), with specific implementations for VM
provisioning:

1. A VM request (ComputeInstance) is submitted to the Fulfillment Service API
2. The Fulfillment Service schedules the request to a Management Cluster
3. The OSAC Controller on the Management Cluster processes the request and triggers Event Driven Ansible
4. Ansible Automation Platform executes the VM provisioning workflow using template-based automation
5. Status is synchronized back through the controller to the Fulfillment Service

## Components

### Fulfillment Service - ComputeInstance API

The Fulfillment Service provides gRPC and REST APIs for managing VM lifecycle operations:

**API Operations** (`fulfillment-api/proto/fulfillment/v1/compute_instances_service.proto`):
- `Create`: Request a new VM deployment
- `Get`: Retrieve VM details and status
- `List`: List all VMs for a tenant
- `Update`: Modify VM configuration or request restart
- `Delete`: Request VM deletion

**ComputeInstance Request Model** (`fulfillment-api/proto/fulfillment/v1/compute_instance_type.proto`):

A ComputeInstance request includes:
- `spec.template`: The VM template ID (e.g., "ocp_virt_vm")
- `spec.template_parameters`: A map of parameters specific to the selected template, using protobuf `Any` type to support various parameter types
- `spec.restart_requested_at`: Optional timestamp to request a VM restart

**Example gRPC API Request** (JSON representation):

```json
{
  "spec": {
    "template": "ocp_virt_vm",
    "template_parameters": {
      "cpu_cores": {
        "@type": "type.googleapis.com/google.protobuf.Int32Value",
        "value": 4
      },
      "memory": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "8Gi"
      },
      "disk_size": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "50Gi"
      },
      "image_source": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "quay.io/containerdisks/fedora:latest"
      },
      "ssh_public_key": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "c3NoLXJzYSBBQUFBQjNOemFDMXljMkVBQUFBREFRQUJBQUFCQVFERXhhbXBsZQ=="
      }
    }
  }
}
```

**ComputeInstance Status**:

The VM status provides information about the deployment:
- `state`: Overall VM state (PROGRESSING, READY, FAILED)
- `conditions`: Detailed conditions tracking specific aspects of the VM
- `ip_address`: IP address assigned to the VM when ready
- `last_restarted_at`: Timestamp of the last restart (when using restart functionality)

### Management Cluster Scheduling

The Fulfillment Service implements a scheduling algorithm to select an appropriate Management Cluster for each request (`fulfillment-service/internal/controllers/computeinstance/computeinstance_reconciler_function.go`):

1. **Hub Selection**: Currently uses random selection from available Management Clusters. Future enhancements may consider:
   - Capacity and resource availability
   - Geographic or availability zone preferences
   - Load balancing across hubs
   - Tenant affinity rules

2. **ComputeInstance Creation**: Once a Management Cluster is selected, the Fulfillment Service creates a `ComputeInstance` custom resource in a dedicated namespace for VM orders. This object contains:
   - The selected template ID
   - Template parameters (JSON-encoded)
   - Labels identifying the compute instance UUID
   - Annotations identifying the tenant

The ComputeInstance serves as the bridge between the Fulfillment Service and the
OSAC Controller running on the Management Cluster.

### OSAC Controller - ComputeInstance Processing

The OSAC Controller is a Kubernetes controller running on each Management
Cluster that reconciles ComputeInstance resources
(`osac-operator/internal/controller/computeinstance_controller.go`).

**ComputeInstance Custom Resource** (`osac-operator/api/v1alpha1/computeinstance_types.go`):

The ComputeInstance CRD defines:
- `TemplateID`: Identifies which VM template to use
- `TemplateParameters`: JSON-encoded map of template-specific parameters

**Example Kubernetes API Resource**:

```yaml
apiVersion: cloudkit.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: vm-abc123
  namespace: user123-compute-instances
  labels:
    cloudkit.openshift.io/compute-instance-uuid: "550e8400-e29b-41d4-a716-446655440000"
  annotations:
    cloudkit.openshift.io/tenant: "tenant-user123"
spec:
  templateID: ocp_virt_vm
  templateParameters: |
    {
      "cpu_cores": 4,
      "memory": "8Gi",
      "disk_size": "50Gi",
      "image_source": "quay.io/containerdisks/fedora:latest",
      "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDExample...",
      "cloud_init_config": "I2Nsb3VkLWNvbmZpZwp1c2VyczogCi0gbmFtZTogZmVkb3Jh...",
      "exposed_ports": "22/tcp,80/tcp,443/tcp"
    }
status:
  phase: Ready
  conditions:
  - type: Accepted
    status: "True"
    lastTransitionTime: "2025-03-12T20:10:00Z"
  - type: Progressing
    status: "False"
    lastTransitionTime: "2025-03-12T20:15:00Z"
  - type: Available
    status: "True"
    lastTransitionTime: "2025-03-12T20:15:00Z"
  virtualMachineReference:
    namespace: tenant-user123
    kubeVirtVirtualMachineName: vm-abc123
  tenantReference:
    name: tenant-user123
    namespace: cloudkit-compute-instances
  desiredConfigVersion: "a1b2c3d4e5f6"
  reconciledConfigVersion: "a1b2c3d4e5f6"
```

**Reconciliation Loop**:

When a ComputeInstance is created or updated, the controller performs these steps:

1. **Status Initialization**: Sets the phase to "Progressing" and initializes status conditions

2. **Tenant Management**: Creates or verifies a Tenant resource exists:
   - The **Tenant** is a Kubernetes custom resource that represents the tenant
   - When the Tenant controller processes a Tenant CR, it creates a dedicated Kubernetes **Namespace** for that tenant, and a **UserDefinedNetwork** for that namespace
   - The name of the created namespace is recorded in the Tenant's `status.namespace` field
   - All of the tenant's VMs and related resources are created in this tenant namespace, and belong to the same UserDefinedNetwork
   - The controller waits for Tenant to reach "Ready" state before proceeding with VM provisioning

3. **Configuration Versioning**:
   - Computes `DesiredConfigVersion`: A hash (FNV-1a) of the ComputeInstance spec
   - Checks `ReconciledConfigVersion`: Set by ansible after successful provisioning via annotation
   - Proceeds with provisioning only if versions don't match

4. **EDA Webhook Trigger**: Calls the Event Driven Ansible webhook endpoint to initiate VM provisioning:
   - The webhook payload includes the complete ComputeInstance specification
   - EDA runs a playbook that creates all necessary Kubernetes resources for the VM and does any required infrastructure management

5. **VirtualMachine Monitoring**: After the webhook is triggered, the controller watches for a VirtualMachine resource (created by the Ansible automation) and monitors its conditions:
   - Waits for the VM to become Ready
   - Updates ComputeInstance status based on VirtualMachine state
   - Stores reference to VirtualMachine in status

6. **Status Synchronization**: Continuously updates the ComputeInstance status, including:
   - Phase transitions (Progressing → Ready or Failed)
   - Condition updates
   - VM reference information (namespace, resource name)

7. **Deletion Handling**: When a ComputeInstance is deleted:
   - Triggers the deletion webhook to EDA
   - Ensures all associated VM resources are cleaned up
   - Uses finalizers to prevent premature deletion

### Ansible Automation Platform - VM Provisioning

Event Driven Ansible (EDA) running on AAP receives webhook events from the OSAC Controller and orchestrates the actual VM provisioning.

**EDA Rulebook** (`osac-aap/rulebooks/cluster_fulfillment.yml`):

The rulebook listens on port 5000 and defines rules for VM lifecycle events:
- Endpoint: `create-compute-instance` → Triggers the VM creation job
- Endpoint: `delete-compute-instance` → Triggers the VM deletion job

When triggered, EDA launches the appropriate job template in AAP.

**VM Creation Playbook** (`osac-aap/playbook_cloudkit_create_compute_instance.yml`):

The main VM creation workflow consists of these phases:

1. **Information Extraction**:
   - Receives the ComputeInstance as EDA event payload
   - Extracts template ID and parameters
   - Determines the working namespace (Tenant namespace)

2. **Finalizer Management**:
   - Adds an AAP finalizer to the ComputeInstance to track cleanup responsibility

3. **Template Execution**:
   - Templates are Ansible roles located in the path specified by `CLOUDKIT_TEMPLATE_COLLECTIONS` environment variable
   - Each template provides a standardized interface while allowing customization

4. **Version Annotation**:
   - After successful provisioning, annotates the ComputeInstance with `cloudkit.openshift.io/reconciled-config-version`
   - This signals to the controller that provisioning is complete

**VM Deletion Playbook** (`osac-aap/playbook_cloudkit_delete_compute_instance.yml`):

The deletion workflow:
1. Extracts template information from the ComputeInstance
2. Calls the template's `delete` tasks
3. Removes the AAP finalizer

## VM Templates

VM templates are implemented as Ansible roles that define how VMs are provisioned. Each template:

- Accepts standardized parameters (defined via role argument validation)
- Can customize VM configuration, resource allocation, and initialization
- Implements a `create.yaml` tasks file that performs the provisioning
- Implements a `delete.yaml` tasks file that deletes resources associated with the VM

**Template Metadata** (`osac-templates/roles/ocp_virt_vm/meta/cloudkit.yaml`):

Each template defines metadata:
- `title`: Human-readable name
- `description`: Explanation of what the template provides
- `template_type`: Set to "vm" (vs "cluster")

**Template Parameters** (`osac-templates/roles/ocp_virt_vm/meta/argument_specs.yaml`):

The default `ocp_virt_vm` template accepts these parameters:
- `cpu_cores`: Number of CPU cores (default: 2)
- `memory`: Amount of memory (default: 2Gi)
- `disk_size`: Size of the VM root disk (default: 10Gi)
- `image_source`: Container disk image URL (default: "quay.io/containerdisks/fedora:latest")
- `cloud_init_config`: Base64-encoded cloud-init configuration
- `ssh_public_key`: SSH public key for VM access
- `exposed_ports`: Comma-separated list of ports to expose (default: "22/tcp")

**Template Implementation** (`osac-templates/roles/ocp_virt_vm/tasks/create.yaml`):

This particular template creates the following Kubernetes resources:

1. **cloud-init Secret**: Contains user data for VM initialization
2. **SSH Public Key Secret**: Optional, contains the SSH public key
3. **DataVolume**: Provisions the root disk from a container image
4. **VirtualMachine**: The main VM resource with:
   - CPU and memory specifications
   - Disk configuration (root disk + cloud-init disk)
   - Network interface (masquerade mode for pod networking)
   - VirtIO device configuration
   - HyperV optimizations

**Example VirtualMachine Resource Created by Template**:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-abc123
  namespace: tenant-user123
  labels:
    cloudkit.openshift.io/compute-instance-name: vm-abc123
spec:
  running: true
  template:
    metadata:
      labels:
        cloudkit.openshift.io/compute-instance-name: vm-abc123
    spec:
      domain:
        cpu:
          cores: 4
        memory:
          guest: 8Gi
        devices:
          disks:
          - name: root-disk
            disk:
              bus: virtio
          - name: cloud-init-disk
            disk:
              bus: virtio
            serial: cloud-init
          interfaces:
          - name: default
            masquerade: {}
          rng: {}
        features:
          smm:
            enabled: true
          acpi: {}
          apic: {}
          hyperv:
            relaxed: {}
            vapic: {}
            spinlocks:
              spinlocks: 8191
      networks:
      - name: default
        pod: {}
      volumes:
      - name: root-disk
        dataVolume:
          name: vm-abc123-root-disk
      - name: cloud-init-disk
        cloudInitNoCloud:
          secretRef:
            name: vm-abc123-cloud-init
      accessCredentials:
      - sshPublicKey:
          source:
            secret:
              secretName: vm-abc123-ssh-public-key
          propagationMethod:
            noCloud: {}
```

CSPs can create custom templates to offer differentiated VM configurations, such as:
- VMs with specific images or operating systems
- Different resource tiers (small, medium, large)
- Specialized hardware configurations (GPU-enabled VMs)
- Pre-configured software installations
- Integrations with CSP infrastructure (network fabric, secret store, DNS, etc.)

## Infrastructure Components

The VM provisioning workflow integrates with several infrastructure components:

**OpenShift Virtualization (KubeVirt)**:
- VMs run as pods on the Management Cluster using KubeVirt
- Each VM consists of:
  - **VirtualMachine**: Defines the desired state of the VM
  - **VirtualMachineInstance**: The running instance (created when VM is started)
  - **DataVolume**: Persistent storage for the VM disk
  - **Secrets**: Store cloud-init and SSH key data
- This architecture enables:
  - Fast VM provisioning (no bare metal allocation needed)
  - Kubernetes-native management and monitoring
  - Live migration capabilities
  - Integration with Kubernetes storage and networking

**Storage**:
- VMs use DataVolumes (part of Containerized Data Importer / CDI)
- DataVolumes provision PersistentVolumeClaims from various sources:
  - Container images (containerdisks)
  - HTTP sources
  - Existing PVCs (cloning)
  - S3 buckets
- Storage class is selected based on:
  - Default storage class in the cluster
  - Environment variable `CLOUDKIT_VM_OPERATIONS_STORAGE_CLASS`
- CSPs can customize storage options per template

**Tenant Isolation**:
- Each tenant receives a dedicated namespace and UserDefinedNetwork
- All VM resources are created in the tenant namespace
- Kubernetes RBAC provides access control
- Network policies can provide additional isolation

## Status Tracking and Reporting

VM status flows through multiple levels:

1. **KubeVirt VirtualMachine Status**: The source of truth for VM state
   - Tracks if VM is Running, Stopped, or in other states
   - Reports Ready condition when VM is fully operational
   - Provides printable status (Running, Stopped, Starting, etc.)

2. **ComputeInstance Status**: Maintained by the OSAC Controller
   - Aggregates VirtualMachine status
   - Tracks provisioning phase (Progressing/Ready/Failed/Deleting)
   - Includes detailed conditions for troubleshooting
   - Stores reference to the VirtualMachine resource

3. **Fulfillment Service ComputeInstance Status**: Synchronized from k8s ComputeInstance
   - Provides tenant-facing status information
   - Includes IP address when ready
   - Maintains conditions visible through the API

**Key Status Conditions**:
- `Accepted`: ComputeInstance has been accepted for processing
- `Progressing`: VM provisioning is in progress
- `Available`: VM is running and ready to use
- `Deleting`: VM is being deleted
- `RestartInProgress`: A restart operation is underway (when using restart_requested_at)
- `RestartFailed`: A restart operation failed

## VM Operations

### VM Restart

VMs can be restarted using a declarative timestamp-based mechanism:

1. User updates `spec.restart_requested_at` to the current time
2. Controller detects that `restart_requested_at` > `status.last_restarted_at`
3. Controller triggers restart workflow
4. After restart completes, controller sets `status.last_restarted_at` to `spec.restart_requested_at`

This mechanism enables:
- Immediate restarts (set timestamp to NOW)
- Scheduled restarts (external scheduler sets timestamp)
- Idempotent restart requests (same timestamp won't trigger multiple restarts)

### VM Deletion

When a VM deletion is requested:

1. **Fulfillment Service**: Deletes the ComputeInstance, triggering deletion timestamp
2. **OSAC Controller**: Detects deletion and:
   - Triggers the deletion webhook to EDA
   - Sets ComputeInstance phase to "Deleting"
3. **Ansible Automation**: Executes the deletion playbook which:
   - Stops the VirtualMachine
   - Deletes the VirtualMachine resource
   - Deletes associated DataVolumes (root disk)
   - Deletes Secrets (cloud-init, SSH keys)
   - Deletes any Services created for the VM
4. **OSAC Controller**: Finalizes ComputeInstance deletion after AAP removes its finalizer

## Execution Flow

The complete VM fulfillment execution path:

1. **API Request**: User calls `ComputeInstances.Create()` on Fulfillment Service
2. **Fulfillment Service**:
   - Validates tenant
   - Selects Management Cluster (hub) randomly
   - Creates ComputeInstance CR in hub namespace with:
     - Label: `cloudkit.openshift.io/compute-instance-uuid: <uuid>`
     - Annotation: `cloudkit.openshift.io/tenant: <tenant-id>`
     - Spec containing template ID and parameters
3. **OSAC Controller** (reconcile loop):
   - Detects new ComputeInstance
   - Adds controller finalizer
   - Creates or verifies Tenant resource exists
   - Computes DesiredConfigVersion (hash of spec)
   - Checks ReconciledConfigVersion (from AAP annotation)
   - If versions differ, triggers webhook: POST to `http://eda-service:5000/create-compute-instance`
4. **Event Driven Ansible**:
   - Receives webhook at endpoint `create-compute-instance`
   - Routes to job template: `<prefix>-create-compute-instance`
   - Launches Ansible job
5. **Ansible Playbook**:
   - Receives ComputeInstance as event payload
   - Extracts template ID (e.g., "ocp_virt_vm") and parameters
   - Determines tenant working namespace
   - Adds AAP finalizer: `cloudkit.openshift.io/aap`
   - Includes template role with `tasks_from: create`
6. **VM Template Role** (`ocp_virt_vm/tasks/create.yaml`):
   - Creates cloud-init Secret in tenant namespace
   - Creates SSH public key Secret (if provided)
   - Creates DataVolume for root disk
   - Creates VirtualMachine resource with:
     - CPU, memory, disk specifications
     - Network interfaces
     - Volume mounts
   - Waits for VirtualMachine to reach Ready state
   - Annotates ComputeInstance with reconciled config version
7. **OSAC Controller** (watching VirtualMachine):
   - Detects VirtualMachine created/updated
   - Maps back to ComputeInstance via label
   - Updates ComputeInstance status:
     - Stores VM reference
     - Sets Available condition to True when VM is Ready
     - Copies reconciled version from annotation
   - When DesiredConfigVersion == ReconciledConfigVersion:
     - Sets phase to "Ready"
     - Sets Progressing condition to False
8. **Fulfillment Service** (feedback controller):
   - Watches ComputeInstance on hub
   - Syncs status back to API
   - Updates conditions, state, and IP address
9. **API Response**: User receives updated status with VM IP address

## Scalability and Performance

The VM fulfillment system is designed for scale:

**Management Cluster Capacity**:
- Each Management Cluster can host a large number of VMs
- Capacity depends on:
  - Available CPU, memory, and storage resources
  - Network bandwidth
  - Node count and size
- VMs share the same infrastructure as hosted control planes

**Provisioning Throughput**:
- Multiple VMs can be provisioned concurrently on a single Management Cluster
- EDA webhook rate limiting prevents overwhelming AAP
- VM creation is typically faster than bare metal allocation

**Horizontal Scaling**:
- Additional Management Clusters can be added to increase total capacity
- The Fulfillment Service load balances across available Management Clusters
- Each Management Cluster operates independently

## Customization and Extension

CSPs can customize VM fulfillment in several ways:

**Custom Templates**: Create new Ansible role-based templates that:
- Use different base images or operating systems
- Pre-install required software
- Configure VMs for specialized workloads (databases, web servers, etc.)
- Integrate with CSP-specific infrastructure

**Template Parameters**: Expose configuration options to end users:
- Operating system selection
- Software package installation
- Networking configuration
- Storage options

**Workflow Extensions**: Add pre- or post-provisioning steps:
- Registration with external systems
- Monitoring and observability setup
- Backup configuration
- Compliance scanning

**Integration Points**: Connect with external systems:
- DNS registration for VM hostnames
- IPAM for IP address management
- Backup systems
- Configuration management tools
