***Note: This is a working document and intended to evolve with input.***

# AI-in-a-Box Design Document

### High-Level Component Summary
![OSAC overview diagram](images/overview-diagram.jpg)
The above diagram shows the components of our architecture. Two core sets of services at the bottom are: 1)  **Bare Metal Fulfillment (BM-F)**, which keeps track of hardware and is responsible for managing physical servers and network switches, and 2) **Cluster Fulfillment (C-F)**, which provisions and manages the OpenShift clusters using computers and networks from BM-F.  Both the BM-F and C-F have agents that expose the new APIs required for their services. In a large-scale environment there may be many instances of C-F and BM-F.  

At the top, the **Web Interface** is an (example) UI for our service built on top of an API provided by the **Fulfillment Service**, a new component that directs requests to the appropriate BM-F or C-F agents. 
- **Bare Metal Fulfillment (BM-F)** components are: 1) **Bare Metal Fulfillment Service**: A service that orchestrates bare metal clusters using the Bare Metal/Layer 2/Layer 3 Services. 2) **Bare Metal Service**: A service that permits operations on servers, such as power control, boot order configuration, etc. 3) **Layer 2 Service**: A service that interacts with network switches to create isolated layer 2 networks. 4) **Layer 3 Service**: A service that manages IP address allocation, DHCP services, and routing (including default gateways). 
- **Cluster Fulfillment (C-F)** is based on Red Hat’s Advanced Cluster Management (ACM) with the associated Hosted Control Plane (HCP) services.  New functionality includes: 1) **Cluster Fulfillment Service**: Responsible for executing cluster requests on a specific ACM. Configures hardware resources via bare metal fulfillment before calling HCP; 2) **Config Management**: Centralized mechanism for further configuring an installed User Cluster; and 3) **Hotpool Service**: Responsible for maintaining a minimum number of free bare metal resources available for cluster installation by calling **Bare Metal Fulfillment**. Doing so allows for rapid responses to cluster requests. 

Other services required for our solution include: 
- **Observability**: A single point of access for metrics, logs, and traces for all resources managed by this solution.
- **Identity Provider**: The identity provider provides information about people and projects and is used to support authorization for access to clusters.
- **Quota Provider**: The quota provider populates systems and services with quota information, and also provides a mechanism to query quotas for a project.
- **DNS**: DNS records are required for new OpenShift clusters.
- **Storage**: In order to provide a fully configured cluster for AI workloads, our solution must be able to acquire and attach storage for newly created clusters.
- **Inventory**: The source of truth regarding what available hardware resources exist, where they exist (e.g., which rack, which row, which power domain), and the network interface attachments by port for hardware resources.

In the MOC today, we use ESI for Bare Metal and Network Management, [Keycloak](https://www.keycloak.org/) for the Identity Provider,  [Coldfront](https://coldfront.readthedocs.io/en/latest/) for the Quota Provider, and [Amazon Route53](https://aws.amazon.com/route53/) for DNS.  We have also started using Netbox for inventory management.

## Design Decisions
This section discusses the “what” and “why” regarding the key design decisions we have made, and points out some of the open questions. More detailed implementation discussions will be available in later implementation documents.

### Bare Metal Fulfillment
#### Minimal Bare Metal use case
Although our primary use case is the AI cluster service, it is also vital to support the bare metal use case. We have made the design decision to minimize what we are exposing for this use case, and require sophisticated tenants to provide all other functionality (DHCP, elastic IP address, etc) outside of our solution.

The key requirements for the bare metal use case are:  1) allocation of compute nodes, 2) creation of networks, 3) import of networks for external services, 4) placement of compute nodes on networks, 5) power control of compute nodes, and 6) serial console access. All other requirements can be supported outside of our service.

The cluster fulfillment use case requires all but the last requirement.  Requirements 1, 2, 4 and 5 are needed by any provider offering strongly isolated AI clusters.  Requirement 3 is needed in scenarios where a provider wants to enable tenants to connect their clusters to other tenant specific resources.  For example, an Equinix tenant may want to stand up a cluster that uses tenant storage in the same data center or exploit networking in the data center to stitch together different open shift clusters.   In the MOC, different institutions will need to connect clusters they stand up to institution specific resources and identify clusters as “on-premis” to institution users.  The only feature exclusive to the bare metal use case is serial console access.

#### Introduction of Bare Metal Service
The Bare Metal Service is a new component that is intended to support a minimal API for controlling servers (those being the operations required for our solution). All the required functionality is already supported by the [ESI](https://esi.readthedocs.io/en/latest/) service used at the MOC, and the Bare Metal Service will be a simple shim on top of ESI. This choice isolates our solution from ESI, enabling us to replace ESI in the future with alternative implementations.  

ESI is primarily just a particular configuration of OpenStack services, and there are a number of reasons why we may eventually replace it:
- ESI is hard to deploy and configure. Red Hat’s OpenStack installers as of RHOS 17.1 have been difficult to configure and maintain, and ESI has added additional capabilities that complicate the deployment.
- Network management is provided by Neutron, leading to challenges  when managing bare metal network interfaces (since we need to make API calls to two separate services and reconcile the information retrieved from both).
- Due to a variety of design decisions, OpenStack services are noticeably slow for this use, leading to a frustrating user experience.

The full range of features we wish to expose through the Bare Metal Service remains an open question. As discussed below, the cluster service requires ESI-supported features such as DHCP, routing, and IP allocation. However, it is uncertain whether these features are necessary for the BM-F use case, and if we can reduce the requirements for the underlying Bare Metal Management and Network Management, then we will do so.

#### Introduction and use of an Inventory Service
Both ESI and ACM include a limited node inventory that is sufficient for their core functionality; however we expect this solution to include a large-scale inventory service. This inventory information is critical to enable cluster resource allocation that meets locality and availability requirements. Note that large-scale service providers will already have their own inventory systems, so it is not necessary for us to develop a new inventory service for this solution as long as we provide a suitable programmable interface to existing inventory services. The MOC currently uses Netbox for inventory (still under construction).

### Cluster Fulfillment
#### Introduction of Cluster Service
We expect the majority of customers to replace the Web UI developed for demonstration as part of this solution. We also expect many tenants (as well as providers) to interact with the solution programmatically through APIs and CLI tools. The Cluster Service component will provide the APIs necessary for the user to interact with Cluster Fulfillment. This agent will interact with both the BM-F services to manage the networks used for clusters and allocate nodes, and with ACM services to provision clusters.

The Cluster Service enables us to work around the current state of ACM - namely, that ACM is not a multi-tenant tool, and we do not want to expose the full Kubernetes API of the management cluster to tenants.

#### Introduction of a HotPool Service
The process of adding new nodes into ACM is time consuming; as a result, initial PoCs will have the required Bare Metal Resource pre-added into ACM. In the long term, we would like to develop a HotPool Service that interacts directly with the Inventory Service in order to automatically maintain a modest number of free nodes.

#### Use of ACM Agents for Bare Metal Resources
In ACM, there are two abstractions for representing bare metal resources: “BareMetalHosts” and “Agents”.
- Agents: In the Agent model, registered hosts are booted with a discovery image that includes the agent. ACM then communicates with this software agent (rather than managing the host directly).
- BareMetalHost: In the BareMetalHost model, ACM directly manages bare metal hosts via their BMC through Metal3, which is a thinly disguised instance of Ironic that allows ACM to perform bare metal operations such as power control and boot order configuration. This model allows us to automate the process of host discovery. However it also requires additional infrastructure support in the form of BMC proxy, since we do not want ACM to have direct knowledge of a node’s BMC credentials in the multitenant environment. 

The MOC can automate the creation of both Agents and BareMetalHosts from our bare metal inventory by using the ESI API to boot the bare metal host using the appropriate discovery image. 

Since the Bare Metal Service needs to talk to the BMCs, if we want ACM to interact with the BMCs we will need to modify it to invoke the Bare Metal Service or develop a BMC proxy so both talk directly to the BMC. We believe that the right long term design is to have ACM use the Bare Metal Service, and we will use the agent model in our PoC to let us make progress without potentially unnecessary work to develop a proxy and without requiring product changes.

#### Config Management
We are still exploring the right mechanism to configure OpenShift clusters. In the MOC environment today we use ArgoCD (in the form of Red Hat’s [OpenShift GitOps product](https://www.redhat.com/en/technologies/cloud-computing/openshift/gitops)).  However, this is insufficient since we need the ability to parametrize the configuration for each individual cluster.  
- Mechanism for further configuring an installed User Cluster
  - installing operators, generating certificates, etc
- Possible solutions:
  - [Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible) (AAP)
  - ACM [Policies](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.12/html-single/governance/index)

### Networking requirements for clusters and bare metal
In order to provide the strong isolation guarantees required by this solution, we need to be able to deploy OpenShift clusters on dedicated L2 networks. Additionally, in order for OpenShift to function, we need to provide a variety of other network infrastructure services:
- Provide address assignment via DHCP to bare metal nodes, and provide a mechanism to control the selection of address ranges to avoid conflicts with other internal networks (and to ensure a proper address range for the planned size of the cluster)
- Allocate floating/elastic IP addresses and assign them to internal addresses (or forward specific ports) in order to expose the API and Ingress endpoints.
- Provide routing so that machines on isolated networks have access to external addresses (for software updates, access to remove services/apis. etc)

These capabilities are supported by ESI, and in the initial PoC we will support these capabilities by exposing these ESI features through the Bare Metal Service (and having the Cluster Service invoke them when needed).  Over time the capabilities may migrate to other components of the solution.   We will reconsider this as we progress through the PoCs.

### User experience
For the PoCs, we will build any missing user interfaces needed to demonstrate the solution functionality if it is not already provided in an existing Red Hat product.  The goal is to provide the user with “one stop shopping” for requesting, modifying or deleting bare metal or cluster services (including RHOAI platform services) for AI use cases. As an example, there is no Red Hat product end user GUI available that can specify the type of cluster an end user desires in a multi-tenant environment.   We will leverage observability tools that have been implemented in the MOC as the fastest path to demonstrate the solution functions.  We will need additional support on the team to translate these demos to products, as the team does not yet include user experience designers.

### Observability
#### Design Decision 1: Multi-Tenant Observability with Fine-Grained Access Control
Context:

The AI-in-a-Box platform requires robust observability that supports multi-tenant usage for AI cluster application end users, tenants, and service providers.  Because tenants and end users may belong to different organizations, the observability solution must also enforce fine-grained access control to keep each tenant’s observable cluster information accessible only to the service provider and the tenant owners and end users associated with each individual cluster.   This requires three types of observability functions.
- Cluster-level observability (service provider visibility across clusters)
- Single-cluster observability (restricted visibility for a specific cluster owner/administrators)
- Project-level end user observability (observability for project information within a specific namespace of a specific cluster)

Observability targets multiple personas.   Use cases and personas will be specified in a separate document.  While ACM currently provides cluster-wide metrics, it does not offer fine-grained access control for metrics from different clusters or different projects (namespaces) within clusters.  Fine-grained access control is currently implemented in the MOC using AI Telemetry and the Prometheus Keycloak Proxy.

Decision:

Implementation options for providing these functions are under discussion in the Observability Working group.   

Rationale:
- The AI-in-a-box use cases require access to observability metrics be provided with different scopes (full cluster, single cluster, project-level).
- Authentication and authorization are needed to grant access to observability metrics with different scope.  For example, the Prometheus Keycloak Proxy provides fine-grained authentication and authorization via Keycloak based on user identification and project membership(s).
- Users need to gain access to appropriate dashboards based on their roles.   For example AI tContext: Telemetry in the MOC provides access to dashboards based on Keycloak group permissions.
- Ensuring this integration enables secure, role-based observability access at the desired granularity for the offered tenant services.

#### Design Decision 2: Approximate resource costs information for tenants and end users managing their cluster and project resources
We know from our experience with MOC that there is a need for project owners to understand roughly how their choice of resources will add up when selecting services.   For example, different GPUs have different hourly rates in the MOC.  The project owner will want to know something about the rates when specifying clusters or bare metal machines in order to manage their budget.  This is absolutely not the same as billing, but the information allows the potential tenant to make appropriate choices when requesting clusters (e.g. request that fast but expensive GPU or use a slower one?).

Cluster admins and end users can make more informed choices about resource usage when they have at least rough information about resource costs.  Some observability use cases require approximate cost information about resources, and access to that information should be handled with the same type of fine-grained access control described previously for other cluster metrics.

While our implementation will focus on the MOC charging model, we hope that the reference implementation will enable providers to adapt the solution to their own business model and hopefully contribute changes back.   This feature should be configurable so providers, e.g. enterprises, can totally disable it. 

Decision:

Introduce a metrics ServiceMonitor and microservice with these capabilities:
- Query CPU, GPU, memory, and node utilization per node, cluster, project, and end user.
- Compute approximate resource cost estimates, and display the percentage of the project quota that has been used to date.
- Expose approximate cost metrics in Observability dashboards.

Rationale:
- Help tenants and end users understand day-to-day resource costs
- Enable better cluster and project-level resource management.
- Encourage cost optimization for end users managing AI workloads

#### Design Decision 3: Dashboards for Different Personas in OpenShift and RHOAI
Context:

The observability platform for OpenShift clusters serves different personas with distinct roles and information needs:
- Overall Admins: Require visibility into all clusters, projects, and system-wide health.
- Cluster Admins: Need insights into a specific cluster’s performance, logs, and resource utilization.
- Project Leaders: Require project-specific observability, tracking usage and costs for their namespaces.
- Users/Project Members: Need access to relevant project-level metrics but not full cluster details.

A single-pane-of-glass approach is unrealistic due to the scope of data and access restrictions. However, a centralized starting UI that presents tailored dashboards per persona is feasible.  The personas who will be able to request or change a cluster should also be able to choose whether to include full observability or a minimal level of observability in their requested cluster.  (The minimal level would be that needed by the resource provider to manage and track usage of the cluster.)

Decision:

The solution will use one or more dashboards per persona/role, ensuring that users only see dashboards relevant to them. If an easy mechanism exists to hide non-relevant dashboards (e.g., based on Keycloak roles), it will be implemented in this iteration; otherwise, it will be considered for a future iteration.

Options Considered:
- Drill-Down Experience (Start at cluster level, navigate to project level)
- Dashboards Per Persona (Each role gets specific dashboards)
- Hiding Non-Relevant Dashboards (Dynamic UI based on role)

Final Choice:
+ Dashboards per persona/user role – ensures clarity and ease of use.
+ If hiding non-relevant dashboards can be easily implemented (e.g., via Keycloak roles in AI Telemetry/Grafana), it will be added in this iteration. Otherwise, role-based dashboard filtering will be considered in a future iteration.

See User Interface section for additional information.

### Storage support
There are three types of storage that we need to think through, namely: 1) Volume storage that will be used by services in the cluster during the cluster lifetime, and 2) Long term storage that can be accessed by multiple clusters, and 3) Storage cache for long term storage.  In our PoCs we will provide minimal implementations for key use cases discussed below with storage available in the MOC, and over time define appropriate interfaces to enable different storage services.  

**Volume storage**: For the first, when provisioning a new cluster, we need a way to allocate storage that can be accessed read/write from multiple containers in the OpenShift cluster.   This means, 
- Create credentials for the new cluster
- Create a storage pool for the new cluster
- Configure a CSI driver on the new cluster with credentials and connection information.

This means that the solution needs the appropriate access to programmatically manage credentials and allocate storage; normally an administrative control.  For the PoCs we will have such access to a spectrum scale system of modest size (1PB) and can use that as well as a non-production Ceph cluster, and potentially a pure storage solution.   While not in scope for the general solution, at the MOC we will need to explore how to add the right mechanisms on top of the NESE production service.

**Long term storage**:  We assume that the storage will be S3 object storage, in the case of the MOC deployed using Ceph RGW.   This can be outside of the solution, for example, tenants may want to use AWS S3, in which case the solution will not interact with it.  Having said that, we expect many cloud provider customers will deploy their own S3 service, and the solution should enable tenants to create new access credentials and assign granular permissions to their storage (e.g., creating read-only credentials, or creating credentials that only have access to specific buckets).  Again, for the MOC we will need to add the right mechanisms to the RGW currently deployed by NERC or add a new RGW that enables this.

**Storage caching**: We assume that most data is in long term storage outside of the cluster, but we want to be able to access it efficiently, for example, enabling GPU direct access using RDMA when data is hot.  This means that the data needs to be visible through a file system interface, since that is what NVIDIA supports.  Also, all the data needs to be cached on storage that is directly attached to the same kind of switches that the computer is using, since RoCE does not work over switches from different vendors.  We plan to use an IBM Spectrum Scale system in the PoC that supports our requirements.  In the long term, it would be nice to have an open source solution especially for less demanding workloads.

### Ethernet support
We have made the design decision to support only Ethernet in our solution.  We require the ability to configure strong network multi-tenancy, and it is not clear how well InfiniBand supports that today.  To support high performance we will be evaluating RoCE during the PoCs to enable direct GPU-to-GPU and GPU-to-storage DMA.

### Meeting Compliance requirements
The MOC will, for the AI Hub, need to meet various compliance requirements including HIPAA and GDPR.  In fact, one major advantage of OpenShift on demand clusters is that different tenants, with different compliance needs, can be strongly isolated from each other.  An important goal of the solution is to automate to the extent possible what we need to do for each regime.  This will likely include automating logging and retention.  As another example, for HIPAA we will need to ensure that physical computers used are allocated from racks where access is available only to staff with the appropriate training, and our solution will need to track that in the inventory system.  Additionally, we will need to ensure that all services deployed in the cluster encrypt data stored in shared environments.

### Multiple cluster & bare metal services
For both scalability and fault tolerance, we will support an architecture where a single Fulfillment Service API endpoint can interact with several Cluster Services and Bare Metal Services, where the scale of each is independent.  For example, in a data center with 100K nodes, a bare metal service may be appropriate to handle 10K servers, and a cluster service might only handle 5K nodes, so we would deploy in the data center 10 bare metal and 20 cluster services.  Cluster services will be able to create clusters on any of the bare metal services as well as clusters that span multiple bare metal services for fault tolerance.

Most of the state will be in the bare metal and cluster services that are dependent on specific implementations today (ACM & ESI).  We have not yet investigated the failure model of each of these components.  A key part of this project that we have deferred for now is defining the fault characteristics of the overall solution, and how the overall solution will orchestrate overall recovery in the presence of component failure. 
![scalability diagram](images/scalability.jpg)

### In PoCs focus purely on Physical Servers
Despite extensive conversations, we will for the PoCs covered in this document focus only on physical hosts.   Longer term, we will want to integrate virtual hosts directly in two use cases:
1. Have openshift clusters where the hosts are virtual rather than physical.  This should be natural with ACM, and will enable greater elasticity and supporting clusters where we don’t have to allocate all the GPUs in a physical host to a tenant.  There may also be security reasons we want to do this; e.g., for containers where the tenant wants root access. 
2. A simple interface where tenants can request directly from our solution a VM or container, with the GPUs they want, without creating an openshift cluster. This should support both terminal access and jupyter notebooks.  This functionality would be natural on top of what we are doing, and would be a simple way for users to get started. 

# Proof of Concepts
This section describes a possible series of proof-of-concepts that aims to show a steady progression of key deliverables that leads towards the final design. A feature mentioned in a particular PoC is simply the first version of that feature; we expect to continue iteration through successive development efforts. These PoCs are built towards the specific use cases outlined in the [AI-in-a-Box Use Cases](https://docs.google.com/document/d/1tMDsnHSWavdwogUTIvfvSi2W4R8HekHxVT7r9blyqdU/edit?usp=sharing) document, ensuring that they align with the real world user needs and requirements.

In order to build up the elements of the AI-in-a-Box solution in stages we defined the seven PoC Demos listed in this section.   We expect to use the PoC Demos to share progress and get feedback from both the MOC Operations Engineering teams and the Red Hat product teams.  As we get feedback, and discover more through implementing functions, we expect to improve each PoC to better serve the Red Hat, MOC and future customer needs.  Functions can be added, deleted or changed in a future PoC depending on the results from the previous demo.  We will schedule PoC demos on a regular cadence (e.g. monthly) to make sure we get feedback regularly.  

Even though the PoC demos may evolve over time, it is important to have a clear common understanding of the functions to be implemented for each PoC stage before that stage begins.   This description is the basis for plans and estimates for tasks at each stage.  This document includes a draft description of the purpose and expected functions for each proposed PoC, as a baseline for review.  We expect to complete more detailed PoC demo descriptions as the project continues.

## PoC 1: Initial Proof-of-Concept
This initial PoC will demonstrate a first example of the expected end-user functionality for tenants deploying OpenShift AI clusters on top of a greatly simplified service that merges many of the fulfillment components described above. We will make many design compromises for the sake of a quick turnaround (full details can be found in a future Demo 1 Design Document).

This simple demo will create an initial graphical user interface, implement a trivial API and first version of fulfillment service, and instantiate new OpenShift clusters using nodes pre-allocated from ESI. These OpenShift clusters will be attached to the Observability cluster. 
- The Fulfillment Service, Cluster Service, and Bare Metal Service will be combined into a single Fulfillment Service. Later PoCs will separate these services.
- ACM will not automatically pull nodes from ESI. We will create a project to represent ACM and lease a number of nodes to this project, and then boot the nodes off an appropriate discovery ISO from the InfraEnv in order to register them with ACM.
- We will not demonstrate dynamic network isolation. We will use ESI to pre-configure node networking as needed for cluster operation.
- The operations we will demonstrate are limited to cluster creation and teardown.
- In the user cluster, all per-cluster storage will be provided by the local hard drives on their systems in the cluster.  (If this storage is insufficient, standalone storage service may be needed.

This will be reviewed with the MOC operations team to collect feedback and requirements to be integrated into future PoCs. 

## PoC 2: Fulfillment Service Re-Architecture & Observability
This PoC will focus on re-factoring the Fulfillment Service to separate out the Fulfillment Service, Cluster Service, and Bare Metal Service.

In parallel, we will identify the personas that are primary targets for AI-in-a-Box.  Because there are existing observability GUIs in multiple Red Hat products, we will identify which GUIs will be used by which personas.   For this PoC stage, we must support fine-grained access control and multitenancy, which has so far only been implemented in the MOC Observability tools.   Therefore, we will use links to the existing observability features in the appropriate new user GUI or existing product GUI dashboards (with modifications for the PoC) as a way to demonstrate how each persona can view the basic status of their resources as their AI environment requests are fulfilled and continue to be used (and eventually removed).  Observability at this stage will provide information for both OCP cluster users and bare metal resource use cases.  Additional observability functions will be added to demonstrate how status and data on new functions added in each PoC is also observable in each stage of integration after this one.

## PoC 3: Additional User Cluster Configuration
This PoC aims to provide additional key configuration options for User Clusters: network isolation, storage, and the ability to add and remove nodes from an existing cluster. This POC will also be the first in which we can show a complete template with compute and storage resources sufficient to run an ML use case, so some example templates should be included in the POC (e.g. S/M/L templates). These features should allow us to demonstrate the automated execution of an ML environment that the user selected.
- ESI node tenancy and dynamic network isolation
- Integration with IBM storage, if available in the MOC.
- Add/remove nodes from an existing cluster
- Improvements to the User GUI, appropriate dashboards and Observability to provide node add/delete, provide dynamic network isolation information (e.g. VLANIDs) and allow the user to select storage options that are automatically provisioned through IBM storage, instead of the local node storage in use for PoCs 1 and 2.  Displaying estimated monthly MOC storage charges for the user’s selection as part of this functionality is highly desirable.
- Improvements to the User GUI, appropriate dashboards and Observability to provide status on automated deployment, operation, and eventual removal of the ML environment selected by the user (e.g. RHOAI) and possibly a default application for the demo (e.g. a small “T-shirt size” ML application).  This state of the PoC will require a more detailed definition of personas.
- Improvements to the existing MOC event services for observability that inform the admin user/PI of critical issues with operating nodes or clusters in their project.  (This may be implemented through one of the existing GUIs if that is more appropriate.   We will need to refine the admin/project leader persona at this stage to determine how best to provide event information.)
- Enable RoCE for GPUs

## PoC 4: Quotas and Hotpool Service
This PoC focuses on enabling and enforcing tenant resource quotas, allowing infrastructure owners to control resource access. It also aims to create the Hotpool Service, enabling the quick and dynamic addition of ACM Agents in response to cluster creation requests.
- ESI lease quotas
- Cluster quotas
- Improvements to the User GUI, appropriate dashboards and Observability to provide information on existing quotas in the MOC (which are set in ColdFront), and to allow a project leader or administrator to change quotas, with an appropriate update to ColdFront.  (Note that this is the first PoC to require specific implementation of functionality that is only for the project leader persona.  This persona has not yet been implemented in MOC Observability or any existing Red Hat production functionality.)
- Hotpool Service 

## PoC 5: Bare Metal Fulfillment and Resource Usage Tracking
This PoC exposes bare metal fulfillment to users, giving them bare metal access to resources that are not necessarily in an OpenShift cluster. It also ensures that we expose resource usage tracking to administrators, allowing them to track tenant resource usage and integrate that information with their own billing systems.
- Bare metal fulfillment
- Resource usage tracking
- Improvements to the User GUI to support bare metal functionality and telemetry.   Bare metal observability will need to be defined more clearly at this stage, because it is assumed to be only generic node allocation/removal in previous PoCs.  (There are several options that have been requested by the bare metal persona from Jonathan Appavoo already, but options for supporting these depend on the type of enhancements we implement in observability at this stage.)

## PoC 6: Group Bare Metal Operations, RDMA, Observability Enhancements
This PoC advances various features: Observability, group bare metal operations, and RDMA support.
- Support for group bare metal operations
- Observability and user GUI enhancements to support group bare metal operations.
- Storage w/ RDMA support via RoCE
- Observability enhancements to show performance improvements from RoCE.

## PoC 7: Multiple Management Clusters and Compliance
This PoC adds support for multiple management clusters, and develops strategies for installing/configuring resources in compliance with specific regulatory and security requirements (e.g. GDPR, AI Act, and the infrastructure owner’s security policy).
- Support for multiple management clusters
- Compliance
- GUI and observability improvements to support critical compliance and regulatory data.   This will require refinement or possibly addition of new personas for this PoC.
