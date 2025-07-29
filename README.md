
# **VX Protocol** – Value Transport Layer

**VX (Value Transport Layer)** is a **blockchain-native routing protocol** that moves value and state the way TCP/IP moves data.
Instead of bridges or wrapped tokens, VX routes transactions as packets, verifies them via decentralized finalizers, and snapshots truth in a global ledger.

> **TL;DR**: VX is **TCP/IP for value** – cross-chain, stateless, modular, and secure.

---

## **🔹 Why VX?**

Blockchains today work in isolated silos. Cross-chain solutions rely on:

* Wrapped tokens and trusted multisigs (fragile, opaque)
* Centralized sequencers and “trusted oracles”
* Heavy global state sync that doesn’t scale

VX takes a different approach:

* **Packet-based transactions:** Each transfer is decomposed into independently routed packets (like internet data packets)
* **Stateless routers:** Routers forward packets without holding ledger state
* **Global ledger snapshots:** Miners finalize state changes and update a single truth ledger
* **Cross-chain handshake:** Every participating chain’s nodes verify their own side

---

## **🔹 Key Concepts**

### 1️⃣ Global Ledger

* VX maintains a **global ledger** tracking:

  * Which chain holds what assets
  * Transfers in/out
  * Balances for every asset
* Think of it as the **accountant** – simple and compact, not a full transaction history.

### 2️⃣ Blocks as Snapshots

* VX blocks aren’t like Ethereum blocks; they’re **time-based snapshots**.
* They record state updates (ledger diffs) from the last period, not full transaction lists.

### 3️⃣ Packets

* Transactions are split into **packets** (max size: 8192 bytes).
* Each packet contains:

  * Source and destination VX address
  * Asset and amount
  * Routing metadata
  * Onion-wrapped signature trail

### 4️⃣ Onion Routing + Stateless Routers

* Routers don’t maintain ledger state; they:

  1. Verify checksum and signature integrity
  2. Soft-query nearby miners for nonce validation
  3. Forward the packet to the **next hop** only
* Each transaction takes **20 random hops** for security and privacy.

### 5️⃣ Beacons & Chain Integration

* Chains run **lightweight Beacons**:

  * Broadcast gas prices and chain state
  * Relay confirmation events to VX ledger
  * Are “pinged” by routers to verify availability

### 6️⃣ Triple Handshake Finality

1. **Source chain node** signals intent
2. **Destination chain node** preps to receive
3. **VX ledger** confirms transfer with finalizers

Only then is the transfer finalized in the global ledger.

---

## **🔹 VX Addressing (IP/MAC Hybrid)**

* VX uses **64-bit addresses**, designed for speed and routing awareness:

  * **48 bits:** MAC-style unique identifier
  * **8 bits:** Node role (router, miner, beacon)
  * **8 bits:** Checksum or version ID

This compact format allows:

* Leaner packets (smaller headers)
* Region/shard-aware routing (like IPv6)
* Compatibility with embedded devices (e.g., Raspberry Pi routers)

---

## **🔹 Node Roles**

### **Routers (stateless)**

* Forward packets without holding global state
* Use routing tables and onion logic for blind delivery
* Query Miners for soft nonce checks
* Anyone can run one with minimal hardware

### **Miners (finalizers)**

* Maintain the global VX ledger
* Aggregate packet trails and finalize state diffs into blocks
* Provide nonce and signature validation to routers
* Earn protocol fees

### **Beacons**

* Chain-specific lightweight relays
* Sync chain state (gas prices, confirmations)
* Validate chain activity for VX ledger
* Routers ping Beacons for availability checks

---

## **🔹 Security Model**

* **20-hop routing:** Probability of an attacker controlling all hops is negligible
* **Onion encryption:** Routers only know previous and next hop
* **Nonce-based validation:** Double-spends are rejected at finalizer stage
* **Reputation & fallback:** Routers that drop/tamper with packets lose reputation and are bypassed

---

## **🔹 Current Status: MVP**

We’re building the first **Proof of Concept**:

* **Source:** Ethereum Sepolia (or Solana Devnet)
* **Destination:** Sui Testnet
* VX relayer will:

  1. Accept deposits on source chain
  2. Route the “packet” through simulated hops
  3. Mint equivalent VX tokens on destination chain
  4. Update the VX ledger snapshot

---

## **🛠️ Getting Started**

### Clone the repository:

```bash
git clone https://github.com/YOUR-ORG/vx-protocol.git
cd vx-protocol
```

### Install dependencies:

```bash
# Backend (Go)
go mod tidy

# Frontend / scripts (Node)
npm install
```

### Environment setup:

```bash
cp .env.example .env
# Fill in RPC keys (Ethereum, Sui, Supabase, Fly.io)
```

### Run the relayer:

```bash
go run ./relayer
```

### Run the UI (optional):

```bash
npm run dev
```

---

## **🔹 Roadmap**

* [ ] Define packet spec
* [ ] Ledger schema (Supabase)
* [ ] Sepolia + Sui testnet contracts
* [ ] Fly.io relayer MVP
* [ ] Public demo UI
* [ ] Multi-chain Beacon integration
* [ ] Router reputation system

---

## **🔹 Contributing**

VX is 100% open source and welcomes contributions:

1. Fork the repo
2. Create a branch (`feature/my-feature`)
3. Submit a pull request

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## **🔹 License**

Licensed under the [Apache 2.0 License](LICENSE).

---

## **🔹 FAQ**

### 1. Are stateless routers safe?

Yes – they cannot forge or reorder packets, only pass them.
Packets are onion-encrypted and checked by multiple hops before reaching miners.

### 1.5. Why Routers?

Routers allow VX to skip unnecessary internal transactions and streamline packetflow.
 - Each packet specifies whether the transfer is INTERNAL (within the VX chain) or EXTERNAL (cross-chain).
 - INTERNAL:
    - Routers can process and resolve the transfer without involving the full cross-chain handshake.
    - this prevents bloating the ledger with redundant entries/
 - EXTERNAL:
    - Routers route the packet across chains using the standard 20-hop flow and triple handshake.

### 2. What stops double-spends?

Nonce checks and finalizer confirmation. Only finalized packets update the global ledger.

### 3. Can VX be censored?

No – packets automatically reroute if a router drops them, and finalizers cannot reject valid packets.

### 4. How is VX different from bridges?

VX is a **protocol** not a bridge: no wrapped tokens, no central trusted oracles, no fragile multisigs.

See [full FAQ](docs/FAQ.md) for more.

---

## **🔹 Links**

* [Whitepaper (WIP)](docs/WHITEPAPER.md)
* [VX Packet Spec](docs/PACKET_SPEC.md)
* [Ledger Schema](docs/LEDGER_SCHEMA.md)

---

### **VX is TCP/IP for value. Build with us.**

---

