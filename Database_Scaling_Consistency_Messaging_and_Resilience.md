# Database Scaling, Consistency, Messaging & Resilience

*Distributed Systems Study Library · Author: Jayesh Sharma*

## Contents
- [1. Database scaling](#1-database-scaling)
- [2. Replication and read replicas](#2-replication-and-read-replicas)
- [3. Sharding](#3-sharding)
- [4. Replication vs sharding](#4-replication-vs-sharding)
- [5. Consistency](#5-consistency)
- [6. Message queues and event-driven architecture](#6-message-queues-and-event-driven-architecture)
- [7. Retries and dead-letter queues](#7-retries-and-dead-letter-queues)
- [8. Reliability and resilience](#8-reliability-and-resilience)
- [9. Revision questions](#9-revision-questions)

## 1. Database scaling

**Database scaling** helps a database handle more users, requests and stored data without becoming too slow.

Why scale? More simultaneous reads/writes, increasing data size, slow queries, or CPU/memory/storage limits.

| Approach | Meaning | Shop analogy |
|---|---|---|
| Vertical scaling | Give one database server more CPU, RAM or storage | Make the existing shop bigger |
| Horizontal scaling | Use multiple servers to distribute the workload | Open more shop branches |

```mermaid
flowchart TB
    G["Growing application load"] --> V["Vertical: upgrade one server"]
    G --> H["Horizontal: use multiple servers"]
    V --> V1[("Larger database server")]
    H --> H1[("DB server A")]
    H --> H2[("DB server B")]
```

**Remember:** Vertical scaling increases one machine's capacity; horizontal scaling distributes work.

## 2. Replication and read replicas

**Replication** creates copies of the same records on multiple database servers. Updates are sent from one server to other servers; those copies may lag behind briefly.

### Leader–follower architecture

The **leader** accepts writes; **followers** receive copies of those changes and can serve suitable reads. If the leader fails, a follower may be promoted depending on the setup.

```mermaid
flowchart TD
    W["Order placement: WRITE"] --> L[("Leader / Primary")]
    L -->|"Replicate updates"| R1[("Follower / Read replica 1")]
    L -->|"Replicate updates"| R2[("Follower / Read replica 2")]
    Q1["Product browsing: READ"] --> R1
    Q2["Other read requests"] --> R2
```

A **read replica** takes read traffic away from the primary. It is useful when reads outnumber writes. Because replication can be delayed, critical immediate reads may need to go to the primary.

**Example:** Place an order through the primary database; browse product details through a read replica.

## 3. Sharding

**Sharding** splits records into different portions called *shards*. Each shard stores only part of the overall data.

For example, customers might be distributed by customer ID:

```mermaid
flowchart TD
    A["Application"] --> RT["Shard routing by customer ID"]
    RT -->|"IDs 1–1000"| S1[("Shard 1")]
    RT -->|"IDs 1001–2000"| S2[("Shard 2")]
    RT -->|"IDs 2001–3000"| S3[("Shard 3")]
```

The routing layer must know where to find a given record. The ranges above are illustrative.

**Analogy:** Instead of every shop holding every customer record, each branch handles a portion.

## 4. Replication vs sharding

| Question | Replication | Sharding |
|---|---|---|
| What happens? | Same data is copied | Data is divided |
| Each server holds | Copy of the records | Subset of records |
| Main purpose | Improve read capacity and availability | Spread data and workload |
| Example | Multiple copies of product catalog | Customer records divided by ID |

**You can combine both.** Each shard can have its own replica:

```mermaid
flowchart TD
    A["Application / shard router"] --> P1[("Shard 1 primary")]
    A --> P2[("Shard 2 primary")]
    P1 -->|"Replicate"| R1[("Shard 1 replica")]
    P2 -->|"Replicate"| R2[("Shard 2 replica")]
```

**Key idea:** Replication makes more copies; sharding divides the data.

## 5. Consistency

When data exists on multiple servers, **consistency** describes what users see after updates.

### Strong consistency

After an update succeeds, later reads see the latest confirmed value. This can require coordination across servers.

**Example:** A customer updates a delivery address; a later read returns the new address.

### Eventual consistency

Different copies may temporarily show different values. If updates stop, the copies are expected to converge.

**Example:** A changed profile picture may still look old on one server for a short period. “Eventually” does not specify an exact delay.

```mermaid
sequenceDiagram
    participant U as User
    participant A as Server A
    participant B as Server B
    U->>A: Update profile picture
    A-->>U: Update confirmed
    Note over A,B: Replication has not reached B yet
    U->>B: Read profile
    B-->>U: Old picture (eventual consistency)
    A-->>B: Replicate change
    U->>B: Read again
    B-->>U: New picture
```

| Aspect | Strong consistency | Eventual consistency |
|---|---|---|
| Read after confirmed update | Latest confirmed value | May temporarily be older |
| Coordination | Typically more coordination | Copies may synchronize later |
| Good fit | Stale data causes serious harm | Short delay is acceptable |

### Where is eventual consistency acceptable?

- Social media like counts
- News feeds
- Product reviews
- DNS changes
- Analytics dashboards

### Where are stronger guarantees important?

- Bank transfers and accurate balances
- Seat bookings, avoiding double allocation
- Inventory checks at checkout
- Confirmed payment status

**Design question:** What happens if a user sees an old value? The impact helps determine the right consistency guarantee.

## 6. Message queues and event-driven architecture

A **message queue** temporarily stores messages between producers and consumers.

- **Producer:** Creates a message.
- **Queue:** Holds the message until processing.
- **Consumer:** Receives, processes and acknowledges it.

```mermaid
flowchart LR
    O["Order Service: producer"] -->|"OrderPlaced"| Q[["Message queue"]]
    Q --> E["Email Service: consumer"]
    E -->|"On success"| ACK["Acknowledge message"]
```

**Example:** An online store accepts an order immediately, while the email service processes the confirmation later.

### Event-driven architecture

An event describes something that has happened. Other services react independently.

```mermaid
flowchart TD
    O["Order Service"] -->|"Publish OrderPlaced"| T[["Event topic / broker"]]
    T --> E["Email: send confirmation"]
    T --> I["Inventory: update stock"]
    T --> S["Shipping: prepare shipment"]
```

### Queue vs publish/subscribe

| Queue | Pub/Sub |
|---|---|
| Distributes a task to one consumer within a group | Sends an event to each subscribed service/group |
| Useful for background work | Useful when multiple services need the event |
| Example: one worker processes an email task | Example: email, inventory and shipping each react to an order |

```mermaid
flowchart TB
    subgraph Queue["Work queue"]
      P1["Producer"] --> Q[["Queue"]]
      Q --> W1["Worker A OR Worker B"]
    end
    subgraph PubSub["Publish / Subscribe"]
      P2["Publisher"] --> T[["Topic"]]
      T --> S1["Subscriber A"]
      T --> S2["Subscriber B"]
      T --> S3["Subscriber C"]
    end
```

**Remember:** Queues distribute work; pub/sub broadcasts events to interested subscribers.

## 7. Retries and dead-letter queues

A consumer may fail because of a temporary network issue or an unavailable dependency.

**Retry flow:**
1. Consumer receives a message.
2. Processing fails.
3. The system waits and retries.
4. The message succeeds, or the retry limit is reached.

A **Dead Letter Queue (DLQ)** stores messages that continue failing after the allowed attempts. Teams can inspect the message, investigate and decide whether to replay it.

```mermaid
flowchart TD
    Q[["Main queue"]] --> C["Consumer"]
    C --> D{"Processing successful?"}
    D -->|"Yes"| ACK["Acknowledge"]
    D -->|"No"| LIMIT{"Retry limit reached?"}
    LIMIT -->|"No"| WAIT["Wait before retrying"]
    WAIT --> C
    LIMIT -->|"Yes"| DLQ[["Dead Letter Queue"]]
    DLQ --> OPS["Investigate and decide whether to replay"]
```

**Reliability note from the presentation:** Messages may be delivered again if acknowledgement is lost. Consumers should handle duplicates safely.

## 8. Reliability and resilience

- **Reliability:** The system works correctly over time.
- **Resilience:** The system handles failures and recovers.

### Single point of failure (SPOF)

A **SPOF** is one component whose failure can stop the entire system. A single database server with no backup is a common example.

### Redundancy and failover

- **Redundancy:** Keep extra healthy components.
- **Failover:** Switch work to a healthy backup after failure.

```mermaid
flowchart TD
    U["Users"] --> LB["Load balancer"]
    LB --> A1["Application server A"]
    LB --> A2["Application server B"]
    A1 --> DB1[("Primary DB")]
    A2 --> DB1
    DB1 -->|"Replicate"| DB2[("Standby DB")]
    F["Primary failure detected"] -.-> SW["Failover procedure"]
    SW -.-> DB2
```

**Analogy:** Redundancy is having a backup; failover is switching to it.

### Timeouts and retries

A **timeout** prevents waiting too long for a response. A **retry** tries again after a failure.

Good practices from the presentation:

- Limit the number of retries.
- Wait before retrying.
- Retry only potentially temporary failures.
- Make repeated requests safe, especially for payments.

### Circuit breaker

A circuit breaker temporarily stops calls to a service that keeps failing.

- **Closed:** Requests flow normally.
- **Open:** Requests are blocked temporarily.
- **Half-open:** Allow a small number of test requests.

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Repeated failures
    Open --> HalfOpen: Cooldown expires
    HalfOpen --> Closed: Test requests succeed
    HalfOpen --> Open: Test requests fail
```

### Putting it together

A resilient service can combine multiple application servers, a standby database, timeouts, bounded retries and a circuit breaker.

```mermaid
flowchart LR
    C["Client"] --> LB["Load balancer"]
    LB --> A["App servers with timeouts and retries"]
    A --> CB["Circuit breaker"]
    CB --> DEP["Dependent service"]
    A --> DB[("Primary DB")]
    DB --> ST[("Standby DB")]
```

## 9. Revision questions

1. Why do databases need scaling?
2. What is the difference between vertical and horizontal scaling?
3. How do leader–follower replication and read replicas work?
4. How is sharding different from replication?
5. Can each shard also have replicas?
6. How do strong and eventual consistency differ?
7. Where is temporary staleness acceptable?
8. What are producer, queue and consumer?
9. How does pub/sub differ from a work queue?
10. Why do we need retries and a DLQ?
11. What are redundancy and failover?
12. What do closed, open and half-open mean in a circuit breaker?

---
