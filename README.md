
# Data Center Fabric Load Balancing

In a Leaf-Spine fabric, every leaf switch has multiple equal-cost uplinks to the spine tier, creating inherent path diversity. How the network distributes traffic across those paths — and how it adapts when conditions change — determines fabric throughput, latency, and resilience.

This project covers the full load-balancing stack, from static ECMP hashing through congestion-aware adaptive routing to the specific constraints of RDMA transport in AI training fabrics.

## Documentation and Learning Path

The documents are structured progressively — each builds on the concepts from the previous one.

- [Fabric Load Balancing](docs/01_README_LB.md): ECMP fundamentals — path discovery, hash functions, flow pinning, hash polarization, the elephant flow problem, and centralized traffic engineering.

- [ECMP Variants](docs/02_ECMP_VAR.md): The disruption problem (consistent hashing, resilient hashing), fine-grained ECMP with bank-scoped failover, and weighted ECMP.

- [Adaptive Routing](docs/03_README_ARS.md): How modern switch ASICs overcome ECMP's congestion blindness — port grading, quality scores, flowlet switching, packet spraying, global adaptive routing with upstream notifications (ARN), and the interplay with centralized TE.

- [Load Balancing for RoCEv2](docs/04_README_ROCE_LB.md): Why RDMA's single-path Reliable Connection breaks standard load balancing at AI scale — the RC ordering constraint, QP scaling, adaptive routing for RoCE, measuring balance with CoV, and cell-based switching.
