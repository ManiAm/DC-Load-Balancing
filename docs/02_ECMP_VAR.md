
# ECMP Variants


## The Disruption Problem

In static ECMP, the number of buckets exactly equals the number of active next-hops, and each next-hop owns exactly one bucket.

<img src="../pics/1.png" alt="segment" width="350">

This means any membership change forces a complete rebuild of the hash-to-bucket mapping.

**Addition**: When a new next-hop is added (e.g., Server E joins), the bucket count increases and the hash modulus changes. The ASIC recalculates the mapping for all flows, not just the ones that will use the new path. Flows that were stable on their existing paths get reassigned to different next-hops, causing out-of-order packets.

<img src="../pics/2.png" alt="segment" width="350">

**Removal**: The same reshuffling occurs when a next-hop fails or is withdrawn. For example, when Server B fails, its bucket is removed and the total shrinks from four to three. The hash recalculation disrupts all remaining flows including those that had no relationship to Server B.

<img src="../pics/3.png" alt="segment" width="750">


### The Partial Fix — Consistent Hashing

To fix the reshuffling caused by static ECMP, engineers introduced Consistent Hashing. Consistent hashing decouples the number of buckets from the number of next-hops by creating a fixed-size container. Instead of having exactly 4 buckets for 4 servers, the switch might be configured with a fixed pool of 12 buckets, which are dealt out evenly (e.g., 3 buckets per server).

When an ECMP member fails or is removed, the number of hash buckets does not shrink. Instead, the specific buckets that belonged to the dead member are redistributed to the surviving next-hops.

- Advantage: All other flows remain completely undisturbed.
- Drawback: It does not prevent disruption when a new next-hop is added, as space must be cleared for the new member.

The following figure shows next-hops assigned in a round-robin fashion to a fixed, larger pool of buckets.

<img src="../pics/5.png" alt="segment" width="350">

Because the number of buckets is fixed, removing a next-hop no longer changes the math for the entire group. When Next-Hop B fails, the total number of buckets (12) does not change.

<img src="../pics/6.png" alt="segment" width="350">

The system only takes the specific buckets mapped to the failed node and redistributes them to surviving next-hops. Healthy flows remain completely undisturbed.

<img src="../pics/7.png" alt="segment" width="350">


### The Complete Solution — Resilient Hashing

Consistent Hashing eliminates disruption on member removal (surviving flows are undisturbed), but it does not handle member additions gracefully. Because the bucket count is fixed, adding a new next-hop means stealing buckets from existing next-hops to give to the new one, which can still disrupt active flows. The following figures show adding Next-Hop E requires shifting some existing buckets (3 and 8) to make room, which can disrupt active sessions.

<img src="../pics/8.png" alt="segment" width="750">

Resilient Hashing builds directly upon Consistent Hashing to solve the final puzzle piece: how to add a new server gracefully without breaking active sessions. It uses the same fixed-size bucket pool, but instead of immediately stealing buckets when a new node is added, it introduces patience. The core idea is:

1. The new next-hop is brought online but initially receives **zero buckets**.
2. The switch monitors which existing buckets are currently carrying active traffic and which are idle.
3. When a bucket goes idle (no recent traffic detected), the switch safely migrates that empty bucket to the new next-hop.
4. Active sessions are protected and allowed to naturally conclude on their original paths, while new traffic gradually begins balancing onto the new node.

This concept is universal to resilient hashing, but the specific implementation differs by ASIC vendor:

#### Broadcom (Immediate Redistribution)

On Broadcom ASICs (Tomahawk, Trident series), resilient hashing operates without timers or background monitoring:
- When a next-hop is **removed**, its buckets are immediately redistributed to the surviving next-hops.
- When a next-hop is **added**, some buckets are immediately migrated from existing next-hops to the new one.
- The algorithm balances buckets as evenly as possible across all next-hops. Assignment does not change for any reason other than membership changes (it does not react to traffic load or bucket activity).

This is simple and deterministic, but the immediate migration on addition can disrupt active flows that happen to occupy the stolen buckets.

#### Nvidia / Mellanox Spectrum (Timer-based, Gradual)

On Nvidia Spectrum ASICs, a background thread manages bucket migration using two configurable timers:
- `active_flow_timer`: Defines how long a bucket must be continuously idle (no traffic) before it is considered safe to migrate. The default is typically 120 seconds.
- `max_unbalanced_timer`: A hard ceiling on how long the system is allowed to remain unbalanced. If this timer expires and there are still not enough idle buckets, the switch forces a migration as a safety net.

The step-by-step process:
1. A new next-hop is added. It receives zero buckets.
2. The background thread continuously scans all buckets for activity.
3. When a bucket has been idle for the full `active_flow_timer` duration, it is migrated to the new next-hop.
4. If the `max_unbalanced_timer` expires before enough idle buckets are found, the thread forces a rebalance (which may disrupt active sessions).

This timer-based approach protects long-lived sessions from disruption during scale-out events, at the cost of temporarily uneven distribution until buckets naturally become idle. In networks with highly persistent flows, the new next-hop may remain underutilized for an extended period.

#### Bucket Count Trade-offs

Resilient hashing uses a finite, hardware-level pool of buckets that is shared across all ECMP groups on the switch. The number of buckets per ECMP group is configurable (common options: 64, 128, 256, 512, or 1024), and this choice represents a direct trade-off:

- **More buckets per group:** Reduces the disruption impact when a next-hop is added or removed (a smaller percentage of flows are affected). However, the total number of ECMP groups the switch can support decreases because the shared hardware pool is consumed faster.

- **Fewer buckets per group:** Supports more ECMP groups simultaneously, but each add/remove event disrupts a larger proportion of flows.

If the maximum number of ECMP groups is reached, new ECMP routes cannot be installed and are rejected by the hardware.



## Fine-Grained ECMP

Consistent hashing and resilient hashing solve the disruption problem in hardware, but their bucket redistribution is coarse — they have no awareness of which upstream router a next hop belongs to, so a failure behind one spine can still reshuffle flows destined for a completely different spine.

**Fine-Grained ECMP (FG-ECMP)** takes a different approach: instead of letting the ASIC manage the bucket array, **software pre-computes the entire bucket-to-member mapping** and programs it explicitly. The ASIC itself just sees a regular ECMP group with a pre-populated bucket array — all the "fine-grained" intelligence lives in the control plane.

This software ownership enables two capabilities that hardware-managed modes cannot provide on their own:

1. **Keep the bucket array size constant** when a member disappears (no modulus change, no global rehash).

2. **Rewrite only the specific buckets** that belonged to the failed member — and restrict that rewrite to a chosen subset of survivors.

### How It Works

The key difference from standard ECMP is the bucket count. In standard ECMP, the bucket count equals the member count (3 members = 3 buckets). In FG-ECMP, the operator configures a **much larger, fixed bucket pool** (e.g. 30 buckets for 3 members). Each member receives an equal share (10 buckets each). The larger count gives finer granularity for redistribution on failures.

```text
Standard ECMP (3 next hops, 3 buckets):

    Bucket 0 → NH-A    ┐
    Bucket 1 → NH-B    ├─ ASIC decides; removal of NH-B rehashes ALL flows
    Bucket 2 → NH-C    ┘

Fine-Grained ECMP (3 next hops, 30 buckets):

    Bucket  0–9  → NH-A   ┐
    Bucket 10–19 → NH-B   ├─ Software decides (10 buckets per NH)
    Bucket 20–29 → NH-C   ┘
```

When NH-B fails, the software rewrites only buckets 10–19 and redistributes them to the surviving members. The array size stays at 30, so the hash modulus (`h mod 30`) does not change. Flows on NH-A and NH-C stay exactly where they were:

```text
Before NH-B fails                           After NH-B fails (only buckets 10–19 rewritten)

  Bucket  0–9   → NH-A                         Bucket  0–9   → NH-A      unchanged
  Bucket 10–19  → NH-B                         Bucket 10–19  → NH-A/NH-C rewritten
  Bucket 20–29  → NH-C                         Bucket 20–29  → NH-C      unchanged

  h mod 30                                     h mod 30   (array size unchanged)
```

With standard ECMP, the same failure shrinks the array from 3 to 2 buckets, changes the modulus, and moves nearly every flow. With FG-ECMP, only the dead member's flows must move — everyone else is untouched.

### Banks: Limiting the Blast Radius

The above example redistributed NH-B's buckets to *all* survivors. FG-ECMP can do better using **banks** — a grouping of next hops that share a common property, typically one bank per upstream router or spine.

A bank is a label attached to each member. Members that share a label form a bank, and each bank owns a contiguous range of the bucket array, sized in proportion to its member count. The critical rule: **when a member fails, its buckets are given only to other members in the same bank.** Members in other banks receive nothing — their bucket ranges, their flows, and their traffic load are all untouched.

Example: 60 buckets, three next hops, two banks (one per upstream spine):

```text
Fine-Grained ECMP (3 next hops, 60 buckets, 2 banks):

  bank 0 = { NH-1, NH-3 }   owns buckets  0–39   (20 per member)
  bank 1 = { NH-5 }         owns buckets 40–59   (20 per member)

  NH-3 fails  →  buckets 20–39 are rewritten to NH-1 (same bank)
                 buckets 40–59 (bank 1) are not touched
```

A failure behind Spine 1 cannot ripple into traffic that was going through Spine 2. This is **bank-scoped failover** — the core value of FG-ECMP.

> **Note:** Banks are a failover isolation concept, not a traffic-weighting concept. Every member still receives the same number of buckets (`total_buckets / total_members`). A bank's range is larger only because it has more members. With 60 buckets and 3 members, each member gets exactly 20 buckets (33.3% of traffic) — bank 0 owns 40 buckets because it has two members, not because it has higher weight.

### FG-ECMP vs. Standard and Consistent ECMP

| Property                 | Fine-Grained ECMP                        | Static ECMP         | Consistent ECMP |
|--------------------------|------------------------------------------|---------------------|-----------------|
| **Bucket management**    | Software-controlled                      | Hardware (ASIC)     | Hardware (ASIC) |
| **Bucket assignment**    | Explicitly programmed per bucket         | ASIC-managed        | ASIC-managed |
| **Bucket count**         | Fixed (operator-configured)              | Equals member count | Fixed (pre-configured) |
| **Traffic distribution** | Equal per next hop                       | Equal per next hop  | Equal per next hop |
| **Stable on removal**    | Yes (bank-scoped redistribution)         | No (full rehash)    | Yes (hardware) |
| **Stable on addition**   | Partial (proportional steal within bank) | No (full rehash)    | No (bucket steal) |
| **Bank-based failover**  | Yes                                      | No                  | No                |

Two things to remember about FG-ECMP:

- It is about **stability**, not weighting. Every member always receives the same number of buckets. Per-member weights are not supported.
- It is configured **per destination prefix**, not automatically for all routes. The operator explicitly selects which route prefixes use FG-ECMP and assigns their members to banks.


## Weighted ECMP

Everything covered so far splits traffic **equally** across next hops. **Weighted ECMP** removes this assumption by allowing each next hop to receive a **proportional** share of traffic based on an assigned weight.

> The industry has not settled on a single name. Cumulus Linux (Nvidia) and Nokia use **Weighted ECMP**; SONiC, Google, and SAI use **WCMP** (Weighted Cost Multipath); Cisco and Juniper use **UCMP** (Unequal Cost Multipath). **BGP Link Bandwidth** is the BGP extended community that signals per-path capacity to derive weights. The underlying mechanism is the same in all cases.

### Why It Is Needed

Standard ECMP gives every next hop an equal share of the hash space. That is the wrong answer when the paths themselves are unequal:

- **Mixed link speeds**: Consider a fabric during a rolling upgrade from 40G to 100G. If a leaf has one 100G uplink to Spine A and one 40G uplink to Spine B, standard ECMP splits flows 50/50. The 40G link saturates while the 100G link sits underutilized. Weights of 5:2 match the split to the capacity ratio, steering approximately 71% of flows toward the faster link.

- **Traffic engineering**: Deliberately directing more traffic toward a preferred path, even when link speeds are identical — or gracefully draining a path by ramping its weight down (e.g. weight 4 → 2 → 1 → remove) before taking it out of service.

### How Weights Are Implemented in Hardware

The weight is a **per-member attribute** — it lives on each next hop *inside* the group, not on the route and not on the group itself. Weighted ECMP uses the exact same hash-and-bucket mechanism as standard ECMP. The hash function is unchanged, and a given flow still sticks to exactly one next hop. The only difference is **how buckets are allocated**: a next hop with higher weight owns more buckets.

In standard ECMP, the bucket count equals the member count — one bucket per member. In weighted ECMP, the bucket count is the **sum of the weights**, and each member owns buckets proportional to its weight. The most common hardware technique is **entry replication**: the same physical next hop is listed multiple times in the bucket array.

Take four members and give NH-A weight 2 while the others keep weight 1. The bucket count becomes 2+1+1+1 = 5, the hash becomes `h mod 5`, and two buckets point to NH-A:

```
                                          ┌─ Next Hop Group ──────────────────────────────────────┐
                                          │                                                       │
                                          │   Hash Buckets          Members                       │
                                          │   (bucket array)        (next hops, with weight)      │
                                          │  ┌─────────────┐      ┌───────────────────────────┐   │
  flow 1 ──────┐                          │  │  Bucket 0   │ ──┐  │ #0  NH-A  Eth0   weight 2 │   │
               │    ┌────────┐            │  ├─────────────┤   ├─►│      (2 of 5 buckets)     │   │
  flow 2 ──────┼──► │  Hash  │─ h mod 5 ─►│  │  Bucket 1   │ ──┘  ├───────────────────────────┤   │
               │    │ 5-tuple│            │  ├─────────────┤      │ #1  NH-B  Eth4   weight 1 │   │
  flow 3 ──────┘    └────────┘            │  │  Bucket 2   │ ───► │      (1 of 5 buckets)     │   │
                                          │  ├─────────────┤      ├───────────────────────────┤   │
  (route lookup already                   │  │  Bucket 3   │ ───► │ #2  NH-C  Eth128 weight 1 │   │
   selected this NHG)                     │  ├─────────────┤      │      (1 of 5 buckets)     │   │
                                          │  │  Bucket 4   │ ──┐  ├───────────────────────────┤   │
                                          │  └─────────────┘   └─►│ #3  NH-D  Eth132 weight 1 │   │
                                          │                       │      (1 of 5 buckets)     │   │
                                          │                       └───────────────────────────┘   │
                                          └───────────────────────────────────────────────────────┘

  Result:  NH-A receives ~40% of flows; NH-B, NH-C, NH-D ~20% each.
```

Because each bucket still attracts an equal share of flows (`1/num_buckets`), the hash itself never changes — only the bucket-to-member ownership changes, and that is what shapes the traffic split.

Weights are **ratios**: 2:1:1:1 and 4:2:2:2 produce the same 40/20/20/20 split, just with 5 or 10 buckets. When the summed weights get large, the ASIC may rescale them to a smaller total while preserving the same ratio. Standard ECMP is simply the special case where every weight is 1, so bucket count equals member count.

### Weight Sources

Weights can be assigned in two ways:

- **Static configuration**: The operator explicitly sets a weight on each next hop when defining a route (e.g. weight 3 on one path, weight 1 on another).

- **BGP Link Bandwidth**: With the BGP link-bandwidth extended community, routers advertise the capacity of each path. The routing protocol derives weights automatically from these advertised bandwidths — no per-route configuration is needed.

### Limitations

Weighted ECMP inherits the same fundamental constraint as standard ECMP: each flow is pinned to a single bucket and **cannot be split**. The configured weight ratio only holds statistically when there are many small mice flows. A small number of elephant flows can easily skew the actual load far from the intended weights.

Additionally, weighted ECMP complicates [resilient hashing](#the-complete-solution--resilient-hashing). When a next-hop group is resized, redistributing replicated entries can inadvertently alter the intended weight ratio — a problem known as **weight skew**.
