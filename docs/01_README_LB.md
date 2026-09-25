
# Fabric Load Balancing

Modern data centers are built on a **Leaf-Spine** topology (a practical implementation of the Clos network architecture). In this design, the network consists of two tiers of switches: **leaf switches** at the edge, where servers connect, and **spine switches** forming the fabric core. The defining characteristic is full-mesh connectivity between the tiers — every leaf switch connects to every spine switch.

<img src="../pics/leaf-spine-new.png" width="480"/>

This full-mesh wiring creates **multiple equal-cost paths** between any two servers on different leaves. Traffic from Server A on Leaf 1 to Server B on Leaf 4 can traverse any of the spine switches and arrive in the same number of hops with the same latency. This inherent path diversity is what gives Leaf-Spine fabrics their bandwidth and resilience — but it also raises a critical question: when a leaf switch has multiple equally valid uplinks, how does it decide which spine to send each packet through?

That decision is the core of **fabric load balancing**, and it involves two distinct systems working together: the control plane discovers the available paths, and the switching ASIC (NPU) makes the actual per-packet forwarding decision in hardware.

> For a comprehensive treatment of data center topologies (Leaf-Spine, Fat-Tree, Dragonfly, and more), physical wiring models, and scaling strategies, see the [Network Topology](https://github.com/ManiAm/DC-Fundamentals/blob/master/docs/03_README_TOPOLOGY.md) document in the DC-Fundamentals project.


## Path Discovery (Control Plane)

Before any load balancing can happen, the switch must first discover that multiple equal-cost paths actually exist. This is the responsibility of the control plane. Routing protocols like BGP (the standard in modern data center fabrics) or OSPF run as software processes on the switch's CPU. They constantly exchange reachability information with neighboring switches to compute the shortest path to every destination. In a modern Leaf-Spine architecture, every leaf connects to every spine, meaning the routing protocol will naturally discover multiple paths with the exact same cost.

When this happens, the process looks like this:

- The routing protocol identifies multiple equal-cost next hops.
- It installs all of them into the Routing Information Base (RIB) — the software routing table.
- The best routes from the RIB are then programmed into the Forwarding Information Base (FIB) — the hardware forwarding tables of the switching ASIC.

While static routes can achieve the same result, dynamic protocols are preferred in production environments. They automatically detect link failures and remove the affected next hop from the hardware tables without manual intervention.

Once the FIB is programmed, the control plane's job is done. From this point forward, the ASIC handles every forwarding decision entirely in hardware at line rate, with zero involvement from the switch CPU.


## What is a Network Flow?

To understand how the hardware load-balances traffic, we first must understand how network traffic is grouped. A **flow** is a sequence of packets sent from a specific source to a specific destination representing a single session (like a file download or a web request). To identify a flow, the switch looks at five specific fields in the packet header, collectively known as the 5-tuple:

- Source IP Address
- Destination IP Address
- Source Port
- Destination Port
- Protocol (e.g., TCP or UDP)

If a stream of packets shares the exact same 5-tuple, the switch knows they belong to the exact same conversation. Crucially, packets within the same flow must be sent down the exact same physical path. If they are split across different links, variations in network latency could cause them to arrive out of order, crippling application performance.

Network flows in a data center generally fall into two categories:

- **Mice Flows**: Small, short-lived connections (such as DNS queries, small API calls, or basic HTTP requests). They make up the vast majority of total flows by count.

- **Elephant Flows**: Large, continuous, high-bandwidth connections (such as storage backups, virtual machine migrations, or massive dataset transfers). While they are few in number, they make up the vast majority of total data volume.


## ECMP (Equal-Cost Multi-Pathing)

ECMP is the standard data-plane load-balancing mechanism in IP fabrics. It ensures that aggregate traffic is distributed across all available links, while strictly keeping individual flows pinned to a single link.

The diagram below illustrates ECMP distributing traffic on a per-flow basis. Each colored square represents an individual packet, with each distinct color representing a unique flow. When these packets arrive at the router, the switch sorts each flow into one of the available equal-cost paths. In this example, the router has three available paths, so flows are distributed across three buckets:

- Pink Flow: Pinned to the top path.
- Blue Flow: Pinned to the middle path.
- Orange & Green Flows: Both routed across the bottom path.

<img src="../pics/ecmp_example.png" width="650"/>

The Orange and Green flows sharing the same path is an example of a **hash collision**. Because there are nearly infinite flow combinations but only a few physical links, different flows will inevitably map to the same link.


### Hash-Based Forwarding

To understand how the ASIC picks a path, follow one packet through the switch:

- **Step 1**: Packet arrives; the ASIC extracts the destination IP.
- **Step 2**: Routing table (FIB) lookup → the route points to a next-hop group (NHG) with multiple equal-cost members.
- **Step 3**: The ASIC extracts the 5-tuple and computes a hash over it (the hash output is a wide number, e.g. `0x9C41`).
- **Step 4**: The hash is reduced to a bucket index: `index = hash mod NHG_size` (e.g. `0x9C41 mod 4 = 1`).
- **Step 5**: A lookup table (**bucket array**) is read at that index. The entry (**bucket**) points to a specific member (next hop, MAC, egress port).

The one line to remember: **the hash picks an index; the bucket at that index picks the member.**

<img src="../pics/ecmp_oper.png" width="800"/>

> **Note:** ECMP hashing does not track how many packets or how much bandwidth has been sent to each next hop. It provides no guarantee that traffic is evenly distributed across all links. Distribution is purely statistical — it depends on the number and size of active flows and how their headers happen to hash. With a large number of **mice flows**, distribution tends to be even. With a small number of **elephant flows** or flows with similar headers, significant imbalance is possible. See [The Elephant Flow Problem](#the-elephant-flow-problem) for a detailed analysis.


### Advanced Configuration: Customizing the Hash

While the standard 5-tuple works for most traffic, modern network ASICs allow operators to customize which header fields are included in the hash calculation. On some platforms, the ASIC hashes additional fields by default beyond the 5-tuple, including **Source MAC**, **Destination MAC**, **Ethertype**, **VLAN ID**, and **Ingress Interface**. These extra fields improve distribution for non-IP traffic and provide additional entropy.

Operators can enable or disable individual fields to solve specific network challenges:

- **Encapsulated Traffic (Tunnels)**: For traffic wrapped in tunnels (like GRE or IPsec), hashing only the outer IP headers would map all tunneled traffic to a single link. Configuring the switch to include inner headers (inner source/destination IP, inner ports, inner protocol) in the hash prevents congestion by distributing the underlying flows across multiple links.

- **Fragmented Packets**: Fragmented IP packets often lack Layer 4 port information after the first fragment. If ports are included in the hash, subsequent fragments might take a different path than the first, leading to reassembly failures. Reconfiguring the hash to a 2-tuple (Source & Destination IP only) ensures consistent routing for all fragments.

- **IPv6 Flow Labels**: In IPv6 networks, the IPv6 Flow Label can be added to the hash. This provides the switch with high-quality randomness (entropy) without forcing the ASIC to parse deep into Layer 4 headers.

> **Note: Symmetric Hashing**
> By default on some platforms, **symmetric hashing** is enabled. This ensures that traffic flowing in both directions of a conversation (A→B and B→A) always hashes to the same physical path. The ASIC achieves this by treating source and destination fields as an unordered pair — swapping source IP with destination IP (and source port with destination port) produces the same hash output. Symmetric hashing is important for stateful monitoring tools (like NetFlow collectors or TAP aggregators) that need to see both sides of a flow on the same link. If the source and destination IP hash settings or the source and destination port hash settings are mismatched (one enabled, the other disabled), symmetric hashing is automatically disabled.


### ECMP in Leaf-Spine Topology

In a standard 2-tier fabric, the leaf switch is the primary device performing ECMP, choosing which spine to send traffic to. The spine typically has exactly one direct link to each destination leaf, so it simply forwards the packet without needing to load-balance. However, in a larger 3-tier fabric (leaf → spine → super-spine), the spine ASICs will also perform ECMP when selecting which super-spine to traverse to reach a completely different pod.

<img src="../pics/ecmp_leaf_spine.png" alt="segment" width="400">

This asymmetry has a powerful scaling implication: adding a new spine switch to the fabric instantly adds one more ECMP bucket to *every* leaf in the fabric simultaneously. If a leaf previously had 4 spines (4 equal-cost paths), adding a 5th spine gives every leaf a 5th path, increasing total fabric bandwidth by 25% without touching a single existing switch configuration. This is one of the reasons Leaf-Spine scales predictably: bandwidth is a function of spine count, and ECMP distributes it automatically.


### The Multi-Hop Challenge: Hash Polarization

In a multi-tier topology, each switch independently computes its own ECMP hash on the same packet headers. If every switch uses the exact same hash algorithm and the same input fields, the same 5-tuple will produce the exact same hash result at every hop. The result is **hash polarization**: flows cluster onto the exact same physical paths across different tiers of the network, leaving some links heavily congested while parallel links sit completely idle.

#### Visualizing the Hash Polarization Problem

The following diagram illustrates hash polarization. Consider four flows (F1–F4), each with a different 5-tuple, arriving at Switch A:

<img src="../pics/hash-pol.png" alt="segment" width="500">

Switch A computes a hash on each flow's 5-tuple to select a next hop. According to the first hash table in the diagram, flows that hash to 0 are forwarded to Switch B (green dashed path) and flows that hash to 1 are forwarded to Switch C (red dashed path). Suppose F1 and F2 hash to 0, while F3 and F4 hash to 1. So far, the traffic is successfully split.

- F1 → hash 0 → Switch B
- F2 → hash 0 → Switch B
- F3 → hash 1 → Switch C
- F4 → hash 1 → Switch C

The polarization problem occurs at the second tier. Switch B receives F1 and F2 — the flows that produced hash 0 at Switch A. Because Switch B uses the exact same algorithm on the exact same 5-tuples, it computes the exact same hash values: F1 → 0, F2 → 0. Because every flow at Switch B hashes to 0, Switch B forwards 100% of its traffic to Switch D. The link from B to E sits completely idle.

- F1 → hash 0 again → Switch D
- F2 → hash 0 again → Switch D

The same occurs at Switch C. F3 and F4 both produced hash 1 at Switch A, and Switch C recalculates hash 1 for both. All traffic goes to Switch E. The link from C to D is never used.

Even though 4 equal-cost paths exist between A and F, the identical computation at each hop forces all traffic onto just 2 of them (A → B → D → F and A → C → E → F). The other two cross-links are wasted. Breaking this correlation requires each switch to produce a statistically independent hash result for the same 5-tuple. To achieve this, ASICs use Hash Seeds, Hash Offsets, and specific Hash Algorithms.

#### Hash Seed

The hash seed is a 32-bit integer (0–4,294,967,295) that is mixed into the hash computation before the algorithm processes the packet's 5-tuple fields. It is analogous to a cryptographic salt: feeding the same packet header into the same algorithm, but with a different seed, produces a completely different hash output and therefore a different ECMP member selection.

Assigning a unique seed per switch is the primary method for preventing hash polarization. With different seeds, the same packet produces a unique hash at each tier, ensuring flows are statistically distributed across the entire fabric.

#### Hash Offset

The hash offset dictates which bits of the computed hash value are actually used for the ECMP member selection. After a deterministic hash algorithm (like CRC or XOR) produces a multi-bit result, the ASIC extracts a specific window of bits starting at position offset to determine the final bucket index.

Even with unique seeds, the same algorithm processing the same 5-tuple can sometimes produce correlated bit patterns across tiers. The offset breaks this subtle correlation by forcing each tier to look at a completely different region of the hash output.

> **Note: Offset vs. Seed**
> Both mechanisms prevent hash polarization, but they work differently. The seed changes the hash computation itself (different input → different output). The offset keeps the hash output the same but extracts different bits for the bucket selection. Using both together provides the strongest anti-polarization across fabric tiers. (The offset has no effect on random or round-robin algorithms, as those do not compute a hash from packet headers).



### Hash Algorithms

The hash algorithm determines how the selected header fields and the seed are mathematically combined into the final hash value.

| Algorithm   | Description                                                                                                                                                                                                                                                                        |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| crc         | Standard CRC—the default and most widely used. Provides good distribution across buckets for typical traffic patterns.                                                                                                                                                             |
| crc-ccitt   | CRC-CCITT variant (polynomial 0x1021). Features slightly different bit-mixing than standard CRC; useful when the default CRC produces uneven distribution for a specific, highly predictable traffic profile.                                                                      |
| crc-32lo    | Uses the lower 32 bits of a CRC-32 computation. On platforms with wide hash pipelines, this selects the lower half.                                                                                                                                                                |
| crc-32hi    | Uses the upper 32 bits of a CRC-32 computation. Paired with crc-32lo, these two variants allow a network operator to use different halves of the same CRC calculation for ECMP vs. LAG. This avoids polarization between routing and switching without needing to change the seed. |
| xor         | Bitwise XOR of the hash fields. It is simpler and faster than CRC but produces weaker distribution. Fields that are numerically close (e.g., sequential IP addresses) tend to map to adjacent buckets.                                                                             |
| crc-xor     | Hybrid: A CRC calculation followed by an XOR fold. Combines CRC's distribution strength with XOR's ability to compress the result into fewer bits.                                                                                                                                 |
| random      | Ignores packet headers entirely and selects a bucket at random. Every packet is independently assigned, providing perfect statistical balance but zero flow affinity—packets from the same flow will likely take different paths, causing out-of-order delivery.                   |
| round-robin | Cycles through buckets sequentially (bucket 0, 1, 2, …, wrap). Like random, this provides no flow affinity and will cause significant packet reordering within flows.                                                                                                              |


Deployment Recommendations:

- Use `crc` for most deployments to ensure good distribution and flow path consistency.
- Mix `crc-32lo` and `crc-32hi` across ECMP/LAG with unique seeds for advanced anti-polarization.
- Avoid `random` and `round-robin` unless the application specifically tolerates packet reordering.


## The Elephant Flow Problem

ECMP was designed around the "law of large numbers." It assumes that hashing thousands of random, small flows (Mice) will naturally and evenly distribute traffic across all available links. For Mice flows, this statistical assumption works perfectly. However, because ECMP is strictly bound by the rule that it cannot split a single flow across multiple paths (to preserve packet ordering), an entire Elephant flow is forced to traverse a single physical link.

When an Elephant flow enters the fabric, it exposes the rigidity of hardware-based hashing. Because the underlying ASIC hash function is static (meaning the same 5-tuple will always produce the exact same physical port assignment) it cannot dynamically react to bandwidth utilization. This leads to a highly inefficient scenario:

- A massive Elephant flow hashes to a specific spine link, instantly consuming a huge portion of that link's total capacity.

- If a second Elephant flow enters the switch and happens to compute to the exact same hash remainder (a **hash collision**), the ASIC blindly assigns it to that same already-burdened link.

- The link quickly hits 100% saturation and begins dropping packets.

While that single link is completely overwhelmed and dropping traffic, neighboring parallel spine links connected to the exact same destination might be sitting completely idle. Because standard ECMP is completely unaware of link utilization or flow size, it will happily forward an Elephant flow into a congested bottleneck while ignoring perfectly good, unused bandwidth right next door.

### Hash Collision in Practice

The diagram below shows a concrete hash collision in an 8-leaf, 4-spine fabric. Two independent elephant flows happen to hash to the same spine, congesting a single downlink while parallel paths sit idle.

<img src="../pics/ecmp_collision.jpg" alt="ECMP hash collision" width="900">

Server 2 (on leaf 1) is sending a large flow to server 9 (on leaf 3). Leaf 1's ASIC hashes the flow's 5-tuple and selects spine 2 as the next hop. The green path (server 2 → leaf 1 → spine 2 → leaf 3 → server 9) carries this flow without issue.

Independently, server 16 (on leaf 6) is sending a large flow to server 7 (also on leaf 3). Leaf 6's ASIC hashes this flow's 5-tuple and also selects spine 2. The blue path (server 16 → leaf 6 → spine 2 → leaf 3 → server 7) now shares the spine 2 → leaf 3 downlink with the green flow.

Because both are elephant flows, they demand more bandwidth than the single spine 2 → leaf 3 link can provide. The link saturates, forcing the switch to drop packets and cutting the throughput of both flows roughly in half. Note that if these were small mice flows, they would share the link without noticeable degradation.

This is the blind spot of ECMP. Spine 1, spine 3, and spine 4 all have completely idle, unobstructed paths to leaf 3 — yet the ASIC cannot detect the congestion or reroute either flow because ECMP decisions are based solely on the static hash, not on link utilization.


### Hash Collisions Are Statistically Inevitable

ECMP distributes flows based on hash values, not bandwidth. It has no awareness of how much traffic each flow carries. When two elephant flows independently hash to the same spine link, they compete for the same bandwidth: one link is saturated while parallel links sit idle. With mice flows, this rarely matters because each flow is tiny. With elephant flows, a single collision can halve the throughput of both flows.

#### The Birthday Paradox

The probability of hash collisions follows the same mathematics as the [birthday paradox](https://en.wikipedia.org/wiki/Birthday_problem). For *N* flows across *K* equal-cost paths, the probability of at least one collision is:

    P(collision) = 1 − K! / (K^N × (K−N)!)    when N ≤ K
                 = 1.0                        when N > K

With just *K* + 1 flows, collisions are guaranteed. But they become overwhelmingly likely long before that. With *K* = 8 spine links, the probability exceeds 50% at only 4 flows and reaches near certainty well before 8.

#### Performance Impact

Collisions cause *uneven* load, not just duplicate assignments. The expected number of flows on the most congested path is:

    Expected max load = (N/K) × (1 + ln(K) / ln(N/K))

For a leaf switch with *K* = 8 spine uplinks and *N* = 32 active elephant flows:

    Fair share     = N/K = 32/8 = 4 flows per path
    Expected max   = 4 × (1 + ln(8)/ln(4)) = 4 × (1 + 1.5) = 10 flows

The most congested spine link is expected to carry **10 flows** (2.5× its fair share) while lightly loaded links sit underutilized. This imbalance costs the fabric approximately 10–15% in total throughput, even though aggregate capacity is more than sufficient.


## Centralized Traffic Engineering (The SDN Approach)

### The Fundamental Limitation of ECMP

Every mechanism discussed so far — hash tuning, polarization mitigation — improves ECMP but cannot escape its core constraint: **the switch makes forwarding decisions using only local information**. Each switch hashes packet headers and selects a bucket independently, with no knowledge of how much traffic other switches are placing on the same links. When multiple leaf switches independently hash elephant flows onto the same spine link, the result is congestion on that link while parallel links sit idle — and no individual switch can detect or correct this because each one sees only its own egress ports.

> For mechanisms that address ECMP's disruption and weighting limitations — resilient hashing, consistent hashing, and weighted ECMP — see [ECMP Variants](02_ECMP_VAR.md).

This is not a configuration problem. It is an architectural limitation of any system where forwarding decisions are made independently at each hop using only locally available information.

### Centralized Traffic Engineering

An entirely different approach is to remove per-switch decision-making and hand it to a **central controller** that has a global view of the entire fabric. This is the SDN (Software-Defined Networking) approach to traffic engineering, where the controller collects real-time demand information from every switch, computes globally optimal flow placement, and programs the forwarding tables accordingly.

**MicroTE** is the foundational example of this approach (["MicroTE: Fine Grained Traffic Engineering for Data Centers,"](https://dl.acm.org/doi/abs/10.1145/2079296.2079304) ACM CoNEXT, 2011). The key insight is that data center traffic, while appearing random in aggregate, contains a significant fraction of large, predictable flows that persist long enough for a centralized system to detect and optimize.

MicroTE operates in three phases:

1. **Demand Collection:** ToR switches periodically report their current traffic matrix to the central controller — which source is sending how much traffic to which destination. This telemetry runs on timescales of a few seconds.

2. **Global Computation:** The controller, armed with a complete view of every flow and every link in the fabric, computes an optimal routing plan. It can simultaneously place TOR 1's elephant flow on Spine 1 and TOR 2's elephant flow on Spine 2, achieving a globally balanced solution that no independent hash function could guarantee.

3. **Route Programming:** The controller pushes the computed routes back into the switches' forwarding tables, overriding the default ECMP hash for the targeted flows.

MicroTE focuses its optimization on the **predictable** portion of traffic (large, stable [elephant flows](#the-elephant-flow-problem)). Short-lived [mice flows](#what-is-a-network-flow) change too rapidly for a centralized system to track, so they are left to standard ECMP, where their small size means hash collisions cause negligible harm.

<img src="../pics/microTE.png" alt="segment" width="400">


### ECMP vs. Centralized TE

| Dimension                    | ECMP (Distributed Hash-Based)                                                             | Centralized TE (MicroTE / SDN)                                                              |
|:-----------------------------|:------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------|
| **Architecture**             | Fully distributed — each switch independently hashes and forwards                         | Centralized — a controller collects demand, computes routes, and programs switches            |
| **Congestion Visibility**    | None — the hash function has no awareness of link utilization                              | Complete — the controller sees every flow and every link in the entire fabric simultaneously  |
| **Reaction Speed**           | Instantaneous — hash lookup is performed in hardware at line rate                          | Seconds (collect telemetry → compute routes → push to switches)                               |
| **Decision Quality**         | Statistical — distribution depends on flow count and header entropy, not actual demand     | Globally optimal — the controller can coordinate all switches simultaneously                  |
| **Elephant Flow Handling**   | Blind — elephant flows are pinned by hash with no regard for link load                     | Explicitly detects and places elephant flows on optimal paths                                 |
| **Microburst Handling**      | No reaction — hash assignments are static regardless of transient load                     | Poor — microbursts appear and vanish before the controller can react                          |
| **Hardware Requirement**     | Standard ECMP support (universal in modern ASICs)                                          | Standard switches with programmable forwarding tables; no special ASIC features required     |
| **Infrastructure Overhead**  | None — runs entirely within existing switch hardware                                       | Requires an external controller, telemetry pipeline, and high-availability design             |
| **Failure Mode**             | Always available — hash-based forwarding requires no external dependencies                 | Controller is a single point of failure — if it goes down, switches fall back to static ECMP  |


### Industry Adoption

In practice, centralized TE and distributed ECMP are not competing alternatives — they are complementary layers deployed together:

- **ECMP everywhere (universal):** Every modern data center fabric uses ECMP as the baseline forwarding mechanism. It handles the vast majority of traffic (mice flows) effectively with zero infrastructure overhead.

- **Centralized TE on top (hyperscale operators):** Google, Microsoft, and Meta layer centralized SDN controllers over their ECMP fabrics to optimize elephant flow placement. [Google's Jupiter and B4 networks](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43837.pdf), Microsoft's SWAN, and Meta's fabric all use centralized controllers to compute traffic-engineered paths. These operators have the engineering resources to build and maintain the controller infrastructure, telemetry pipelines, and high-availability systems that centralized TE demands.

- **Smaller operators:** Most enterprise and mid-scale data centers rely on ECMP alone (with hash tuning and resilient hashing) and accept the statistical imperfections. The engineering cost of building and operating a centralized TE system outweighs the throughput gains for fabrics below hyperscale.
