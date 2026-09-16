# Open Sovereign AI Cloud

## Structure

[Features](features/): Description of key features and capabilities, some of
which may not yet be implemented.

[Architecture](architecture/): Description of components, how they work
together, and the design decisions that have been made.

[Guides](guides/): Step-by-step configuration and usage guides for
administrators and developers.

## Introduction

There is a worldwide trend towards local and specialized clouds, where countries
and service providers want to offer their own cloud services under local
jurisdiction and specific compliance regimes. Use cases include traditional
VMaaS clouds, neoclouds, and sovereign clouds.

Open Sovereign AI Cloud (OSAC) is an open-source project for organizations
standing up their own clouds. It offers multi-tenant self-service provisioning
of VMs, OpenShift clusters, bare-metal servers, Model-aaS, and more. OSAC
offers standard cloud features including tenancy, RBAC, quota, metering, and
tenant isolation at every layer.

OSAC interfaces:
* **gRPC API**: scalable, secure, and safe to put in front of unrelated tenants. It is standards-based and designed for automation.
* **CLI**: an out-of-the-box CLI for admins and tenants to accomplish their work with OSAC.
* **UI**: a brandable web interface for service providers who prefer an out-of-the-box UI vs building their own.

## Core Services

**Bare Metal-aaS** allows tenants to allocate groups of computers, place those
computers on isolated networks, and manage/configure those computers themselves.
BMaaS is needed by tenants who want to install their own workload management
software (e.g., SLURM), and tenants who want OpenShift clusters with bare
metal nodes.

**VMaaS** allows tenants to create virtual machines using primitives that are
familiar to users of public clouds. VMaaS utilizes [Kubevirt](https://kubevirt.io/)
as the backend VM platform.

**Cluster-aaS** creates OpenShift clusters on demand. By default it uses [Hosted
Control
Planes](https://www.redhat.com/en/topics/containers/what-are-hosted-control-planes)
to achieve the best compute density, provision quickly, and give the service
provider exclusive access to manage critical parts of the control plane.
Cluster-aaS utilizes BMaaS and VMaaS to provision nodes.

**Model-aaS** (MaaS) delivers token-based access to cloud-local inference
endpoints running a curated selection of models. MaaS builds on [OpenShift AI's
MaaS](https://www.redhat.com/en/products/ai/openshift-ai), which is implemented
with [vLLM](https://vllm.ai/).

In addition to the above, OSAC includes a number of supporting services such as
standard cloud storage features and isolated networking via a full Virtual
Private Cloud (VPC) implementation.

## Customization

Each Cloud Service Provider (CSP) makes their own choices about the supporting
infrastructure on which their cloud runs. Those choices include server hardware,
network gear and fabric, GPU selection, hardware inventory, storage solution,
DNS platform, secret store, etc. OSAC needs to interface with each of those
while provisioning and managing cloud services.

Furthermore, CSPs have good reason to customize the details of how provisionable
assets, such as VMs and Clusters, get implemented. For example a CSP may need to
influence the way kubevirt APIs are utilized in order to include
hardware-specific optimizations or other features. Or they may need to customize
the way OpenShift clusters are created in order to turn on or off certain
features.

OSAC comes out of the box with working default implementations, while enabling
the CSP to customize or even replace portions of OSAC's workflows. OSAC does so
by utilizing Ansible roles to implement those portions of workflows that CSPs
may need to customize.

[Ansible Automation
Platform](https://www.redhat.com/en/technologies/management/ansible) (AAP) comes
with an extensive [ecosystem of
Collections](https://docs.ansible.com/projects/ansible/latest/collections/index.html)
that can interface with most of the infrastructure that would be found in a
datacenter. That ecosystem, combined with AAP's job management capabilities,
make AAP an ideal execution engine for OSAC.

## Development

OSAC is being developed with and continuously deployed at the [Mass Open
Cloud](https://massopen.cloud/) (MOC) to take advantage of the MOC’s scale, to
provide industry and academic partners a public environment where they can
integrate their hardware and services into the solution, and to ensure the
solution can address the needs of a production environment with a large
community of AI users. OSAC began with contributions from the MOC, Red Hat
Ecosystem Engineering, Red Hat Research, and IBM Research, and hopes to attract
a broader community of developers and early adopters that will help prioritize
and help develop features.

## Terms and Definitions

ACM: Advanced Cluster Management (ACM) is a Red Hat product that simplifies the
provisioning and management of multiple Kubernetes (OpenShift) clusters.

Cloud Service Provider: CSPs are organizations that offer compute resources for
rent to multiple unrelated tenants / customers.

HCP: (HyperShift) Hosted Control Plane refers to an architecture where the
control plane of an OpenShift cluster is decoupled from the worker nodes and run
as Pods on a separate hosting cluster.

MOC: The MassOpen Cloud (MOC) is a public computing cloud where the Open
Sovereign AI Cloud is being deployed.

Tenant: A user or group of people with the ability to self-service provision
clusters; acts as a cluster administrator for their own cluster(s). An end user
of the OSAC solution.

See [personas.md](personas.md) for a description of OSAC personas.

## Contribution Guide

The Open Sovereign AI Cloud [organization on
GitHub](https://github.com/osac-project) is a public location to plan and develop
this solution.  Major features, architectural changes, and significant initiatives are proposed and discussed in the [enhancement-proposals](https://github.com/osac-project/enhancement-proposals) repository.  GitHub [issues](https://github.com/osac-project/issues/issues) are also used to track community-reported bugs, feature requests, and other project discussions.

To contribute to the project, first find or open an issue on
https://github.com/osac-project/issues. Prior to beginning work, it is wise to get
feedback from project stakeholders on the change and how it will be implemented.
Then fork the repository you’re interested in contributing to, clone it to your
local machine, and create a descriptive feature branch name. Once you’ve made
your changes and ensured that it passes any lint checks, push and open a pull
request against main on the upstream repo, linking the issue you opened and
summarizing what you changed. Then your pull request will eventually be
reviewed, and you can respond to comments, add follow-up commits, and re-run
tests until all required checks are green. If the team accepts the pull request,
your contribution will be merged and you will have successfully contributed to
the OSAC project.