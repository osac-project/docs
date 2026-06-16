# Architecture

The Open Sovereign AI Cloud (OSAC) solution gives cloud providers a complete
platform to offer self-service provisioning of a range of services and
infrastructure (e.g., VMs, Bare Metal, OpenShift clusters, OpenShift AI, Model
as a Service...) , integrated with the cloud provider’s existing infrastructure
components. It features a fulfillment workflow that is powerful enough to handle
complex cluster deployment while also being flexible enough to accommodate
future expansion into other types of deployments.

## Templates

Self-service provisioning in OSAC is built around a concept of Templates.
Whether an end user wants to provision a VM, a cluster, or something else,
they'll be presented with a selection of templates from which to choose. Upon
selecting a template, they'll provide the required input, and then the system
will proceed with a workflow to allocate the resources (e.g., computers, VMs,
networks), connect those resources together, and install the required software
on them.

The Cloud Service Provider (CSP) can define their own templates, including:

* What input each template requires
* What infrastructure it provisions
* How it provisions that infrastructure

A Template is an Ansible
[Role](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html).
Metadata such as [argument
validation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-argument-validation)
describes how to use a template, while the contents of the Role itself codify
the implementation of provisioning.

Provisioning any kind of compute infrastructure can involve many different
systems, such as:

* DNS
* Network fabric
* Hardware inventory
* Hardware management
* Virtualization platform
* Application deployment

Ansible's huge ecosystem of content can already interact with and automate the
vast majority of systems that would be used by a cloud provider. By utilizing
Ansible for defining the template and the details of how infrastructure gets
provisioned, each cloud provider gets the opportunity to customize templates to
use their chosen infrastructure systems.

Additionally, each cloud provider can customize or create Templates that
provision infrastructure according to the specifications of their cloud
offering. For example, OpenShift clusters can have software pre-installed and
pre-configured based on what a particular cloud provider wants to offer.

## Management Clusters

A Management Cluster is an OpenShift cluster that includes a specific set of
management tooling and that has access to a pool of provisionable compute
resources.

Each Management Cluster includes all of the resources and tooling that is
necessary to provision and sustain the infrastructure requested by end users. A
single Management Cluster may be treated as a failure domain or availability
zone.

Management Clusters include the following software:

* [Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management): provision and manage clusters, especially using Hosted Control Planes.
* [OpenShift Virtualization](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization): provision and manage VMs.
* [Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible): executes provisioning and management workflows via job templates.

## Fulfillment

The Fulfillment Service provides a single API that enables a CSP to access the
wide range of capabilities needed to provision infrastructure on-demand. It does
so by providing a gRPC- and/or REST-based API that allows end-users to create
fulfillment requests. Upon receiving a request, it then conveys that request to
a Management Cluster where that request can be fulfilled by a set of k8s
controllers.

The design of the Fulfillment Service can be understood through three major
components: the core Fulfillment Service; the O-SAC Controller; and Ansible
Automation Platform (AAP). The O-SAC Controller and AAP both run on a
Management Cluster.

### Fulfillment Service

The Fulfillment Service receives and tracks requests for cloud resources. Each
request is scheduled onto a Management Cluster. The Fulfillment Service includes
an API that can be used through REST or gRPC.

The Fulfillment Service exists as an API layer for several reasons:

* Privilege Separation is achieved because the service that external entities interact with does not have the ability to directly affect infrastructure.
* Multi-tenancy that is suitable for cloud use cases can only be achieved by adding a layer of privilege abstraction on top of the Kubernetes APIs that implement OSAC's capabilities.
* While we could directly expose a k8s API with similar capabilities, the k8s API itself is generally understood to not be suitable for putting on the internet or in front of unknown and untrusted clients.
* Additionally, Namespace-based RBAC in k8s is not sufficient for the level of tenant separation required.
* Multiple Management Clusters may exist in a deployment as discussed above, and this layer enables topology awareness and the ability to schedule requests onto different Management Clusters as appropriate.

The `osac` CLI integrates with this API; service providers may also
integrate their own UIs with this API.

### O-SAC Controller

O-SAC Controller is a Kubernetes operator running on each Management Cluster
that watches for requests and then ensures they get fulfilled by launching
Ansible Automation Platform job templates and monitoring their progress.

While the controller has APIs that may appear to duplicate other existing APIs
in the OpenShift ecosystem, this controller offers a much narrower scope of
capabilities, resulting in an API that is safe to expose to untrusted users.
For example, when creating a cluster with this controller's API, the client can
only specify a limited number of parameters about the cluster and how it gets
deployed. If instead we offered direct access to existing cluster provisioning
APIs, the client would have access to far more control over how infrastructure
is configured and deployed, with the ability to deviate from what the CSP
intends to deploy.

Following the k8s controller pattern enables the automation to not only
provision resources, but to continuously ensure that those resource exist in the
desired state. The controllers watch the state of resources and then initiate
automation workflows as needed to maintain desired state.

### Ansible Automation Platform (AAP)

AAP executes the majority of provisioning steps by running the associated
Templates. The O-SAC Controller launches templates through the AAP REST API
and polls job status until provisioning completes or fails.

Ansible roles and playbooks are expected to be idempotent, which is a good match
for the declarative nature of the project's k8s controllers. The roles that
constitute Templates may be run multiple times and should always converge to the
same resulting state. That is a standard best practice for any Ansible
automation content, and in this project it enables the controllers to safely run
their automated workflows with Ansible as needed.

See [AAP Provisioning Architecture](aap-provisioning/) for details on job
lifecycle management and configuration options.

### Workflow

The general workflow for fulfillment is as follows:

* First, the cloud provider makes a request for resources through the API, on behalf of a tenant user, having received that request through their own user interface. Alternatively the cloud provider might expose the fulfillment API directly to end users.
* The Fulfillment Service selects a Management Cluster and then places the request there with the Kubernetes O-SAC operator using Kubernetes APIs.
* The O-SAC operator then performs automated setup and launches AAP job templates to provision infrastructure. It polls job status and reports progress back to the Fulfillment Service.
* AAP runs the deployment automation. The exact automation varies and can be customized by each cloud provider's needs.

The Fulfillment Service is intentionally modeled on the Kubernetes pattern:
providers submit a request that declares the desired state, and the O-SAC
operator reconciles the cluster to match that desired state. To put it simply, the
fulfillment API creates the request, and the O-SAC Operator reconciles the
order. This gives the service a flexible way to achieve reconciliation that
doesn’t limit us to a specific set of APIs or tools. Because this offers a wider
application, further configuration to fit the needs of service providers is done
through Ansible Automation Platform (AAP).

The flexibility of this architecture means that, broadly speaking, there are
only two things that must happen when implementing a fulfillment workflow.

1. Define an request object that has all the attributes a user needs to specify their desired outcome. For cluster fulfillment, that's simply a template and template parameters.
2. Create the playbooks that reconcile the request. For cluster fulfillment, those playbooks call the OpenShift provisioning APIs, as well as our initial implementation of bare metal/L2/L3 workflows.

Details of specific fulfillment types are specified in the following sections.

* [Cluster Fulfillment](cluster-fulfillment.md)
* [Bare Metal Server Fulfillment](bm-server-fulfillment.md)
* [VM Fulfillment](vm-fulfillment.md)
* [PublicIP Networking](publicip-networking.md)

### Integration Patterns

In many cases, a CSP will use its own APIs and services to interact with
end-users, and then the CSP's systems will call the Fulfillment Service in
response to end-user actions. In those scenarios, the end-user never interacts
directly with the Fulfillment Service.

Optionally, a CSP can expose the Fulfillment Service directly to its end-users.

Either way, the Fulfillment Service has multitenancy built into its data and
operational models, and it depends on specific information in each API request
to identify the user and tenant.

### User Interfaces

OSAC includes [a command-line
utility](https://github.com/osac-project/fulfillment-service) that can be used to
interact with the API.

OSAC may also include a web UI that can be used for demos or proofs of concept.
At some point it might make sense to include a production-ready web UI that could
be used directly by CSPs, but that would require additional scope of use cases
that go beyond the initial focus of OSAC.

## Scalability

The solution can scale horizontally in multiple ways:

The Fulfillment API service can run as many instances as necessary behind a load
balancer.

Each Management Cluster can scale as needed to run the management tooling that
constitutes the cloud's control plane.

Each Management Cluster can scale to hundreds of nodes that can run hosted
control planes. Based on typical performance and load characteristics for an
OpenShift control plane, each additional node in the cluster should increase the
cluster's capacity by at least several hosted control planes, if not ten or
more.  See OpenShift's [sizing
guidance](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/hosted_control_planes/preparing-to-deploy-hosted-control-planes#hcp-sizing-calculation_hcp-sizing-guidance)
for hosted control planes for more detail. Each individual hosted control plane
can be attached to hundreds of worker nodes, whether they are bare metal or
virtual.

Likeise, each Management Cluster can scale as needed to host VMs.

A deployment of the solution may include as many Management Clusters as needed
in order to achieve sufficiant scale.

## Lifecycle and Upgrade

The solution may be safely upgraded and maintained without disrupting the VMs
and clusters that have been deployed for end users. Infrastructure that has
already been provisioned for an end user will not be disrupted even if the cloud
solution itself suffers an outage.

When offering clusters with hosted control planes, or when offering VMaaS, it is
a best practice for each Management Cluster to have a separate dedicated cluster
in which to run those hosted control planes and/or VMs. Doing so enables the
Management Cluster itself to be safely upgraded and/or disrupted without
disrupting the end-user infrastructure.

Even the clusters that do host VMs and/or hosted control planes can be upgraded
safely. Following standard k8s best practices enables pods to be automatically
replaced on other nodes, and VMs to be live-migrated, as needed to upgrade each
node in the cluster.

## Key Infrastructure Solutions

### Networking

Isolated networks are an essential element in any multi-tenant cloud. See the
[Networking API enhancement proposal](https://github.com/osac-project/enhancement-proposals/tree/main/enhancements/networking)
for the full design, and [PublicIP Networking](publicip-networking.md) for the
public IP architecture.

### Inventory

In order to provision a variety of physical and virtual compute resources and then
make them available to a variety of tenants, the fulfillment workflow must have
a source of truth for what devices exist and how exactly they are connected.

OSAC expects to be able to utilize multiple different types of inventory source
of truth. Work is underway to determine what inventory source of truth will be
the primary focus for OSAC's reference implementation.
