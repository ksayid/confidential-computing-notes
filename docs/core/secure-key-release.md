---
title: Secure Key Release
parent: Core Concepts
nav_order: 3
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
Confidential containers on Azure Container Instances provide a sidecar open source container for attestation and secure key release. This sidecar instantiates a web server, which exposes a REST API so that other containers can retrieve a hardware attestation report or a Microsoft Azure Attestation token via the POST method. The sidecar integrates with Azure Key vault for releasing a key to the container group after validation has been completed.

## Azure Key Vault Container Types
Azure Key Vault is a service that stores and manages cryptographic keys, secrets, and certificates. It has two main container types:

1. Vaults: A multi-tenant service that comes in two different tiers:
    * Standard SKU: Supports only software-protected keys.
    * Premium SKU: Supports both software-protected keys and keys protected by an HSM.
2. Managed HSM:
    * Standard B1: Supports only HSM-backed keys.

## Accessing Keys in Azure Key Vault
1. Obtain an Access Token
    * Use the Azure Instance Metadata Service (IMDS) to get a token for Azure Key Vault.
    * The managed identity must be enabled for the Azure resource to use IMDS.
2. Invoke Key Vault’s REST API
    * Make an HTTP request to retrieve a key.
    * Include the access token as a Bearer token in the Authorization header.
    * If the list operation isn’t allowed, you must know the exact key name (and optionally its version).

## Controlling Access to Keys
By default, if a security principal has permission to get keys, they can retrieve all keys if they know their names. This can be a potential security risk.

Mitigation Option: Use separate Key Vaults for different applications, environments, and regions, per Azure’s best practices.

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


