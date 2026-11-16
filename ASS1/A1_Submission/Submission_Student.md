# Web Engineering Assignment 1 Submission
**Department:** Software Engineering (BS)  
**Course:** Web Engineering (CSC 332)  
**Submission Type:** Individual

## Part 1: Short Report

This report is based on `cis/Application Communication Technologies.md` ("Web Communication Technologies — A Starting Guide"). I’ve written the full detailed report in `Part1_Report_Student.md` in this folder, but I’ve also included a clear summary here.

### Understanding of concepts

From the document, I understood that web communication technologies are better understood by the problem they solve instead of just by product names. The document looks at communication in a few connected areas: communication patterns, API or interaction models, data representation, application or messaging protocols, transport and socket addressing, and cross-cutting concerns like security and reliability. These areas help in making design decisions, but they are not strict layers that always have to be decided in one fixed order.

Some of the key ideas I took from the reading are the different communication patterns such as request–response, message queues with competing consumers, publish–subscribe/message bus setups, event streams with retained logs and offsets, streaming in different directions, and techniques like long polling and webhooks. The document also explains interaction models like REST, GraphQL, gRPC, SOAP, and JSON/XML-RPC, and it makes clear that WebSocket is a protocol for ongoing bidirectional communication rather than an API style. On top of that, it separates data representation (JSON, XML, Protobuf, Avro, binary/text/media formats, serialization vs compression) from protocols and transport. It then covers HTTP/1.1–3, WebSocket, MQTT, AMQP, Kafka as a platform, and DNS/service discovery, followed by transport details (TCP, UDP, QUIC, framing, IP/ports/sockets) and cross-cutting concerns such as security, reliability, infrastructure, and observability.

The concepts I found most important are: separating these dimensions to avoid mixing things up (like confusing REST with HTTP), understanding delivery topology and guarantees (queue vs pub/sub vs replayable streams, retries/acks/idempotency), knowing what guarantees exist at different boundaries (like TCP byte streams needing framing, or transport security vs message security), and treating reliability, security, and observability as part of the design from the start.

### Technical understanding

I see these dimensions as related steps in thinking through a design. You usually start with how systems need to communicate, then choose how to express that interaction, decide how data will be represented, pick suitable protocols, and rely on a transport and addressing setup, while keeping cross-cutting concerns in mind throughout. In practice, though, real constraints can push you to adjust earlier decisions.

For example, request–response fits well with HTTP APIs or unary RPC calls, server-to-client updates can fit SSE or streaming, and two-way ongoing communication can fit WebSocket or bidirectional streaming. Pub/sub, queues, and streams are commonly handled through brokers or platforms and their protocols. REST works around resources and representations, GraphQL uses a typed query model, and gRPC uses service contracts often with Protobuf and HTTP/2. HTTP/1.1 and HTTP/2 run over TCP, HTTP/3 runs over QUIC over UDP, and WebSocket provides framed bidirectional messaging after setup. TCP gives an ordered byte stream without message boundaries, UDP keeps datagram boundaries without delivery guarantees, and QUIC adds its own transport and security features over UDP.

Some distinctions that stood out to me are: REST vs HTTP, HTTP/2 not being an alternative to REST, SSE not being a transport protocol, WebSocket not being an API style, and the difference between queues, pub/sub fan-out, and event logs. I also found it useful to separate MQTT/AMQP as messaging protocols from Kafka as a platform, and to keep serialization, compression, and transport as different concerns. The document also clearly separates TCP/UDP/QUIC behavior, transport security from message security, and IP addressing from ports and public reachability.

### Critical analysis

Overall, I think the document is strong and explains things carefully, especially where behavior depends on configuration, broker choice, or scope. But there are a few areas where it could be even clearer.

First, even though it says the design flow is not a strict stack, the diagram/order can still feel a bit fixed. A clearer note that real constraints (like browser support, transport options, or existing infrastructure) can feed back into earlier choices would help. Second, the point about HTTP/2 over TCP is correct, but it could be phrased a bit more clearly by saying packet loss can delay delivery for other streams on the same connection until recovery, while still noting the benefit of multiplexing. Third, WebSocket over HTTP/2/3 with Extended CONNECT is mentioned briefly, so it would help to add that support depends on the full client/server/intermediary path.

I also think it would be useful to mention “effectively-once” processing (at-least-once delivery plus idempotent or deduplicated handling within a defined boundary), since the document already explains that exactly-once guarantees are usually limited in scope. The description of DNS as an application-level protocol is reasonable, but a small note about that being in the Internet protocol suite context could avoid confusion. Finally, the warning about confusing broker capability with intended delivery topology (like the same RabbitMQ queue causing competing consumers instead of fan-out) could be highlighted earlier because that’s an easy mistake to make.

For the full explanation, examples, and reasoning, please see `Part1_Report_Student.md`.

## Part 2: Online Resume (CV Website)

### Project links
- **GitHub Repository:** https://github.com/Alionit01/Web_Engineering
- **Live Website (GitHub Pages):** https://alionit01.github.io/Web_Engineering/

### Implementation summary
- **Location in repo:** `ASS1/portfolio/` (`index.html`, `style.css`, `assets/`)
- **Technologies:** I built the resume/portfolio site using hand-written HTML and CSS with a responsive layout, and published it using GitHub Pages (with a GitHub Actions workflow serving `ASS1/portfolio/`).
- **Content on the site:** It includes my personal introduction, skills, projects/work, journey/education-related sections, and source/contact information as shown on the live website.

### Screenshots of completed CV website pages
Screenshots of the live site are included in the `screenshots/` folder (and can be embedded in the final `.docx` submission):
1. `screenshots/01-home.png` — home/hero view (top of the page)
2. `screenshots/01-full-page.png` — full-page capture of the website
3. `screenshots/02-about.png` — about section (if captured separately)
4. `screenshots/03-skills.png` — skills section (if captured separately)
5. `screenshots/04-work.png` — work/projects section (if captured separately)
6. `screenshots/05-journey.png` — journey/education section (if captured separately)
7. `screenshots/06-contact.png` — contact/footer section (if captured separately)

At the moment, `01-home.png` and `01-full-page.png` are already saved in the folder. If separate section screenshots are needed for the final Word file, they can be taken from the live site and added to the same folder before embedding.

## Notes
- The GitHub repository and live website links above are kept accessible as required.
- This submission addresses the requirements given in `ASS1/WE A1.docx` (Part 1 report and Part 2 website evidence/links). `ASS1/0-CIS-WE-F26.docx` was used as course context.
