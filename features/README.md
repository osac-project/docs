# Features

Not all of these requirements will be present in initial iterations of the
solution. However, the solution should eventually support all of the following:

* Provide an API and command line tooling that can be used by tenants or by cloud providers that want to expose their own interfaces.
* Support strong multi-tenancy where different clusters are isolated at L2, and no tenant can observe or compromise other tenants. 
* Provide an inventory component with rich topology information that acts as a source of truth for all hardware and network resources. 
* Provide tenants with the ability to automatically have nodes selected for their cluster based on requirements around topology and hardware metadata.
* Provide tenants with access to all the infrastructure information needed to debug complex distributed AI applications. 
* Provide a simple reference user interface while still permitting advanced users to have granular control over their cluster configuration. 
* Implement per‑tenant CPU, GPU, memory, and storage quotas with API and dashboard visibility, including alerts when usage nears limits.
* Ensure that deployed clusters can meet various security and compliance requirements including HIPAA and GDPR to fulfill sovereignty needs.
* While initial components supported will be driven by the requirements of the MOC, the solution should support alternative implementations after some form of validation. The solution must be feasible to deploy in the production MOC.
* The solution will be designed with clear interfaces between components, allowing customers to replace individual elements (e.g., switch management tooling) with options tailored to their own environment.
* Provide (as a configuration option) approximate cost information to be exposed to users to enable users to understand the implications of their choices.

All of this adds up to an easy-to-use, on-demand solution that cloud providers can tailor to their needs. The solution keeps each customer securely separated, tracks all hardware and network details, meets compliance requirements, and shows clear, upfront costs.

## Feature Documentation

* [Console Access](MGMT-22670-console-access.md) - Serial console access for compute instances
