---
title: Confidential AI
parent: Core Concepts
nav_order: 9
---

Confidential computing relies on trusted execution environments (TEEs), which create a secure perimeter where data remains encrypted in use. These environments provide the abstraction of data being encrypted at all times—from the perspective of an attacker, even someone with root privileges or access to the hypervisor. In essence, TEEs ensure that:
* Data is processed securely within the enclave.
* Even privileged users, such as system administrators, cannot access the plaintext data.
* Cryptographic attestations verify the integrity of the hardware and software within the TEE.

In practice, confidential computing can encapsulate the entire generative AI pipeline within this secure hardware enclave, ensuring that the data is encrypted at every stage—during prompt processing, RAG queries, model inference, and response generation.

Essentially, we have memory encryption, remote attestation, and hardware-based isolation as the core components of this system. Let me break this down:

## Memory Encryption
With memory encryption, here’s how it works:
1. Imagine the processor as the central unit, and The Enclave as a secure area within the processor.
2. The processor has a built-in encryption engine. Before any data leaves The Enclave, it is encrypted by this hardware encryption engine.
3. This means any data traveling on the memory bus to main memory is automatically encrypted by hardware. This encryption is incredibly fast because it’s done at the hardware level, leveraging specific instructions built into the processor.
4. In memory, the data remains encrypted. Even if an attacker with root privileges tries to inspect the memory or snoop on the bus, all they’ll see is encrypted data.
5. When The Enclave needs to compute, the data is fetched from memory, and the encryption engine automatically decrypts it. The decrypted data is then processed inside the CPU.
6. The CPU operates on regular plaintext data internally, maintaining high performance. However, the plaintext data is never exposed outside the CPU die. Attackers with root access or even employees with elevated privileges cannot access the plaintext data—it’s securely isolated within the CPU.

This approach ensures that data is encrypted at all times while in use, from the attacker’s perspective. It’s efficient because the CPU processes the data normally, without noticeable overhead.

## Remote Attestation
### How Confidential VMs Work
1. A secure processor in the hardware manages keys and sets up the confidential VM.
2. The hypervisor requests the secure processor to initialize a VM with specific initial pages.
3. The secure processor measures these pages, which involves computing a hash of all the VM’s pages and associated metadata (like page type, location, etc.). This measurement captures the code and data, defining what the VM will execute.
4. Once the measurement (a digest of these pages) is computed, the secure processor signs it. This is accompanied by a certificate from the hardware vendor (e.g., AMD, Nvidia) to validate it.

### Why This Is Important
The user can now verify that the confidential VM on the cloud is running the exact code and data they intended—without any malware or other unauthorized components. They can also confirm that the hardware vendor has signed and verified the legitimacy of the VM.

### Secure Communication
Additionally, the user knows the public key of the enclave, which allows them to establish a secure communication channel. This is essentially TLS but terminated inside a confidential VM.

The result is that the user remotely verifies the VM’s legitimacy and can securely communicate with the enclave using this established channel.

## Hardware-Based Isolation
Each VM has a unique identifier (ID), allowing multiple confidential VMs to run on the same physical machine. The hardware ensures that the code and data pages associated with a confidential VM are tagged with its unique ID, preventing access by unauthorized entities, including the hypervisor. You can have a mix of encrypted (confidential) and non-encrypted VMs running on the same machine, but each VM remains completely isolated.

## Advances in Confidential Computing
Recently, NVIDIA added enclave support for their H100 GPUs, which is a significant advancement for general-purpose security. Here’s how it works:
1. The GPU is placed inside an enclave, extending the trust boundary from the CPU enclave.
2. The confidential VM running on the CPU attests the GPU. This involves verifying that the GPU is running the correct firmware and is from NVIDIA by checking certificates issued by NVIDIA.
3. Once verified, a secure communication channel is established, ensuring all communication over PCIe is encrypted.
4. This setup enables the CVM to generate a global attestation report, confirming that both the CPU enclave and the GPU are properly configured. Users can rely on this report to ensure their confidential VM is securely connected to the GPU(s). A secure channel is then established to transmit sensitive data to the GPU.

For a deeper look at how NVIDIA actually implements GPU-CC — the architectural engines (FSP, GSP, SEC2, CE), the secure-boot and key-derivation flow, and a security analysis of each runtime data path — see [NVIDIA GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/).

## Using Enclaves for Generative AI (GenAI)
At a high level, using enclaves for GenAI is straightforward:
1. **Model in the Enclave:** Place the AI model inside an enclave.
2. **Secure Data Access:** Keep the database outside the enclave but encrypted. Encrypted data is fetched into the enclave, decrypted, and processed as needed.
3. **Secure Prompt Processing:**
   * Users send prompts over a confidential, encrypted channel (e.g., TLS).
   * The prompt terminates inside the enclave, ensuring no one—including the cloud provider—can see it.
   * The model processes the prompt and sends back the response over the encrypted channel, maintaining confidentiality.

The result is that neither the user’s prompts nor the model’s responses are visible to the cloud provider. This approach also keeps related queries, such as retrieval-augmented generation (RAG) database queries, private.

## Symmetric Cryptography within Enclaves
* Enclaves often use symmetric encryption (e.g., AES in block cipher modes) for internal data protection.
* Symmetric encryption methods are generally considered secure against quantum attacks if key lengths are sufficiently large (e.g., AES-256).
* Enclaves are designed to adapt to advancements in cryptographic standards. As PQC algorithms are standardized and integrated into cryptographic libraries, enclave systems will inherit these updates without requiring fundamental redesigns.

### Encryption Methodology (AES-GCM)
AES-GCM (Galois/Counter Mode) is a widely used cipher combining encryption and authentication. It’s efficient and secure when implemented correctly. However, GCM mode relies on counters to maintain unique initialization vectors. With large data volumes, these counters can "run out" or risk reuse, which compromises security. This indicates that NVIDIA has chosen AES-GCM for its secure encryption, but the design has some scalability considerations that need to be addressed for future-proofing.

If you really want to hide everything, the way to go is to put the model in an enclave and perform private inference, where the whole prompt is encrypted. If that’s not possible because you’re using an external model provider, then you’ll need to sanitize the queries according to whatever policy makes the most sense.

## Confidential AI in More Technical Terms
Confidential AI relies on hardware-based TEEs to shield data and code from unauthorized access—even by the cloud provider’s administrators. Here’s how it works:
1. **Memory Encryption:** The CPU (and in newer cases, the GPU) is equipped with specialized hardware extensions (e.g., Intel TDX, AMD SEV-SNP) that transparently encrypt all memory associated with a given enclave or confidential VM. The encryption keys never leave the hardware’s secure boundary and are not accessible to the host operating system, hypervisor, or physical machine administrators.
2. **Isolated Execution:** The code and data for your AI workload run inside a secure enclave or confidential VM. The CPU enforces isolation so that even with root or administrative privileges on the host machine, an attacker can’t read or modify the enclave’s memory.
3. **Remote Attestation:** Before you trust this environment with your model or data, the hardware can produce an attestation report. This report includes cryptographic measurements of the code and configuration of the enclave. You verify these measurements remotely against expected values. If the measurements match what you expect, you know the code inside the enclave has not been tampered with.
4. **No Visibility for the Provider:** Because the hardware controls the encryption and integrity checking, the cloud provider can’t decrypt the enclave’s memory. Even if the provider dumps RAM or has full control of the hypervisor, what they get is encrypted data they can’t interpret. They essentially own the physical machine, but the hardware security mechanisms prevent them from seeing inside the “black box” running your code.

In short, the cloud provider physically owns the hardware, but the TEEs and attestation protocols ensure that all the data and execution states inside your enclave remain secure, confidential, and verifiably untampered with.

## Step-by-Step Example

This worked example uses Azure-specific services. The Azure-side surfaces named below — **MAA** (Microsoft Azure Attestation), **THIM** (Trusted Hardware Identity Management), and **Managed HSM / Secure Key Release** — are described in detail in Part III of the [Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#12-host-attestation-fleet-internal-verification) note. **OHTTP** (Oblivious HTTP, IETF RFC 9458) is a relay protocol that lets a client encrypt a request to a target service and route it through a proxy that knows the client's network identity but not the request body, while the target service sees the request body but not the client identity — splitting "who's asking" from "what they're asking."

### Step 1: Establish Trust and Obtain Encryption Keys
1. The app contacts the Key Management Service (KMS) to ensure a secure connection with Azure services.
2. The KMS retrieves keys from the Managed HSM.
3. Microsoft Azure Attestation (MAA) validates the security status of the TEE.
4. Trusted Hardware Identity Management (THIM) verifies the TEE's identity using hardware-based credentials.
5. The app generates digests to verify data integrity, and the KMS issues a receipt signature confirming the data has not been tampered with.
6. The app retrieves the OHTTP public key configuration for encrypting data.

### Step 2: Encrypt the Audio File Using OHTTP
The audio file is encrypted using the OHTTP public key obtained in Step 1, ensuring only the intended service can decrypt it.

### Step 3: Send Encrypted Request via OHTTP Proxy
The app sends the encrypted request to the OHTTP proxy, which forwards it to the target service while keeping the service's identity hidden from the public network.

### Step 4: Decrypt and Process the Data in the TEE
The encrypted request is decrypted within the TEE, which is continuously attested to ensure it remains in a trusted state.

### Step 5: Perform Inference Using the Whisper Model in the TEE
The Whisper model converts speech to text securely inside the TEE, with MAA verifying the TEE's security throughout the process.

### Step 6: Encrypt the Response for Secure Transmission Back
The response is encrypted within the TEE before being sent back through the OHTTP proxy to the client.

### Step 7: Receive Encrypted Response and Verify Attestation
The client decrypts the response and checks the attestation records to confirm the data was processed securely in a trusted environment.

## Summary of Key Concepts
* **Trust Establishment:** Ensures all participants (KMS, TEE) are verified and trustworthy.
* **Data Encryption and Anonymity:** Uses OHTTP and proxies to protect data confidentiality and obscure the request path.
* **Secure Execution in TEE:** Provides a controlled environment for secure processing and continuous integrity checks.
* **End-to-End Encryption:** Maintains data confidentiality from initial transmission to the final decryption step.
