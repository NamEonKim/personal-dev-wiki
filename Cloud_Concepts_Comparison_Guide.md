# Cloud Concepts Comparison Guide

> A portfolio guide to why enterprises use cloud computing, how on-premises environments differ from public cloud, and how common concepts map across AWS, Google Cloud, Microsoft Azure, Oracle Cloud Infrastructure (OCI), and VMware.

## 1. Why Do Enterprises Use Cloud Computing?

Cloud computing lets organizations request IT resources such as servers, networks, and storage as needed, provision them quickly, and manage them according to consumption. NIST describes key cloud characteristics as on-demand self-service, network access, resource pooling, rapid elasticity, and measured service. [NIST Definition of Cloud Computing](https://www.nist.gov/publications/nist-definition-cloud-computing)

### Key Business Benefits

1. **Faster time to deliver services**<br>
   Teams can provision infrastructure through a portal, API, or code instead of waiting for hardware procurement, rack installation, and network setup. Development and test environments can be prepared quickly and removed when no longer needed.

2. **Responding to changes in demand**<br>
   Capacity can be increased during peaks such as shopping events, month-end processing, or campaigns, and reduced during quieter periods. Autoscaling settings must be paired with appropriate application design.

3. **Shifting upfront capital expenditure**<br>
   Instead of building a datacenter first, organizations can use consumption-based pricing for cloud resources. Cloud is not always cheaper, however. Long-term steady-state usage, data transfer, licensing, operational staffing, and reservation or commitment terms should all be compared.

4. **Using global infrastructure**<br>
   Regions and availability zones can help place services closer to users or support disaster recovery designs. Availability does not come automatically from simply moving to the cloud; multi-zone architecture and backup and recovery plans are still needed.

5. **Using managed services**<br>
   Managed databases, messaging, analytics, and AI services can reduce the server operations that an organization must handle directly. Service limits, pricing, security configuration, and data portability still need to be considered.

6. **Automation and standardization**<br>
   Infrastructure as Code (IaC), using tools such as Terraform, CloudFormation, and ARM/Bicep, makes infrastructure configuration reviewable, versionable, and repeatable. It helps deploy development, test, and production environments consistently and track changes.

7. **Business and operational agility**<br>
   Teams can spend less time waiting for infrastructure requests and more time experimenting and improving in small increments. To realize this benefit, cost visibility, access controls, logging and monitoring, and deployment processes must also be designed.

### What Cloud Does Not Solve Automatically

- **Security and operational responsibilities do not disappear.** Providers manage physical datacenters and underlying infrastructure, while customers continue to manage data, accounts, permissions, configuration, and applications. The responsibility boundary varies across IaaS, PaaS, and SaaS. [Azure Shared Responsibility Model](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility), [Google Cloud Shared Responsibility and Shared Fate](https://docs.cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate)
- **Costs do not automatically go down.** Idle VMs, oversized instances, unnecessary stored data, and cross-region transfers can increase costs.
- **Availability is an outcome of design.** Backup, recovery exercises, multi-zone deployment, and failover often need to be designed explicitly.
- **Migration alone is not modernization.** Moving existing servers to virtual machines (lift and shift) can speed up migration, but may also carry over operational patterns and application limitations.

## 2. On-Premises vs. Cloud

| Area | On-Premises | Public Cloud |
|---|---|---|
| Hardware ownership | The organization purchases and owns servers, storage, and network equipment | The cloud provider owns and operates the underlying hardware; customers consume services |
| Getting started | Procurement, installation, network, and security setup are required | Resources are provisioned through a console, API, or code |
| Capacity changes | Adding equipment can take time for purchasing and installation | Capacity can often be added or reduced quickly within provider limits |
| Cost structure | Higher share of upfront capital expenditure (CAPEX), maintenance, power, and facility costs | Primarily consumption-based operating expenditure (OPEX); reservations and commitments may also be available |
| Facility management | The organization is responsible for datacenters, power, cooling, and physical security | The provider manages cloud datacenters and underlying hardware |
| Control | Deep control down to hardware and facilities is possible | The level of control depends on the service model |
| Maintenance | The organization handles hardware replacement and capacity planning | The provider maintains underlying hardware; customer responsibility for the OS and application depends on the service |
| Regions and disaster recovery | Additional investment in separate datacenters and connectivity may be required | Multiple regions and zones may be used, but redundancy and recovery still need to be designed |
| Connectivity | An intranet-centered setup is possible | Console and API access, network connectivity, and identity protection must be configured securely |
| Potential fit | Situations where direct control matters due to specialized hardware, latency, regulation, or existing investment | Situations that benefit from rapid provisioning, variable demand, managed services, or global deployment |

**Core difference:** Cloud still runs on physical servers and networks. The difference is who owns and operates the resources, how much of the stack the consumer manages, and how resources are requested, scaled, and billed.

### Hybrid Cloud

Organizations do not have to move every system at once. Some systems can remain on-premises while others run in public cloud, a pattern called hybrid cloud. For example, a regulated database might stay in the corporate environment while the web/API tier runs in public cloud. Network connectivity, identity integration, observability, data synchronization, and incident response need to be designed together.

## 3. Concepts to Know First

### IaaS, PaaS, and SaaS

- **IaaS:** The provider supplies fundamental infrastructure such as virtual machines, virtual networks, and disks. Customers are generally responsible for the OS and applications running on it.
- **PaaS:** Customers use managed platforms such as runtimes and databases. They focus more on applications and data while the provider takes on more OS and server management.
- **SaaS:** Customers use finished software as a service. They still manage accounts, data, permissions, and settings.

Responsibility boundaries vary by service model, even within the same cloud provider. [Azure Shared Responsibility Model](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)

### Region, Availability Zone, and Datacenter

- **Region:** A geographic area where a cloud provider offers services.
- **Availability Zone:** A unit within a region designed to isolate the impact of failures. Its physical and logical implementation can vary by provider.
- **Datacenter:** A facility containing physical servers and network equipment. Cloud customers typically do not directly manage the location or hardware of a particular datacenter.
- **VMware environments:** VMs can run in vSphere clusters or workload domains inside an organization's datacenter, or on VMware services provided by a cloud operator. VMware itself should not be classified simplistically as the same type of public cloud provider as AWS, Azure, or OCI.

## 4. Mapping Concepts Across AWS, Google Cloud, Azure, OCI, and VMware

The table shows the closest functional concepts. **Items on the same row serve similar purposes; this does not mean their features, limitations, pricing, or operating models are identical.** OCI refers to Oracle Cloud Infrastructure. The VMware column primarily refers to private cloud or VMware service environments based on vSphere/VMware Cloud Foundation (VCF).

| Concept | AWS | Google Cloud | Microsoft Azure | Oracle Cloud Infrastructure (OCI) | VMware (primarily vSphere/VCF) | What It Means |
|---|---|---|---|---|---|---|
| Compute VM | Amazon EC2 instance | Compute Engine VM instance | Azure Virtual Machine | OCI Compute VM instance (Compute also offers bare metal instances) | vSphere Virtual Machine | A virtual server with CPU, memory, and an OS. A VMware VM and an EC2 instance are both VMs, but EC2 is a cloud service provisioned, billed, and integrated by AWS, while a vSphere VM is a workload object created and operated on VMware virtualization infrastructure. |
| Image / VM source | AMI (Amazon Machine Image) | Machine image / boot disk image | VM image / Azure Compute Gallery image | Platform image or custom image | VM template / content library template | A deployment source containing an OS and software configuration. Image formats, cloning, patching, and sharing policies differ. |
| Instance specification template | Launch template | Instance template | VM scale set model / ARM or Bicep configuration | Compute instance configuration | VM template or Aria Automation Cloud Template | Reuses VM specifications and deployment settings. AWS/GCP instance templates and VMware VM templates have overlapping uses but are not the same object. |
| Service catalog / blueprint | CloudFormation template/stack, Service Catalog product | Terraform configuration / Infrastructure Manager deployment | ARM template/Bicep deployment, Azure Deployment Stacks | Resource Manager stacks and templates (Terraform IaC; not a direct service-catalog equivalent) | VMware Cloud Foundation Automation Cloud Template and Catalog (also known by the former VMware Aria Automation name) | Blueprint is a general term for a reusable infrastructure design or request item. AWS CloudFormation templates, Azure Bicep, Terraform configurations, and VMware Cloud Templates differ in implementation and lifecycle. |
| Workflow / orchestration | AWS Step Functions, Systems Manager Automation | Google Cloud Workflows, Cloud Composer | Azure Logic Apps, Azure Automation | Events + Functions for event-triggered tasks (not a general workflow equivalent) | VCF Automation workflows/extensions, vCenter or vRO automation (depending on deployment) | Defines multiple steps, conditions, retries, and integrations. An infrastructure deployment template (IaC) and an execution flow (workflow) are related but distinct roles. |
| Account / organization boundary | AWS Organization → Account | Organization → Folder → Project | Microsoft Entra tenant → Management Group → Subscription → Resource Group | Tenancy → compartments | VCF Organization/Project (depending on product), vCenter inventory Datacenter/Folder/Resource Pool | Boundaries for resource ownership, separation, policy, and billing. They do not map one-to-one. In Azure, a tenant and a subscription are different layers. VMware tenant boundaries vary with the VCF/automation products in use. |
| User and permission management (IAM) | AWS IAM; Identity Center | Cloud IAM (roles granted at organization, folder, project, and resource levels) | Microsoft Entra ID + Azure RBAC | OCI IAM users, groups, and policy statements scoped to a tenancy or compartment | vCenter SSO and roles/permissions; VCF Automation projects/roles (depending on setup) | Controls who can perform which actions on which resources. Policy syntax, inheritance scope, and role design differ. “Account IAM” usually means account and identity/access management (IAM). |
| Workload identity | IAM role / instance profile | Service account | Managed identity / service principal | Instance principals (dynamic groups + IAM policies) | vSphere/VCF service account or external identity integration | An identity used by a VM, application, or automation (rather than a person) to call other services. Using roles and short-lived credentials or managed identities is generally preferred to embedding long-lived passwords in source code. |
| Virtual network | Amazon VPC, subnet, security group | VPC network, subnet, firewall rules | Virtual Network (VNet), subnet, Network Security Group | Virtual Cloud Network (VCN), subnet, route table, security list / network security group, gateways | NSX segment, gateway, distributed firewall, or vSphere port group | Logically separates networks and controls connectivity and traffic. VPC and VNet are similar concepts, but their features and network scopes differ. VMware capabilities depend on whether the environment uses vSphere networking alone or NSX. |
| VM block storage | Amazon EBS volume | Persistent Disk / Hyperdisk | Azure Managed Disk | Block Volume and Boot Volume | VMDK on datastore; a VCF environment may use a vSAN datastore | Block devices attached to a VM, such as an OS or data disk. Performance tiers, replication scope, snapshots, and backup policies should be compared service by service. |
| Object storage | Amazon S3 | Cloud Storage | Azure Blob Storage | OCI Object Storage (Standard, Infrequent Access, and Archive tiers) | General-purpose public object storage is not automatically included with VCF/vSphere; a separate product or external storage is configured | Stores data as objects and keys. It serves a different purpose from a block disk attached to a VM. S3-compatible storage or an object platform can be connected to VMware environments, but this is a separate configuration. |
| File storage | Amazon EFS / FSx | Filestore / Google Cloud NetApp Volumes | Azure Files / Azure NetApp Files | OCI File Storage (NFSv3) | NFS/SMB datastore or external NAS/file service | Shared storage accessed by multiple servers through file-sharing protocols. Protocols, performance, and availability vary by service. |
| Autoscaling / VM group | EC2 Auto Scaling Group | Managed Instance Group | Virtual Machine Scale Sets | Compute instance pools with autoscaling configurations | VM group/scaling configured through vSphere/VCF automation or external operations tools | Increases or decreases VM count based on load or policy. VMware automation may require additional implementation based on installed products and design, so it is not necessarily equivalent to a managed public-cloud scale group. |
| Load balancer | Elastic Load Balancing (ALB/NLB, etc.) | Cloud Load Balancing | Azure Load Balancer / Application Gateway | OCI Load Balancer; OCI Network Load Balancer (L3/L4) | NSX Advanced Load Balancer or another separately configured product | Distributes traffic across backends. Compare layer (L4/L7), features, pricing, and deployment scope. |
| Managed relational database | Amazon RDS / Aurora | Cloud SQL / AlloyDB | Azure SQL Database / Azure Database for PostgreSQL, etc. | Autonomous AI Database Serverless; Base Database Service; Exadata Database Service (Dedicated Infrastructure / Exascale) | Installed on a VCF VM or provided through a separate VMware/external database service | Installing a database on a VM and using a managed database service involve substantially different operational responsibilities. VMware virtualization alone does not automatically provide a managed database service equivalent to a public cloud offering. |

### OCI Fit, Pricing, and Regional Qualifications

OCI combines VM and bare metal Compute with managed Oracle database choices. Autonomous AI Database Serverless automates database operations such as provisioning, patching, backups, and scaling; Base Database Service offers database systems on VMs with more customer control. OCI can suit workloads that need Oracle database services or a choice between virtual machines and dedicated physical compute, but the management responsibilities differ by service. Its tenancy/compartment policy model, VCN design, and Terraform-oriented Resource Manager are provider-specific concepts, not drop-in equivalents.

OCI charges for services according to the selected resources and usage; use the [OCI Cost Estimator and current price list](https://www.oracle.com/cloud/pricing/) rather than assuming one price applies to every configuration. The Free Tier includes a US$300 trial credit for up to 30 days and separate Always Free amounts, subject to eligibility and service limits. Always Free Compute and Autonomous AI Database resources can be provisioned only in the tenancy's home region, and Free Tier resources are unavailable in US Government Cloud regions. Check current quotas, regional service availability, and terms before relying on a free allocation.

OCI regions can have one or multiple availability domains, and service or shape availability can vary by region. Check the target region and service documentation when planning capacity, availability, and disaster recovery.

### Distinguishing Blueprints, Templates, and Workflows

1. **Image:** An OS/disk source used to boot a VM.
2. **VM template / instance template:** A reusable VM specification and base configuration.
3. **Infrastructure template / blueprint:** A design that declares the desired configuration of multiple resources, such as VMs, networks, disks, and permissions.
4. **Workflow:** A procedure that runs tasks such as approval, resource creation, configuration, validation, and notification in a sequence with conditions and error handling.

For example, declaring “a Linux VM with two disks, on a specific subnet, with a monitoring agent” is closer to a template/IaC. “Request received → administrator approval → VM deployment → configuration management → result notification” is closer to a workflow. A product may connect or combine both roles.

## 5. Is “VM = EC2”?

**At the conceptual level, yes.** Amazon EC2 is AWS's service for providing virtual servers, and an EC2 instance is a VM running in that service. AWS documentation describes an EC2 instance as a virtual server in the AWS Cloud. [Amazon EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

More precisely:

- **VM** is a general concept: a virtualized computer.
- **EC2** is the name of AWS's compute service.
- **OCI Compute** is Oracle's compute service; a Compute instance can be a VM or bare metal, so compare VM instances specifically when mapping the VM concept.
- **EC2 instance** is a VM resource created in EC2.
- **vSphere VM** is a VM resource managed on VMware hypervisor infrastructure.
- Therefore, `VM ≈ EC2 instance ≈ Compute Engine VM ≈ Azure VM ≈ OCI Compute VM instance` is useful for an introductory concept comparison, but it does not mean the service spectrum, account structure, networking, storage, pricing, or operational features are the same.

Cloud providers offer more than a VM: they bundle billing, images, networking, security groups/firewalls, disks, autoscaling, monitoring, APIs, and SLAs. These service bundles create platform-specific differences. AWS EC2 is an IaaS example where customers manage the guest OS and applications running inside the VM. [AWS EC2 Concepts](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

## 6. Practical Example: Deploying a Web Application

The diagram below illustrates the roles different cloud components can play.

```text
User
  │
  ▼
DNS → Load Balancer
          │
          ▼
     VM / Compute Instance
       ├── Boot disk (block storage)
       ├── Service identity (least privilege)
       └── Object storage (images, logs, etc.)

Configuration: Terraform / CloudFormation / Bicep / OCI Resource Manager / VMware Cloud Template, etc.
Step automation: CI/CD pipeline or workflow engine
Observability and operations: logs + metrics + alerts + backup + patching
```

For AWS, representative services include EC2, EBS, S3, VPC, and IAM. For Google Cloud, corresponding roles may be served by Compute Engine, Persistent Disk, Cloud Storage, VPC, and Cloud IAM; for Azure, by Azure VM, Managed Disk, Blob Storage, VNet, and Entra ID/RBAC; and for OCI, by Compute, Block Volume, Object Storage, VCN, and IAM. In a VMware-based private cloud, vSphere VM, VMDK/datastore, NSX, and vCenter/VCF Automation may divide these roles depending on the deployment. This is a conceptual example; components cannot always be swapped directly.

## 7. What Carries Across Multi-Cloud—and What Changes

### Operational Concepts That Carry Across

- Compute, networking, storage, identity, images, templates, policies, logs, monitoring, backups, tags/labels, and cost management
- IaC to declare desired state and manage deployments repeatably
- Least privilege, environment separation (dev/test/prod), and change history
- Recovery objectives (RPO/RTO), availability, failure domains, and capacity planning

### What to Recheck for Each Provider

- Hierarchy and the inheritance scope of policies and permissions
- Physical meaning of regions/zones and service availability
- Network addressing, routing, firewalls, and private connectivity
- Performance, replication, and cost models for disks and object/file storage
- Image and template formats, sharing, and update procedures
- Constraints of APIs, IaC providers, workflow engines, and managed services
- Consumption pricing, reservations/commitments, data transfer costs, and licensing

**A useful comparison method:** Instead of forcing a one-to-one mapping between product names, ask: “What capability does this service provide?”, “Which layers do I manage?”, and “Who owns failure, security, and billing responsibilities?”

## 8. Summary

- Enterprises use cloud for faster provisioning, elastic scaling, managed services, global deployment, and automation.
- On-premises and cloud differ in physical infrastructure ownership and operations, resource procurement, and control boundaries.
- `VM` is a general concept; `EC2` is an AWS service. An EC2 instance is a VM provided through AWS.
- AWS, Google Cloud, Azure, and OCI provide common infrastructure concepts, but their product features and hierarchies are not identical.
- OCI's VM and bare metal Compute options and managed Oracle database services can fit workloads with those requirements; check service availability, regional limits, and cost for the intended configuration.
- VMware is a virtualization and private-cloud platform based on vSphere/VCF. Compare it to public cloud by capability layers—VMs, networking, storage, and automation—rather than treating the entire platform as one equivalent service.
- IAM, VMs, storage, templates, and workflows matter in every environment, but similar names do not imply identical policies, behavior, or pricing.

## References

- [NIST — The NIST Definition of Cloud Computing](https://www.nist.gov/publications/nist-definition-cloud-computing)
- [Microsoft Learn — Shared responsibility in the cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)
- [Google Cloud — Shared responsibilities and shared fate](https://docs.cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate)
- [AWS — What is Amazon EC2? / EC2 concepts](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [Google Cloud — Infrastructure Manager overview](https://docs.cloud.google.com/infrastructure-manager/docs/overview)
- [Google Cloud — Grant a workflow permission to access resources](https://docs.cloud.google.com/workflows/docs/authentication)
- [Google Cloud — Compute Engine IAM roles and permissions](https://docs.cloud.google.com/compute/docs/access/iam)
- [Microsoft Learn — Azure Resource Manager templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-create-first-template)
- [Microsoft Learn — Connect Azure Blob Storage from Logic Apps workflows](https://learn.microsoft.com/en-us/azure/connectors/connectors-create-api-azureblobstorage)
- [Oracle — Compute service overview](https://docs.oracle.com/en-us/iaas/Content/Compute/Concepts/computeoverview.htm)
- [Oracle — Networking and VCN overview](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/overview.htm)
- [Oracle — Identity and Access Management overview](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/overview.htm)
- [Oracle — Calling services from an instance using instance principals](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
- [Oracle — Resource Manager overview](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/Concepts/resourcemanager.htm)
- [Oracle — Block Volume overview](https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/overview.htm)
- [Oracle — Object Storage overview](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorageoverview.htm)
- [Oracle — File Storage overview](https://docs.oracle.com/en-us/iaas/Content/File/Concepts/filestorageoverview.htm)
- [Oracle — Creating instance pools](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/creatinginstancepool.htm)
- [Oracle — Load Balancer overview](https://docs.oracle.com/en-us/iaas/Content/Balance/Concepts/balanceoverview.htm)
- [Oracle — Network Load Balancer overview](https://docs.oracle.com/en-us/iaas/Content/NetworkLoadBalancer/overview.htm)
- [Oracle — Events overview](https://docs.oracle.com/en-us/iaas/Content/Events/Concepts/eventsoverview.htm)
- [Oracle — Functions overview](https://docs.oracle.com/en-us/iaas/Content/Functions/Concepts/functionsoverview.htm)
- [Oracle — OCI Database service overview](https://docs.oracle.com/en-us/iaas/Content/Database/Concepts/databaseoverview.htm)
- [Oracle — Autonomous AI Database Serverless documentation](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/index.html)
- [Oracle — Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm)
- [Oracle — OCI Free Tier terms and limits](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm)
- [Oracle — OCI pricing and cost estimator](https://www.oracle.com/cloud/pricing/)
- [Google Cloud — VMware Cloud Foundation components and networking](https://docs.cloud.google.com/vmware-engine/docs/vmware-solution/concepts-vcf)
- [VMware — VCF technical validation](https://www.vmware.com/docs/vmware-esg-technical-validation-cloud-foundation)

---

*Document date: 2026-11-12. Vendor products and terminology can change. Check current official documentation for availability and configuration requirements before designing or implementing a solution.*
