---
title: Encrypted Filesystem
parent: Core Concepts
nav_order: 4
---

A **filesystem** is responsible for organizing, storing, and retrieving data on a storage device, like a hard drive or SSD. It manages files and directories and controls how data is stored and retrieved. The goal of a cryptographic file system is to secure data stored on disk through encryption. Without the correct encryption key, the data is unreadable to unauthorized users. 

Two approaches: 
* **Volume encryption**: encrypts the entire storage or volume.  When data is written to or read from the storage, it passes through an encryption layer that encrypts or decrypts the data on-the-fly. This method uses a single cryptographic key for both data and metadata, effectively making the entire volume secure from unauthorized access. However, because it encrypts everything with a single key, individual file security is not possible.
    * **dm-crypt**: Linux kernel feature that provides a generic way to create encrypted volumes.
* **File system level encryption**: encrypts individual files or directories within the file system itself. Each file or directory can be encrypted with a different key, providing a more granular level of security. This approach is particularly useful for multi-user systems where different users or applications may need access to specific files without having access to the entire volume. 

The **encrypted filesystem sidecar** is a mediator between the encrypted data stored in Azure Blob Storage and the application that needs to access this data. Before the encrypted filesystem can be mounted and used by the application container, the sidecar container must ensure it is authorized to access the encryption key. This involves a security process called **attestation**—proving to Azure's security services that the container is running in a secure and compliant environment. Once attestation is successful, the sidecar container is given the encryption key. With the encryption key, the sidecar container can decrypt the entire filesystem (or volume) so that the application container can use it. This step is where the concept of block encryption is directly applied, though the sidecar container abstracts away the complexity. The application sees a regular filesystem, unaware that it's encrypted at rest and only decrypted in memory or when accessed.

> Imagine a locked chest (the encrypted filesystem) containing treasures (the data). The sidecar container has the key (encryption key) to this chest. When the application (a user of the treasure) wants to access the contents, the sidecar container unlocks the chest and presents the treasures as if they were always out in the open, hiding the fact that they were locked away securely when not needed. The application can use the treasures without ever needing to know about the lock or the key.

The sidecar is a form of **volume encryption**. 
* The sidecar container is responsible for accessing and mounting a filesystem that is encrypted and stored remotely, likely im a block storage managed by Azure Blob Storage. Once decrypted, the entire filesystem becomes accessible to the application container.
*  The process of obtaining the encryption key through hardware attestation and Azure Attestation's authorization token further suggests a volume encryption approach. A single key is used to unlock the entire filesystem, rather than managing separate keys for different files or directories within the filesystem.

* https://techcommunity.microsoft.com/t5/azure-confidential-computing/nlp-inferencing-on-confidential-azure-container-instance/ba-p/3827628
* cryptsetup

## Encrypted file system sidecar
Confidential containers on Azure Container Instances provide a sidecar container to mount a remote encrypted filesystem previously uploaded to Azure Blob Storage. The sidecar container transparently retrieves the hardware attestation and the certificate chain endorsing the attestation’s signing key. It then requests Microsoft Azure Attestation to authorize an attestation token, which is required for securely releasing the filesystem’s encryption key from the managed HSM. The key is released to the sidecar container only if the attestation token is signed by the expected authority and the attestation claims match the key’s release policy. The sidecar container transparently uses the key to mount the remote encrypted filesystem; this process will preserve the confidentiality and integrity of the filesystem upon any operation from a container that is running within the container group.

