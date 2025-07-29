
# **Routers – Thoughts & Logic**

---

## **Why Routers?**

Alright, so: **Routers are really the backbone of VX**. Without them, everything becomes slow and heavy because you’d need every node to know everything, all the time (like Ethereum/BTC full state).

But… **why lets just treat routers like IP routers**

* They don’t keep full ledger state.
* They just **pass packets along hop-by-hop**.
* They ask “does this make sense?” (soft validation) and move it forward.
* Only the finalizers/miners care about full validation and updating the global ledger.

This makes the system **fast and scalable** because routers aren’t weighed down by global state.

---

## **What Do Routers Actually Do?**

At a high level:

1. **Pick up packets** (poll from a packet pool or gossip network).
2. **Check the header**: is it malformed? is the TTL still > 0? does the checksum pass?
3. **Soft validate**: ask a nearby miner or beacon if the packet looks real (e.g. nonce hasn’t already been spent, asset exists, etc.).
4. **Peel the onion layer**: decrypt the next hop info from the Onion Blob.
5. **Forward**: push it to the next hop.

That’s it. No need to keep giant databases or track full balances.

---

## **What They Don’t Do**

* Routers **do not maintain the global ledger**.
* They don’t decide final validity (that’s for finalizers).
* They shouldn’t try to order or batch transactions.

If you think about it, this keeps them “dumb” and **less attackable**: all they’re doing is passing along packets.

---

## **How Do They Know Who to Forward To?**

The packet itself carries the routing logic:

* Each hop is pre-determined by the **Route Commit** and onion-encrypted.
* When a router peels the Onion Blob, it learns:

  1. Who it got the packet from (previous hop)
  2. Who to send it to (next hop)
* It doesn’t know the full path or the final destination.

This prevents routers from doing anything shady like trying to censor or collude—they just don’t know enough.

---

## **Soft Verification – How “Dumb” is Too Dumb?**

One thing I keep going back to:

* If routers are *too dumb* and don’t validate anything, they could just spam invalid packets forward.
* But if they’re *too smart* and do full validation, we’re back to the global state problem.

**Middle ground idea:**

* Routers do lightweight math checks: nonce format, checksum, signatures that can be verified quickly.
* For deeper checks (e.g. “has this nonce been spent?”), they **ask a beacon or nearby miner** before forwarding.
* This gives us a balance: routers don’t hold full state, but they’re not blind.

---

## **Routing Tables & Network Graph**

Routers should maintain a minimal “map” of the network:

* Ping other routers, beacons, and miners every few seconds.
* Track who’s online, who’s slow, who’s overloaded.
* This creates a simple routing table (like BGP or SNMP does for the internet).

Why? Because we can use it to:

* Re-route around congestion.
* Drop unresponsive nodes.
* Assign routes more intelligently in the randomizer.

---

## **What Happens if a Hop Fails?**

We’ve already got the **Backup Index** inside the packet header.

* If a router can’t reach the next hop (timeout, offline, whatever), it pulls the next backup from the pre-selected list and forwards it there.
* TTL decrements each hop, so packets don’t get stuck in infinite retry loops.

This gives us redundancy without having to re-route everything from scratch.

---

## **Internal vs External Transfers**

Routers should be able to tell from the **Packet Type**:

* **Internal:** Just update the ledger on the same chain. Maybe the packet doesn’t even need all 20 hops.
* **External:** Full routing + beacon handshake + multi-sig treasury calls.

This avoids wasting bandwidth on internal transfers that don’t need full cross-chain machinery.

---

## **Security Questions I Keep Asking**

1. **What if a router tampers with the packet?**
   → The Onion Blob and checksum should catch it; finalizer will reject mismatches.

2. **Can routers censor packets by dropping them?**
   → They could try, but:

   * Packets have backups and retries.
   * Routers that keep failing pings or dropping packets can be flagged and eventually removed from the registry.

3. **What if someone runs a ton of malicious routers?**
   → We need some anti-Sybil mechanism (small stake or membership requirement) so it’s expensive to flood the network.

---

## **Implementation Thoughts**

1. MVP: routers poll a central **packet pool** every 4s and forward what they pick up.
2. Later: move to **gossip-based pools** so no single packet source exists.
3. Keep routers as stateless as possible—if they crash, no big deal.
4. Implement soft validation (cheap checks) first, then add deeper beacon/miner queries later.

---

## **Future Work**

* Build a lightweight reputation system (routers earn trust by successfully forwarding packets).
* Figure out if we need **region-aware routing**: e.g., limit how many hops can be in the same data center or country.
* Can we make routers **self-auditing** (periodic reports of what packets they processed)?
* Explore **router diversity incentives** so one entity can’t dominate a route.
