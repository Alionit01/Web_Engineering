# Assignment 1 — Part 1: Short Report
**Course:** Web Engineering (CSC 332)  
**Student submission package:** Portfolio/resume site + this report  
**Reading basis:** `cis/Application Communication Technologies.md` ("Web Communication Technologies — A Starting Guide")

---

## 1. Understanding of Concepts

The document presents web/application communication as a set of **design decisions** rather than a menu of trendy product names. Its core idea is to classify technologies by the **problem they solve** and to reason across complementary dimensions: communication pattern, API/interaction model, data representation, application/messaging protocol, transport (and socket addressing), and cross-cutting concerns. A single technology can span several dimensions (e.g., gRPC combines an RPC framework, common Protobuf serialization, and HTTP/2 transport; QUIC combines transport behavior with TLS 1.3–based security mechanisms), so labels are not strict layers.

### Key concepts and technologies (in my own words)

- **Communication patterns (who talks, how, and with what delivery expectations):**
  - **Request–response:** a caller asks for an operation/resource and receives a result, status, or error (sync or async at the programming level). Examples include HTTP APIs, unary gRPC, SOAP operations, JSON-RPC. Best when the caller needs an answer to proceed.
  - **Message queue (point-to-point / competing consumers):** producers enqueue work for independent processing; typically one consumer in a group handles a given item, enabling load distribution. Delivery guarantees (acks, retries, dead-lettering, persistence/redelivery) are broker/configuration-dependent—queues do not automatically imply exactly-once or no replay.
  - **Message bus / publish–subscribe:** one publication is routed (by topics/bindings/rules) to multiple subscriber destinations so independent services can react to the same event. Pub/sub is the interaction pattern; a “message bus” is an architectural integration approach, not a wire protocol.
  - **Event stream / append-only log:** events are appended to a retained, often partitioned log; consumers track offsets and can replay within retention/scope. Ordering is usually per partition/defined scope. Distinct from “a queue with many readers” because retention and independent consumer positions are central (e.g., Kafka consumer groups share partitions within a group, but separate groups read independently).
  - **Streaming:** continuous exchange over an ongoing interaction—client-to-server, server-to-client, or bidirectional. SSE is highlighted as a browser-facing server-push facility (`EventSource`, `text/event-stream` over HTTP), not a transport protocol.
  - **Polling/webhooks:** long polling holds client-initiated requests until data/timeout then repeats; webhooks are server-to-server HTTP callbacks to a registered endpoint (requiring reachability, validation, duplicate handling, and retry-aware responses).

- **API and interaction models (how operations/messages are expressed):** REST as an architectural style (resources/uniform interface, commonly over HTTP but not synonymous with HTTP), GraphQL as a query language + typed execution model, gRPC as an RPC framework (commonly Protobuf + HTTP/2), SOAP as structured messaging/envelope (often XML), JSON-RPC/XML-RPC as RPC encodings. WebSocket is a protocol enabling persistent bidirectional messaging, not an API style by itself.

- **Data representation/serialization:** payload encoding (JSON, XML, Protobuf, Avro, MessagePack/CBOR, CSV, media/document formats) is separate from pattern/protocol/transport. Serialization/deserialization converts objects to bytes and back; `Content-Type`/`Accept` help interpret HTTP representations; compression is distinct from encoding. Schema-based formats support controlled evolution subject to compatibility rules.

- **Application and messaging protocols:** HTTP/1.1, HTTP/2 (multiplexed streams over TCP; head-of-line blocking at TCP layer can still affect shared connection on loss), HTTP/3 (HTTP semantics over QUIC/UDP with independent streams, TLS 1.3 handshake, migration). WebSocket provides framed bidirectional messaging after upgrade/bootstrap (101 for HTTP/1.1, Extended CONNECT for HTTP/2/3). Messaging technologies distinguished as protocols (MQTT, AMQP family/versions) vs platforms (Kafka with native protocol, partitions, offsets, groups). DNS/service discovery resolve/locate endpoints (DNS is application-level; can use UDP/TCP; DoT/DoH protect exchanges).

- **Transport and socket addressing:** TCP provides reliable ordered byte streams (no message boundaries—apps must frame), UDP preserves datagram boundaries without delivery/order guarantees, QUIC provides secure multiplexed transport over UDP (reliable streams, optional datagrams, flow control, migration). IP addresses identify network endpoints/hosts; ports identify transport/application endpoints on a host; sockets combine protocol+IP+port (TCP connections identified by 4-tuple). Public vs private addressing/NAT/proxies affect exposure but not the conceptual addressing roles.

- **Cross-cutting concerns:** security/identity/authorization (TLS/QUIC security in transit vs message security like JWS/JWE; HTTPS vs authorization; OAuth2 vs OIDC; JWT as claims container that may be signed/encrypted), reliability/fault tolerance (timeouts, bounded retries with backoff/jitter, idempotency, delivery semantics and scoped exactly-once claims, circuit breakers, bulkheads, backpressure, acknowledgements/DLQ handling), infrastructure (gateways, load balancers, proxies/meshes—order/composition varies), observability (logs/metrics/traces, correlation/trace context such as W3C `traceparent`, health checks, caution on high-cardinality metric labels).

### Most important concepts and why

1. **Classify by problem and separate the dimensions (pattern → model → representation → protocol → transport → cross-cutting).** This is the most important idea because it prevents category errors (e.g., treating REST as a protocol identical to HTTP, treating a data format as a transport, or assuming a broker name dictates delivery topology). It leads to defensible designs under real constraints (browser support, latency, security, operations).
2. **Delivery topology and semantics are design choices, not product slogans.** Distinguishing competing-consumer queues, pub/sub fan-out, and retained replayable streams (and understanding acks/retries/at-least-once/offsets) is critical for correctness in distributed systems—especially avoiding unintended duplicate effects and false exactly-once assumptions.
3. **Boundaries and guarantees differ by layer.** TCP’s byte stream vs application framing, transport security vs message security, and HTTP/2 multiplexing vs TCP-level loss effects (contrasted with HTTP/3/QUIC streams) are foundational to building interoperable, reliable, and secure systems.
4. **Cross-cutting concerns must be designed in, not bolted on.** Idempotency, timeouts/retry budgets, observability with trace propagation, and clear authn/authz boundaries determine whether a system survives partial failure and can be operated/debugged in production.

---

## 2. Technical Understanding

### How the dimensions relate

The document implies a **reasoning flow**: start from the communication need (pattern), choose how to express interactions (API/interaction model), decide how to encode payloads (representation), select application/messaging rules (protocol/platform), rely on a transport/addressing mechanism for delivery, and apply cross-cutting controls throughout. These are **complementary dimensions**: you can often change one choice while keeping others (e.g., a REST-style API over HTTP/1.1, HTTP/2, or HTTP/3; JSON or Protobuf payloads carried over HTTP or inside broker messages), but constraints couple them (browsers favor HTTP/WebSocket/SSE; gRPC commonly assumes HTTP/2; IoT may favor MQTT over TCP).

- **Patterns ↔ protocols/models:** request–response maps naturally to HTTP APIs, unary RPC, or SOAP operations; one-way server push to browsers aligns with SSE or HTTP streaming; ongoing bidirectional exchange aligns with WebSocket or bidirectional gRPC streaming (and WebRTC for peer media/data). Pub/sub, queues, and streams are realized via brokers/platforms and their protocols (AMQP/MQTT/Kafka protocol) rather than plain HTTP request–response alone.
- **Models ↔ representations:** REST is representation-oriented (media types, `Content-Type`/`Accept`); GraphQL defines a typed query/response shape typically serialized as JSON; gRPC pairs service contracts with an IDL/serialization (commonly Protobuf); SOAP/JSON-RPC/XML-RPC prescribe message encodings/envelopes.
- **Application protocols ↔ transport:** HTTP/1.1 and HTTP/2 run over TCP; HTTP/3 runs over QUIC over UDP; classic WebSocket upgrades from HTTP/1.1 over TCP (with alternate bootstrapping over HTTP/2/3). TCP needs application framing for discrete messages; UDP preserves datagrams but shifts reliability/ordering concerns upward if needed.
- **Transport ↔ addressing:** IP routing delivers to an endpoint/host; ports demultiplex to the correct service; NAT/proxies/public-private addressing shape reachability and exposure.
- **Cross-cutting concerns span all:** TLS/QUIC security protects transit at the connection/path level while JWS/JWE protect messages/tokens; retries/timeouts/idempotency interact with request–response and at-least-once messaging; observability and trace context must propagate across protocols/services.

### Important distinctions emphasized

| Distinction | Why it matters |
|---|---|
| **REST vs HTTP** | REST is an architectural style; HTTP is a protocol. RESTful APIs commonly use HTTP, but they are not the same thing. |
| **HTTP/2 vs “alternative to REST”** | HTTP/2 changes transport/framing behavior under HTTP semantics; API style (e.g., REST) is orthogonal. |
| **SSE vs transport** | SSE is a server-push mechanism/format+API over HTTP, not a transport protocol. |
| **WebSocket vs API style** | WebSocket is a bidirectional protocol; applications still define their own message contracts on top. |
| **Queue vs pub/sub vs event log** | Work distribution to competing consumers differs from fan-out to multiple subscribers and from retained, offset-based replayable streams; broker capability ≠ intended topology. |
| **MQTT/AMQP (protocols) vs Kafka (platform)** | Protocols define messaging exchange rules; Kafka’s value centers on partitioned retained logs, offsets, and consumer-group semantics (with its own client-broker protocol). |
| **Serialization vs compression vs transport** | Encoding structure, compressing bytes, and delivering bytes are separate transformations/services. |
| **TCP stream vs UDP datagrams vs QUIC** | Different delivery guarantees/boundaries; QUIC is not “TLS over TCP” drawn under HTTP/3 and not just “UDP but faster.” |
| **Transport security vs message security** | TLS/HTTPS/QUIC security protect in-transit paths; JWS/JWE can protect messages/tokens beyond the channel. Authn, authz, and identity layers (OAuth2/OIDC) are distinct concerns. |
| **IP vs port vs public addressability** | IP routes to an endpoint; port selects the service; public IP on every device is not required (NAT/proxies/load balancers common). |

The worked student-registration example ties these together: a pub/sub event pattern, asynchronous event interaction, schema-governed representation (JSON/Avro/Protobuf as needed), broker choice driven by replay/operational needs (separate queues in RabbitMQ vs separate consumer groups/topics in Kafka), TCP in common deployments, and cross-cutting TLS/acks/idempotency/retries/DLQ/trace context.

---

## 3. Critical Analysis

Overall, the document is strong: it is careful about common misconceptions, gives precise distinctions, and consistently qualifies behavior as broker-, configuration-, or scope-dependent. The following points identify areas that are **slightly unclear, easy to misread, or could be made more explicit** (with suggested corrections/clarifications), rather than fundamental errors.

1. **“Complementary design dimensions, not … a strict protocol stack” vs. diagrammed ordering**
   - *Issue/ambiguity:* The text correctly states the sequence is a reasoning order (not mandatory implementation) and notes real-world constraints may influence earlier choices. However, readers may still interpret the top-to-bottom flowchart as a layered stack where each choice strictly depends on the previous one.
   - *Reasoning:* In practice, transport/browser/infrastructure constraints (e.g., need for HTTP/3, WebSocket support, existing Kafka deployment) often drive upstream decisions; the relationship is iterative.
   - *Suggested correction/clarification:* Add an explicit note on the diagram: “Iterative: constraints at any dimension (e.g., transport/browser/broker capabilities) can feed back into earlier choices.”

2. **HTTP/2 multiplexing and TCP head-of-line blocking (HOL) phrasing**
   - *Issue/ambiguity:* The document states, “Because TCP delivers an ordered byte stream, packet loss can still delay data for multiple HTTP/2 streams sharing that connection.” This is accurate, but concise readers may overgeneralize that *all* progress halts identically in every case or miss that HTTP/2 still improves over HTTP/1.1 connection-per-request patterns in many scenarios.
   - *Reasoning:* The key point is loss-induced delay coupling at the TCP layer for streams on the same connection; stating it as “can delay delivery/processing of data for other streams on that connection until loss recovery” is slightly more precise.
   - *Suggested correction/clarification:* Rephrase to emphasize *coupled delivery on loss* at the TCP layer and contrast with QUIC’s per-stream independence, while noting HTTP/2 still multiplexes application streams over one connection.

3. **WebSocket bootstrapping over HTTP/2/3 (Extended CONNECT) — level of detail**
   - *Issue/ambiguity:* The document correctly mentions Extended CONNECT for HTTP/2/3 but gives little detail on negotiation/support variability.
   - *Reasoning:* Implementers may assume identical behavior and universal support across intermediaries/clients; in reality, support and configuration differ.
   - *Suggested correction/clarification:* Add a short qualifier: “Availability depends on client, server, and intermediary/proxy support for the relevant HTTP version and Extended CONNECT; validate deployment paths end-to-end.”

4. **Exactly-once semantics — strong caution, but could name “effectively-once”**
   - *Issue/ambiguity:* The text rightly states guarantees are generally limited to a defined system/transaction boundary and that end-to-end business effects across independent services/external side effects are much harder. Some readers still equate “exactly-once processing” marketing claims with global business exactly-once.
   - *Reasoning:* Distributed systems commonly achieve *effectively-once* outcomes via idempotent consumers, deduplication, and transactional/outbox patterns within bounded scopes.
   - *Suggested correction/clarification:* Explicitly introduce “effectively-once (at-least-once delivery + idempotent/deduplicated processing within a defined boundary)” as the practical goal in many architectures.

5. **DNS described as “application-level protocol” — contextual nuance**
   - *Issue/ambiguity:* The document classifies DNS as an application-level protocol (common in the Internet protocol suite’s practical layering). Some formal models/texts describe naming/resolution differently, which can confuse readers comparing sources.
   - *Reasoning:* The classification is reasonable and widely used; the potential confusion is terminological, not technical.
   - *Suggested correction/clarification:* Add a brief parenthetical: “(in the Internet protocol suite, DNS operates at the application layer, providing a naming/resolution service used by applications).”

6. **Queue/broker capabilities vs. delivery intent — could be even more explicit with a counterexample**
   - *Issue/ambiguity:* The doc states “A broker can support both models; the distinction is the intended delivery topology…” which is correct. A concrete misconfiguration example is given later (RabbitMQ same queue → competing consumers), but an early explicit warning would reinforce it.
   - *Reasoning:* Many production bugs come from topology misconfiguration rather than broker choice.
   - *Suggested correction/clarification:* Promote the RabbitMQ “same queue = competition” note as a general caution near the queue/bus comparison: “verify bindings/queues/subscriptions implement the intended fan-out vs work-sharing topology.”

**Conclusion:** These are refinements for precision and teachability. The document’s central guidance—classify by problem, separate dimensions, and treat guarantees/security/operations as explicit, scoped design concerns—is sound and technically well grounded.

---

## References (from the source document)
- RFCs/standards cited in `cis/Application Communication Technologies.md` (HTTP semantics/2/3, QUIC, WebSocket, TCP, DNS, OAuth2/OIDC, etc.) and product documentation (RabbitMQ exchanges, Kafka consumer groups/offsets) as listed in its “Selected Standards and Primary References” section.
- Local project artifacts: repository and live deployment links provided in `README.md` / submission document.
