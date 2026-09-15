# Per-service enablement

This guide explains how a Cloud Provider Admin selects the OSAC service tiers
that are deployed, verifies the resulting configuration, and diagnoses requests
to disabled services.

The examples describe the current OSAC implementation. The umbrella chart uses
the values path global.services.<service>.enabled. The shorter
services.*.enabled wording used in the feature requirements refers to these
values under the chart's global key.

## Service tiers

OSAC has four service-tier switches:

| Service | Value | Default | Scope |
| --- | --- | --- | --- |
| CaaS | global.services.caas.enabled | true | Cluster orders, templates, catalog items, and versions |
| VMaaS | global.services.vmaas.enabled | true | Compute instances, disk images, instance types, volumes, and console sessions |
| BMaaS | global.services.bmaas.enabled | true | Bare-metal instances and bare-metal instance types |
| MaaS | global.services.maas.enabled | true | MaaS enablement signal; no MaaS gRPC services or operator controller currently exist |

The default values keep all service tiers enabled. A service is disabled by
setting its enabled value to false in the umbrella chart values:

    global:
      services:
        caas:
          enabled: true
        vmaas:
          enabled: true
        bmaas:
          enabled: false
        maas:
          enabled: false

The service values are consumed by the fulfillment-service chart, the
osac-operator chart, and the umbrella chart's bare-metal operator dependency.

## Service-to-component mapping

### Fulfillment-service

The fulfillment-service gRPC server and REST gateway receive the same boolean
command-line flags:

| Service | Flag |
| --- | --- |
| CaaS | --enable-caas |
| VMaaS | --enable-vmaas |
| BMaaS | --enable-bmaas |
| MaaS | --enable-maas |

The chart adds a flag to each container command when the corresponding global
service value is enabled. If the binary is started without any enable flags,
the fulfillment-service enables all four services for backward compatibility.

The REST gateway currently registers its generated handlers for all services.
Calls for a disabled service are rejected by the fulfillment-service gRPC
server and are returned by the gateway as HTTP 503. See Disabled-service
behavior for the verification procedure.

### osac-operator

The operator maps service tiers to controller environment variables:

| Service | Controller | Environment variable |
| --- | --- | --- |
| CaaS | ClusterOrder | OSAC_ENABLE_CLUSTER_CONTROLLER |
| VMaaS | ComputeInstance | OSAC_ENABLE_COMPUTE_INSTANCE_CONTROLLER |
| BMaaS | BareMetalInstance | OSAC_ENABLE_BAREMETAL_INSTANCE_CONTROLLER |

The chart's controllerEnabled helper uses an explicit
operator.controllers.* value when one is set. Otherwise it falls back to the
matching global.services.*.enabled value. Tenant, storage, volume, and
networking controllers are separate shared-infrastructure settings.

MaaS currently has no operator controller. Its value is still propagated to
the fulfillment-service flag set and Capabilities response so clients can
discover the configured tier.

### Bare-metal fulfillment operator

The umbrella chart uses global.services.bmaas.enabled as the dependency
condition for the bare-metal fulfillment operator. When BMaaS is false, the
BMF operator deployment is not installed. The separate BMF CRD dependency
remains installed so disabling the operator does not remove its CRDs.

## Selecting services

Use a values file containing the global.services settings, then pass it to the
umbrella chart. The dependency rules in the chart must be satisfied before
Helm can install or upgrade the release.

The next sections show the complete installation, upgrade, verification, and
troubleshooting procedures.

## Dependency validation

The umbrella chart validates the service combination before it creates or
updates workloads:

- CaaS can be enabled only when VMaaS or BMaaS is also enabled.
- MaaS can be enabled only when CaaS is enabled.

These rules are expressed in the chart's values.schema.json. Helm applies
them to both helm install and helm upgrade. An invalid combination fails
schema validation before the release is deployed or updated.

For example, this configuration is invalid because CaaS has no compute
backing service:

    helm template osac charts/osac \
      --set global.services.caas.enabled=true \
      --set global.services.vmaas.enabled=false \
      --set global.services.bmaas.enabled=false

This configuration is invalid because MaaS requires CaaS:

    helm template osac charts/osac \
      --set global.services.maas.enabled=true \
      --set global.services.caas.enabled=false

The fulfillment-service validates the same service dependencies when it starts
outside Helm. Its startup sequence first enables all four services when no
enable flags are set, then validates the resulting flags before creating the
server. The current error messages are:

- invalid service flags: CaaS requires at least one of VMaaS or BMaaS
- invalid service flags: MaaS requires CaaS

The current osac-operator startup validation checks the CaaS dependency on at
least one compute controller. MaaS has no operator controller, so the MaaS
dependency is enforced by the Helm schema and fulfillment-service validation.

## Install with selected services

1. Copy the default values and set the service choices. This example keeps
   CaaS and VMaaS enabled and disables BMaaS and MaaS:

       global:
         services:
           caas:
             enabled: true
           vmaas:
             enabled: true
           bmaas:
             enabled: false
           maas:
             enabled: false

2. Install the umbrella chart:

       helm install osac charts/osac \
         --namespace <namespace> \
         --create-namespace \
         --values values.yaml

3. Confirm the rendered result before applying it in a change-controlled
   environment:

       helm template osac charts/osac \
         --namespace <namespace> \
         --values values.yaml

The rendered fulfillment-service containers contain only the enable flags for
the selected tiers. The operator receives false for the BMaaS controller, and
the BMF operator deployment is omitted. Shared infrastructure controllers and
the BMF CRD dependency remain available.

## Enable a service after installation

To enable an additional service, update the matching global value and run
helm upgrade. For example, to enable BMaaS:

    global:
      services:
        bmaas:
          enabled: true

Then run:

    helm upgrade osac charts/osac \
      --namespace <namespace> \
      --values values.yaml

The upgrade rolls the affected workloads. The fulfillment-service starts with
the BMaaS flag, the operator enables its bare-metal controller unless an
explicit controller override says otherwise, and Helm creates the BMF operator
deployment because its dependency condition is now true. Verify the effective
state through the Capabilities endpoint after the rollout completes.

This guide covers initial selective enablement and enabling an additional
service. The lifecycle of resources that already exist when a service is
disabled is not defined by this feature.
