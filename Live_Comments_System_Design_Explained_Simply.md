# Live Comments System Design — Explained Simply

*Distributed Systems Study Library · Author: Jayesh Sharma*

**A beginner-friendly system design guide to comments that appear live during a sports match, concert, livestream or online event.**

> The publicly accessible introduction to [AlgoMaster's Design Live Comments](https://algomaster.io/learn/system-design-interviews/design-live-comments) describes the problem and highlights latency, fanout and ordering. The detailed article is premium-only. These notes are an independently written design based on the public problem statement and established distributed-systems patterns; they do not claim to reproduce the inaccessible sections.

## Contents

- [1. What are live comments?](#1-what-are-live-comments)
- [2. Requirements](#2-requirements)
- [3. Start with the simplest design](#3-start-with-the-simplest-design)
- [4. Scalable architecture](#4-scalable-architecture)
- [5. Comment journey](#5-comment-journey)
- [6. Why WebSockets?](#6-why-websockets)
- [7. Fanout: one comment to many viewers](#7-fanout-one-comment-to-many-viewers)
- [8. Redis Pub/Sub and Kafka](#8-redis-pubsub-and-kafka)
- [9. Storage and pagination](#9-storage-and-pagination)
- [10. Ordering, duplicates and recovery](#10-ordering-duplicates-and-recovery)
- [11. Capacity estimation](#11-capacity-estimation)
- [12. Hot events and traffic spikes](#12-hot-events-and-traffic-spikes)
- [13. Failure scenarios](#13-failure-scenarios)
- [14. Security and moderation](#14-security-and-moderation)
- [15. Interview approach](#15-interview-approach)
- [16. Quick revision](#16-quick-revision)

---

## 1. What are live comments?

Imagine watching a live cricket match on an application. Thousands of people are watching the same match.

- Priya writes: "What a fantastic shot!"
- Rahul should see Priya's comment almost immediately.
- Other viewers watching the same match should also see it.
- Someone joining late should be able to load comments already posted.

This is a **live comments system**.

A comment is not a private message to one person. It belongs to an event or room, and many viewers may be interested in it.

~~~mermaid
flowchart TD
    MATCH["Live cricket match"] --> ROOM["Match comment room"]
    ROOM --> A["Priya posts a comment"]
    A --> B["Rahul sees it"]
    A --> C["Amit sees it"]
    A --> D["Thousands of other viewers see it"]
~~~

**Memory trick:** One event, one room, many viewers.

## 2. Requirements

Before selecting technologies, identify what the application must support.

### Functional requirements

| Requirement | Meaning |
|---|---|
| Post a comment | A viewer can submit text |
| View live comments | New comments appear without refreshing |
| View comment history | A late viewer can load older comments |
| Join or leave a room | A viewer subscribes to a particular event |
| Moderate comments | Remove spam or inappropriate content |

### Nonfunctional requirements

| Requirement | Meaning |
|---|---|
| Low latency | New comments should appear quickly |
| Scalability | Support many concurrent viewers |
| Availability | Continue working during server failures |
| Durability | Accepted comments should not disappear |
| Ordering | Display comments in a sensible order |
| Abuse protection | Prevent spam and excessive posting |

### Two different kinds of data

- **Durable data:** A successfully accepted comment. Store it in a database.
- **Temporary data:** Typing indicators, connection status or viewer presence. These can usually be kept in memory or a short-lived cache.

This distinction helps avoid storing unnecessary information permanently.

## 3. Start with the simplest design

A basic implementation could have a client application, a backend API and a database. When a user posts a comment, the backend stores it and returns the result.

~~~mermaid
flowchart LR
    USER["User / Browser"] --> API["Comment API"]
    API --> DB[("Comments Database")]
    DB --> API
    API --> USER
~~~

This is enough to save and retrieve comments, but it does not solve live delivery efficiently.

Suppose 10,000 people are watching the same match. If each browser repeatedly asks the server, "Is there a new comment?", the system receives a large number of unnecessary requests.

We need a way for the server to push new comments to connected viewers.

## 4. Scalable architecture

Let's introduce a few components one at a time.

~~~mermaid
flowchart TD
    CLIENT["Browsers / Mobile Apps"] --> LB["Load Balancer"]
    LB --> WS["WebSocket Gateways"]
    WS --> CS["Comment Service"]
    CS --> DB[("Comments Database")]
    CS --> OUTBOX["Outbox / Event Publisher"]
    OUTBOX --> KAFKA[("Kafka: comment events")]
    KAFKA --> WORKER["Fanout Workers"]
    WORKER --> PUB["Room Pub/Sub"]
    PUB --> WS
    WS --> CLIENT
    DB --> READ["Comment History API"]
    READ --> CLIENT
~~~

| Component | Simple responsibility |
|---|---|
| Load balancer | Distributes connections across gateway servers |
| WebSocket gateway | Maintains live client connections |
| Comment service | Validates and accepts comments |
| Database | Permanently stores accepted comments |
| Outbox / publisher | Publishes an event after the database transaction succeeds |
| Kafka | Keeps a durable stream of comment events for processing |
| Fanout worker | Finds the room and distributes the event |
| Room Pub/Sub | Notifies gateways that have viewers in that room |

**Important:** This is a practical scalable design, not the only possible architecture. A small application can begin with fewer components and introduce Kafka and separate fanout workers when scale requires them.

## 5. Comment journey

Let's follow Priya's comment: "What a fantastic shot!"

### Step 1: Connect to the event room

Priya opens the match page. Her application establishes a WebSocket connection and joins the room for that match.

### Step 2: Submit the comment

The client sends a message containing the room ID, comment text and a unique client-generated request ID.

Example:

~~~json
{
  "roomId": "match-101",
  "requestId": "req-789",
  "text": "What a fantastic shot!"
}
~~~

The server checks authentication, room membership, message size and rate limits.

### Step 3: Save the comment

The Comment Service stores the comment in the database. It should only acknowledge successful acceptance after the durable write succeeds.

### Step 4: Publish a comment event

The system publishes a CommentCreated event. A transactional outbox is one way to ensure the database write and the intention to publish are not lost between two separate operations.

### Step 5: Distribute to viewers

A fanout worker reads the event and publishes it to the relevant room channel. Gateways with connected viewers in that room receive it.

### Step 6: Display it

Each gateway sends the comment to its connected viewers through their WebSocket connections.

~~~mermaid
sequenceDiagram
    participant P as Priya
    participant G as WebSocket Gateway
    participant C as Comment Service
    participant D as Database
    participant K as Event Stream
    participant F as Fanout Worker
    participant R as Room Pub/Sub
    participant V as Other Viewers

    P->>G: Send comment
    G->>C: Validate and submit
    C->>D: Save comment + outbox event
    D-->>C: Commit successful
    C-->>P: Comment accepted
    K->>F: CommentCreated event
    F->>R: Publish to match-101
    R->>G: Notify subscribed gateways
    G->>V: Push comment over WebSocket
~~~

The comment may be acknowledged to Priya before every other viewer has received it. **Accepted** and **delivered to all viewers** are different states.

## 6. Why WebSockets?

A live comment system needs frequent server-to-client updates.

### Polling

The browser repeatedly asks: "Any new comments?"

If it asks every 2 seconds, that creates 30 requests per minute per user, even when no one has commented.

### WebSockets

A WebSocket connection stays open. The client and server can both send messages over the same connection.

~~~mermaid
flowchart LR
    subgraph POLL["Polling"]
      P1["Client"] -->|"Request"| S1["Server"]
      S1 -->|"No update"| P1
      P1 -->|"Request again"| S1
    end
    subgraph SOCKET["WebSocket"]
      C["Client"] <-->|"Persistent two-way connection"| S["Server"]
    end
~~~

| Approach | Best suited for |
|---|---|
| Short polling | Simple applications with infrequent updates |
| Long polling | Near-real-time updates without WebSockets |
| Server-Sent Events | One-way server-to-client updates |
| WebSockets | Frequent two-way real-time communication |

For live comments, WebSockets are a natural choice because users send comments and receive other people's comments.

**Remember:** WebSocket solves the connection and transport problem. It does not by itself solve persistence, room membership, fanout or recovery.

## 7. Fanout: one comment to many viewers

Fanout means distributing one event to multiple interested recipients.

Suppose 20,000 viewers are watching the same match. Priya posts one comment. That comment must reach all connected viewers of that match.

A naive design might make the Comment Service send 20,000 individual requests. This tightly couples comment processing with delivery and makes bursts difficult to handle.

Instead, use room-based fanout.

~~~mermaid
flowchart TD
    C["One comment for match-101"] --> F["Fanout service"]
    F --> R["Room: match-101"]
    R --> G1["Gateway 1"]
    R --> G2["Gateway 2"]
    R --> G3["Gateway 3"]
    G1 --> U1["Viewers connected to gateway 1"]
    G2 --> U2["Viewers connected to gateway 2"]
    G3 --> U3["Viewers connected to gateway 3"]
~~~

### What is a room?

A room is a logical group of users interested in the same event. Examples: match-101, concert-202 or webinar-303.

A gateway tracks which local connections belong to which rooms. A distributed Pub/Sub mechanism lets all relevant gateways learn about a new comment.

### Why not broadcast to everyone?

A user watching one match should not receive comments from another match. Room-based fanout avoids unnecessary traffic.

### A very large room: the hot-room problem

Imagine one major cricket final with 1 million viewers. One room becomes extremely popular.

Possible improvements:

- Spread WebSocket connections across many gateway instances.
- Partition fanout work by room or sub-room.
- Batch comments before sending them to clients.
- Apply per-user and per-room rate limits.
- Use regional gateways to keep delivery close to viewers.

Do not create one single overloaded server for a popular room.

## 8. Redis Pub/Sub and Kafka

These technologies are often confused. They solve related but different problems.

### Redis Pub/Sub

Redis Pub/Sub is useful for quickly notifying interested gateway servers.

For example, publish a new comment to channel room:match-101. Gateways subscribed to that channel receive the notification.

~~~mermaid
flowchart LR
    W["Fanout Worker"] --> R[("Redis Pub/Sub")]
    R --> G1["Gateway 1"]
    R --> G2["Gateway 2"]
    R --> G3["Gateway 3"]
~~~

**Limitation:** Redis Pub/Sub is not a durable message queue. If a subscriber is disconnected when a message is published, it can miss that message.

### Kafka

Kafka stores events in a durable, ordered log, subject to its retention and replication configuration.

It is useful for durable event processing, consumer groups, replay after a consumer failure, partition-level ordering and buffering when downstream processing is slower.

~~~mermaid
flowchart LR
    C["Comment Service"] --> K[("Kafka Topic: comments")]
    K --> F1["Fanout Worker Group"]
    K --> A["Analytics Consumer"]
    K --> M["Moderation Consumer"]
~~~

| Redis Pub/Sub | Kafka |
|---|---|
| Fast live notification | Durable event stream |
| No built-in message history for disconnected subscribers | Retained messages can be replayed |
| Useful for gateway notification | Useful for asynchronous processing |
| Subscriber can miss a message while disconnected | Consumer can resume from an offset |

A common design uses Kafka for durable processing and Redis Pub/Sub for fast gateway notification. They are complementary, not interchangeable.

## 9. Storage and pagination

Live delivery is only half the problem. A viewer joining late must see previous comments.

A simple comment record might contain:

| Field | Purpose |
|---|---|
| comment_id | Unique comment identifier |
| room_id | Event or match identifier |
| user_id | Author |
| text | Comment content |
| sequence_number | Room-level ordering value, if assigned |
| created_at | Creation timestamp |

### Example table

~~~sql
CREATE TABLE comments (
    comment_id       VARCHAR(64) PRIMARY KEY,
    room_id          VARCHAR(64) NOT NULL,
    user_id          VARCHAR(64) NOT NULL,
    sequence_number  BIGINT NOT NULL,
    text             VARCHAR(2000) NOT NULL,
    created_at       TIMESTAMP NOT NULL,
    UNIQUE (room_id, sequence_number)
);
~~~

An index on (room_id, sequence_number) helps retrieve comments for one room in order.

### Pagination

Do not load all 50,000 comments when someone opens a match. Load the latest 50 comments first. When the user scrolls upward, fetch an older page.

~~~mermaid
flowchart TD
    U["Viewer opens match"] --> API["Comment History API"]
    API --> DB[("Comments Database")]
    DB --> L["Latest 50 comments"]
    L --> U
    U --> SCROLL["Viewer scrolls up"]
    SCROLL --> API2["Request older page using cursor"]
    API2 --> DB
    DB --> OLD["Previous 50 comments"]
    OLD --> U
~~~

Use **cursor-based pagination** rather than large offset values.

- First request: latest 50 comments.
- Next request: comments older than the last comment's sequence number.
- Repeat when the viewer scrolls further.

This avoids repeatedly scanning and skipping large numbers of rows.

## 10. Ordering, duplicates and recovery

Distributed systems can receive retries, network delays and concurrent writes. We need to handle these carefully.

### A. Ordering

Suppose Priya posts comment A and Rahul posts comment B almost simultaneously. Different servers may process them at slightly different times. Their arrival timestamps alone may not provide a consistent room-wide order.

A practical approach is to assign a monotonically increasing sequence number per room.

| Comment | Room sequence |
|---|---:|
| A | 501 |
| B | 502 |
| C | 503 |

The client displays comments by sequence number.

For ordering at scale, partition events by room_id in Kafka. Events with the same room key go to the same partition and retain partition order. This does not automatically establish a perfect global real-time order across all producers; the system still needs a clear sequence-assignment rule.

### B. Duplicate comments

Imagine Priya clicks Send. The server saves the comment, but the acknowledgement is lost. Her application retries.

Without protection, the same comment could be stored twice.

Use a unique requestId or idempotency key.

~~~mermaid
flowchart TD
    R["Receive comment request"] --> CHECK{"Request ID already processed?"}
    CHECK -->|"Yes"| OLD["Return previous result"]
    CHECK -->|"No"| SAVE["Save comment with unique request ID"]
    SAVE --> ACK["Return accepted result"]
~~~

A unique database constraint or an atomic idempotency record prevents duplicate acceptance.

### C. Delivery acknowledgements

Keep these states separate:

1. **Accepted:** The server has durably stored the comment.
2. **Published:** The event has been made available to the fanout pipeline.
3. **Delivered:** A connected gateway/client has received it.

Do not claim that a comment reached every viewer merely because it was saved successfully.

### D. Reconnection and missed comments

A viewer may lose internet connectivity for 10 seconds.

When reconnecting:

1. Re-establish the WebSocket.
2. Send the last sequence number received.
3. Fetch any missing comments after that sequence.
4. Resume live delivery.
5. Deduplicate using comment ID or sequence number.

~~~mermaid
sequenceDiagram
    participant U as Viewer
    participant G as Gateway
    participant API as History API
    participant DB as Database

    U->>G: Reconnect, last sequence = 501
    U->>API: Fetch comments after 501
    API->>DB: Query sequence > 501
    DB-->>API: Comments 502 to 506
    API-->>U: Return missing comments
    G-->>U: Continue live WebSocket updates
~~~

**Memory trick:** Persist first, publish reliably, deduplicate retries, and replay after reconnect.

## 11. Capacity estimation

Let's use an illustrative example. These are exercise assumptions, not actual traffic figures.

Assume:
- 100,000 concurrent viewers.
- 1% of viewers post one comment every 10 seconds.
- Each comment is approximately 500 bytes before protocol overhead.

### Step 1: Calculate comment writers

100,000 × 1% = **1,000 active writers**.

### Step 2: Calculate incoming comments per second

1,000 ÷ 10 = **100 comments per second**.

### Step 3: Calculate outgoing deliveries

Each comment goes to 100,000 viewers:

100 × 100,000 = **10,000,000 viewer-deliveries per second** if every comment is sent individually to every connected viewer.

### Step 4: Estimate payload bandwidth

10,000,000 × 500 bytes = 5,000,000,000 bytes/second, or approximately **5 GB/second** of comment payload, before WebSocket framing, network and other overhead.

### What does this tell us?

The incoming write rate is only 100 comments per second, but the outgoing delivery rate is enormous.

This is the key challenge of live comments: **fanout can be much more expensive than accepting comments.**

Real systems reduce the cost through batching, compression, regional distribution, audience controls and rate limits. Not every viewer needs every event at the same instant.

## 12. Hot events and traffic spikes

A major final, celebrity livestream or concert can suddenly attract millions of viewers.

- **Too many connections:** Use multiple WebSocket gateway instances behind a load balancer.
- **Too many comments:** Use rate limiting and buffering. A user should not be able to send hundreds of comments per second.
- **One room receives most traffic:** Partition fanout work and distribute room subscribers across multiple gateways.
- **Clients cannot consume updates fast enough:** Use bounded buffers and backpressure. Batch messages or drop nonessential updates where appropriate.

Never silently drop a comment that the system has promised to deliver durably; the client should be able to recover it from stored history.

~~~mermaid
flowchart TD
    USERS["Sudden viewer / comment spike"] --> LB["Load Balancer"]
    LB --> G["Many WebSocket Gateways"]
    G --> RATE["Rate Limiter"]
    RATE --> Q["Durable Event Queue"]
    Q --> WORKERS["Scaled Fanout Workers"]
    WORKERS --> ROOMS["Distributed Room Channels"]
    ROOMS --> VIEWERS["Connected Viewers"]
~~~

**Important:** Autoscaling takes time. Queues, rate limits and admission control help protect the system while additional capacity starts.

## 13. Failure scenarios

| Failure | What can happen? | Recovery |
|---|---|---|
| WebSocket gateway crashes | Connected viewers disconnect | Reconnect to another gateway and replay missed comments |
| Database unavailable | New comments cannot be durably accepted | Return a clear temporary error; do not falsely acknowledge |
| Kafka consumer crashes | Fanout processing pauses | Restart consumer and resume from committed offset |
| Redis subscriber disconnects | It may miss live notifications | Reconnect and recover using durable history / event replay |
| Duplicate request | Same comment may be submitted again | Idempotency key and unique constraint |
| Popular room overload | Delivery latency increases | Partition work, batch, rate-limit and scale gateways |
| Network interruption | Client misses live events | Resume using last received sequence number |

A useful principle:

**The database is the durable source for accepted comments. The live delivery path is a fast way to distribute them.**

If a live notification is missed, the client should be able to recover from durable history.

## 14. Security and moderation

A public comment system needs protection against abuse.

- Authenticate users before accepting comments.
- Verify that the user is allowed to access the room.
- Limit comment length and validate input.
- Apply per-user and per-room rate limits.
- Detect spam and abusive content.
- Escape user-generated text before rendering it in a browser.
- Provide reporting, blocking and moderation controls.
- Protect WebSocket connections with TLS and validate the connection origin where appropriate.

For very large events, moderation may use asynchronous processing. However, the product must decide whether comments are visible immediately, held for review, or filtered before fanout.

## 15. Interview approach

When asked to design live comments, do not start by listing Kafka, Redis and WebSockets. Explain the problem first.

1. **Clarify requirements:** Ask about concurrent viewers, comments per second, acceptable latency, history, moderation and whether every viewer must receive every comment.
2. **Identify APIs:** Join room, send comment, receive live comment, get history and leave room.
3. **Draw architecture:** Start with clients, WebSocket gateways, Comment Service and database. Then introduce event streaming and fanout.
4. **Explain write path:** Validate → persist → publish event → fanout → push to viewers.
5. **Explain read path:** Load latest comments from the database using cursor-based pagination. Use WebSockets for new comments.
6. **Discuss difficult parts:** Fanout, hot rooms, ordering, duplicate requests, reconnect recovery and backpressure.
7. **Explain trade-offs:** WebSockets need connection management; Kafka adds operational complexity; Redis Pub/Sub is not durable; per-room ordering needs a sequence strategy; broadcasting to every viewer is expensive.

### Important APIs / operations

| API / operation | Purpose |
|---|---|
| Join room | Subscribe to an event |
| Send comment | Submit a new comment |
| Live comment event | Receive new comments over WebSocket |
| Get comment history | Load older comments with a cursor |
| Leave room | Stop receiving event updates |

## 16. Quick revision

| Question | Short answer |
|---|---|
| What is a live comment? | A comment shared with viewers of the same event in near real time |
| Why WebSockets? | Persistent two-way communication |
| What is a room? | A group of users interested in one event |
| What is fanout? | One comment distributed to many viewers |
| Why Kafka? | Durable event stream, buffering and replay |
| Why Redis Pub/Sub? | Fast notification to interested gateways |
| Where are comments stored? | Durable database |
| How do we handle duplicates? | Idempotency key and unique constraint |
| How do we maintain order? | Per-room sequence and partitioning by room |
| How do reconnecting clients recover? | Fetch comments after their last received cursor |
| What is the biggest scaling challenge? | Outgoing fanout, not just incoming writes |

### The entire design in one picture

~~~mermaid
flowchart TD
    USER["Users watching an event"] --> LB["Load Balancer"]
    LB --> WS["WebSocket Gateways"]
    WS --> CS["Comment Service"]
    CS --> DB[("Durable Comments DB")]
    CS --> Q["Outbox / Kafka"]
    Q --> FW["Fanout Workers"]
    FW --> RP["Room Pub/Sub"]
    RP --> WS
    DB --> HISTORY["History API / Cursor Pagination"]
    HISTORY --> USER
    WS --> USER
~~~

**Final memory trick:**

- **WebSockets** = keep viewers connected.
- **Comment Service** = validate and accept comments.
- **Database** = remember comments.
- **Kafka** = process comment events reliably.
- **Fanout** = distribute one comment to many viewers.
- **Redis Pub/Sub** = notify gateways quickly.
- **Sequence number / cursor** = maintain order and recover missed comments.

The core idea is simple: **save a comment once, distribute it to the right room, and let clients recover anything they miss.**
