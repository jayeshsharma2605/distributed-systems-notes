# HTTP vs HTTPS — Explained Simply with Diagrams

*Distributed Systems Study Library · Author: Jayesh Sharma*

**A practical beginner's guide to HTTP requests, responses, TLS, certificates, HTTP versions and real-world system design.**


## Contents

- [1. What are HTTP and HTTPS?](#1-what-are-http-and-https)
- [2. How HTTP fits in the network stack](#2-how-http-fits-in-the-network-stack)
- [3. HTTP request structure](#3-http-request-structure)
- [4. HTTP response and status codes](#4-http-response-and-status-codes)
- [5. HTTP methods, safety and idempotency](#5-http-methods-safety-and-idempotency)
- [6. HTTP is stateless but apps can maintain sessions](#6-http-is-stateless-but-apps-can-maintain-sessions)
- [7. Why plain HTTP is risky](#7-why-plain-http-is-risky)
- [8. How HTTPS and TLS protect a connection](#8-how-https-and-tls-protect-a-connection)
- [9. Certificates and TLS 1.3 handshake](#9-certificates-and-tls-13-handshake)
- [10. HTTP/1.1 vs HTTP/2 vs HTTP/3](#10-http11-vs-http2-vs-http3)
- [11. HTTPS in real system architectures](#11-https-in-real-system-architectures)
- [12. Connection reuse and certificate lifecycle](#12-connection-reuse-and-certificate-lifecycle)
- [13. Worked examples and interview questions](#13-worked-examples-and-interview-questions)

---

## 1. What are HTTP and HTTPS?

Think of ordering a book online. Your browser **asks** the store for product details, and the store **replies**. **HTTP (Hypertext Transfer Protocol)** defines the structure and meaning of that conversation.

**HTTPS** uses the same HTTP request/response concepts but protects them with **TLS (Transport Layer Security)**. It isn't an unrelated replacement for HTTP.

| | HTTP | HTTPS |
|---|---|---|
| Meaning | Hypertext Transfer Protocol | HTTP protected using TLS |
| Default port | 80 | 443 |
| Transport | Commonly TCP for HTTP/1.1 and HTTP/2 | HTTP/1.1 or HTTP/2 over TLS/TCP; HTTP/3 over QUIC |
| Protection | No built-in encryption or authenticated server | TLS encryption, integrity and server authentication |
| Example | `http://example.com` | `https://example.com` |

```mermaid
flowchart LR
    B["Browser"] -->|"HTTP request"| SERVER["Web/API server"]
    SERVER -->|"HTTP response"| B
```

**Memory trick:** **HTTP is the conversation. HTTPS is the protected conversation.**

---

## 2. How HTTP fits in the network stack

HTTP is an **application-layer** protocol. It describes methods, resource paths, headers, and message content. To transport its bytes, it relies on lower layers.

```mermaid
flowchart TD
    APP["HTTP application messages"] --> V{"HTTP version"}
    V -->|"HTTP/1.1, HTTP/2"| TCP["TCP transport"]
    V -->|"HTTP/3"| QUIC["QUIC transport over UDP"]
    TCP --> IP["IP routing"]
    QUIC --> IP
```

HTTP/1.1 and HTTP/2 normally use TCP; **HTTP/3 uses QUIC over UDP**. HTTPS protects HTTP/1.1 and HTTP/2 using TLS over TCP, while QUIC integrates TLS 1.3 for HTTP/3. These differences become important when discussing latency and multiplexing.

---

## 3. HTTP request structure

An HTTP/1.1 request has a **request line**, **headers**, an empty line, and sometimes a **body**.

Here is an **illustrative** API request:

```http
POST /api/orders HTTP/1.1
Host: shop.example.com
Content-Type: application/json
Authorization: Bearer example-redacted-token

{"productId":42,"quantity":2}
```

| Part | What it tells the server |
|---|---|
| `POST` | Which action/method is being requested |
| `/api/orders` | Which resource or route is targeted |
| `HTTP/1.1` | HTTP version |
| `Host` | Requested host, important with multiple sites |
| `Content-Type` | How the body is encoded |
| `Authorization` | Credentials, when applicable |
| JSON body | Order details |

```mermaid
flowchart TD
    REQUEST["HTTP request"] --> LINE["Request line: method + path + version"]
    REQUEST --> HEADERS["Headers: metadata / credentials"]
    REQUEST --> BODY["Optional body: JSON, form, bytes"]
```

**Important:** This text representation illustrates **HTTP/1.1**. HTTP/2 and HTTP/3 use binary framing rather than sending this exact textual wire format.

A GET request typically retrieves data and does not need a body.

---

## 4. HTTP response and status codes

After processing a request, the server returns a status and often headers and a body.

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store

{"orderId":9001,"status":"created"}
```

| Family | Meaning | Common examples |
|---|---|---|
| **1xx** | Informational | 100 Continue |
| **2xx** | Success | 200 OK, 201 Created |
| **3xx** | Redirects and cache validation | 301 Moved Permanently, 304 Not Modified |
| **4xx** | Client-side request or authorization issues | 400, 401, 404, 429 |
| **5xx** | Server-side or intermediary failures | 500, 502, 503 |

**Practical rule:** Retrying an unchanged 400 request rarely helps; a temporary 503 may recover. But **not every 4xx should be treated the same**: 429 often calls for backoff, and **5xx retries also require idempotency and limits**.

```mermaid
flowchart TD
    API["HTTP response"] --> STATUS["Status line / code"]
    API --> HEAD["Headers: caching, type, metadata"]
    API --> BODY["Optional body: HTML, JSON, bytes"]
```

---

## 5. HTTP methods, safety and idempotency

HTTP methods describe what the client intends to do.

| Method | Typical purpose | Safe? | Idempotent by HTTP semantics? |
|---|---|---|---|
| **GET** | Read data | Yes | Yes |
| **POST** | Create or trigger work | No | Not inherently |
| **PUT** | Replace a resource | No | Yes |
| **PATCH** | Apply a partial modification | No | Not inherently |
| **DELETE** | Remove a resource | No | Yes |

**Safe** means the client does not *request* a state change (a server may still log or count requests). **Idempotent** means repeating the same operation has the same **intended effect on server state** as performing it once; the response codes need not be identical.

### Why this matters for distributed systems

A payment client sends a POST. The payment succeeds, but the response is lost. If it retries blindly, the customer could be charged twice.

**Idempotency keys** are an application-level pattern to associate repeated POSTs with the same business operation.

```mermaid
sequenceDiagram
    participant C as Checkout
    participant A as Payment API
    C->>A: POST /payments (key=payment-123)
    Note over A: Payment committed
    A--xC: Response lost
    C->>A: Retry POST (same key)
    A-->>C: Return original result
```

**Remember:** Transport reliability cannot prevent duplicate business operations. Retry design still matters.

---

## 6. HTTP is stateless but apps can maintain sessions

**Stateless HTTP** means each request is a self-contained exchange; HTTP itself does not require the server to remember a previous request.

Applications can still have login sessions, carts, and user preferences. They commonly send a **session cookie** or **authentication token** with requests and store necessary session information in a shared system.

```mermaid
flowchart LR
    CLIENT["Client: cookie or token"] --> LB["Load balancer"]
    LB --> A["App A"]
    LB --> B["App B"]
    A --> DB[("Shared session / user store")]
    B --> DB
```

**Why this helps scaling:** Any healthy app instance can process a request if it has the data and authorization context it needs. Stateful business logic still exists, but it does not need to depend on one machine's memory.

---

## 7. Why plain HTTP is risky

On a network, packets traverse devices and infrastructure outside the application. If an attacker can observe or interfere with **plain HTTP**, they may be able to:

- **Read** sensitive content such as login details or cookies (**eavesdropping**).
- **Change** a request or response on the way (**tampering**).
- **Pretend** to be the destination through an attack on routing or interception (**impersonation**).

```mermaid
flowchart LR
    B["Browser"] -->|"Readable HTTP"| X["Untrusted network / possible observer"]
    X -->|"Readable HTTP"| S["Website"]
```

Using HTTP for passwords, tokens, or payment information over an untrusted network is unsafe. HTTPS reduces these risks by authenticating the server and protecting data with TLS.

**Nuance:** HTTPS cannot fix a malicious endpoint, compromised browser, stolen credentials, or an application vulnerability by itself.

---

## 8. How HTTPS and TLS protect a connection

TLS gives HTTPS three essential security properties:

| Property | Simple explanation | What it prevents |
|---|---|---|
| **Confidentiality** | Encrypted content is not readable to network observers | Eavesdropping on HTTP content |
| **Integrity** | Altered encrypted records are detected and rejected | Undetected modification in transit |
| **Authentication** | Client verifies the server identity through certificate validation | Many server impersonation attacks |

```mermaid
flowchart LR
    B["Browser"] -->|"Encrypted TLS data"| N["Network observers see encrypted traffic"]
    N -->|"Encrypted TLS data"| S["Verified HTTPS endpoint"]
```

**Important:** HTTPS hides the protected HTTP content, but it does **not** hide all metadata. IP addresses, connection timing, and sometimes destination naming information can remain observable. TLS is usually **hop-to-hop between its two endpoints**, not automatically end-to-end across every internal proxy.

---

## 9. Certificates and TLS 1.3 handshake

A **digital certificate** helps a browser verify that a server is authorized to represent the requested hostname. The browser checks factors such as trust chain, validity period and hostname. The server proves possession of the private key corresponding to the certificate.

### Simplified TLS 1.3 process over TCP

1. First, **TCP connects**.
2. The client sends a **ClientHello** with supported parameters and usually a key share.
3. The server replies with **ServerHello**, its key share, and later its certificate and cryptographic proof.
4. Both derive shared traffic keys using the handshake key exchange.
5. **Finished** verification authenticates the handshake transcript.
6. Protected HTTP messages can be exchanged.

```mermaid
sequenceDiagram
    participant C as Browser
    participant S as Server
    C->>S: TCP: SYN
    S-->>C: TCP: SYN-ACK
    C->>S: TCP: ACK
    C->>S: TLS ClientHello + key share
    S-->>C: ServerHello + key share
    S-->>C: Encrypted certificate, proof, Finished
    Note over C: Validate certificate and identity
    C->>S: Finished
    C->>S: Encrypted HTTP request
    S-->>C: Encrypted HTTP response
```

**What does a “shared secret” mean?** Both endpoints can calculate compatible encryption keys without directly transmitting those secret keys as cleartext over the network.

For a typical **fresh** HTTPS-over-TCP connection, TCP setup costs approximately **one RTT**, and a full TLS 1.3 handshake generally adds approximately **one RTT** before protected application requests can be sent (details vary with resumption and transport settings).

**Remember:** A certificate is not the encryption key for every HTTP message. After handshake negotiation, the connection typically protects data using efficient **symmetric encryption**.

---

## 10. HTTP/1.1 vs HTTP/2 vs HTTP/3

HTTP versions preserve broadly the same core methods and semantics but improve how exchanges use connections.

| Version | Transport | Main improvement | Key limitation/trade-off |
|---|---|---|---|
| **HTTP/1.1** | TCP | Persistent connections can be reused | Serial response ordering on a connection; practical limits around pipelining |
| **HTTP/2** | TCP | Binary frames, multiplexed streams, header compression | TCP packet loss can hold up bytes needed across streams |
| **HTTP/3** | QUIC over UDP | Independent QUIC streams, integrated TLS 1.3, fast setup | Requires QUIC support and UDP connectivity |

```mermaid
flowchart TD
    H1["HTTP/1.1"] --> T1["TCP"]
    H2["HTTP/2"] --> T2["TCP"]
    H3["HTTP/3"] --> Q["QUIC + TLS 1.3"]
    Q --> U["UDP"]
    T1 --> IP["IP"]
    T2 --> IP
    U --> IP
```

### Head-of-line blocking, simply explained

Imagine three deliveries sharing a road. If the receiver insists on bytes arriving in one ordered sequence, a missing portion can delay delivery of later bytes.

- **HTTP/1.1:** Commonly uses serial exchanges on one connection; slow responses can hold up those queued behind them.
- **HTTP/2:** Multiplexes HTTP streams so app-layer responses can be interleaved, but **TCP** still exposes one ordered byte stream, so packet loss can stall delivery across streams.
- **HTTP/3:** QUIC gives streams independent ordering. Lost data on one stream does not necessarily prevent progress on other streams, although network congestion still affects them all.

```mermaid
flowchart LR
    LOSS["One stream loses needed data"] --> TCP["HTTP/2 over TCP: later bytes may wait"]
    LOSS --> QUIC["HTTP/3 over QUIC: other streams may progress"]
```

**Latency nuance:** QUIC can establish a fresh secure connection in roughly **one RTT**, and eligible resumption may use **0-RTT** application data, with replay-risk safeguards. Neither implies that every HTTP/3 request is always faster.

---

## 11. HTTPS in real system architectures

Production applications often place a **CDN, reverse proxy, API gateway, or load balancer** in front of application instances.

### TLS termination at the edge

A client's TLS connection may end at the edge proxy. After examining the HTTP request, the proxy chooses an internal backend.

```mermaid
flowchart LR
    C["Browser"] -->|"HTTPS/TLS connection 1"| EDGE["CDN / load balancer: TLS termination"]
    EDGE -->|"Optional TLS connection 2"| APP["Application service"]
```

Backend traffic can use plain HTTP in narrowly trusted deployments, or a **new encrypted TLS connection**, depending on the security requirements.

### Mutual TLS (mTLS)

In many service-to-service systems, **both caller and server** authenticate with certificates. This is called **mTLS** and can help establish workload identity, though authorization policy is still necessary.

```mermaid
sequenceDiagram
    participant A as Service A
    participant B as Service B
    A->>B: Start mTLS handshake
    B-->>A: Present server certificate
    A->>B: Present client certificate
    Note over A,B: Validate peer identities and negotiate encryption
    A->>B: Protected API call
```

**Important:** TLS termination in a reverse proxy does **not** mean HTTPS encryption automatically continues all the way to the application. Decide separately whether internal traffic must be re-encrypted.

---

## 12. Connection reuse and certificate lifecycle

Fresh connections have setup costs. Reusing an established connection avoids repeatedly paying for transport and security handshakes.

- Use **keep-alive** and suitable **connection pools** for repeated calls.
- Monitor connection limits and idle timeouts; blindly keeping every connection forever wastes resources.
- Use geographically close edge endpoints when appropriate to reduce network round-trip time.
- Monitor TLS certificate validity and automate renewals.

The transcript mentions **ACME**, the protocol used by services such as Let's Encrypt to automate issuance and renewal. This helps reduce outages caused by expired certificates.

```mermaid
flowchart TD
    SITE["HTTPS service"] --> CERT["Certificate configured"]
    CERT --> MON["Monitor expiry and renewal"]
    MON --> RENEW["Automated ACME renewal"]
    RENEW --> RELOAD["Deploy / activate updated certificate"]
    RELOAD --> MON
```

**Memory trick:** **Reuse connections. Rotate certificates. Protect every required hop.**

---

## 13. Worked examples and interview questions

### Example A: why connection reuse helps

Suppose network RTT is **100 ms** (illustrative).

A fresh HTTPS-over-TCP connection with 1 RTT TCP setup and 1 RTT full TLS 1.3 setup incurs about **200 ms of setup round-trip time**, excluding DNS resolution, server processing, transfer time, and additional variability. Reusing a healthy existing connection can avoid much of that setup cost.

### Example B: a safe retry

A POST to `/payments` times out after server-side processing. Repeating it without safeguards may duplicate the operation. An idempotency key lets the API recognize the same logical request and return an equivalent result **if implemented correctly**.

### Example C: what exactly does 304 mean?

A **304 Not Modified** tells the client it can reuse a valid cached representation under conditional-request semantics. It is a **3xx** status but does not mean “send the user to a different webpage.”

### Quick revision

| Topic | Remember |
|---|---|
| HTTP | Application-level request/response rules |
| HTTPS | HTTP protected by TLS |
| Default ports | HTTP 80, HTTPS 443 |
| HTTP/1.1 request | Method + path + version + headers + optional body |
| 2xx / 4xx / 5xx | Success / client or request issue / server-side issue |
| Safe | Intended not to modify server state |
| Idempotent | Repetition has same intended effect |
| TLS | Confidentiality, integrity, server authentication |
| Certificate | Helps verify intended server identity |
| TLS 1.3 | Modern handshake and session protection |
| HTTP/2 | Binary framing and multiplexing over TCP |
| HTTP/3 | HTTP over QUIC |
| TLS termination | Client TLS ends at an edge or proxy |
| mTLS | Mutual certificate-based authentication |
| ACME | Automated certificate management |

### Interview questions

1. What is the difference between HTTP and HTTPS?
2. Explain an HTTP request line, headers and body.
3. What do 2xx, 3xx, 4xx and 5xx status codes signify?
4. How is idempotency different from safety?
5. Why might retrying a POST create duplicate business work?
6. If HTTP is stateless, how does an app remember the current user?
7. Name the three main protections TLS provides.
8. What does a certificate validate in a TLS handshake?
9. Explain TLS 1.3 in simple steps.
10. Why did HTTP/2 introduce multiplexing?
11. How does HTTP/3 differ from HTTP/2?
12. What does it mean to terminate TLS at a load balancer?
13. When would you use mTLS for microservices?
14. Why do certificate renewal and connection reuse matter?

**Final takeaway:** **HTTP tells computers how to communicate. HTTPS adds an authenticated encrypted channel. Modern HTTP versions improve how efficiently these conversations travel, while your architecture must still handle retries, certificates and protection between internal services.**

---
