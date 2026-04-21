# LoRaWAN-Ready-for-BLE-integration
<p align="center">
  <img src="https://img.shields.io/badge/LoRaWAN-Protocol-6236FF?style=for-the-badge" />
  <img src="https://img.shields.io/badge/C%2FC%2B%2B-Embedded-00599C?style=for-the-badge&logo=cplusplus&logoColor=white" />
  <img src="https://img.shields.io/badge/LoRa-PHY%20Layer-EF4444?style=for-the-badge" />
  <img src="https://img.shields.io/badge/AES--128-Encryption-10B981?style=for-the-badge" />
  <img src="https://img.shields.io/badge/IEEE-EUI64-F59E0B?style=for-the-badge" />
</p>

# 📡 LoRaWAN End-Device Implementation

> A **C/C++ implementation** of the LoRaWAN MAC layer for end-devices — supporting OTAA (Over-the-Air Activation), secure key management, multi-channel communication, and reliable message transmission with automatic retransmission.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Features](#-features)
- [API Reference](#-api-reference)
  - [Configuration Setters](#configuration-setters)
  - [Initialization](#initialization)
  - [Join Procedure (OTAA)](#join-procedure-otaa)
  - [Message Transmission](#message-transmission)
- [Join Flow (OTAA)](#-join-flow-otaa)
- [Security Model](#-security-model)
- [Key Identifiers](#-key-identifiers)

---

## 🔭 Overview

This project implements the **LoRaWAN MAC layer** for end-devices, following the LoRaWAN specification. It handles the complete lifecycle of a LoRaWAN device:

1. **Configuration** — Set device identifiers (DevEUI, JoinEUI) and cryptographic keys
2. **Initialization** — Configure radio parameters (frequency, power, data rate, code rate)
3. **Network Join** — OTAA join procedure via Join-Request / Join-Accept exchange
4. **Data Transmission** — Reliable uplink messaging with automatic channel selection and retransmission

The implementation sits on top of a **LoRaPhy** (physical layer) driver and communicates with a LoRaWAN Network Server.

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│              (User code / Sensor data)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  LoRaWAN MAC LAYER                       │
│                  (This Implementation)                   │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐  │
│  │  Key     │  │  Join    │  │  Message Transmit     │  │
│  │  Mgmt    │  │  (OTAA)  │  │  + Channel Selection  │  │
│  │          │  │          │  │  + Retransmission      │  │
│  └──────────┘  └──────────┘  └───────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────┐       │
│  │  Security: AES-128 Encryption + MIC (CMAC)   │       │
│  └──────────────────────────────────────────────┘       │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  LoRa PHYSICAL LAYER                     │
│           (SX1276/SX1278 radio driver)                   │
│                                                         │
│  Frequency · Spreading Factor · Bandwidth · TX Power     │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
                   ┌─────────┐
                   │   RF    │
                   │ (Radio) │
                   └─────────┘
```

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🔑 **OTAA Activation** | Full Over-the-Air Activation with Join-Request / Join-Accept |
| 🔐 **AES-128 Security** | AppKey, NwkKey, AppSKey, NetSKey, JSIntKey, JSEncKey management |
| 📻 **Multi-Channel** | 16-channel support with Channel Frequency List (CFList) |
| 🔄 **Auto Retransmission** | Automatic retry with fallback frequency and data rate |
| 📊 **Adaptive Data Rate** | Upstream/downstream data rate management with DLSettings |
| 🛡️ **Message Integrity** | MIC (Message Integrity Code) calculation on every frame |
| 🎲 **Random Channel Selection** | Channel randomization for load balancing |
| 🔁 **Replay Protection** | DevNonce counter prevents replay attacks |

---

## 📖 API Reference

### Configuration Setters

These functions configure the device identifiers and cryptographic keys before joining a network.

#### Device Identifiers

| Function | Parameters | Description |
|----------|-----------|-------------|
| `setDevAddr(d0, d1, d2, d3)` | 4 × `uint8_t` | Sets the 4-byte device address (DevAddr) |
| `setNetID(d0, d1, d2)` | 3 × `uint8_t` | Sets the 3-byte Network Server identifier (NetID) |
| `setDevEUI(a0..a7)` | 8 × `uint8_t` | Sets the globally unique end-device ID (IEEE EUI64) |
| `setJoinEUI(a0..a7)` | 8 × `uint8_t` | Sets the Join Server identifier (IEEE EUI64) |

#### Cryptographic Keys

| Function | Parameters | Description |
|----------|-----------|-------------|
| `setappKey(n0..n15)` | 16 × `uint8_t` | Sets the **AppKey** — encrypts join requests/responses |
| `setnwkKey(n0..n15)` | 16 × `uint8_t` | Sets the **NwkKey** — secures device-to-network communication |
| `setNetSKey(n0..n15)` | 16 × `uint8_t` | Sets the **NetSKey** — network session key for MAC commands |
| `setAppSKey(a0..a15)` | 16 × `uint8_t` | Sets the **AppSKey** — application session key for payload encryption |

#### Communication Parameters

| Function | Parameters | Description |
|----------|-----------|-------------|
| `setCFList(a0..a15)` | 16 × `uint8_t` | Sets channel frequencies from the Channel Frequency List |
| `setDownStreamDataRate1()` | — | Computes RX1 data rate from upstream DR and DLSettings |
| `setDownStreamDataRate2()` | — | Extracts RX2 data rate from the lower nibble of DLSettings |

---

### Initialization

```cpp
int16_t init();
```

Initializes the LoRa radio and derives internal security keys:

| Action | Detail |
|--------|--------|
| Sets radio frequency | Default carrier frequency |
| Sets TX power | Transmission power level |
| Sets data rate | Spreading factor + bandwidth |
| Sets code rate | Forward error correction ratio |
| Derives **JSIntKey** | Used to MIC Rejoin-Request type 1 messages and Join-Accept answers |
| Derives **JSEncKey** | Used to encrypt Join-Accept triggered by a Rejoin-Request |

**Returns:** `int16_t` status code (0 = success)

---

### Join Procedure (OTAA)

#### `join_request(uint8_t* const mpduPayload)` → `uint8_t`

Constructs a Join-Request MAC frame:

1. Sets the MHDR MType to **Join-Request**
2. Fills the frame with **AppEUI**, **JoinEUI**, and **DevEUI**
3. Appends the **DevNonce** (monotonically incrementing counter — prevents replay attacks)
4. Calculates and appends the **MIC** (Message Integrity Code)

**Returns:** Length of the constructed MPDU payload.

#### `join()` → `bool`

Executes the full OTAA join sequence:

```
┌─────────────┐                          ┌─────────────┐
│  End-Device │                          │   Network   │
│             │                          │   Server    │
└──────┬──────┘                          └──────┬──────┘
       │                                        │
       │  1. defaultSetting()                   │
       │     (reset channels, DR ranges,        │
       │      frequencies, rxParamSetup)        │
       │                                        │
       │  2. Build Join-Request                 │
       │     [AppEUI | JoinEUI | DevEUI |       │
       │      DevNonce | MIC]                   │
       │                                        │
       │──── TX on random channel (1 of 16) ───▶│
       │                                        │
       │  3. Switch to RX1                      │
       │     setDownStreamDataRate1()           │
       │     Set DL frequency for channel       │
       │                                        │
       │◀──────── Join-Accept (RX1) ────────────│
       │     ✅ return true                     │
       │                                        │
       │  — OR if RX1 timeout —                 │
       │                                        │
       │  4. Switch to RX2                      │
       │     setDownStreamDataRate2()           │
       │     Set LORA_FREQUENCY2                │
       │                                        │
       │◀──────── Join-Accept (RX2) ────────────│
       │     ✅ return true                     │
       │                                        │
```

**Returns:** `true` if the device successfully joined the network.

---

### Message Transmission

```cpp
void send_message(uint8_t* const message, uint8_t len);
```

Sends an uplink data message from the end-device to the server:

| Step | Action |
|------|--------|
| 1 | Select a **random channel** from the 16 available |
| 2 | Verify channel **availability** |
| 3 | Set **transmission frequency** and **data rate** for the selected channel |
| 4 | **Transmit** the message via LoRaPhy |
| 5 | Wait for server response within the **RX delay** window |
| 6 | If no ACK in RX1: adjust to **downstream DR2** and **fallback frequency** |
| 7 | **Retry** transmission up to the maximum retransmission count |

**Parameters:**
- `message` — Pointer to the payload buffer
- `len` — Length of the payload in bytes

---

## 🔄 Join Flow (OTAA)

```mermaid
sequenceDiagram
    participant ED as End-Device
    participant NS as Network Server

    Note over ED: Reset to default settings
    ED->>ED: defaultSetting()
    ED->>ED: Build Join-Request frame
    Note over ED: AppEUI + JoinEUI + DevEUI + DevNonce + MIC

    ED->>NS: Join-Request (random channel)

    alt RX1 Window
        NS-->>ED: Join-Accept
        Note over ED: ✅ Joined
    else RX1 Timeout → RX2 Window
        ED->>ED: setDownStreamDataRate2()
        ED->>ED: Switch to LORA_FREQUENCY2
        NS-->>ED: Join-Accept
        Note over ED: ✅ Joined
    end
```

---

## 🔐 Security Model

The implementation uses **AES-128** based security with multiple key layers:

```
Root Keys (Pre-provisioned)
├── AppKey ──── Used for Join-Request/Accept encryption
└── NwkKey ──── Used for device-to-network security
    ├── JSIntKey ── Derived at init() → MIC for Rejoin-Request type 1
    └── JSEncKey ── Derived at init() → Encrypt Join-Accept on Rejoin

Session Keys (Derived after Join)
├── AppSKey ── Encrypts application payload (end-to-end)
└── NetSKey ── Secures MAC commands (hop-by-hop)
```

| Key | Size | Purpose |
|-----|------|---------|
| **AppKey** | 128-bit | Encrypts join requests and responses |
| **NwkKey** | 128-bit | Secures communication between device and network server |
| **JSIntKey** | 128-bit | MIC calculation for Rejoin-Request type 1 and Join-Accept |
| **JSEncKey** | 128-bit | Encrypts Join-Accept triggered by Rejoin-Request |
| **AppSKey** | 128-bit | Application session key — encrypts application data |
| **NetSKey** | 128-bit | Network session key — secures MAC layer commands |

---

## 🏷 Key Identifiers

| Identifier | Format | Description |
|------------|--------|-------------|
| **DevEUI** | IEEE EUI64 (8 bytes) | Globally unique end-device ID; printed on device labels |
| **JoinEUI** | IEEE EUI64 (8 bytes) | Identifies the Join Server that processes join procedures |
| **AppEUI** | IEEE EUI64 (8 bytes) | Identifies the application entity processing JoinReq frames |
| **DevAddr** | 4 bytes | Short device address assigned by the network |
| **NetID** | 3 bytes | Network Server's unique identifier |
| **DevNonce** | Counter | Starts at 0; incremented per join — prevents replay attacks |
| **CFList** | 16 bytes | Channel Frequency List — configures up/downlink frequencies |

---

## 🛠 Tech Stack

| Component | Technology |
|-----------|------------|
| **Language** | C / C++ (Embedded) |
| **Protocol** | LoRaWAN 1.1 |
| **Physical Layer** | LoRa (via LoRaPhy driver) |
| **Encryption** | AES-128 (CMAC for MIC) |
| **Addressing** | IEEE EUI64 |
| **Channels** | 16 configurable frequencies |

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

<p align="center">
  Built with 📡 for IoT and LPWAN Communication
</p>
