# Web Communication Technologies
## A Starting Guide

> **Important:**
> Classify communication technologies by the problem they solve, not by their popularity or product name.
>
> Start with the **communication pattern**, then identify the **interaction/API model**, **data representation**, **application or messaging protocol**, **transport**, and **network path**. Finally, address the cross-cutting concerns that apply throughout the design.
>
> These are complementary design dimensions, not competing alternatives and not a strict protocol stack. 
>
> A technology may span several dimensions: for example, gRPC combines an RPC framework, a serialization format, and an HTTP-based transport; WebSocket combines a communication capability with a protocol; and QUIC combines transport and security-related functions.

---

## 0. The Big Questions

| # | Design question | Category |
|---:|---|---|
| 1 | Who initiates communication, in which direction, and with what delivery model? | 1 Communication patterns |
| 2 | How does an application invoke an operation or exchange a message? | 2 API and interaction models |
| 3 | How is the payload represented and encoded? | 3 Data representation |
| 4 | Which application-level rules define message exchange, routing, or session behavior? | 4 Application and messaging protocols |
| 5 | How are data units delivered between endpoints? | 5 Transport |
| 6 | How are communication secured, made resilient, and operated? | 6 Cross-cutting concerns |

**Typical design reasoning order**

```text
Communication pattern
        ↓
API / interaction model
        ↓
Data representation
        ↓
Application / messaging protocol
        ↓
Transport and socket addressing
        ↓
Cross-cutting concerns
```

This is a reasoning sequence, not a mandatory implementation order. In practice, constraints such as browser support, infrastructure, latency, security policy, and operational capability may influence earlier choices.

---

## 1. Communication Patterns

> **Question:** How do applications exchange information? Consider initiation, direction, coupling, and delivery semantics before selecting a technology.

### 1.1 Request–response

```text
Client ───── Request ─────▶ Server
Client ◀──── Response ──── Server
```

- A caller requests an operation or resource and expects a corresponding result, status, or error.
- The caller may wait synchronously or use an asynchronous programming model. The communication pattern remains request–response either way.
- The client normally initiates each exchange.
- **Examples:** HTTP APIs, SOAP request–response operations, unary gRPC, JSON-RPC.
- **Use when:** the caller needs a result to continue, such as retrieving a student record or submitting a payment request.

### 1.2 Message queue — point-to-point work distribution

```text
Producer ─────▶ [ Queue: M1 | M2 | M3 ] ─────▶ Consumer
                                              (one worker handles each message)
```

- A producer places work on a queue; a consumer processes it independently of the producer's request lifecycle.
- In the common **competing-consumer** model, one consumer in a consumer group receives a given queue item for processing. Multiple consumers can share the workload.
- Acknowledgements, retries, visibility timeouts, dead-letter handling, persistence, and redelivery depend on the broker and its configuration.
- A queue does **not** inherently mean exactly-once processing, nor does it imply that messages can never be retained or replayed.
- **Examples:** RabbitMQ queues, Amazon SQS queues, Azure Queue Storage.
- **Use when:** distributing background jobs, email delivery, report generation, or image processing across workers.

### 1.3 Message bus / publish–subscribe — one publication, multiple subscribers

```text
Publisher ─────▶ Message bus / broker
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Subscriber A Subscriber B Subscriber C
```

- A publisher sends a message or event to a logical destination; the bus or broker routes it according to topics, subscriptions, bindings, or routing rules.
- Multiple independent subscribers can receive the publication. In brokered systems, each subscriber commonly has its own queue or subscription endpoint.
- The publisher does not normally wait for each subscriber to finish processing.
- **Publish–subscribe** is the messaging interaction pattern. A **message bus** is an architectural integration approach that provides shared routing and distribution; it is not a specific wire protocol.
- **Examples:** RabbitMQ topic/fanout exchanges with separate queues, cloud pub/sub services, MQTT topics.
- **Use when:** several services must react independently to the same business event, such as `StudentRegistered` being consumed by attendance, billing, and notification services.

> **Queue versus bus:** A work queue commonly distributes each item among competing workers. A pub/sub design distributes a publication to multiple subscriber destinations. A broker can support both models; the distinction is the intended delivery topology, not simply the broker's brand.

### 1.4 Event stream / append-only log

```text
Producer ─────▶ Event log: [ E1 ][ E2 ][ E3 ][ E4 ][ E5 ] ─────▶
                              │             │
                              ▼             ▼
                       Consumer group A  Consumer group B
                         offset 2          offset 4
                         (re-read)         (independent position)
```

- Events are appended to a retained, ordered log, commonly partitioned for scale.
- Consumers track their own positions (offsets) and can often replay retained events by moving their positions backwards.
- In systems such as Kafka, consumers within the **same consumer group** share partitions; separate groups can read the same topic independently.
- Ordering is generally guaranteed only within the relevant partition or defined stream scope, not automatically across every partition.
- **Examples:** Apache Kafka, Amazon Kinesis Data Streams, Azure Event Hubs.
- **Use when:** event history, replay, multiple independent processing applications, stream processing, or audit-oriented pipelines are important.

> **Important distinction:** An event stream is more than a queue with many readers. Its retained log and independently managed consumer positions are central design properties. Retention and replay limits are platform-specific.

### 1.5 Streaming communication — continuous data in one or both directions

Streaming describes how data is exchanged over an ongoing interaction. It is useful to distinguish the direction of the stream.

**Client-to-server streaming**

```text
Client ─── Data 1 ───▶ Server
Client ─── Data 2 ───▶ Server
Client ─── Data 3 ───▶ Server
```

- The client sends a sequence of data items over one logical request or stream.
- **Examples:** client-streaming gRPC; streaming request bodies in HTTP; continuous telemetry uploads.
- **Use when:** a client uploads a large or incremental data set, telemetry, or a sequence of measurements.

**Server-to-client streaming**

```text
Server ─── Data 1 ───▶ Client
Server ─── Data 2 ───▶ Client
Server ─── Data 3 ───▶ Client
```

- The server sends multiple data items over one response or stream.
- **Examples:** server-streaming gRPC; HTTP response streaming; Server-Sent Events (SSE).
- **Use when:** a client needs live updates, progress notifications, logs, or AI-generated output delivered incrementally.

**Bidirectional streaming / live session**

```text
Client ◀──────── persistent logical session ────────▶ Server
       ── data ──▶                         ◀── data ──
       ◀── data ──                         ── data ──▶
```

- Both endpoints can send multiple messages independently during the same interaction.
- **Examples:** bidirectional gRPC streaming and WebSocket.
- **Use when:** both sides need ongoing interaction, such as collaborative editing, multiplayer interaction, or live control.

> **SSE classification:** Server-Sent Events is not a transport protocol. It is a browser-facing server-push facility defined by the HTML standard, using the `EventSource` API and the UTF-8 `text/event-stream` format over HTTP. It is an implementation choice for the server-to-client pattern.

### 1.6 Polling and webhooks — event notification techniques

**Long polling**

```text
Client ───── HTTP request (held open) ─────▶ Server
Client ◀──── response when data / timeout ── Server
Client ───── next request ──────────────────▶ Server
```

- The client sends a request that the server holds until data becomes available or a timeout is reached.
- The client then issues another request.
- It can reduce repeated empty responses compared with short polling, but it still uses repeated client-initiated requests.
- **Use when:** server push is unavailable or unsuitable and the client must be notified without frequent short-interval polling.

**Webhook (server-to-server callback)**

```text
An event occurs in System A
          │
          ▼
System A ───── HTTP POST ─────▶ Callback endpoint in System B
                                      │
                                      ▼
                                System B reacts
```

- A provider sends an HTTP request to a URL registered by the receiving application when an event occurs.
- The receiving application must expose a reachable endpoint and should validate the sender, handle duplicate deliveries, and respond appropriately to retries.
- **Use when:** one system must notify another system about an event without requiring the receiver to poll continuously.

### 1.7 Choosing a communication pattern

| Requirement | Suitable starting point |
|---|---|
| Caller needs an immediate result | Request–response |
| Background work should be distributed among workers | Message queue |
| Several independent services should receive the same publication | Message bus / publish–subscribe |
| Consumers need retained history and replay | Event stream / log |
| Client continuously sends data to a server | Client-to-server streaming |
| Client needs one-way live updates | Server-to-client streaming, often SSE or HTTP streaming |
| Both endpoints need ongoing interaction | Bidirectional streaming / WebSocket; WebRTC for peer media or data use cases |
| A system should notify another system through its registered endpoint | Webhook |

Patterns can be combined. For example, a service may accept a request–response API call, publish an event to a message bus, and expose an SSE endpoint to stream processing status to a browser.

---

## 2. API and Interaction Models

> **Question:** How does one application express an operation, query, or message contract to another application?

| Technology | Classification | Main idea | Example |
|---|---|---|---|
| REST | Architectural style | Resources and representations exposed through a uniform interface; commonly implemented with HTTP semantics | `GET /students/101` |
| GraphQL | Query language and execution model | Client specifies the data fields it needs through a typed schema | `student(id: 101) { name email }` |
| gRPC | RPC framework | Clients invoke methods defined by service contracts; commonly uses Protocol Buffers and HTTP/2 | `StudentService.GetStudent(id=101)` |
| SOAP | Messaging protocol / framework | Structured SOAP messages, commonly XML-based, used to exchange operations and data | `GetStudent` SOAP request |
| JSON-RPC | RPC protocol | Remote procedure calls encoded as JSON messages | `{"method":"getStudent","params":[101]}` |
| XML-RPC | RPC protocol | Remote procedure calls encoded as XML; mainly encountered in legacy integrations | XML method call |

**The same logical operation expressed in different models**

```text
REST      : GET /students/101
GraphQL   : query { student(id: 101) { name email } }
SOAP      : SOAP envelope containing a GetStudent request
JSON-RPC  : {"jsonrpc":"2.0","method":"getStudent","params":[101],"id":1}
gRPC      : StudentService.GetStudent(id=101)
```

**Important distinctions**

- REST is an architectural style, not a protocol. RESTful web APIs commonly use HTTP, but REST and HTTP are not synonyms.
- GraphQL defines a query language, type system, and execution model. It is commonly exposed over HTTP; subscription transport varies by implementation.
- gRPC is a framework, not merely a data format. Protocol Buffers are its common interface-definition and serialization technology; HTTP/2 is its common underlying application protocol.
- SOAP defines a structured messaging model and envelope. It commonly uses XML and HTTP, but SOAP is not the same thing as HTTP.
- WebSocket is not an API style. It is a protocol used to support persistent, bidirectional communication; an application still defines its message contract.

**A common gRPC composition**

```text
gRPC service contract / RPC framework
              │
              ├── Protocol Buffers (default IDL and serialization)
              │
              └── HTTP/2
                    │
                    └── TCP
                          │
                          └── TLS (commonly used; transport security is configurable)
```

This describes a common deployment, not an unconditional rule. gRPC supports multiple RPC shapes (unary, client-streaming, server-streaming, and bidirectional-streaming), and its serialization and security configuration are not limited to one universal combination.

---

## 3. Data Representation and Serialization

> **Question:** How is the application payload represented and encoded? The format does not, by itself, determine the communication pattern, application protocol, or transport.

### 3.1 Structured data and message serialization

| Format | Representation | Typical use |
|---|---|---|
| JSON | Text-based structured data | Web APIs, configuration, event payloads |
| XML | Text-based structured data | SOAP, document-centric and enterprise integration |
| Protocol Buffers (Protobuf) | Compact binary encoding with a schema / message definition | gRPC and service-to-service messaging |
| Apache Avro | Schema-based serialization with binary and JSON encodings | Event pipelines and data platforms |
| MessagePack / CBOR | Binary serialization of structured data | Compact messages, constrained devices, interoperable APIs |
| CSV | Delimited text for tabular data | Data import/export and reporting |

### 3.2 Media and document formats

| Format | Representation | Typical use |
|---|---|---|
| JPEG / PNG | Binary image formats | Image payloads |
| MP4 | Media container format | Video and audio distribution |
| PDF | Document format | Portable document exchange |

These formats are payload formats, not general-purpose API message protocols. They are commonly carried as HTTP request or response bodies, message-broker payloads, or file-transfer content and identified using media types where applicable.

**Same logical object, different encodings**

```text
Student { id: 101, name: "Ada" }

JSON      : {"id":101,"name":"Ada"}
XML       : <student><id>101</id><name>Ada</name></student>
Protobuf  : binary encoding defined by the Student message schema
```

### 3.3 Serialization pipeline

```text
Application object
       │
       ▼
Serialize / encode ──▶ bytes ──▶ application protocol / transport ──▶ bytes
                                                                    │
                                                                    ▼
Application object ◀── Deserialize / decode ◀───────────────────────┘
```

**Key considerations**

- **Schema and compatibility:** Protobuf and Avro use schemas to define message structure. They support controlled schema evolution, but compatibility depends on following their field, default, and reader/writer compatibility rules.
- **Content type:** In HTTP, `Content-Type` identifies the representation being sent, while `Accept` communicates acceptable response media types. These headers help endpoints negotiate and interpret representations; they do not select the transport.
- **Encoding versus compression:** Serialization defines how structured values are encoded. Compression (for example, gzip or Brotli) is a separate transformation that may be applied to the encoded representation.

---

## 4. Application and Messaging Protocols

> **Question:** What rules define messages, framing, operations, routing, or sessions at the application level?

### 4.1 Web and API protocols

| Protocol | Purpose | Common underlying transport |
|---|---|---|
| HTTP/1.1 | Request–response messaging for web resources and APIs | TCP |
| HTTP/2 | HTTP semantics with binary framing and multiplexed streams | TCP |
| HTTP/3 | HTTP semantics mapped over QUIC | QUIC over UDP |
| WebSocket | Framed, bidirectional messaging over a persistent connection | TCP in the classic HTTP/1.1 deployment; also bootstrappable over HTTP/2 and HTTP/3 |

**HTTP evolution**

```text
HTTP/1.1  ──▶ TCP
HTTP/2    ──▶ TCP  (multiplexed HTTP streams within one connection)
HTTP/3    ──▶ QUIC ──▶ UDP
```

- HTTP/2 multiplexes concurrent streams over one TCP connection. Because TCP delivers an ordered byte stream, packet loss can still delay data for multiple HTTP/2 streams sharing that connection.
- HTTP/3 maps HTTP semantics onto QUIC. QUIC provides independently flow-controlled streams, transport security using the TLS 1.3 handshake, and connection migration support.
- HTTP/2 is not an alternative to REST. A RESTful API can use HTTP/1.1, HTTP/2, or HTTP/3, subject to the API's deployment and client/server support.

**WebSocket connection establishment**

For the classic HTTP/1.1 handshake:

```text
Client ─── HTTP GET + Upgrade: websocket ───▶ Server
Client ◀── HTTP 101 Switching Protocols ──── Server
Client ◀════════ WebSocket frames ═════════▶ Server
                  (bidirectional)
```

After the HTTP/1.1 upgrade succeeds, the connection carries WebSocket frames. With HTTP/2 and HTTP/3, WebSocket bootstrapping uses the Extended CONNECT mechanism rather than the HTTP/1.1 `101 Switching Protocols` exchange.

### 4.2 Messaging protocols and platforms

| Technology | Classification | Purpose | Common transport |
|---|---|---|---|
| MQTT | Messaging protocol | Lightweight broker-based publish–subscribe messaging, commonly used in IoT | TCP; WebSocket bindings are also used |
| AMQP | Messaging protocol family / standard | Message transfer, routing, queues, and broker interoperability | Commonly TCP |
| Apache Kafka | Event-streaming platform | Durable, partitioned event logs, producer/consumer APIs, and stream processing | Kafka's native client-broker protocol over TCP |

**Conceptual examples**

```text
MQTT:
Device ── publish to topic ──▶ MQTT broker ──▶ topic subscribers

AMQP 0-9-1 with RabbitMQ:
Producer ──▶ Exchange ── routing rules ──▶ Queue(s) ──▶ Consumer(s)

Kafka:
Producer ──▶ Topic partitions (retained event log) ◀── Consumers / groups
```

**Important distinctions**

- MQTT and AMQP are messaging protocols. They define rules for exchanging messages between clients and brokers or messaging peers.
- AMQP exists in multiple versions, including AMQP 1.0 and AMQP 0-9-1. RabbitMQ commonly uses AMQP 0-9-1 by default; do not assume every AMQP version has identical topology or wire behavior.
- In RabbitMQ's AMQP 0-9-1 model, exchanges route published messages to queues or streams using exchange type and binding rules. Separate subscriber queues can implement fan-out; consumers sharing one queue commonly implement competing-consumer work distribution.
- Kafka is a platform, not simply another name for a message queue or a messaging protocol. Its native wire protocol is used by clients and brokers; the event log, partitions, offsets, and consumer groups are central to its model.
- Kafka payloads may use JSON, Avro, Protobuf, or other formats. A schema registry and a particular serialization format are optional design choices, not built-in requirements of every Kafka topic.

### 4.3 Naming and service discovery

| Mechanism / technology | Role | Notes |
|---|---|---|
| DNS | Resolves domain names to records, commonly including IP addresses | DNS is an application-level protocol; it commonly uses UDP and also supports TCP |
| DNS over TLS (DoT) / DNS over HTTPS (DoH) | Protect DNS exchanges in transit | DoT uses TLS; DoH maps DNS queries to HTTP exchanges |
| Service discovery | Helps clients find available service instances or endpoints | Implemented through mechanisms such as Kubernetes DNS, Consul, or etcd-backed registries |

```text
Application hostname
       │
       ▼
DNS resolver ──▶ IP address / DNS records
       │
       ▼
Network routing and transport connection
       │
       ▼
Service endpoint
```

DNS name resolution and service discovery are related but not identical. DNS resolves names to records; a service-discovery system may additionally track service health, instance registration, metadata, and dynamic endpoint changes.


## 5. Transport

> **Question:** What delivery service does the application use between endpoints?

Transport protocols provide services such as connection management, delivery, ordering, flow control, congestion control, and multiplexing. They do not all provide the same guarantees.

| Protocol | Responsibility | Key distinction |
|---|---|---|
| TCP | Reliable, ordered byte-stream transport | Provides a continuous byte stream; it does not preserve application message boundaries |
| UDP | Datagram transport | Preserves datagram boundaries but does not guarantee delivery, ordering, or retransmission |
| QUIC | Secure, multiplexed transport over UDP | Provides reliable streams, optional datagram support, per-stream flow control, connection migration, and TLS 1.3-based security |

### 5.1 TCP framing: the message-boundary issue

```text
Application writes:  [MSG-A] [MSG-B] [MSG-C]

TCP delivers:        [M S G - A M S G - B M S G - C]
                     (an ordered byte stream)

Application must:    recover message boundaries using an application rule
                     such as a length prefix, delimiter, or fixed-size frame
```

TCP may split one application write across multiple reads, or return bytes from multiple writes in one read. Applications must implement or use a framing protocol when they need discrete messages.

UDP, by contrast, preserves datagram boundaries, but applications must handle loss, duplication, reordering, and size constraints where relevant.

### 5.2 Common protocol-to-transport mappings

```text
HTTP/1.1  ──▶ TCP
HTTP/2    ──▶ TCP
HTTP/3    ──▶ QUIC ──▶ UDP
WebSocket ──▶ TCP (classic deployment); can also use HTTP/2 or HTTP/3 Extended CONNECT
MQTT     ──▶ TCP (or a WebSocket binding)
AMQP     ──▶ commonly TCP

gRPC     ──▶ HTTP/2 ──▶ TCP (TLS commonly enabled)
```


### 5.3 Socket addressing: reaching an application on a machine

A network connection must identify both **where the destination is** and **which application endpoint should receive the data**. IP addresses and port numbers provide these two levels of addressing.

```text
Destination IP address  ──▶ identifies a network interface / host endpoint
Destination port       ──▶ identifies the application service endpoint
Transport protocol     ──▶ TCP or UDP (or QUIC over UDP)

Example: 203.0.113.20:443 over TCP (illustrative address)
         IP address                  port
```

- **IP address:** IPv4 or IPv6 addressing allows packets to be routed toward a destination interface. An IP address identifies a network interface or endpoint, not an application. IP networks are described using prefixes (CIDR notation), such as `/24`, which identify the network portion used for routing and subnetting. A public IP address is globally routable; private IPv4 ranges commonly used inside local networks are `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16`.
- **Port number:** A TCP or UDP port identifies the relevant transport-layer endpoint on that host, allowing multiple applications or services to communicate through the same IP address. Port numbers range from **0 to 65535**. The commonly defined ranges are 0–1023 (System / Well-Known), 1024–49151 (User / Registered), and 49152–65535 (Dynamic / Private).
- **Socket endpoint:** In practical application programming, a socket endpoint is commonly described by a transport protocol, IP address, and port number. A TCP connection is identified by the source IP and port together with the destination IP and port (plus the transport protocol).
- **Client and server ports:** A server commonly listens on a known port, such as TCP port 443 for HTTPS. A client typically uses an operating-system-assigned ephemeral source port for its connection.
- **Public and private addressing:** A server does not always need a public IP address assigned directly to its machine. Private IPv4 addresses are commonly used behind routers and Network Address Translation (NAT); a public address and port mapping, reverse proxy, or load balancer may expose the service externally. IPv6 can also provide globally routable addresses, subject to network and firewall configuration.

**Example connection**

```text
Client application                         Server application
10.0.0.25:53014  ───── TCP connection ───▶ 203.0.113.20:443
(source IP:port)                            (destination IP:port)

The server listens on port 443.
The client's operating system assigns source port 53014.
The example addresses are illustrative (203.0.113.0/24 is reserved for documentation).
A router/NAT may translate the client's private address and port
when the connection crosses to the public Internet.
```

> **Key idea:** The IP address gets packets to the correct network endpoint; the port directs the data to the appropriate transport/application endpoint on that host. A port does not identify an application globally, and a public IP address is not required on every device that initiates or receives communication.

**Common misconceptions**

- UDP is not automatically faster or better than TCP. It provides a different service; applications that need reliability, ordering, congestion control, or security must obtain them elsewhere.
- TCP is not a data format and does not understand application messages.
- QUIC is a transport protocol that uses UDP as its network-facing carrier. It provides its own transport mechanisms rather than inheriting TCP's byte-stream behavior.
- QUIC incorporates TLS 1.3 handshake mechanisms. It is misleading to draw HTTP/3 as `HTTP → TLS → QUIC → UDP` in the same way as HTTPS over TCP.

---

## 6. Cross-Cutting Concerns

> These concerns apply across patterns, API models, protocols, transports, and infrastructure. They should not be treated as one additional protocol layer.

### 6.1 Security, identity, and authorization

| Technology | Purpose | Scope / placement |
|---|---|---|
| TLS | Protects data in transit through encryption, integrity, and peer authentication | Commonly protects application protocols over TCP; QUIC integrates TLS 1.3 handshake mechanisms |
| HTTPS | HTTP secured using TLS or the security mechanisms of QUIC for HTTP/3 | Web and API communication |
| mTLS | Mutual TLS authentication, where both peers present certificates | Connection / peer authentication |
| OAuth 2.0 | Authorization framework for delegated access | Application authorization and token acquisition |
| OpenID Connect (OIDC) | Authentication layer built on OAuth 2.0 | User identity and authentication |
| JWT | Compact claims representation | Common token format; may be protected using JWS or JWE |
| JWS | JSON-based digital signature or MAC representation | Message/token integrity and authenticity |
| JWE | JSON-based encrypted representation | Message/token confidentiality |
| API keys | Application-specific client credentials or identifiers | Application-level access control; security depends on storage, scope, rotation, and server-side policy |

**Where transport security fits**

```text
HTTP / MQTT / other application protocol
                 │
                 ▼
       TLS over TCP, where used
                 │
                 ▼
               TCP
```

For HTTP/3, the relationship is different: HTTP/3 runs over QUIC, and QUIC incorporates the TLS 1.3 handshake and packet protection. Do not model HTTP/3 as ordinary TLS-over-TCP.

**Transport security versus message security**

```text
Transport security (TLS / QUIC security)
    Protects a communication connection or path while data is in transit.

Message security (JWS / JWE)
    Protects a message or token itself, which may remain protected after
    it is stored, forwarded, or passes through intermediaries.
```

**Non-interchangeable concepts**

- A JWT is a claims format; the term does not mean that the content is encrypted. A JWT may be represented as a JWS (signed or MAC-protected) or a JWE (encrypted).
- HTTPS protects the communication channel; it does not, by itself, decide what an authenticated user is authorized to do.
- OAuth 2.0 is an authorization framework. OIDC adds an authentication and identity layer.
- TLS peer authentication and application-level user authorization solve different problems.

### 6.2 Reliability and fault tolerance

```text
Request
   │
   ▼
Timeout / transient failure?
   │
   ├── Yes ──▶ Retry with bounded exponential backoff and jitter
   │                    │
   │                    ▼
   │             Retry budget exhausted?
   │                    │
   │                    ▼
   │          Circuit breaker / fallback / fail fast
   │
   └── No ───▶ Continue normal processing
```

Important techniques include:

- **Timeouts:** Bound how long a caller waits. A timeout does not prove that the remote operation failed; it may have completed while the response was delayed or lost.
- **Retries:** Retry only when appropriate, with bounded attempts, exponential backoff, and jitter. Uncontrolled retries can amplify an outage.
- **Idempotency:** Design operations so repeating the same request does not create unintended repeated effects. Idempotency keys can help APIs identify duplicate requests, but their behavior must be defined by the application.
- **Delivery semantics:** At-most-once, at-least-once, and exactly-once describe different delivery or processing guarantees. At-least-once delivery can produce duplicates; consumers commonly use deduplication or idempotent handling.
- **Exactly-once scope:** Exactly-once guarantees are generally limited to a defined system or transaction boundary. Exactly-once end-to-end business effects across independent services and external side effects are much harder and should not be assumed.
- **Circuit breakers:** Temporarily stop calls to a failing dependency to prevent cascading failures.
- **Bulkheads:** Isolate resources so one failing workload does not exhaust the entire application.
- **Backpressure and flow control:** Prevent a fast producer from overwhelming a slower consumer or receiver.
- **Acknowledgements and dead-letter handling:** Make message-processing outcomes explicit in broker-based systems.

### 6.3 Communication infrastructure

```text
Client
  │
  ▼
API gateway / reverse proxy
  │  ├── authentication / authorization policy
  │  ├── routing and request transformation
  │  └── rate limiting
  │
  ▼
Load balancer
  │
  ├────────▶ Service instance A
  ├────────▶ Service instance B
  └────────▶ Service instance C
```

- **API gateway:** Common entry point for API routing, authentication policy enforcement, rate limiting, and other edge concerns.
- **Load balancer:** Distributes traffic across service instances according to configured health and routing policies.
- **Reverse proxy / service mesh:** May provide routing, retries, mutual TLS, traffic policy, and telemetry. The exact responsibility depends on the architecture.

These functions are not tied to one exact physical order. A deployment may place a load balancer before a gateway, combine functions, or distribute them across proxies and services.

### 6.4 Observability

```text
Service A ── trace context ──▶ Service B ── trace context ──▶ Service C
    │                              │                              │
    ├── logs                       ├── logs                       ├── logs
    ├── metrics                    ├── metrics                    ├── metrics
    └── spans                      └── spans                      └── spans
                       │
                       ▼
                Distributed trace
```

- **Logs:** Explain what happened, ideally as structured records with useful context.
- **Metrics:** Quantify system behavior, such as request rate, error rate, latency, queue depth, and resource use.
- **Traces:** Show how one request or transaction travels across services through related spans.
- **Correlation IDs and trace context:** Help connect related operations and logs. W3C Trace Context (`traceparent`) is a standard mechanism for propagating trace context across supported components.
- **Health checks:** Help infrastructure determine whether an instance is ready to receive traffic or is alive.

Avoid using unique trace IDs as ordinary metric labels: high-cardinality metric dimensions can make metrics expensive and difficult to operate. Put trace identifiers in logs and trace data, and use exemplars or trace links where supported.

---

## 7. Worked Example — Student Registration Notification

**Scenario:** When a student registers for a course, the attendance system and notification system must react to the registration.

### 7.1 Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Communication pattern (1) | Message bus / publish–subscribe | Attendance and notification are independent subscribers to the same business event |
| Interaction model (2) | Event message, not a synchronous RPC | Registration should not wait for each downstream service to finish its work |
| Data representation (3) | JSON for simplicity, or Avro / Protobuf where schema governance is required | The event needs a stable, versioned contract |
| Messaging technology (4) | RabbitMQ with separate subscriber queues, or Kafka when retained event history and replay are required | The operational requirements determine the broker/platform choice |
| Transport (5) | Commonly TCP | Both example technologies commonly use TCP connections |
| Cross-cutting concerns (6) | TLS, publisher confirmation / acknowledgement strategy, idempotent consumers, retry policy, dead-letter handling, and trace context | Addresses security, reliability, and operations |

### 7.2 Example flow

```text
Registration service
        │
        │ Student registers
        ▼
StudentRegistered event
        │
        ├── serialized using the agreed event schema
        │
        ▼
Message broker / event platform
        │
        ├───────────────┐
        ▼               ▼
Attendance queue     Notification queue
        │               │
        ▼               ▼
Attendance service  Notification service
        │               │
        └── idempotent processing, retries, and trace context ──┘
```

With RabbitMQ, configure separate queues for the independent subscribers and bind them to an exchange. If both services consume from the **same** queue, they will normally compete for messages rather than each receiving its own copy.

With Kafka, publish the event to a topic. Attendance and notification can use separate consumer groups so each group processes the event independently. Consumers within one group share the topic's partitions; they do not each receive every record independently.

### 7.3 When an alternative is more appropriate

| Alternative | Choose it when |
|---|---|
| REST over HTTPS | The registration service needs an immediate response from a downstream service, or a simple synchronous resource-oriented interaction is sufficient |
| gRPC | A service needs a typed, contract-driven RPC interaction, potentially with streaming, between supported service environments |
| SSE | A browser needs one-way live registration-status updates from a server |
| WebSocket | A browser and server need ongoing two-way messaging |
| WebRTC | Peers need interactive real-time audio, video, or peer-to-peer data communication |

---

## Selected Standards and Primary References

The following primary references support the protocol distinctions and technical notes in this document. Product documentation is included where implementation behavior is product-specific.

1. [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
2. [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
3. [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
4. [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000.html)
5. [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
6. [RFC 8441 — Bootstrapping WebSockets with HTTP/2](https://www.rfc-editor.org/rfc/rfc8441.html) and [RFC 9220 — Bootstrapping WebSockets with HTTP/3](https://www.rfc-editor.org/rfc/rfc9220.html)
7. [WHATWG HTML — Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
8. [RFC 9293 — Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc9293.html)
9. [RFC 7766 — DNS Transport over TCP](https://www.rfc-editor.org/rfc/rfc7766.html) and [RFC 8484 — DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484.html)
10. [RFC 6749 — The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749.html) and [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
11. [RabbitMQ — Exchanges](https://www.rabbitmq.com/docs/exchanges)
12. [Confluent — Kafka Consumer Groups and Offsets](https://docs.confluent.io/platform/current/clients/consumer.html)
