
# **Packets – Thoughts & Logic**

---

## **Why Packets?**

Okay, so: **IP addresses, TCP, routing logic – all of this already exists**. The entire internet moves trillions of data packets every single day without needing global state sync.

So, logically, if we’re trying to do the same thing for *value*, why shouldn’t we adopt the same idea?

* Instead of broadcasting entire transactions to every node (like Bitcoin/Ethereum), **just packetize it**.
* Routers forward the packet hop-by-hop (like IP routers do).
* Only at the “edges” (finalizers/miners) do we **snapshot the result** into the global ledger.

This cuts out the overhead of trying to make every participant store and verify everything.

---

## **How Do We Build VX Packets?**

### **Key Question:** What needs to go inside the header?

If we look at TCP/IP:

* IP packet header has **source/destination addresses**, TTL, checksum, etc.
* TCP header has **sequence numbers**, flags, and so on.

VX should be similar but value-focused. So here's a **draft VX packet header structure**:

```
VX Packet Header:
─────────────────────────────────────────────
1. Version             (1 byte)
2. Epoch ID            (4 bytes)
3. Packet ID (hash)    (32 bytes)
4. Route Commit (hash) (32 bytes)
5. Source VX Address   (8 bytes)
6. Destination VX Addr (8 bytes)
7. Packet Type         (1 byte)
   - 0x01 = Internal Transfer
   - 0x02 = External Transfer
   - 0x03 = Beacon Sync
   - 0x04 = Gas Oracle Update
8. TTL (hops left)     (1 byte)
9. Hop Index           (1 byte)
10. Backup Index       (1 byte)
11. Nonce              (8 bytes)
12. Header Checksum    (4 bytes)
13. Onion Blob         (variable)
─────────────────────────────────────────────
Payload (encrypted)
─────────────────────────────────────────────
```

### **Why Each Field Exists?**

* **Version:** Future-proofing. If we update packet spec, old nodes can still parse v1.
* **Epoch ID:** Ties packet to a specific randomness epoch (`R_e`) so route selection is auditable.
* **Packet ID:** Unique hash of the packet (header + payload).
* **Route Commit:** Commitment to the full path and backups (prevents sender from grinding routes).
* **Source/Destination Address:** 64-bit VX addresses (MAC-style).
* **Packet Type:** Is this just an internal transfer? Cross-chain? Beacon heartbeat?
* **TTL:** Decrements each hop; stops infinite loops.
* **Hop Index:** Which hop we’re at (0–19).
* **Backup Index:** Which backup path to try if hop fails.
* **Nonce:** Stops replay attacks.
* **Header Checksum:** Quick way to know if someone tampered.
* **Onion Blob:** Encrypted “next hop + MAC” layers, peeled off at each hop.

---

## **The Payload – What’s Inside?**

This depends on the packet type, but for a **transfer packet**:

```
Payload:
─────────────────────────────────────────────
1. Asset ID            (32 bytes)
2. Amount              (8 bytes)
3. Origin Chain        (2 bytes)
4. Destination Chain   (2 bytes)
5. Dest Chain Address  (32 bytes)
6. Expiry Timestamp    (8 bytes)
7. Signatures[]        (variable)
─────────────────────────────────────────────
```

* **Asset ID:** Could be ETH, USDC, VX coin, etc.
* **Amount:** Value being moved.
* **Origin/Dest Chains:** Which chains are involved.
* **Destination Chain Address:** Where final funds should land.
* **Expiry:** If packet isn’t finalized by this time, drop it.
* **Signatures:** Proofs from routers, finalizers, and/or multi-sig treasuries.

---

## **Internal vs External Packets**

We need a clear flag:

* **Internal (Type = 0x01):** Stays on the same chain. Routers can short-circuit it; finalizers on the same chain just update the ledger.
* **External (Type = 0x02):** Must route cross-chain through Beacons and multi-sig treasuries.

This matters because it changes how **heavy** the finality process is: internal doesn’t need to wake up the whole cross-chain handshake system.

---

## **API Routers & Multi-Sig Treasuries**

Here’s the logic:

* Each chain has a **multi-sig treasury** (think Circle’s mint/burn system).
* VX routers and beacons are the “API watchers”: they see if the treasury moved funds and feed that into the ledger.
* This way VX can **burn VX-USDC on Chain A** and **mint VX-USDC on Chain B** – fully backed, fully tracked in the ledger.

This also opens the door for **third parties (like Circle or Coinbase)** to plug in their own treasuries as “approved endpoints.”

---

## **Security Questions I Keep Coming Back To**

1. **Can a router tamper with the payload?**
   → No, onion encryption + header checksum should catch it.

2. **Can someone grind nonces to pick friendly routes?**
   → No, because route selection uses epoch VRF randomness + commitment.

3. **What if a hop goes offline?**
   → Use the backup index → pull next hop from the pre-selected backups.

4. **How does the finalizer know this packet was valid?**
   → Verify:

   * Nonce hasn’t been used
   * Route matches the commitment hash
   * Signatures line up
   * Asset was locked in the origin treasury

---

## **Future Work**

* Do we compress some of these fields to shrink the header? (Right now \~120 bytes just for header)
* Should we embed a Merkle proof of prior hops for extra auditability?
* Can we split large transfers into multiple packets (like TCP segmentation) and reassemble at the destination?
* Ensure packet-fall back also works if next hop router cannot be called. 

---

## **Implementation Thoughts**

1. MVP: Hard-code packet size (8kb max).
2. Use a **Supabase packet pool** as a holding queue. Routers poll every 4s to pick up packets.
3. Later: gossip-based pool (no central DB).
4. Onion wrapping = use Sphinx/Noise protocol library (don’t reinvent).
5. Validate packet structure at every hop to kill malformed packets early.

