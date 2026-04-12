---
title: Kubernetes
parent: Container Technologies
nav_order: 4
---

Kubernetes is a container orchestration platform. In the context of confidential computing, it raises questions about trust boundaries, secret storage, and workload isolation.

## Confidential Computing in Kubernetes
1. **Option 1: "Wrap the whole cluster"**  
   - Run the control plane and worker nodes inside confidential VMs so that the cloud provider (and other tenants) cannot access your cluster data.  
   - Straightforward if you trust all cluster components and just want to shield them from the outside world.

2. **Option 2: Per-Node or Per-Workload TEE**  
   - Harder if you need to protect a single worker node from an untrusted admin in the same cluster.  
   - The kubelet generally has broad control over pods, which can undermine TEE guarantees.  
   - True multi-tenant "untrusted admin" approaches require more careful design (e.g., ephemeral micro-VMs for each pod).

## Multi-tenant vs. Single-tenant Security
- Major cloud providers often run Kubernetes in multi-tenant ways (shared control plane, multi-tenant etcd).  
- Confidential computing (enclaves, CVMs) shields your workload from external threats, but not necessarily from other components in the same logical environment if they share trust boundaries.

## Storing Secrets & etcd
- Confidential computing alone does not fully solve storing secrets in Kubernetes etcd.  
- You could isolate etcd in an enclave, but architectural changes might be required.  
- Evaluate your threat model to see if enclaves add enough protection for your use case.

**See also:** [Kata Containers]({{ "docs/containers/kata-containers/" | relative_url }}), [Confidential Containers]({{ "docs/containers/confidential-containers/" | relative_url }})

## Helm
Helm is an open-source tool that helps you define, install, and manage applications in Kubernetes. Often described as the “package manager for Kubernetes,” Helm bundles Kubernetes manifests (such as Deployments, Services, and ConfigMaps) into reusable, version-controlled “charts.” Using charts, you can:

* Quickly deploy pre-configured apps and services.
* Easily upgrade or roll back an application to a previous version.
* Customize each installation via parameters without duplicating configuration files.

In essence, Helm streamlines the process of packaging all your Kubernetes YAML files for an application or microservice and distributing them as a chart—making Kubernetes deployments more repeatable, maintainable, and shareable.

### Key Concepts
* Charts: a Helm package containing all the resource definitions necessary to run an application, tool, or service in a Kubernetes cluster. Think of it as the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file. A single chart might include everything from Deployments and Services to ConfigMaps and Ingress definitions.
* Repository: a collection of charts that can be searched, shared, and downloaded. This is analogous to Perl's CPAN or the Fedora Package Database, but for Kubernetes packages.
* Release: a specific instance of a chart running in a Kubernetes cluster. Because one chart can often be installed multiple times into the same cluster, each installation is associated with a unique release. For example, if you install a MySQL chart twice for two separate databases, you end up with two releases—each with its own release name and associated resources.

### How Helm Works
* Installation & Release: When you run helm install, Helm interacts with the Kubernetes API to create the necessary resources. Helm also keeps track of the release state, including which chart version was deployed and what values were used. This makes it straightforward to upgrade or roll back a release later using helm upgrade or helm rollback.
* Templating: Helm uses the Go templating engine to merge YAML manifests in the templates/ directory with user-specified values. This allows you to dynamically configure Kubernetes resources without duplicating configuration files.

### Chart Structure
A typical Helm chart has a standardized file and directory layout:
* Chart.yaml: Contains metadata about the chart, including the chart’s name, version, and a description. This file is also accessible from within your templates for reference.
* values.yaml: Defines the default configuration values for the chart. Users can override these defaults during helm install or helm upgrade by providing their own values file or command-line parameters.
* templates/: Holds the template files. When Helm renders a chart, all files in this directory are processed through the Go templating engine. The resulting Kubernetes manifests are then submitted to the cluster.
* charts/:  May contain other charts, often referred to as "subcharts." These subcharts can be dependencies, allowing you to compose multiple charts together into a larger application stack.
