---
title: Containers
layout: default
---

[← Back to Main Page]({{ "/" | relative_url }})

* TOC
{:toc}


# Containers
## Container Fundamentals
A container is essentially a combination of Linux kernel features (namespaces, cgroups, chroot, and various security mechanisms) that allows processes to be isolated from one another. This isolation can be granular—you can choose which resources to isolate and which to share. Some key points:

* Namespaces: Provide process isolation (e.g., PID, network, filesystem namespaces).
* cgroups: Limit and prioritize resources (e.g., CPU, memory, disk usage).
* chroot: Alters the root directory for a process, contributing to filesystem isolation.
* Lightweight but Shared Kernel: Traditional Linux containers (like Docker) run on the host’s kernel, offering lightweight isolation but a potential security risk—if one container can exploit the kernel, it may affect the host or other containers.

## Open Container Initiative (OCI)
The **Open Container Initiative** (OCI) defines open industry standards for container formats and runtimes. Two key specifications exist:
### Image specification
Describes how container images are packaged, ensuring compatibility across different engines

#### Distribution and Registry
OCI images can be pushed to and pulled from any OCI-compliant registry (e.g., Docker Hub, GitHub Container Registry, Azure Container Registry). The registry stores manifests, layers, and configs in a way that respects the OCI image spec. When you pull an image:
* The client (e.g., Docker CLI) fetches the image’s manifest.
* It then downloads the layers by their hashes if they’re not already cached locally.
* Finally, it retrieves the config JSON that details how the container should be set up at runtime

### Runtime specification.
Standardizes how containers are started, stopped, and managed, allowing for consistent behavior across various runtimes.
* *When a container is launched using an OCI-compliant runtime (like runc or crun), one of the main inputs is a file called `config.json`. This JSON file specifies all the low-level settings needed to run the container,  from which system calls to block (seccomp settings) to which user the container runs as. 
    * Namespaces: For example, `"namespaces": ["pid", "mount", "network"]` might indicate that the container should have separate PID, mount, and network namespaces from the host.
    * Cgroups: Defines resource constraints (CPU, memory, etc.). For instance, you might set `"memory": { "limit": 536870912 }` (512 MB) to control how much RAM the container can use.
    * Root Filesystem: Specifies the path to the container’s root filesystem bundle (the uncompressed image layers). For example, `"root": { "path": "/var/lib/containers/bundle/rootfs" }`.
    * Environment Variables: A list of key-value pairs, e.g., `"env": ["PATH=/usr/local/bin:/usr/bin:/bin", "NODE_ENV=production"]`.
* Defines how a container’s initial process is launched (PID 1 inside the container), and how signals or exit statuses are handled.
* Outlines hooks and events for container creation, execution, pausing, resuming, and deletion.

Several projects implement these standards:
* **runc**: A low-level CLI wrapper around `libcontainer` for directly running OCI-compliant containers. It's minimalistic—unlike Docker, it does not provide a rich API.
* **containerd**: Supports OCI-compliant images. Under the hood, it converts Docker images into OCI bundles, bridging Docker's API calls with lower-level container operations. As an OCI-compliant engine, containerd pulls, caches, and stores images in its content store. This store is keyed by the layer hashes (digests). When you run `containerd pull <image>`, it retrieves the manifest, layers, and config—verifying all hashes—then makes them available to runtimes like `runc`.

## Linux Kernel Features Underlying Containers 
Containers rely on foundational Linux kernel capabilities to provide isolation and resource management.

### Namespaces
Linux namespaces isolate resources at the kernel level, ensuring each container perceives only its own "private" environment. The major namespaces involved are:
* **mnt**: Isolates mount points and filesystems.
* **pid**: Gives processes inside the container their own PID space (with PID 1 inside the container).
* **ipc**: Separates inter-process communication, including shared memory and semaphores.
* **user**: Divides user IDs and privileges, ensuring a container’s root user is not root on the host.
* **net**: Virtualizes the network stack (e.g., each container can have its own loopback interface).
* **uts**: Splits the hostname and domain name (UNIX time sharing).
* **cgroups**: Typically listed separately; manages resource usage constraints.

### Control Groups (cgroups)
* Purpose: Group-level resource allocation, limiting, prioritization, accounting, and process control.
    * Resource limiting (e.g., memory, disk cache).
    * Prioritization (e.g., CPU shares, I/O throughput).
    * Accounting (e.g., track usage for billing or scheduling).
    * Control (e.g., pause/resume processes, checkpoint/restore).
* Hierarchy: Child groups inherit parent resource constraints.
* Controllers: (e.g., "memory" to limit memory usage, “cpuacct” to track CPU usage).
* cgroup filesystem: Typically mounted at `/sys/fs/cgroup`; can browse container resource usage there.

### seccomp (Secure Computing)
* Definition: A kernel mechanism (since version 3.12) that restricts the set of system calls a process can make.
* Purpose: Prevent a compromised container process from invoking “dangerous syscalls,” thus mitigating potential kernel exploits.
* Profiles:
    * Typically use a "denylist" or "allowlist" approach, often referred to as seccomp "profiles."
    * Docker applies a default seccomp profile by default but supports custom profiles

### Linux Security Modules (LSM)
* Goal: Provide additional mandatory access controls (MAC) to confine processes and files.
* Common Examples:
    * SELinux: (Red Hat) Labels files and processes; an SELinux policy determines which process types can access which resource types.
    * AppArmor: (Canonical) Uses policy “profiles” that define file paths, system calls, and other resources processes may access.
* Container Use Case:
    * Prevent containers from reading areas of the host filesystem or other containers’ data.
    * Docker, Podman, and other engines integrate with SELinux/AppArmor profiles.

## Docker Architecture and Components
![Docker architecture overview](../img/docker-architecture-overview.png)
![Docker containerd and shim flow](../img/docker-containerd-shim-flow.png)

### Docker Engine (Client-Server Model)
* Docker CLI: User-facing command-line tool (docker build, docker run, etc.).
* Dockerd (Docker Daemon): Receives CLI requests over HTTP, manages the lifecycle of containers.
* containerd: A separate daemon that handles lower-level container operations (create, start, stop), exposing an API to dockerd.
* runc: The default container runtime for Docker; sets up namespaces, cgroups, seccomp filters, LSM constraints, etc., then forks the container process.
* containerd-shim: A small process that remains after runc creates the container. It keeps the container running if dockerd or containerd restarts.

### Docker’s Container Creation Flow
1. Docker CLI → sends a request to dockerd.
2. dockerd → calls containerd to create the container.
3. containerd → converts Docker images to OCI-compliant bundles, hands them off to runc.
4. runc → performs the namespace/cgroup setup in the kernel and starts the container’s entrypoint process.
5. Shim remains after runc exits to maintain the container process’s I/O, exit codes, etc.

### Overlay Filesystems (OverlayFS / AUFS)
* Union File System: Combines multiple read-only layers and one top-level writable layer into a single mount.
    * Copy-on-write: Modified files get copied to the writable layer; unmodified files remain in the read-only layers.
* Docker Usage:
    * Historically used AUFS, now commonly overlay2.
    * Minimizes duplication across images/containers.

The storage driver is what provides the union filesystem, which manages the layers of the container and how the writeable layer of the container is accessed. In most installations, you won't need to change the default storage driver since a default option will be selected. If you are running a Linux kernel that is at least
version 4.0 or above, your Docker installation will use the overlay2 storage driver; earlier kernels will install the AUFS storage driver.

## Container Runtimes
* Definition: A piece of software that translates user-facing container specs into actual kernel-level isolation (namespaces, cgroups, seccomp, etc.).
* Common Approaches:
    * Native Runtimes (e.g., runC): Container shares the host kernel directly.
    * Sandboxed Runtimes (e.g., gVisor): Interposes a kernel “proxy” to implement syscalls, reducing direct host kernel exposure (not all syscalls are supported).
    * Virtualized Runtimes (e.g., Kata Containers): Runs containers inside lightweight VMs (Firecracker, QEMU). Highest isolation, some performance overhead.

Note: the runtime is not included as a part of Kubernetes. It's a pluggable module that needs to be supplied by you/a vendor to create a functioning cluster.

## Container Isolation
Containers are isolated from the host via namespaces, cgroups, seccomp, and LSMs. To further isolate containers from one another, each container can use separate user namespaces and additional policies via SELinux or AppArmor. A defense-in-depth strategy—combining minimal runtimes (gVisor or Kata) with robust seccomp profiles and LSM rules—further reduces risk.

## Image Layers
* Docker container images are built incrementally. Each step in the Dockerfile creates a new, read-only layer that is stacked on top of the previous layers.
* Each layer has its own 256-bit hash (a content-based identifier). This hash uniquely identifies the layer’s contents.
* Layers are cached and shared among containers. For instance, if multiple images use the same base image layer, Docker only stores it once, which improves efficiency in both disk usage and build time.
* After each layer is downloaded, it is extracted into its own directory on the host filesystem. When you run a container from an image, Docker uses a union filesystem to stack these layers on top of each other, creating a unified view of the filesystem. When the container starts, its root directory is set to this unified directory via chroot.
* If a file already exists in a lower (read-only) layer, Docker doesn’t copy it into the container layer unless you modify it. In other words, files are shared across layers until they need to be changed—only then are they “copied” into the writable layer. Reading a file always checks the topmost layer first. If you replace a file that existed in a lower layer, Docker places a modified copy in your container’s writable layer. The container will then use that copy instead of the original one from the image.
* Any changes made to a running container—new files, modifications, or deletions—are written to a temporary, writable layer called the container layer, which resides on the local host filesystem.
* The Docker storage driver manages requests to this layer and ensures lower layers remain read-only.
* This writable layer is tied to the container ID and persists on the host until you remove or prune the container.
* Because all the layers beneath the container layer are read-only and shared, you can run multiple containers from the same underlying image. Each container simply has its own dedicated (and writable) top layer to capture runtime changes.

## Windows Hyper-V Containers
On Windows, Hyper-V Containers provide stronger isolation by wrapping each container in a lightweight virtual machine, rather than relying on the host kernel directly:
* Utility VM: Each container runs inside a specialized, lightweight VM called a “utility VM.”
* Kernel Isolation: The container’s kernel is not shared with the host or other containers, allowing different kernel versions or patch levels and adding a strong security boundary (no direct host-kernel exposure).
* Stateless Design: Writes inside the container do not persist beyond its lifecycle.
* Startup Optimization: After the initial utility VM is booted, it can be frozen; subsequent containers fork from this frozen state, drastically reducing startup time.
* Management Simplicity: From a Docker perspective, creating a Hyper-V container is similar to creating a normal container (just a different flag).

Hyper-V Containers thus strike a balance between:
* Full Virtual Machines: Very secure but resource-heavy.
* Traditional Containers: Lightweight but share the host kernel (which can be less secure).

## Kata Containers
* Kata Containers combine Intel’s Clear Containers and Hyper.sh’s RunV.
* Uses lightweight VMs to isolate each container’s kernel (Firecracker, QEMU, etc.).
* Stronger Isolation: Similar to Hyper-V containers, but for Linux—no direct host-kernel sharing.
* Performance vs. Security: Adds some overhead compared to native containers, but significantly enhances security boundaries.

## Containers vs. Confidential VMs
- **Containers**: OS-level isolation; rely on kernel namespaces/cgroups.  
- **CVMs**: Hardware-level memory encryption and isolation; the hypervisor cannot see inside them.  
- Confidential VMs are specifically focused on cryptographically securing data "in use" from even the underlying cloud/hardware host.  
- A container can run inside a CVM for additional defense in depth, but that typically comes with extra performance overhead.

<script src="{{ ‘/assets/js/dark-mode.js’ | relative_url }}"></script>
