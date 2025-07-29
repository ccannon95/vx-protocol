
## Randomized Selection thoughts.

* **Use epoch VRF randomness** from beacons + a **deterministic sampler** (HKDF → weighted reservoir sampling).
* **Filter** candidates by health/role/region.
* **Sample 20 primary hops + K backups without replacement**, with **ASN/region caps** to prevent clustering.
* **Onion‑wrap** Sphinx‑style so each hop only knows “who sent” and “who’s next”.
* **Select 3 finalizers** from a separate label on the same seed.
* Include an **audit commitment** so finalizers can verify the route was derived from the public seed (no grinding).

This should give: **unbiased, reproducible, auditable** routing.

---

## 0) Inputs we would need each (short) epoch

* `R_e`: **Epoch randomness** (VRF output + proof) published by the beacon set every 5–15s.
* `M_e`: **Metrics snapshot** (availability, latency, congestion, reputation) for routers.
* `Registry`: Active routers/finalizers with `{id, pubkey, region, asn, capacity, roleFlags}`.

> Epoch randomness prevents the sender from grinding nonces to pick “friendly” paths.

---

## 1) Build the candidate pool

Filter the registry:

* Role = `ROUTER`, status = `healthy`, last\_seen < 10s
* `reputation ≥ threshold`, `capacity ≥ min`
* Exclude same org/ASN if over weight (anti‑collusion)
* For finalizers: role = `FINALIZER`, not in selected route

---

## 2) Derive a per‑packet seed (deterministic, ungameable)

```
seed = H(
  R_e                       ||  // beacon VRF randomness
  packet_id                 ||  // hash of the packet header/payload
  sender_pubkey             ||  // binds to the actor
  sender_nonce              ||  // prevents replay
  "VX/ROUTE/v1"                 // domain separator
)
```

* Everybody can recompute `seed` (once revealed in finality), so selection is **auditable**.
* The sender can’t grind: `R_e` is unknown until the epoch closes.

---

## 3) Sample 20 hops + backups (weighted, without replacement)

**Weights** reflect health/congestion:

```
w_i = f(availability, latency, jitter, reputation, capacity, regionLoad)
```

Use **Efraimidis–Spirakis** (weighted reservoir) driven by a **deterministic PRNG stream** from `HKDF(seed)`. Apply **caps**:

* Max 2 hops per ASN
* Max 4 hops per region
* No same router twice
* Enforce **diversity** early: if a draw violates constraints, skip to next PRNG word.

**Pseudocode (Go‑ish, simplified)**

```go
func PickRoute(seed []byte, routers []Router, n int) ([]Router, []Router) {
    prng := hkdfStream(seed)                     // deterministic stream
    cand := filterHealthy(routers)
    keys := make([]float64, 0, len(cand))
    for _, r := range cand {
        u := nextUniform(prng)                   // (0,1]
        k := math.Pow(u, 1.0/r.Weight)          // ES key
        keys = append(keys, k)
    }
    // sort by descending keys with constraint checks while selecting
    order := argsortDesc(keys)
    route := []Router{}
    backups := []Router{}
    asnCount := map[int]int{}
    regionCount := map[string]int{}

    for _, idx := range order {
        r := cand[idx]
        if violates(asnCount, regionCount, r) { backups = append(backups, r); continue }
        route = append(route, r)
        asnCount[r.ASN]++
        regionCount[r.Region]++
        if len(route) == n { break }
    }
    // remaining go to backups
    for _, idx := range order {
        r := cand[idx]
        if !contains(route, r) { backups = append(backups, r) }
    }
    return route, backups
}
```

> **Deterministic + constraint‑aware** ensures any node can audit that your 20 + backups came from `seed`, not hand‑picked.

---

## 4) Onion‑wrap the packet (Sphinx‑style)

* Fetch the public key of each selected router: `R[0..19]`.
* Generate one ephemeral curve keypair `e`.
* Derive per‑hop shared secret: `s_i = KDF( DH(e, R[i].pubkey) )`.
* For each hop, build **per‑hop header**: `{nextHopID, ttl, mac}` encrypted with `s_i`.
* The outermost layer targets `R[0]`; after each hop peels, it learns only `nextHopID` and forwards.

**Properties:** no hop sees full path, no one can tamper without MAC failure.

---

## 5) Failure + fallback (reshuffle without new randomness)

* Include a **backup index** counter `b` in the header (encrypted).
* If a hop fails or times out, the current hop picks the next backup:

  ```
  next = backups[b]; b++
  ```
* TTL decrements each hop; if exhausted → return to pool / NACK to sender.
* Because **backups** were derived from the same `seed`, fallback is **auditable** and non‑grindable.

---

## 6) Finalizer committee (3‑of‑N, independent)

Same machinery, new domain tag:

```
seed_final = H(R_e || packet_id || sender_pubkey || sender_nonce || "VX/FINAL/v1")
finalizers = PickCommittee(seed_final, FinalizerPool, 3, constraints)
```

* Exclude any router from the route (role separation).
* Committee signs the completion bundle; miners verify 3‑of‑N signatures.

---

## 7) Congestion‑aware routing (weights we should implement today)

Update weights `w_i` each epoch from beacon metrics:

```
w_i = base * AvailabilityFactor * ReputationFactor * CapacityFactor * e^{-alpha * CongestionScore}
```

* `CongestionScore` derived from recent queue length / p95 latency.
* This makes hot zones **less likely** to be chosen without hard bans.

---

## 8) Verifiability / anti‑grinding

* Put `routeCommit = H(routeIDs || salt)` in the packet header.
* On finalization, reveal `{routeIDs, salt}`; anyone can:

  * Recompute `seed` from public inputs
  * Rerun sampler → check `routeIDs` match allowed outputs (subject to constraints)
  * Verify backups and any fallback hops used

If the revealed route is **inconsistent** with the deterministic sampler → slash sender or reject.

> In MVP, the **relayer** can do the selection centrally; later we migh move to beacon VRF with on‑chain verification of the VRF proof.

---

## 9) Minimal data 

**Router registry entry**

```
id, pubkey, asn, region, capacity, reputation, last_seen, roleFlags
```

**Epoch beacons**

```
epoch_id, VRF_value, VRF_proof, metrics_hash (covers all router metrics)
```

**Packet header fields (relevant)**

```
packet_id, sender_pub, sender_nonce, ttl, routeCommit, epoch_id, onion_blob
```

---

## 10) Implementation order Thoughts - 

1. **MVP**: selection in the relayer (central), but **record `R_e` and routeCommit** so it’s audit‑ready.
2. Swap in **beacon VRF** (multiple beacons aggregate; any node verifies proof).
3. Add **constraint caps** (ASN/region) + **weighted sampling**.
4. Add **fallback backups**; measure success.
5. Move onion to **Sphinx** (or Noise) library.
