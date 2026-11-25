🌐 **Language:** [English](README.md) | [Русский](README.ru.md)

# 🐦 m-kolibri

A decentralized Android messenger featuring deep dictionary compression (**"Kolibri-code"**) designed to operate over **LoRa / Meshtastic** mesh networks without the internet, cellular infrastructure, or central servers.

## 🎯 About the Project
Standard LoRa packets in the Meshtastic ecosystem are strictly limited in size (~200–240 bytes), making long UTF-8 text messages (especially in Cyrillic) inefficient. **m-kolibri** solves this bottleneck at the Android application layer:
* **Works with off-the-shelf hardware:** Connects via Bluetooth (BLE) to standard boards (LilyGO T-Beam, Heltec V3, etc.) running official Meshtastic firmware through `PortNum.PRIVATE_APP` — no custom firmware flashing required.
* **True P2P & Offline:** Zero servers, no sign-ups, and no phone numbers.
* **Modular Dictionaries:** Language matrices (`ru.bin`, `en.bin`) are downloaded separately, keeping the core APK lightweight.

## ⚙️ Architecture & Core Technologies

### 1. "Kolibri-code" Dictionary Compression
Instead of transmitting raw character-by-character UTF-8 strings, text is tokenized into fixed **3-byte indices** using static codebooks (fitting up to ~60–65 words into a single LoRa frame):
* **Russian Module (Two-tier binary code, 24-bit):** 2 bytes for word stems (65,536 lemmas) + 1 byte for inflections and endings (256 grammatical modifiers) synthesized on the receiver's device.
* **English Module (`Base32` / `Base36`):** 3-character tokens supporting 32,768 (`Base32`) or 46,656 (`Base36`) word forms.
* **Fallback Mechanism:** Out-of-vocabulary words are transmitted character-by-character using a dedicated control prefix.

### 2. Two-Stage Security Architecture
* **Stage 1 (MVP v0.1 — Native Channel Security):** Relies on Meshtastic's built-in **AES-256-CTR** channel encryption over the air, allowing maximum payload space for compressed words and rapid field testing.
* **Stage 2 (Release v0.2 — App-Layer E2EE):** Adds an independent peer-to-peer encryption layer on top of the mesh channel using **Curve25519 (ECDH)** key exchange (via QR codes) and **ChaCha20 / AES-256-CTR** stream ciphers (0-byte padding overhead), paired with local **SQLCipher** database encryption.

### 3. Payload Frame Structure (Full E2EE Spec)

| Frame Field | Size | Purpose in P2P Mesh |
| :--- | :---: | :--- |
| `Flags / Lang ID` | 1 byte | Protocol version, encryption mode flag, and dictionary ID |
| `Sender ID` | 4 bytes | Local unique address of the source node |
| `Receiver ID` | 4 bytes | Target node address (`0xFFFFFFFF` for broadcast) |
| `Nonce` *(v0.2+)* | 8 bytes | Packet counter for Replay-attack protection |
| `Compressed Payload` | up to 183 bytes | Stream of 3-byte word tokens (up to 61 words per frame) |

## 🗺 Roadmap
- [x] **Phase 1:** Mathematical dictionary modeling, frame specification, and Protobuf integration design.
- [ ] **Phase 2:** Hardware testbench setup (Heltec V3 / LilyGO T-Beam) and BLE $\leftrightarrow$ LoRa transport validation.
- [ ] **Phase 3 (MVP v0.1):** Kotlin tokenization/decompression engine, UI chat screen, and `.apk` release (using native Meshtastic AES-256 channel encryption).
- [ ] **Phase 4 (Release v0.2):** Application-layer E2EE integration (Bouncy Castle: Curve25519 + ChaCha20), QR key exchange UI, and SQLCipher local storage.
- [ ] **Phase 5:** Distribution via **GitHub Releases** and **F-Droid**.

## 📚 Documentation
* 📖 **[Technical Specification — English (Markdown)](docs/SPECIFICATION.md)**
* 📖 **[Технический проект — Русский (Markdown)](docs/SPECIFICATION.ru.md)**
* 📄 **[Download Original Specification (PDF, RU)](docs/m-kolibri_specification.pdf)**

## 🛠 Tech Stack
* **Language & Platform:** Kotlin, Android SDK
* **Protocols:** Meshtastic Android API, Google Protocol Buffers (Protobuf), BLE / Base64
* **Cryptography & Storage (v0.2):** Bouncy Castle, SQLCipher

## 🤝 Contributing & CLA
Contributions are welcome — especially in building and optimizing binary dictionaries (`.bin`) for new languages! 
Please note that before submitting a Pull Request, you must agree to the terms of our Contributor License Agreement (`CLA.md` / `CLA.ru.md`).

## 💖 Support the Project (Donationware)
**m-kolibri** is free, open-source software developed independently. Voluntary donations help fund hardware testbeds and ongoing development:
* **TON / USDT (TON Network):** `YOUR_TON_WALLET_ADDRESS`

## 📄 License
This project (source code, binary dictionaries, and documentation) is licensed under the **GNU General Public License v3.0 (GPLv3)** — see the [LICENSE](LICENSE) file for details.
* Any forks or derivative works must remain open-source under GPLv3.
* Commercial/proprietary use without explicit written permission from the author is prohibited. For commercial licensing inquiries, please contact the repository owner.
