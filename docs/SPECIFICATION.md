🌐 **Language:** [English](SPECIFICATION.md) | [Русский](SPECIFICATION.ru.md)

# TECHNICAL SPECIFICATION: DECENTRALIZED MESSENGER "M-KOLIBRI"

## 1. Introduction and Project Concept
The **m-kolibri** project is a specialized software application for the Android mobile platform. The messenger is designed to enable text communication in fully decentralized, general-purpose mesh networks (based on LoRa technology / the Meshtastic hardware ecosystem) in the complete absence of the global Internet, cellular infrastructure, or intermediate signaling servers.

The architectural concept relies on two fundamental principles:
1. **Maximum data compression** to overcome the strict hardware payload limit of a LoRa packet.
2. **Two-stage security architecture:** starting with native Meshtastic channel encryption in MVP (`v0.1`) and scaling to full application-layer End-to-End Encryption (`E2EE`) between Peer-to-Peer (`P2P`) nodes without a trusted central authority (`v0.2+`).

The application works with off-the-shelf hardware boards (LilyGO T-Beam, Heltec V3, etc.) running standard Meshtastic firmware. Smartphone-to-board communication is handled over Bluetooth (BLE) via the official Meshtastic API using Google Protocol Buffers (`Protobuf`). Custom compressed payloads are transmitted over a dedicated application port (`PortNum.PRIVATE_APP`), eliminating the need to flash custom firmware onto radio modules.

---

## 2. Architectural Security & Network Model: P2P and E2EE
* **P2P (Peer-to-Peer Network):** An interaction model that eliminates dedicated servers. All network nodes have equal privileges and act simultaneously as both clients and routers. Within mesh networks, P2P architecture ensures maximum fault tolerance: having no single point of failure makes the network resilient to external blocking.
* **E2EE (End-to-End Encryption — Release v0.2+):** An architectural approach where data is encrypted directly on the sender's device and decrypted exclusively on the final recipient's device. Intermediate mesh nodes (repeaters, peers) only relay packets in transit without the technical ability to access plaintext messages. *(Note: In MVP v0.1, over-the-air confidentiality is provided by Meshtastic's native AES-256-CTR channel encryption).*

---

## 3. Dictionary Mathematics and Modular Compression
To optimize radio channel throughput, the original **"Kolibri-code"** algorithm is used. Instead of character-by-character UTF-8 transmission, text is converted into fixed indices from a static codebook. Scaling to complex writing systems (e.g., Chinese, Arabic) is excluded from core development due to high morphological/segmentation complexity and is delegated to the open-source community.

### 3.1. English Module (`Base32`)
English words are encoded using a 3-character base-32 code (alphabet includes uppercase Latin letters and digits `2–7`). Dictionary capacity is $32^3 = \mathbf{32,768}$ unique word forms. Given English's weak morphology, this volume covers a broad vocabulary, including technical and specialized terms. Each token weighs strictly **3 bytes**.

### 3.2. Alternative Alphanumeric Module (`Base36`)
As an alternative or extended dictionary design, a 3-character alphanumeric code (Latin letters `a–z` and digits `0–9`) with a 36-character alphabet can be used. Total unique combinations equal $36^3 = \mathbf{46,656}$, covering everyday vocabulary, slang, and core technical terms.

### 3.3. Russian Module (Two-Tier Binary Code)
Rich morphology, cases, declensions, and conjugations in Russian make whole-word encoding inefficient. The **m-kolibri** architecture uses **3 binary bytes (24 bits)** per word, split into two static lookup tables:
* **Stem Table (First 2 bytes):** Provides $2^{16} = \mathbf{65,536}$ combinations for immutable base word forms (lemmas), fully covering the active Russian vocabulary.
* **Inflection & Ending Table (Third byte):** Provides $2^8 = \mathbf{256}$ combinations. Each byte value encodes a grammatical modifier (case, number, gender, verb tense). Upon decompression, the recipient's app synthesizes the exact word form from the stem and inflection.

### 3.4. Modular Architecture, Distribution, and Exception Handling
To minimize the initial size of the mobile app, the core engine is compiled separately. After installation, the user selects a language pack, and the corresponding static binary dictionary (`ru.bin` or `en.bin`) is downloaded.

Each packet includes a version identifier and a **1-byte language/protocol flag** in the header, allowing nodes to process or ignore incoming data types without desynchronizing the network.

> **Exception Handling Mechanism:** When a user enters words missing from the static codebook, the app falls back to character-by-character encoding preceded by a dedicated control prefix, signaling the decompression module to switch to raw character reading mode.

---

## 4. Network Architecture and Physical LoRa Packet Structure
The maximum physical LoRa packet size in the Meshtastic ecosystem is approximately 230–240 bytes (200 bytes pure frame limit in certain configurations). Accounting for standard firmware overhead (network headers, routing, `PRIVATE_APP` Protobuf container), **150 to 183 bytes** of clean payload remain for "Kolibri-code", allowing **50 to 61 compressed words** per single message (and up to ~65 words in v0.1 without E2EE `Nonce` headers).

### Payload Byte Allocation (Full E2EE Frame)

| Frame Element | Size | Purpose in P2P Mesh Network |
| :--- | :---: | :--- |
| **Flags / Lang / Version** | 1 byte | Protocol version, encryption flag (`v0.1` vs `v0.2`), and dictionary ID |
| **Sender ID** | 4 bytes | Local unique address of the source node |
| **Receiver ID** | 4 bytes | Target node address (or `0xFFFFFFFF` for broadcast channel) |
| **Packet Counter (Nonce)** | 8 bytes | Replay-attack protection *(v0.2+ E2EE mode)* |
| **Compressed Payload** | up to 183 bytes | Stream of encoded word tokens (3 bytes per word, up to 61 words) |

For reliable packet transport from Android to the LoRa modem over Bluetooth, **Base64** text encapsulation is used to prevent data corruption from BLE/SPP control characters. The Meshtastic firmware strips Base64 before transmitting raw binary packets over the air.

---

## 5. Cryptographic Protection and Security Protocol (Release v0.2+)
In the absence of centralized authentication servers, generating a shared secret key "over the air" is implemented via a hybrid scheme:

1. **Shared Secret Agreement (`Curve25519`):** Performed via Elliptic-Curve Diffie-Hellman (ECDH) using the ultra-compact **Curve25519** curve. Each user generates a private/public key pair. Public keys (**32 bytes**) are exchanged directly in a secure environment by scanning QR codes on smartphone screens. Multiplying one's private key by the peer's public key yields an identical session secret on both devices.
2. **Symmetric Encryption:** Once the shared secret is derived, the token stream is encrypted using fast stream ciphers — **ChaCha20** or **AES-256 in Counter Mode (CTR)**. Their key advantage is **0-byte size overhead** (output length equals input length byte-for-byte), eliminating padding. Local message history and keys on the smartphone are stored in an encrypted **SQLCipher** database.

### Symmetric Cipher Comparison for LoRa/Mesh Networks

| Criterion / Algorithm | AES-256 (CTR Mode) | ChaCha20 | Kuznyechik (GOST R 34.12-2015) |
| :--- | :--- | :--- | :--- |
| **Cipher Type** | Block (in stream mode) | Stream cipher | Block cipher |
| **Key Length** | 256-bit | 256-bit | 256-bit |
| **Output Size** | Equal to input (0-byte overhead) | Equal to input (0-byte overhead) | Multiple of 16 bytes (requires padding) |
| **Integration Complexity** | Low (native in Android OS) | Low (lightweight libraries available) | High (requires third-party SDK) |
| **Key Features** | Hardware-accelerated on most modern CPUs; high cryptographic strength. | Optimized for software execution; maximum speed on mobile CPUs/MCUs without hardware AES. | 128-bit block size; requires mandatory state certification for government/defense use in Russia. |

---

## 6. Development Roadmap
* **Phase 1: Theoretical Design & Mathematical Modeling** — Defining architecture, dictionary capacities (`Base32` / `Base36` / 24-bit binary), frame structure, and Protobuf specs.
* **Phase 2: Hardware Testbench & Radio Validation** — Configuring two ESP32+LoRa boards (LilyGO T-Beam / Heltec V3, 868/433 MHz) with official Meshtastic firmware and testing raw BLE $\leftrightarrow$ LoRa packet delivery.
* **Phase 3 (MVP v0.1): Compression Engine & Core UI** — Implementing Kotlin tokenizer/decompressor, integrating `ru.bin` / `en.bin` dictionaries, building chat UI, and releasing `.apk` using Meshtastic's native AES-256 channel encryption.
* **Phase 4 (Release v0.2): Application-Layer E2EE** — Integrating Bouncy Castle (`Curve25519` ECDH + `ChaCha20` / `AES-256-CTR`), QR key pairing UI, and `SQLCipher` local storage.
* **Phase 5: Open-Source Release** — Publishing signed `.apk` builds on **GitHub Releases** and **F-Droid**.

---

## 7. Legal Status and Open-Source Governance
* **Donationware Model:** Distributed free of charge as open-source mathematical software supported by voluntary user donations.
* **GNU GPL v3 (`LICENSE`):** Enforces strict *copyleft*. Any modifications or forks must remain open-source under GPLv3; proprietary commercial integration without author consent is prohibited.
* **Contributor License Agreement (`CLA.md`):** Contributors assign exclusive intellectual property rights for submitted Pull Requests to the repository author, ensuring unified copyright ownership and future licensing flexibility.
