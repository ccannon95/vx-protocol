# **Beacons – Thoughts & Logic**

---

## **Why Beacons?**

Beacons are intended to act as **network observers and chain relays**. They watch external chains for relevant activity (e.g., treasury transactions, confirmations, gas price updates) and report those events to the VX Ledger.

The key idea is to keep **routers stateless and lightweight**, while Beacons perform the deeper integration work needed for cross-chain activity.

---

## **Beacon Roles**

1. **Watch external chains:**

   * Run alongside full nodes (ETH, Solana, Sui, etc.).
   * Monitor blocks, transaction logs, and multi-sig treasury addresses.
   * Track chain-level data like finality, gas pricing, and congestion.

2. **Push signed data to VX Ledger:**

   * Report treasury deposits/withdrawals, confirmation status, and metrics.
   * Each push is signed by the Beacon to prove authenticity.
   * Ledger updates require quorum from multiple Beacons (to prevent a single compromised Beacon from affecting state).

3. **Coordinate handshakes:**

   * For external transfers, Beacons must communicate with:

     * Other Beacons watching the same chain.
     * Beacons on the destination chain.
     * The VX Ledger for final confirmation.
   * They also **sign mint/burn transactions using the group’s multi-sig treasury key** to ensure no single Beacon can execute unilaterally.

4. **Periodic heartbeats:**

   * Beacons should also send “I’m alive” signed heartbeats and metrics to the VX Ledger so it can prune stale members and maintain quorum integrity.

---

## **Beacon Groups and Multi-Sig Wallets**

Each chain integrated with VX will likely require a **Beacon group**:

* Each Beacon in the group shares responsibility for watching the chain.
* Beacon groups should be **small but diverse** (probably 8–9 Beacons maximum per chain).
* Each Beacon group controls a **multi-sig treasury wallet** on that chain:

  * Required for holding VX-wrapped assets (e.g., VX-USDC).
  * Multi-sig ensures no single Beacon can move funds without quorum.

---

## **Handshake Flow for Cross-Chain Transfers**

A cross-chain transfer requires a full handshake between:

1. The **origin chain Beacon group**.
2. The **destination chain Beacon group**.
3. The **VX Ledger** (final state authority).

> **Note:** Beacons on both chains should only move forward once the origin deposit has passed sufficient block confirmations (e.g., 15+ confirmations on ETH mainnet) to avoid reorg risk.

### **Steps**

1. **Origin chain detects deposit:**

   * Beacons watching Chain A detect a user deposit to the multi-sig treasury.
   * Beacon group confirms the deposit is finalized after N block confirmations.
   * Group reaches quorum and signs a message:

     ```
     YES: Funds received on Chain A
     ```

2. **Handshake with destination Beacons:**

   * Beacons from Chain A forward the confirmation to Beacons watching Chain B.
   * Chain B Beacons verify that the deposit was correctly signed by the origin group.

3. **Destination chain mints/sends:**

   * Once verified, the Chain B Beacon group mints VX-wrapped tokens (or sends assets) to the user’s destination address **using the group’s multi-sig treasury key**.
   * Chain B group waits for confirmation that this transaction was included on-chain.

4. **Final confirmation:**

   * When the destination transaction finalizes, the Chain B Beacon group returns:

     ```
     YES: Funds delivered on Chain B
     ```
   * This confirmation is sent to the VX Ledger.

5. **VX Ledger snapshot update:**

   * Upon receiving confirmation from both Beacon groups, the VX Ledger finalizers update the global ledger:

     * Debit assets from Chain A’s treasury balance.
     * Credit assets to Chain B’s treasury balance.
   * Ledger snapshot is completed.

---

## **Ledger Update Requirements**

* **Quorum:** At least *M of N* Beacons (e.g., ⅔+) from each group must sign off on origin and destination confirmations.
* **Atomicity:** Ledger update occurs only after both sides confirm.
* **Auditable:** All Beacon signatures and messages should be published so that any node can verify the handshake trail.

---

## **Beacon Push Messages (VX Ledger)**

Each Beacon push should be a structured, signed message:

```json
{
  "beacon_id": "vx-eth-1",
  "chain_id": "1",
  "block_number": 18590232,
  "event": "TreasuryDeposit",
  "amount": "100000000",
  "tx_hash": "0xabcd...",
  "timestamp": 1723482932,
  "signature": "0xSIG..."
}
```

VX Ledger finalizers aggregate pushes from multiple Beacons. Ledger updates only occur when quorum is reached for both sides of a handshake.

---

## **Additional Considerations**

1. **Beacon Diversity:** Groups should be composed of Beacons from different operators, geographies, and infrastructure providers.
2. **Quorum Configuration:** The “max 8–9 Beacons per chain” assumption is to keep quorum manageable while maintaining security.
3. **Failure Handling:** If one or more Beacons in a group go offline, as long as quorum remains, operations continue.
4. **Metrics Collection:** Beacons can also push gas price data, block times, and congestion scores for use in the VX randomizer and fee logic.
5. **Beacon Heartbeats:** Beacons should periodically send “I’m alive” signed heartbeats so VX Ledger can prune stale members and maintain quorum integrity.

---

## **Auditability**

Any node should be able to independently verify the entire handshake trail:

1. Recompute the **handshake\_id** from the origin deposit event.
2. Check all Beacon signatures on the origin and destination attestations meet quorum.
3. Verify the destination chain’s contract shows `used[handshake_id] = true`.
4. Compare the VX Ledger COMMIT snapshot to ensure it matches the final debits and credits.

This makes the VX system fully auditable and resistant to tampering, even without trusting the VX Ledger directly.

---

## **Future Questions**

1. How should Beacons join or leave a group without breaking the multi-sig?
2. Can the handshake be streamlined further (e.g., asynchronous ledger updates with reconciliation)?
3. Do we allow corporate operators (e.g., Circle) to run their own Beacons with direct treasury authority?
4. Should Beacons also provide periodic "heartbeats" to ensure ledger has fresh chain data even without user transfers?


---

# Additional needs: **hard idempotency** across the whole flow so the same transfer can’t be executed (or even *prepared*) twice. 

---

## 0) One canonical **Handshake ID**

Define a **handshake\_id** that *everyone* uses for the same transfer:

```
handshake_id = H(
  origin_chain_id        ||
  origin_vault_address   ||
  origin_tx_hash         ||
  origin_log_index       ||   // or UTXO outpoint (txid:vout)
  depositor_address      ||
  asset_id               ||
  amount                 ||
  dest_chain_id          ||
  dest_address           ||
  epoch_id               ||
  "VX/HANDSHAKE/v1"
)
```

* For account‑based chains (ETH, Sui): use **tx\_hash + log\_index** to disambiguate multiple events in one tx.
* For UTXO chains (BTC): use **txid\:vout**.
* Everyone (beacons, routers, miners, destination contracts) must **carry this exact handshake\_id**.

---

## 1) Two‑phase (really three) state machine on the VX ledger

Implement a tiny, explicit FSM on the ledger with a **unique constraint** on `handshake_id`.

**States:** `NEW → PREPARED → EXECUTED → COMMITTED` (or `ABORTED`)

* **PREPARED:** quorum from **origin beacons** says “deposit finalized on origin.”
* **EXECUTED:** quorum from **destination beacons** says “mint/sends done on destination.”
* **COMMITTED:** miners finalize the ledger delta (snapshot).
* **ABORTED:** timed‑out or reorged‑out origin event.

**Ledger table (simplified):**

```
handshakes(
  handshake_id   PRIMARY KEY,
  origin_chain_id,
  origin_tx_hash,
  origin_log_index,
  dest_chain_id,
  dest_tx_hash,
  asset_id, amount,
  status ENUM('NEW','PREPARED','EXECUTED','COMMITTED','ABORTED'),
  prepared_quorum_root,   // aggregate signature root
  executed_quorum_root,
  created_at, updated_at,
  expiry_ts
)
```

> The **PRIMARY KEY** guarantees only one row per handshake\_id can ever exist. All writes are idempotent.

---

## 2) Beacon attestations are **idempotent and canonical**

A beacon’s message is a signature over a **canonical digest**:

```
attest_digest = H(
  "VX/ATTEST/v1" ||
  handshake_id   ||
  phase          ||   // PREPARE or EXECUTE
  origin_block_hash / dest_block_hash ||
  confirmations  ||   // depth waited
  dest_tx_hash?       // only in EXECUTE
)
```

* Ledger **only counts** attestations whose `attest_digest` matches exactly.
* If a beacon signs conflicting digests for the same `(handshake_id, phase)`, it’s **slashable** (or kicked).

Quorum example: **M‑of‑N** (e.g., 5 of 8 beacons) per side.

---

## 3) Destination contract must also be idempotent

At the execution chain (e.g., Sui Move / Solidity):

* Maintain `used[handshake_id] = true` on first mint/transfer.
* `require(!used[handshake_id])` before mint; set it, then mint (checks‑effects‑interactions).
* This protects against any off‑chain duplication: the chain itself refuses a second “execute”.

---

## 4) Packet‑level dedupe (optional but helpful)

At the routing layer:

* Each router keeps an **LRU Bloom filter** per epoch of seen `packet_id`s and `handshake_id`s.
* Duplicates are dropped early; packets still reaching a router again are forwarded only if TTL/backups require it.

---

## 5) Locks + retries without double‑exec

When the first **PREPARE quorum** arrives:

* The ledger **atomically**:

  * Upserts `handshake_id` with `status=PREPARED` (if NEW), else no‑op.
  * Stores the `prepared_quorum_root`.

Any other PREPARE pushes for the **same digest** are **no‑ops** (idempotent).
If a different digest appears → mark **CONFLICT**, alert, and ignore (or slash).

When **EXECUTE quorum** arrives:

* The ledger checks `status == PREPARED` and `dest_tx_hash` hasn’t been set.
* Atomically sets `status=EXECUTED`, stamps `dest_tx_hash`.

Finalizers then write the **COMMIT** snapshot (ledger delta) → idempotent as well.

---

## 6) Reorg handling (origin/destination)

* Beacons include **block number + block hash + confirmations** in their attestations.
* You only PREPARE after N confirmations (e.g., 15 on Sepolia).
* If a deep reorg invalidates the origin deposit *before COMMIT*, ledger moves `PREPARED → ABORTED`.
* If destination reorgs after EXECUTE but before COMMIT, request **re‑EXECUTE attestation** for the corrected `dest_tx_hash`.

---

## 7) Timeouts & garbage collection

* Include `expiry_ts` in the ledger record when NEW is created.
* If not PREPARED by expiry → delete or mark ABORTED.
* If PREPARED but not EXECUTED by expiry → unlock origin if policy allows (human or automated).
* GC old `handshake_id`s into an archive table to keep the hot set small.

---

## 8) Slashing / membership penalties (possibly)

* A beacon that signs **two different PREPARE digests** for the same handshake\_id → **slash** (or eject).
* Same for EXECUTE conflicts.
* Keep a public **misbehavior log** so operators know who’s untrustworthy.

---

## 9) Minimal code patterns (idempotency everywhere)

**Ledger upsert (pseudocode):**

```sql
INSERT INTO handshakes (handshake_id, status, created_at, updated_at, expiry_ts, ...)
VALUES ($id, 'NEW', now(), now(), now()+interval '2 hours', ...)
ON CONFLICT (handshake_id) DO NOTHING;
```

**Transition with guard:**

```sql
UPDATE handshakes
SET status='PREPARED', prepared_quorum_root=$root, updated_at=now()
WHERE handshake_id=$id AND status IN ('NEW','PREPARED') 
  AND prepared_quorum_root IS DISTINCT FROM $root;
-- Check rowcount; if 0 and root matches, treat as no-op; if different, mark conflict.
```

**Destination contract (Solidity sketch):**

```solidity
mapping(bytes32 => bool) public used;

function mint(bytes32 handshakeId, address to, uint256 amount, bytes calldata proof) external onlyRole(EXECUTOR) {
    require(!used[handshakeId], "already-executed");
    // verify proof binds (handshakeId, amount, to)
    used[handshakeId] = true;
    _mint(to, amount);
}
```

**Sui Move sketch:**

```move
struct Used: has_key { ids: Table<vector<u8>, bool> }

public entry fun execute(handshake_id: vector<u8>, to: address, amount: u64, ctx: &mut TxContext) {
    assert!(!Table::contains(&used.ids, &handshake_id), E_ALREADY_EXECUTED);
    // verify proof...
    Table::insert(&mut used.ids, handshake_id, true);
    coin::mint_to_address<VX>(amount, to);
}
```

---

## 10) Summary: where duplication should be killed

* **Naming:** deterministic `handshake_id` (derived from origin event).
* **Ledger:** unique key + FSM transitions are **atomic & idempotent**.
* **Beacons:** sign canonical digest; conflicting signatures are punishable.
* **Destination contract:** `used[handshake_id]` guard prevents double execution even if off‑chain fails.
* **Routers:** LRU/Bloom dedupe reduces noise but is not relied on for correctness.



