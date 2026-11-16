# Web Engineering — Assignment #1 Submission
**Department:** Software Engineering (BS)  
**Course:** Web Engineering (CSC 332)  
**Submission Type:** Individual

---

## Part 1: Short Report
*(Based on `cis/Application Communication Technologies.md` — see `Part1_Report.md` for the full report.)*

### 1) Understanding of Concepts
The source document frames web communication as problem-driven design across complementary dimensions: **communication patterns** (request–response, queues/competing consumers, pub/sub message bus, event streams/logs, streaming modes, long polling/webhooks), **API/interaction models** (REST, GraphQL, gRPC, SOAP, JSON-RPC/XML-RPC), **data representation/serialization** (JSON, XML, Protobuf, Avro, binary formats, media; encoding vs compression; content negotiation), **application/messaging protocols** (HTTP/1.1–3, WebSocket, MQTT, AMQP, Kafka as a platform, DNS/service discovery), **transport & socket addressing** (TCP/UDP/QUIC, framing, IP/ports/sockets, public/private addressing), and **cross-cutting concerns** (security/identity, reliability/fault tolerance, infrastructure, observability).

**Most important concepts and significance:** (a) separating dimensions to avoid category errors and make defensible designs; (b) delivery topology/semantics (queue vs pub/sub vs retained stream, acks/retries/offsets/idempotency) for correctness; (c) layer/boundary guarantees (TCP framing, HTTP/2 vs HTTP/3/QUIC behavior, transport vs message security); (d) designing in reliability, security, and observability (timeouts/retry budgets, idempotency, trace context, clear authn/authz).

### 2) Technical Understanding
These dimensions relate as a reasoning flow (pattern → interaction model → representation → protocol → transport → cross-cutting concerns), though real constraints can feed back into choices. Key relationships include: patterns realized by suitable protocols/models (e.g., request–response over HTTP/RPC, server push via SSE/streaming, bidirectional sessions via WebSocket/gRPC streaming; pub/sub/queues/streams via brokers/platforms); models paired with representations and carried by protocols; protocols mapped onto transports with different guarantees (HTTP/1.1/2 over TCP, HTTP/3 over QUIC/UDP; TCP byte stream requiring framing vs UDP datagrams; QUIC integrating TLS 1.3 mechanisms); and cross-cutting controls spanning all layers (TLS/QUIC vs JWS/JWE, OAuth2/OIDC/JWT roles, retries/idempotency/delivery semantics, gateways/LBs, logs/metrics/traces).

**Important distinctions:** REST vs HTTP; HTTP/2 not an alternative to REST; SSE not a transport; WebSocket not an API style; queue vs pub/sub vs event log; MQTT/AMQP protocols vs Kafka platform; serialization vs compression vs transport; TCP vs UDP vs QUIC (and HTTP/3 security model); transport vs message security; IP vs port vs public addressability.

### 3) Critical Analysis
The source is technically careful and well qualified. Refinements suggested for clarity/teachability: (1) explicitly mark the design flow as iterative with constraint feedback; (2) phrase HTTP/2-over-TCP loss impact as coupled delivery/progress on a shared connection (contrast with QUIC per-stream independence); (3) note that WebSocket over HTTP/2/3 (Extended CONNECT) depends on end-to-end client/server/intermediary support; (4) add “effectively-once” (at-least-once + idempotent/deduplicated processing within a boundary) alongside scoped exactly-once guarantees; (5) briefly contextualize DNS as application-layer in the Internet suite; (6) foreground topology-misconfiguration cautions (e.g., same queue ⇒ competing consumers vs separate subscriber queues for fan-out).

> Full, detailed report: **`Part1_Report.md`** (in this folder).

---

## Part 2: Online Resume (CV Website)

### Project links (required)
- **GitHub Repository:** https://github.com/Alionit01/Web_Engineering
- **Live Website (GitHub Pages):** https://alionit01.github.io/Web_Engineering/

### Implementation summary
- **Location in repo:** `ASS1/portfolio/` (`index.html`, `style.css`, `assets/`)
- **Technologies:** Hand-written HTML and CSS (responsive layout), deployed via GitHub Pages (Actions workflow serving `ASS1/portfolio/`).
- **Content:** Personal introduction, skills, projects/work, journey/education context, source/contact information (as presented on the live site).

### Screenshots of completed CV website pages
Include screenshots of the live site (save in `screenshots/` and embed in the final `.docx`). Recommended captures:
1. `screenshots/01-home.png` — Hero / landing (top of page)
2. `screenshots/02-about.png` — About section
3. `screenshots/03-skills.png` — Skills section
4. `screenshots/04-work.png` — Work/projects section
5. `screenshots/05-journey.png` — Journey/education section
6. `screenshots/06-contact.png` — Contact/footer section

*(If image files are present in `screenshots/`, embed them here in the Word document submission. Example references: `./screenshots/01-home.png`, etc.)*

---

## Declaration / Notes
- Links are maintained for repository and live site as required.
- This package addresses the requirements stated in `ASS1/WE A1.docx` (Part 1 report and Part 2 website evidence/links). `ASS1/0-CIS-WE-F26.docx` provides course/assignment context.
