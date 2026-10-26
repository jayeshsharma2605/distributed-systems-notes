# Distributed Systems

*Distributed Systems Study Library · Author: Jayesh Sharma*

A collection of easy-to-follow study notes about distributed systems and system design, with practical examples, trade-offs, and architecture diagrams.

## Networking

- [OSI Model — All 7 Layers Explained Simply](Networking/OSI_Model_7_Layers.md) — Physical through Application layers, encapsulation, HTTPS request journey, TCP/IP comparison, troubleshooting and Mermaid diagrams.

- [IP Addressing Explained Simply — IPv4, IPv6, CIDR, Subnets and NAT](Networking/IP_Addressing_Explained.md) — IP routing, IPv4/IPv6, subnet maths, public/private IPs, VPC, NAT, DNS and service discovery.

- [TCP vs UDP — Explained Simply](Networking/TCP_vs_UDP_Explained.md) — Transport layer, TCP handshake, acknowledgments, flow control, congestion control, UDP, QUIC and HTTP/3.

- [HTTP vs HTTPS — Explained Simply](Networking/HTTP_vs_HTTPS_Explained.md) — HTTP messages, methods, status codes, TLS, certificates, HTTP/1.1, HTTP/2, HTTP/3 and HTTPS system design.

- [DNS Resolution — Explained Simply](Networking/DNS_Resolution_Explained.md) — Root, TLD and authoritative name servers, recursive/iterative lookups, caches, TTL, DNS records and propagation.

- [Forward Proxy vs Reverse Proxy — Explained Simply](Networking/Forward_Proxy_vs_Reverse_Proxy.md) — Client-side vs server-side proxies, VPN comparison, TLS termination, caching, routing, security and diagrams.

## Communication Patterns

### Real-Time Communication

- [Long Polling — Explained Simply](Communication_Patterns/Real-Time Communication/Long_Polling_Explained.md) — Regular vs long polling, held HTTP requests, timeout tuning, async servers, event brokers, cursor-based replay and retries.

- [Server-Sent Events (SSE) — Explained Simply](Communication_Patterns/Real-Time Communication/Server_Sent_Events_Explained.md) — EventSource, live streaming, Last-Event-ID, replay, scalable brokers, heartbeats, backpressure and deployments.

- [WebSockets — Explained Simply](Communication_Patterns/Real-Time Communication/WebSockets_Explained.md) — Full-duplex messaging, HTTP Upgrade, frames, horizontal scaling, Pub/Sub, recovery, heartbeats and graceful deployments.

- [Webhooks — Explained Simply](Communication_Patterns/Real-Time Communication/Webhooks_Explained.md) — HTTP callbacks, registration, secure signatures, idempotency, retries, queues, ordering and reliable delivery.

## Load Balancing

- [What Are Load Balancers? — Layer 4 vs Layer 7](Load_Balancing/What_Are_Load_Balancers.md) — Routing, OSI layers, session affinity, health checks, TLS termination, draining, availability and original Mermaid diagrams.
- [Load Balancing Algorithms — Explained Simply](Load_Balancing/Load_Balancing_Algorithms.md) — Round Robin, weighted routing, least connections/requests, EWMA, P2C, consistent hashing, locality-aware and adaptive routing.
- [DNS Load Balancing — Explained Simply](Load_Balancing/DNS_Load_Balancing.md) — DNS lookups, round-robin DNS, health checks, TTL, weighted/geographic routing and multi-region failover.

## System Design Topics

- [Distributed Unique ID Generation — UUID, Snowflake and Base62](Distributed_Unique_ID_Generation_System_Design.md) — Auto-increment and ticket servers, UUIDv4/v7, Snowflake bit layouts and clocks, ID blocks, Base62 and opaque IDs.

- [Location-Based System Design — Geohashes, Quadtrees and Nearby Search](Location_Based_System_Design_Geohash_and_Quadtrees.md) — Static versus moving location data, geospatial indexing, Redis GEO, PostGIS, nearest-driver matching, geofencing and hotspots.

- [Search and Type-Ahead (Autocomplete) — System Design](Search_and_Typeahead_System_Design.md) — Inverted indexes, analyzers, BM25 ranking, search shards, CDC indexing, prefix tries, autocomplete suggestions and caching.

- [Handling Long-Running Tasks — System Design Explained Simply](Handling_Long_Running_Tasks_System_Design.md) — Async jobs, HTTP 202, queues and workers, visibility leases, retries, checkpoints, workflow engines, progress and cancellation.

- [Preventing Double Booking — Reservation System Design](Preventing_Double_Booking_and_Reservation_System_Design.md) — Atomic seat booking, unique constraints, row locks, optimistic concurrency, expiring holds, payment recovery and high-demand ticket queues.

- [Data Synchronization, CDC and Reconciliation — Explained Simply](Data_Synchronization_CDC_and_Reconciliation.md) — Sources of truth, dual writes, outbox, Debezium CDC, Kafka ordering, backfills, reconciliation and offline conflict handling.

- [Distributed Transactions, 2PC and Sagas — Explained Simply](Distributed_Transactions_2PC_and_Sagas.md) — Atomicity across services, two-phase commit, saga orchestration/choreography, compensation, TCC reservations and transactional outbox.

- [Multi-Region System Design — Explained Simply](Multi_Region_System_Design_Explained.md) — Global routing, active-passive and active-active, replication, RPO/RTO, consistency, conflict resolution, failover and data residency.

- [Idempotency and Deduplication — Explained Simply](Idempotency_and_Deduplication_in_Distributed_Systems.md) — Safe retries, API idempotency keys, atomic uniqueness, message consumers, Kafka, Bloom filters and content hashing.

- [Video and Audio Streaming at Scale — Explained Simply](Video_and_Audio_Streaming_at_Scale.md) — Encoding ladders, HLS/DASH, ABR playback, CDNs, live streaming, DRM, playback metrics and system design interviews.

- [Large File Uploads and Downloads — Explained Simply](Large_File_Uploads_and_Downloads_System_Design.md) — Object storage and metadata, direct uploads, multipart resume, checksums, processing workers, CDN, Range requests and file synchronization.

- [Fanout-on-Read vs Fanout-on-Write — Explained Simply](Fanout_on_Read_vs_Fanout_on_Write.md) — Pull vs push feeds, hybrid celebrity handling, batched queues, fanout idempotency and real-world messaging examples.

- [Designing Real-Time Updates at Scale — Explained Simply](Designing_Real_Time_Updates_at_Scale.md) — Choosing polling, SSE and WebSockets, gateway fan-out, durable replay, capacity planning and interview architecture.

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
- [CAP Theorem and PACELC](CAP_Theorem_and_PACELC.md) — Consistency, availability, partitions, CP/AP trade-offs, online-store examples, PACELC and interview revision.
