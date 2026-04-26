---
title: Microsoft Azure Security Model
parent: Reference
nav_order: 5
---

Four Microsoft Learn pages — the security overview, the shared responsibility model, isolation choices, and protection of customer data — describe one coherent system: **how Microsoft and the customer split security work, and what each side commits to do**.

The shared-responsibility document is the contract: it draws a line down the technology stack and says who patches what, who configures what, who is accountable for what. The overview catalogs Microsoft's side of the line — the layered security services Azure ships and the ones the customer is expected to wire up. The isolation-choices document zooms in on one specific commitment Microsoft makes on its side: keeping customers from interfering with each other on shared infrastructure. The customer-data-protection document zooms in on another: keeping Microsoft itself from interfering with customer workloads and data.

This note pulls all four into one map and connects them to the [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note (infrastructure, encryption, double encryption) and the confidential computing material elsewhere in this repo.

## 1. Shared Responsibility — The Framework

### The matrix

Cloud security duties are split between Microsoft and the customer along a layered stack. The boundary moves depending on the service category:

| Layer | On-prem | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Customer data | Customer | Customer | Customer | Customer |
| Configurations & settings | Customer | Customer | Customer | Customer |
| Identities & users | Customer | Customer | Customer | Customer |
| Client devices | Customer | Customer | Customer | Shared |
| Applications | Customer | Customer | Shared | Shared |
| Network controls | Customer | Customer | Shared | Microsoft |
| Operating system | Customer | Customer | Microsoft | Microsoft |
| Physical hosts | Customer | Microsoft | Microsoft | Microsoft |
| Physical network | Customer | Microsoft | Microsoft | Microsoft |
| Physical datacenter | Customer | Microsoft | Microsoft | Microsoft |

A useful intuition for the categories:

- **IaaS** (Infrastructure as a Service, e.g., Azure Virtual Machines): renting an empty plot with utilities. Microsoft gives you raw VMs, disks, virtual networks; you install and patch the operating system, runtime, and application.
- **PaaS** (Platform as a Service, e.g., Azure App Service, Azure Functions, Azure SQL Database): renting an unfurnished house. Microsoft runs the operating system and runtime; you deploy code and configuration.
- **SaaS** (Software as a Service, e.g., Microsoft 365, Dynamics 365): renting a serviced hotel room. Microsoft runs the entire application; you log in and use it.
- **On-premises**: your datacenter, your problem at every layer.

### Things that never transfer

Regardless of service category, four areas always belong to the customer:

- **Data classification and protection decisions** — only you know what is sensitive.
- **Identity and account lifecycle** — Microsoft Entra ID (the cloud identity service, formerly Azure Active Directory) is operated by Microsoft, but enrollment, multi-factor authentication (MFA) enforcement, conditional access policies, and account hygiene are yours. The matrix's "Customer" label for identity hides this nuance: Microsoft *runs* the service, you *configure* it.
- **Endpoint security** — laptops and phones touching the cloud are yours.
- **Access management** — role-based access control (RBAC), role assignments, conditional access.

### "Responsible for" is not "configurable by"

The most common shared-responsibility mistake is conflating those two. Microsoft is responsible for hypervisor security; you cannot configure it. You are responsible for data classification; nobody else can do it for you. The matrix tells you the *accountability* split, not the *control* split.

Common failure modes that follow from misreading the model:

- Spinning up an IaaS VM and assuming Azure auto-patches the operating system. It doesn't.
- Treating identity as Microsoft's job because Entra ID is hosted. You still own MFA enforcement, conditional access, and account hygiene.
- In PaaS, forgetting that application code, secrets, and app configuration are still yours.
- Assuming SaaS removes endpoint responsibility. It doesn't.

## 2. Microsoft's Side, Layered: Defense in Depth

Once the line is drawn, the next question is what Microsoft actually delivers below it. The Azure security overview groups customer-configurable services into six functional areas. Each addresses a different layer of the stack.

### Identity — the new perimeter

**Microsoft Entra ID** (the Azure cloud identity directory; renamed from Azure Active Directory in July 2023) provides single sign-on, multi-factor authentication, conditional access, and identity governance.

Sub-features worth naming:

- **Privileged Identity Management (PIM)** — just-in-time, time-bound, approval-gated activation of administrator roles, so that elevated permissions are granted only when needed and only for a bounded window. Requires a Microsoft Entra ID P2 or Entra ID Governance license.
- **Managed identities** — Azure-managed credentials attached to compute resources so they can authenticate to other Azure services without storing secrets in code.
- **Conditional Access** — a policy engine that applies signals (user, device, location, sign-in risk) to gate logins.

### Perimeter and network

- **Azure DDoS Protection** — auto-tuned mitigation of distributed denial-of-service attacks at the network and transport layers (OSI layers 3 and 4). Two tiers: Network Protection (per virtual network) and IP Protection (per public IP).
- **Azure Firewall** — stateful firewall delivered as a managed service in three tiers (Basic, Standard, Premium). Premium adds inspection of encrypted TLS traffic, an intrusion detection and prevention system (IDPS), and URL filtering.
- **Web Application Firewall (WAF)** — runs in Application Gateway or Azure Front Door. Pre-configured rule sets defend against the OWASP Top 10 (the Open Web Application Security Project's published list of the most common web-app vulnerability classes, e.g., injection, broken authentication, cross-site scripting).
- **Network Security Groups (NSGs)** — stateful packet filters matched on the standard 5-tuple (source IP, source port, destination IP, destination port, protocol), attached to subnets or network interface cards. They do not inspect application-layer content.
- **Azure Virtual Network Manager** — central authority for security admin rules that override individual NSGs across many virtual networks.
- **Private Link / Private Endpoint** — consume PaaS services over private IPs on Microsoft's backbone instead of public endpoints.
- **VPN Gateway / ExpressRoute** — IPsec site-to-site links and dedicated private circuits.

### Compute

- **Trusted Launch** — adds Secure Boot, a virtual Trusted Platform Module (vTPM, a software TPM 2.0 instance per VM for measurement and attestation keys), and boot integrity monitoring on Generation 2 Azure VMs. The same primitives that protect the host fleet, exposed inside the guest. See the [platform trust chain]({{ site.baseurl }}/docs/misc/microsoft-azure-platform-trust-chain/) note for how Secure Boot, TPM-based measured boot, and code integrity actually work. **Flag:** the original synthesis says Trusted Launch is "default for Generation-2 VMs." As of mid-2025 this was still in public preview; verify current status against Microsoft Learn before relying on it.
- **Confidential computing** — memory-encrypted VMs and enclaves: AMD SEV-SNP, Intel TDX, Intel SGX, and confidential GPU workloads on NVIDIA H100. See [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/).
- **Defender for Servers** — endpoint detection and response (EDR) layered on Microsoft Defender for Endpoint.
- **Encryption at host** — VM disk encryption that supersedes the older Azure Disk Encryption (ADE). ADE is scheduled for retirement on September 15, 2028; existing ADE workloads must migrate to encryption at host before then. (See the [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note for ADE specifics.)

### Storage and data

Access to storage is controlled by Azure RBAC plus **Shared Access Signatures (SAS)** — short-lived URL tokens that grant a specific client a specific permission set on a specific resource for a specific window, without sharing the storage account key. Storage Service Encryption is on by default for new accounts. Storage Analytics logs all authentication requests. (Detailed encryption mechanics are in the [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note.)

### Application

- **App Service Authentication / Authorization** — drop-in identity for web apps without code changes.
- **App Service Environments (ASE v3)** — single-tenant deployment of the App Service stack into your virtual network.
- Customer-driven penetration testing is allowed without prior notification, subject to Microsoft's Rules of Engagement.

### Operations — the cross-cutting layer

- **Microsoft Defender for Cloud** (formerly Azure Security Center plus Azure Defender; merged and renamed at Microsoft Ignite, November 2021). Microsoft positions it as a Cloud-Native Application Protection Platform (CNAPP): a single product that combines Cloud Security Posture Management (CSPM, which continuously scans cloud configurations against best practices) with Cloud Workload Protection (CWPP, runtime threat protection for VMs, containers, storage, databases, and serverless functions). The free tier emits prioritized recommendations; workload-specific plans (Defender for Servers, Containers, Storage, Databases) are paid add-ons.
- **Microsoft Sentinel** (formerly Azure Sentinel; renamed November 2021). Cloud-native SIEM (Security Information and Event Management — log collection, correlation, and analytics) plus SOAR (Security Orchestration, Automation, and Response — automated playbooks that act on detections), with built-in user/entity behavior analytics and threat intelligence.
- **Azure Monitor** plus **Log Analytics workspaces**, queried with **KQL** (Kusto Query Language, Microsoft's read-only SQL-like query language for telemetry data). Without diagnostic settings sending logs to Log Analytics, there is no forensic trail.
- **Azure Policy** — JSON-defined rules evaluated against resource properties. *Initiatives* bundle policies into compliance baselines.
- **Just-in-time (JIT) VM access** (a Defender for Cloud feature) — locks inbound management ports (SSH 22, RDP 3389) at the NSG, opens them on authorized request for a bounded window, then auto-closes.

### Renamed services to map across older docs

| Old name | Current name | Year |
|---|---|---|
| Azure Active Directory | Microsoft Entra ID | 2023 |
| Azure Security Center + Azure Defender | Microsoft Defender for Cloud | 2021 |
| Azure Sentinel | Microsoft Sentinel | 2021 |
| Online Services Terms (OST) | Microsoft Product Terms | rolling since July 2020 |

### Defaults vs. configurables

The overview implicitly distinguishes **default platform security** (DDoS auto-protection, encryption-at-rest defaults for Storage and SQL, Entra ID authentication, built-in threat detection in Defender for Cloud's free tier) from **advanced services the customer must enable** (paid Defender plans, Sentinel, Confidential VMs, Private Endpoints, Customer Lockbox, PIM). Treat the first set as baseline; treat the second set as design decisions per workload.

## 3. Tenant Isolation: How Microsoft Keeps Customers Apart

Defense in depth describes the controls available across the platform. A separate question is how Microsoft keeps two customers from interfering with each other on the same shared infrastructure. This is the topic the isolation-choices document dedicates itself to.

### The hierarchy

The customer's view of Azure has four levels of grouping:

- **Tenant** — a Microsoft Entra ID directory instance. The identity boundary. Two tenants cannot see each other's directories.
- **Subscription** — a billing and resource container associated with exactly one tenant. The primary blast-radius and RBAC scope for resources. One tenant can hold many subscriptions.
- **Management group** — optional grouping above subscriptions for applying policy and RBAC across many subscriptions at once. The hierarchy supports up to six levels of depth below the tenant root group, though Microsoft's own guidance is to keep it to three or four for manageability.
- **Resource group** — a namespace inside a subscription for grouping related resources.

Tenant is not the same as subscription. Tenant is *who can log in*; subscription is *where stuff lives and who pays*.

### The compute isolation spectrum

Five rungs, weakest to strongest tenancy:

| Option | What it gives you | Threat addressed |
|---|---|---|
| Standard multi-tenant VM | Hyper-V hypervisor isolation; per-host packet filters | Untrusted neighbor tenant; baseline side-channel mitigations |
| **Isolated VM SKU** | A SKU sized to fill an entire physical host — guaranteed sole tenant on that machine | Compliance/regulatory single-tenancy without managing hosts |
| **Azure Dedicated Host** | You rent the whole physical server, place your own VMs on it, and control maintenance windows | Single-tenancy plus placement and patch-window control |
| **Confidential VM** (AMD SEV-SNP, Intel TDX) | Hardware-encrypted VM memory; the host and hypervisor cannot read VM memory; remote attestation | Malicious cloud operator; compromised hypervisor |
| **Confidential VM on Dedicated Host** | All of the above stacked | Both neighbor-tenant and operator threats |

A key conceptual split: **isolated and dedicated options protect against neighbor tenants. Confidential VMs protect against Azure itself.** They address orthogonal threats and can be combined.

The Hyper-V hypervisor itself runs as a microkernel directly on bare metal with one privileged root partition (host OS plus Fabric Agent) and many guest VMs. Guests reach hardware only through the VM Bus — Hyper-V's inter-partition message channel that brokers I/O between guest virtualization service clients and the providers running in the root partition. Per-host packet filters block spoofing and unauthorized broadcast traffic. Communication from the cluster's Fabric Controller down to its agents is unidirectional and TLS-protected: agents cannot initiate connections back. (These mechanisms are described more fully in the [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note.)

The current confidential VM SKU families are DCasv5 and ECasv5 (AMD SEV-SNP, general-purpose and memory-optimized) and DCesv5 and ECesv5 (Intel TDX, same split). Confidential GPU support runs on NCC H100 v5 (NVIDIA H100, paired with AMD SEV-SNP host VMs).

### Network isolation

- **Virtual Network (VNet)** — the traffic-isolation boundary. VMs in different VNets cannot talk directly, even within one customer or subscription. Subnets subdivide a VNet.
- **NSG** — stateful 5-tuple allow/deny rules.
- **Service Endpoint vs. Private Endpoint** — both let you reach PaaS services from your VNet, but they differ. A *service endpoint* extends your VNet identity to a multi-tenant PaaS service over Azure's backbone; the service still has a public IP. A *private endpoint* injects a private network interface for the service into your VNet; the service gets an IP from your address space and traffic never leaves the VNet. Private endpoints are stronger; service endpoints are simpler.

### Storage, SQL, App Service, AKS

- **Storage** runs on physically separate hardware from compute, with virtual disks that are sparse — physical blocks are allocated only on first write, preventing data remanence from a prior tenant.
- **Azure SQL Database** is a multi-tenant PaaS with logical isolation. A "logical SQL server" is a metadata grouping, not a VM; databases under one logical server can land on different physical instances. A stateless gateway tier proxies connections (over the SQL Server Tabular Data Stream, or TDS, application-layer protocol) to a backend tier of replicated database instances, which validates logins, applies firewall rules, and routes the request to the physical machine that holds the database.
- **App Service** runs each app in a Windows sandbox per app, rooted at an IIS worker process running under a low-privilege, randomly-generated app-pool identity. Apps share VMs with other tenants but cannot read each other's memory. **App Service Environment (ASE) v3** is a single-tenant deployment of the entire App Service stack into your VNet; use it for regulated workloads or when default network egress controls are insufficient.
- **AKS (Azure Kubernetes Service)** layers isolation through Namespaces (RBAC and quota scope), Network Policies (default-deny pod-to-pod traffic), Pod Security, and separate Node Pools per tenant. For hard isolation, use separate clusters or confidential node pools.

### Decision criteria

- **Subscription-level isolation is enough** when your concern is RBAC blast radius, billing separation, or policy scoping — not infrastructure neighbors.
- **Choose Dedicated Host** when you need single-tenant hardware *plus* control over VM placement and patch windows.
- **Choose an Isolated VM SKU** when you need single-tenancy on one large VM without managing hosts — but accept that the SKU has a hardware-deprecation lifecycle.
- **Choose Confidential VM** when the threat model includes Microsoft itself or a compromised hypervisor.
- **Choose ASE over multi-tenant App Service** when you need VNet-native deployment or hardware-level isolation for web workloads.
- **Choose Private Endpoint over Service Endpoint** when traffic must traverse on-premises links or must never appear on a public IP.

## 4. Customer Data Protection: Constraints on Microsoft Itself

Tenant isolation answers "how does Microsoft keep customers apart from each other?" The customer-data-protection document answers a different question: "how does Microsoft keep itself from accessing customer data?"

### The legal stance

Microsoft's commitment is contractual: under the **Data Protection Addendum (DPA)**, Microsoft is the *processor* and the customer is the *controller* (GDPR terms — the controller decides why and how personal data is used; the processor handles it on the controller's behalf under documented instructions). The page's claim that "Microsoft doesn't claim ownership over customer data" is a legal statement, not a physical one — it means Microsoft contractually commits not to use, mine, or share the data outside service-delivery scope. Physical isolation is enforced by the controls below, not by ownership.

The contractual stack to know:

- **DPA** — defines processor/controller obligations and security commitments. Available at `aka.ms/DPA`.
- **Microsoft Product Terms** — the umbrella terms document. **Replaced the Online Services Terms (OST)**; any reference to "OST" is outdated.
- **GDPR** alignment is delivered through the DPA's EU/EEA-specific terms.

### Microsoft personnel access

Stated policy: access to customer data is **denied by default**. When access is needed for a support case, three controls apply:

- **Just-in-time access** — privilege granted for a bounded window, audited.
- **Multi-factor authentication** is required, from secured workstations.
- **No Microsoft accounts on customer VMs** by default.

### Customer Lockbox

Customer Lockbox is the explicit pre-approval workflow on top of just-in-time access. When a Microsoft support engineer requests access to customer data, a designated approver in the customer's organization (Subscription Owner, Entra Global Administrator, or Customer Lockbox Approver role) must accept or deny the request through the Azure portal.

Specifics:

- The request expires after **12 hours** if no one in the customer organization actions it. It remains visible in the queue for up to **4 days** during that escalation window.
- Once approved, access is granted only for the duration in the request (with a 4-hour maximum) and auto-revoked after.
- **Customer Lockbox is opt-in for Azure** — it is not on by default. Without it, just-in-time access still applies but customer pre-approval does not.

This is a meaningful default to be aware of: just-in-time *limits* Microsoft access, but Customer Lockbox *gates* it on customer consent.

### Data residency: where the bits live

Two related but distinct concepts:

- **Data residency** = the physical location of the bits.
- **Data sovereignty** = which legal regime governs them. Azure offers residency choice, but sovereignty is harder. The U.S. CLOUD Act (the 2018 *Clarifying Lawful Overseas Use of Data Act*) lets U.S. authorities compel U.S.-based providers to disclose data they hold regardless of where it is stored — Microsoft itself has publicly acknowledged it cannot offer an absolute guarantee against this for non-U.S. customer data.

Three storage replication options affect residency:

- **LRS (Locally Redundant Storage)** — three copies, one facility, one region.
- **ZRS (Zone-Redundant Storage)** — three copies across two or three availability zones.
- **GRS (Geo-Redundant Storage)** — six copies; three in the primary region plus three in a paired region hundreds of miles away. **GRS is the default at storage-account creation.**

If residency is a hard requirement, switch from GRS to LRS or ZRS deliberately — the default replicates your data into a *second region*.

### Hardware disposition

When data-bearing devices retire, Microsoft sanitizes them on-site following **NIST SP 800-88** guidelines (the U.S. National Institute of Standards and Technology's standard for media sanitization, which defines three escalating categories: *Clear*, *Purge*, and *Destroy*), with physical and logical chain-of-custody tracking through final disposition. A disk delete triggers a NIST 800-88 *Clear* before space is reallocated.

### Government data requests

Stated commitments: no direct or unfettered government access; no provision of encryption keys; a court order or warrant is required for content. For enterprise data, Microsoft redirects requesters to the customer wherever possible. Microsoft publishes a biannual **Law Enforcement Requests Report**; well under 1% of demands target enterprise customers (the H1 2025 report records 168 such requests against many tens of thousands of consumer-data requests).

### Subscription termination timeline

- Billing stops immediately on cancellation.
- A retention window of 30 to 90 days allows recovery.
- The subscription auto-deletes once that window closes (90 days at the latest).
- Immutable or retention-locked data persists read-only until its retention policy expires, even after the subscription is gone.

## 5. Practical Takeaways

### Day-one baseline

- **Microsoft Entra ID** with MFA, conditional access, and RBAC. Identity is the new perimeter.
- **NSGs on every subnet** as the cheapest baseline traffic control.
- **Encryption at rest** — on by default; verify the customer-managed-key story if you operate under regulatory constraints. (See [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/).)
- **Microsoft Defender for Cloud (free tier)** for a posture score and recommendations.
- **Azure Policy** with at least one regulatory initiative assigned at the management-group scope.
- **Diagnostic settings forwarding to a Log Analytics workspace** — without this, no forensic trail.

### Decisions per workload

- **Defender plans, Sentinel, Confidential VMs, Private Endpoints, PIM, JIT VM access** — paid or higher-tier; enable per workload as the threat model demands.
- **Customer Lockbox** — opt in if you want pre-approval over Microsoft engineer access during support.
- **Replication tier (LRS / ZRS / GRS)** — pick deliberately; the default is GRS, which crosses region boundaries.

### Most-common shared-responsibility traps

- Assuming Azure auto-patches your IaaS VM operating system. It doesn't.
- Treating identity configuration as Microsoft's job. It isn't.
- Forgetting that PaaS still leaves application code, secrets, and configuration to you.
- Letting the GRS default carry data into a second region when residency is contractually constrained.

### Connecting to confidential computing

The isolation-choices document explicitly distinguishes neighbor isolation from operator isolation. Standard multi-tenant VMs, dedicated hosts, and isolated SKUs all defend against neighbor tenants but ultimately trust the hypervisor and the host operating system. Confidential VMs (AMD SEV-SNP, Intel TDX) and confidential GPU workloads (NVIDIA H100) remove that trust by encrypting VM memory in hardware and proving system state via remote attestation. For background, see [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/), [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), and [TDX Architecture]({{ site.baseurl }}/docs/core/tdx-architecture/). The [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note describes how the underlying infrastructure layers — hypervisor, host OS, Fabric Controller — fit together.

## Sources

- [Introduction to Azure security](https://learn.microsoft.com/en-us/azure/security/fundamentals/overview)
- [Shared responsibility in the cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)
- [Isolation in the Azure Public Cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/isolation-choices)
- [Protection of customer data in Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/protection-customer-data)
- [Customer Lockbox for Microsoft Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/customer-lockbox-overview)
- [Microsoft Products and Services Data Protection Addendum (DPA)](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA)
- [Azure Dedicated Hosts overview](https://learn.microsoft.com/en-us/azure/virtual-machines/dedicated-hosts)
- [App Service Environment overview](https://learn.microsoft.com/en-us/azure/app-service/environment/overview)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction)
- [What is Microsoft Sentinel?](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [Azure storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Data-bearing device destruction (Microsoft Service Assurance)](https://learn.microsoft.com/en-us/compliance/assurance/assurance-data-bearing-device-destruction)
- [NIST SP 800-88 Rev. 1 — Guidelines for Media Sanitization](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-88r1.pdf)
