---
title: Microsoft Azure Overview
parent: Reference
nav_order: 1
---

Microsoft Azure is Microsoft's public cloud platform — a global network of data centers offering more than 200 services that organizations rent to run software, store data, build applications, and increasingly to develop and deploy AI systems. Originally launched in 2010 as "Windows Azure," the platform was renamed Microsoft Azure in 2014 to reflect a strategic pivot: rather than being a Windows-centric extension of Microsoft's server stack, Azure became a multi-platform cloud that runs Linux as readily as Windows, supports nearly every major programming language, and competes head-on with Amazon Web Services and Google Cloud.

For a customer, Azure functions as a flexible utility. Instead of buying servers, configuring data centers, and provisioning storage hardware, a business pays for what it consumes — compute time, storage capacity, network bandwidth, database queries, AI model invocations — and scales those resources up or down on demand. Azure organizes its offerings into broad categories: **compute** (running code and applications), **storage** (holding files and data), **networking** (moving traffic securely), **databases** (managing structured information), **AI and machine learning**, **identity and security**, **developer tools**, **Internet of Things**, and a growing roster of industry-specific solutions. The sections below walk through the most foundational of these.

## Compute

Compute is the part of Azure that runs your code. Modern Azure offers a spectrum of compute options that trade control for convenience: at one end, you manage everything down to the operating system; at the other, you hand Microsoft a piece of code and they run it for you.

**Virtual Machines (VMs).** A VM is a software-emulated computer running its own operating system inside one of Microsoft's physical servers. Azure VMs are the most flexible compute option: you choose the size (CPU, memory, disk), the operating system (Windows Server or various Linux distributions, with specialty images for SAP, HPC, or GPU workloads), and you install whatever software you want. VMs are the right fit when you need full control, when you're migrating an existing on-premises workload that expects a traditional server, or when you depend on software that isn't available as a managed service.

**App Service.** App Service is a managed environment for hosting web applications and APIs. You upload your code — written in .NET, Java, Node.js, Python, PHP, or Ruby — and Azure handles the underlying server, patching, load balancing, and scaling. App Service shifts the line of responsibility: you own the application; Azure owns the platform. For most modern websites and back-end APIs, App Service is the path of least resistance.

**Azure Functions.** Functions takes the managed model further by removing servers from the picture entirely (the model commonly called "serverless"). You write a small piece of code that responds to an event — a file landing in storage, an HTTP request, a message arriving in a queue, a timer firing — and Azure runs it on demand, charging only for the milliseconds it executes. Functions excels at glue code, scheduled jobs, and event-driven processing where workloads are sporadic and you don't want to pay for an idle server.

**Containers and Kubernetes.** Containers package an application together with everything it needs to run, so the same artifact behaves identically on a developer's laptop and in production. Azure offers three container services along a complexity gradient. **Azure Container Instances** runs a single container with no orchestration — useful for short tasks. **Azure Container Apps** is a serverless container platform that handles scaling, networking, and updates automatically; it's a strong default for microservices-style architectures where you want containers without operating Kubernetes yourself. **Azure Kubernetes Service (AKS)** provides a managed Kubernetes cluster for teams that need fine-grained control. Kubernetes is an open-source system for orchestrating containers across many machines; AKS handles the operational burden of running it.

**Service Fabric** is Microsoft's distributed-systems platform and the official migration target for legacy "Cloud Services" workloads (the old Web, Worker, and VM Roles, all of which are being retired by March 2027). For new applications, most teams will choose AKS or Container Apps instead.

A practical example: if you're building an online store, you might run the storefront on App Service, process payment confirmations through Azure Functions triggered by queue messages, package an inventory microservice in Container Apps, and run a legacy reporting tool on a Virtual Machine.

## Storage

Azure Storage provides durable, internet-accessible places to put data. Each region replicates data across multiple physical locations, and standard accounts keep at least three synchronous copies; geo-redundant tiers replicate again to a paired region hundreds of miles away. The core storage types are:

**Blob Storage.** "Blob" stands for *Binary Large Object* — any unstructured data: images, video, backups, log files, machine-learning datasets. Blob storage is Azure's most heavily used service and the default landing zone for almost any large data. Blobs come in tiers (Hot, Cool, Cold, Archive) that trade access speed for cost — perfect for keeping rarely-touched compliance archives cheap while keeping active product images fast.

**Azure Data Lake Storage** is not a separate service but a hierarchical-namespace mode of Blob Storage optimized for analytics. It exposes folders and access control lists the way a traditional file system does, which makes it efficient for big-data tools like Apache Spark and Microsoft Fabric to scan billions of files.

**Azure Files** provides managed file shares accessible over standard protocols (SMB and NFS), so existing applications that expect a network drive can use Azure Files without code changes. A typical use is replacing an aging on-premises file server while keeping the same drive letter for end users.

**Managed Disks** are block-level storage volumes that act as virtual hard drives for Azure VMs. You pick a tier (from cheap standard HDDs to ultra-premium SSDs) and a size; Azure handles redundancy and snapshots.

**Queue Storage** is a simple messaging service that lets one component drop a message in a queue and another component pick it up later. It decouples producers from consumers — a payment service might place a "send receipt" message that an email worker picks up seconds or minutes later. (For richer messaging — topics, ordering, transactions — Microsoft now points customers at the more capable **Azure Service Bus**.)

**Table Storage** is a NoSQL key-value store for very large numbers of structured records (a NoSQL database stores data without the strict tables and relationships of SQL). Microsoft now positions Table Storage as a starting point and steers customers toward **Azure Cosmos DB for Table** for new workloads — a more capable, globally-distributed version with the same API.

The original "XDrive" feature (mounting storage as an NTFS drive on a VM) has been folded into the broader story: today the equivalent is to attach a Managed Disk to a VM, or to use Azure Files when a network share is what you actually need.

## Resource Manager

Early Azure described a "Fabric Controller" that tracked deployments, replaced failed VMs, and managed the data center. That low-level orchestrator still exists deep in Microsoft's data centers, but customers no longer interact with it directly. The customer-facing management surface is now **Azure Resource Manager (ARM)**.

ARM is the control plane for Azure — the unified API behind every action a customer takes, whether through the Azure portal (the web UI), the command line, an SDK, or an Infrastructure-as-Code template. When you click "create database" in the portal, the portal calls ARM; ARM authenticates the request, applies role-based access control rules, and routes the action to the responsible service. Because every tool funnels through the same API, results stay consistent regardless of how the request was initiated.

Two concepts make ARM central:

- **Resource groups.** Every Azure resource (a VM, a database, a storage account) lives inside a *resource group*, a logical container that bundles related items. You can apply tags, permissions, budgets, and lifecycle rules at the group level — and delete the entire group in one operation when a project ends, instead of hunting down dozens of individual resources.
- **Declarative deployment.** Instead of running a sequence of imperative commands, you can describe a desired state in a template (ARM JSON templates, or the friendlier Bicep language) and ARM figures out what to create, update, or leave alone. This is what makes "infrastructure-as-code" — versioning your infrastructure in Git the way you version source code — practical on Azure.

ARM is also where governance lives. Azure Policy enforces rules ("no public IP addresses on storage accounts in production"), Cost Management tracks spend per resource group, and **Microsoft Entra ID** (the identity service formerly called Azure Active Directory) decides who is allowed to do what.

## Databases

The original "SQL Azure" was renamed **Azure SQL Database** and is now one of many database services. Choosing among them mostly comes down to data shape and compatibility requirements.

**Azure SQL Database** is a fully managed relational database compatible with Microsoft SQL Server. It's the right choice when your application speaks T-SQL (Microsoft's SQL dialect) and you want Microsoft to handle backups, patching, high availability, and scaling. A serverless tier exists for workloads that idle most of the time.

**Azure SQL Managed Instance** is a close-to-on-premises variant of SQL Database that supports nearly the entire SQL Server feature set, designed for "lift-and-shift" migrations of legacy databases that depend on features SQL Database doesn't expose.

**Azure Database for PostgreSQL** and **Azure Database for MySQL** are managed offerings of the two most popular open-source relational databases. Picking these instead of SQL Database is usually about ecosystem fit — using existing PostgreSQL extensions, avoiding licensing concerns, or matching what an open-source application stack expects.

**Azure Cosmos DB** is a globally distributed NoSQL database for applications that need single-digit-millisecond latency in many regions at once. Cosmos DB supports multiple APIs (document, graph, key-value, column-family) and, since 2024, has become a central piece of Azure's AI story by acting as a vector database — storing the numerical "embeddings" that AI applications use to find similar items. A retail site can use Cosmos DB to power both its product catalog and the semantic search behind a chatbot.

**Microsoft Fabric** is Microsoft's unified analytics and data platform, launched in 2023. In March 2026 Microsoft introduced a **Database Hub** in Fabric, providing a single management surface across Azure SQL, Cosmos DB, PostgreSQL, MySQL, and Arc-enabled SQL Server. Direction here is still evolving; treat any specific Fabric capability as fast-moving until it stabilizes.

A practical example: a SaaS company might keep customer accounts in Azure SQL Database, store product telemetry in Cosmos DB for low-latency lookups around the world, and load both into Fabric for analytics dashboards.

## AI Services

The most significant change to Azure since 2023 is the AI tier. **Microsoft Foundry** (rebranded from Azure AI Foundry in late 2025) is Azure's integrated platform for building AI applications — including agents, chatbots, copilots, and retrieval-augmented search systems. Foundry hosts a catalog of more than 11,000 models, including OpenAI's GPT family, Anthropic's Claude, Meta's Llama, Mistral, DeepSeek, and Cohere, and exposes them through a consistent API surface.

The current emphasis is on *agents* — AI components that can reason about a goal, call tools, and chain steps autonomously. As of early 2026 the **Foundry Agent Service** ships with more than 70 prebuilt agents, and the Agent2Agent (A2A) protocol lets agents from different vendors collaborate on a task. Foundry-built agents can also extend **Microsoft 365 Copilot**, surfacing custom enterprise capabilities directly inside Word, Outlook, and Teams. **Agent Framework 1.0**, released in April 2026, is the official .NET and Python SDK for building production agents on this stack.

This part of Azure is moving faster than any other; specific service names, model availability, and pricing have changed multiple times per year. Treat any AI-specific claim here as a snapshot rather than a settled feature set.

## Networking

Underneath all of the above is the network. **Azure Virtual Network (VNet)** is the private network your Azure resources sit in, similar to a corporate LAN but software-defined. VNets connect to on-premises networks via VPN or **ExpressRoute** (a dedicated private link from your data center into Azure).

For traffic *into* applications, two services matter. **Application Gateway** is a regional Layer-7 load balancer with a built-in web application firewall (note: the V1 SKU retires April 28, 2026 in favor of V2). **Azure Front Door** is the global equivalent — a content delivery and acceleration service that routes a user in Singapore to the nearest copy of your application without per-region configuration. For most public-facing applications, Front Door at the edge plus Application Gateway at the regional boundary is the standard pattern.

## Notable Changes from Pre-2014 Azure

The platform has been substantially renamed, restructured, or replaced since its early "Windows Azure" days. The most consequential shifts:

- **Rebranding.** "Windows Azure" became "Microsoft Azure" in 2014, reflecting the platform's pivot to multi-OS and multi-language support. "SQL Azure" became "Azure SQL Database." "Azure Active Directory" became "Microsoft Entra ID."
- **Web/Worker/VM Roles are gone.** The role-based deployment model that defined early Windows Azure has been retired. Cloud Services (classic) was deprecated September 2024; Cloud Services (extended support) retires March 31, 2027. Equivalent workloads now run on App Service, Container Apps, AKS, or Service Fabric.
- **AppFabric is gone.** The original AppFabric was broken up: caching became **Azure Cache for Redis**, messaging became **Azure Service Bus**, and access control merged into Microsoft Entra ID.
- **XDrive is gone.** Mounting blob storage as an NTFS drive is no longer offered; the modern equivalents are Managed Disks for VM-attached storage and Azure Files for shared file access.
- **The Fabric Controller is no longer customer-facing.** Customers now interact with **Azure Resource Manager**, the unified control plane introduced in 2014. The Fabric Controller still operates inside Microsoft data centers but is hidden from users.
- **Containers and Kubernetes.** A whole compute category — containers, orchestrated by AKS or simplified through Container Apps — did not exist in early Azure.
- **AI is now central.** Microsoft Foundry, the Agent Framework, and the Azure OpenAI partnership represent the largest expansion of Azure's surface area in the last several years.
- **Cosmos DB** (globally distributed NoSQL/vector database) is now arguably as important to the database story as Azure SQL.
- **Microsoft Fabric** (the data analytics platform, introduced 2023) is not to be confused with the original "Fabric Controller." Direction is still evolving.

## Sources

- [Azure Products – Browse by Category](https://azure.microsoft.com/en-us/products/category)
- [Azure Storage 2026: Built for Agentic Scale and Cloud-Native Apps](https://azure.microsoft.com/en-us/blog/beyond-boundaries-the-future-of-azure-storage-in-2026/)
- [Choose an Azure Compute Service – Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Cloud Services (extended support) retirement – Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5840515/cloud-services-(extended-support)-will-be-retired)
- [Introduction to Azure Storage – Microsoft Learn](https://learn.microsoft.com/en-us/azure/storage/common/storage-introduction)
- [What is Azure Resource Manager? – Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)
- [Azure Cosmos DB Overview – Microsoft Learn](https://learn.microsoft.com/en-us/azure/cosmos-db/overview)
- [Microsoft Fabric Database Hub – The Register, March 2026](https://www.theregister.com/2026/03/30/microsoft_fabric_database_hub_partial_solution/)
- [Microsoft Foundry – Azure](https://azure.microsoft.com/en-us/products/ai-foundry)
- [Microsoft Ships Agent Framework 1.0 – Visual Studio Magazine, April 2026](https://visualstudiomagazine.com/articles/2026/04/06/microsoft-ships-production-ready-agent-framework-1-0-for-net-and-python.aspx)
- [Application Gateway V1 retirement – Azure Look](https://azurelook.com/azure-update/application-gateway-v1-will-be-retired-on-28-april-2026-transition-to-application-gateway-v2/)
- [Azure networking services overview – Microsoft Learn](https://learn.microsoft.com/en-us/azure/networking/fundamentals/networking-overview)
