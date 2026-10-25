# Distributed Systems

A collection of easy-to-follow study notes about distributed systems and system design, with practical examples, trade-offs, and architecture diagrams.

## System Design Topics

- [Introduction to System Design](Introduction_to_System_Design.md) — Requirements, core components, photo-upload and feed flows, trade-offs, evolution and interview approach.
- [Scalability Explained — Step-by-Step System Scaling](Scaling_Systems_Step_by_Step.md) — Vertical and horizontal scaling, bottlenecks, stateless services, caching, replicas, sharding, queues and practical diagrams.
- [High Availability and Failover](High_Availability_and_Failover.md) — Uptime maths, availability nines, redundancy, active–active/passive, standby recovery, multi-zone resilience and failure isolation.
- [Reliability in Distributed Systems](Reliability_in_Distributed_Systems.md) — Correctness, MTBF/MTTR maths, error rates, redundancy, failover, graceful degradation, circuit breakers and idempotency.
- [Latency, Throughput and Bandwidth](Latency_Throughput_and_Bandwidth.md) — p50/p95/p99 latency, network delays, bit/byte maths, transfer times, TCP windows, Little's Law, bottlenecks and practical optimization.

- [Scaling Read-Heavy Traffic](Scaling_Read_Traffic_Study_Notes.md) — Database optimization, read replicas, caching, CDNs, hot keys, cache stampedes, and a Mermaid architecture diagram.
- [Scaling Write-Heavy Systems](Scaling_Write_Heavy_Systems.md) — Write optimization, batching, durable queues, sharding, hot spots, aggregation, durability and a driver-location architecture.
- [Handling Hot Keys in Distributed Systems](Handling_Hot_Keys_in_Distributed_Systems.md) — Read and write hot keys, local caching, cache replication, sharded counters, Kafka partition skew, flash-sale inventory and detection strategies.
- [Handling Traffic Spikes in Distributed Systems](Handling_Traffic_Spikes_in_Distributed_Systems.md) — Spikes, autoscaling, CDN caching, queues, waiting rooms, rate limiting, overload control and resilient flash-sale design.
- [System Design — Request Journey, APIs, Communication & Distributed Databases](Web_Request_Flow_APIs_and_Distributed_Systems.md) — DNS, TCP/UDP, HTTPS, REST APIs, idempotency, messaging, SQL/NoSQL, CAP theorem, and a notification system design exercise with Mermaid diagrams.
- [Database Scaling, Consistency, Messaging & Resilience](Database_Scaling_Consistency_Messaging_and_Resilience.md) — Replication, sharding, consistency models, queues, pub/sub, retries, DLQs, failover and circuit breakers.
- [URL Shortener — System Design Exercise](URL_Shortener_System_Design.md) — Requirements, QPS/storage estimates, APIs, read/write flows, database design and Base62 encoding.
- [Back-of-the-Envelope Capacity Estimation](Back_of_the_Envelope_Capacity_Estimation.md) — Easy-to-remember numbers, simplified RPS maths, latency, storage, bandwidth, cache sizing, server capacity and worked examples.
- [Consistent Hashing and Virtual Nodes](Consistent_Hashing_and_Virtual_Nodes.md) — Modulo hashing, hash-ring routing, server changes, vnodes, replicas, binary search and easy key-movement maths.
