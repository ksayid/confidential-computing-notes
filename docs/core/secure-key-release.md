---
title: Secure Key Release
parent: Core Concepts
nav_order: 8
---

## Key Management System (KMS)
A KMS is a piece of software that performs cryptographic operations (such as encryption and managing private keys). It is usually embedded inside a secure hardware component or inside hardware security modules (also referred to as HSMs).

For instance, let's take a running web application: a particular attention must be given to passwords and credit card details when storing them. Usually, these issues are resolved by encryption.

For the encryption to be secure, the key to decrypt must be stored securely, and that key must be encrypted by another key to protect it.

This key chain can quickly become quite complicated, especially in company setting. But at the root of the concept, there is always going to be a master key that we must securely store, and it cannot be done by simply encrypting it.

This where a KMS comes in handy. One of its features is to manage keys: it will import them, manage the users and the roles, etc. It will do so in a secure and protected way, completely isolated from the services that use it. That is because KMSs can perform multiple cryptographic operations. They can store private keys and certificates, perform encryption and key rotation...

Attester — The Attester is the entity which informs the relying party of the state of the system by sending evidence. The evidence informs the relying party about the state of the system, the Trusted Computing Base (the part of hardware, firmware and software that is trusted) and other aspects of the system. The evidence comprises a set of claims, later asserted or denied by the verifier. The evidence is cryptographically signed with a key (e.g. coming from the silicon vendor) that is used during the verification process.

If you are looking for a deep dive into the attestation flows and how they differ between different vendors, I would recommend reading this paper — https://systex22.github.io/papers/systex22-final79.pdf

Relying Party/Key Broker Service (KBS) — The KBS is the relying party and the following are its primary function:

Receives evidence from the attester (confidential VM or container) via challenge-response protocol
Relay the evidence to the Attestation Service (Verifier) for verification.
Apply appraisal policy for the returned Attestation Results to assess the trustworthiness of the attester.
Interact with Key Management Service to retrieve the keys and send them to the attester
Verifier (Attestation Service) — The Attestation service verifies the evidence, based on configured policy and reference values. Think of reference values as “good” and “trusted” values which are known beforehand and used for verification of the evidence sent by the attester. These reference values are typically generated when building the system software and firmware layers, through a coupled CI/CD pipeline.

Key management service — A service for securely storing, managing, and backing up cryptographic keys used by applications and users.

## Secure key release sidecar
Confidential containers on Azure Container Instances provide a sidecar open source container for attestation and secure key release. This sidecar instantiates a web server, which exposes a REST API so that other containers can retrieve a hardware attestation report or a Microsoft Azure Attestation (MAA) token via the POST method. The sidecar integrates with Azure Key Vault for releasing a key to the container group after validation has been completed. (MAA, the Azure attestation service that validates TEE evidence and issues signed JWTs, is described in detail in §12.4 of the [Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#microsoft-azure-attestation-maa-the-customer-facing-service) note.)

## Azure-specific KMS surfaces

The Azure-side KMS lineup (Key Vault Standard, Key Vault Premium, Managed HSM, Cloud HSM, Payment HSM) and the per-option tradeoffs around tenancy, FIPS level, and use case are catalogued in §16 of the [Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#16-choosing-a-key-manager-the-five-way-decision) note. Operationally, an Azure VM obtains a Key Vault access token through the Instance Metadata Service (IMDS) using a managed identity, then invokes the vault's REST API with the token as a `Bearer` credential; if list permission is withheld, callers must know the exact key name. A common access-control hardening pattern is to use **separate vaults per application, environment, and region** so that "permission to get keys" does not leak across blast-radius boundaries.

## Introducing Secure Key Release
Secure Key Release is a policy-based method for releasing keys with additional checks to ensure the requesting environment is trustworthy. This is especially useful for Trusted Execution Environments (TEEs) like Confidential VMs.

Key differences in Secure Key Release:
1. Access Policies:
    * Add release key permissions for a specific managed identity.
2. Key Properties:
    * The key must have a Release Policy that defines the criteria for trusted environments.
    * The key must be marked as exportable.
3. HTTP Request:
    * Use a modified URL to target the release operation instead of the standard get operation.
4. Environment Assertion:
    * Include the attested platform report (environment assertion) in the request body.
    * The Microsoft Azure Attestation service ensures that the environment is compliant (e.g., correct firmware version, region) and the TEE is trustworthy.

## How Environment Assertion Works
* Environment Assertion:
    * A signed JSON Web Token (JWT) from a trusted authority.
    * Contains details (claims) about the environment such as TEE type, publisher, version, etc.
* Key Vault’s Release Policy:
    * Matches the claims in the Environment Assertion against the Key Vault’s requirements.

## Encrypted filesystems

A natural downstream use of secure key release is **encrypted-at-rest filesystems**: a TEE attests, the KMS releases a disk encryption key to it, and the TEE uses the key to mount an encrypted volume.

A **filesystem** is responsible for organizing, storing, and retrieving data on a storage device, like a hard drive or SSD. The goal of a cryptographic file system is to secure data stored on disk through encryption — without the correct encryption key, the data is unreadable to unauthorized users.

Two approaches:

* **Volume encryption** encrypts the entire storage or volume. When data is written to or read from the storage, it passes through an encryption layer that encrypts or decrypts the data on-the-fly. This method uses a single cryptographic key for both data and metadata, effectively making the entire volume secure from unauthorized access. However, because it encrypts everything with a single key, individual file security is not possible.
    * **dm-crypt**: Linux kernel feature that provides a generic way to create encrypted volumes.
* **File system level encryption** encrypts individual files or directories within the file system itself. Each file or directory can be encrypted with a different key, providing a more granular level of security. This approach is particularly useful for multi-user systems where different users or applications may need access to specific files without having access to the entire volume.

### Encrypted filesystem sidecar

Confidential containers on Azure Container Instances provide a sidecar container to mount a remote encrypted filesystem previously uploaded to Azure Blob Storage. The sidecar transparently retrieves the hardware attestation and the certificate chain endorsing the attestation’s signing key. It then requests Microsoft Azure Attestation to authorize an attestation token, which is required for securely releasing the filesystem’s encryption key from the managed HSM. The key is released to the sidecar container only if the attestation token is signed by the expected authority and the attestation claims match the key’s release policy. The sidecar transparently uses the key to mount the remote encrypted filesystem; this preserves the confidentiality and integrity of the filesystem upon any operation from a container in the container group.

The sidecar is a form of **volume encryption**:

* The sidecar container is responsible for accessing and mounting a filesystem that is encrypted and stored remotely (Azure Blob Storage). Once decrypted, the entire filesystem becomes accessible to the application container.
* The process of obtaining the encryption key through hardware attestation and the Azure Attestation token further suggests a volume encryption approach: a single key is used to unlock the entire filesystem, rather than managing separate keys per file or directory.

> Imagine a locked chest (the encrypted filesystem) containing treasures (the data). The sidecar container has the key (encryption key) to this chest. When the application wants to access the contents, the sidecar unlocks the chest and presents the treasures as if they were always out in the open, hiding the fact that they were locked away when not needed. The application can use the treasures without ever needing to know about the lock or the key.

References:

* https://techcommunity.microsoft.com/t5/azure-confidential-computing/nlp-inferencing-on-confidential-azure-container-instance/ba-p/3827628
* `cryptsetup`
