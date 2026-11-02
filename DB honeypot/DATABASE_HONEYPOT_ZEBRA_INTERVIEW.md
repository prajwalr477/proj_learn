# Database Honeypot — Zebra Interview Drilling

**Goal:** Explain the system confidently, justify choices, and survive follow-up questions. **Read sections 1–4 first.** Answers below are based on your uploaded project notes, not a direct audit of the repository. Several uploaded notes conflict; the final section flags them.

## 1. The answer to “Tell me about this project” (45–60 seconds)

> “I worked on a database honeypot for PostgreSQL and MySQL. The idea is to expose realistic-looking database endpoints to suspicious clients without exposing real business data, then study what they try to do. Go-based proxies speak the native database wire protocols and inspect authentication and SQL requests. Depending on the identity and routing policy, traffic can go to a backend or a FastAPI deception engine that returns realistic synthetic records. Redis maintains session state and fake identities, so actions across queries remain consistent. The proxies publish events to Redpanda; consumers reconstruct sessions, identify behavior patterns, and preserve evidence. Prometheus, Grafana, and Loki help us monitor the system and investigate activity. My main design focus was **realistic interaction, safe isolation, and useful behavioral telemetry**.”

**If asked “What problem are you solving?”** Connection attempts alone tell you *someone connected*. This system aims to reveal *what the client searched for, how it escalated, which tables it targeted, and whether it returned*. The core is **deception plus behavioral analytics**, not merely packet capture.

**Important qualification:** A honeypot does not conclusively identify a human attacker; traffic can come from scanners, testers, or misconfigured clients. Protocol fidelity makes it harder to spot as a decoy, not impossible.

## 2. Architecture you must be able to draw

```text
psql / mysql client
       |
       v
Go PostgreSQL / MySQL proxy (TCP + native wire protocol)
       |-- authentication, query parsing, policy, session ID
       |-- legitimate/backend route --> isolated physical DB backend
       `-- deceptive route -------> FastAPI deception engine
                                      |-- fake business schemas
                                      |-- session-specific responses/mutations
                                      `-- Redis session/principal state
       |
       `-- query/auth/session events --> Redpanda (Kafka API)
                                         |--> Session module (timeline + profiling)
                                         |--> Evidence store (forensic records)
                                         `--> Scaling/metrics consumers

Metrics: services --> Prometheus --> Grafana
Logs:    containers --> Promtail --> Loki --> Grafana
```

### Example from connection to dashboard

1. A client opens a TCP connection on PostgreSQL port **5432** or MySQL port **3306** and initiates the normal database protocol.
2. The **Go proxy** receives startup/authentication messages. The notes describe PostgreSQL **SCRAM-SHA-256** and MySQL **mysql_native_password** for virtual accounts. Virtual accounts are kept distinct from real backend credentials.
3. The proxy establishes a **session ID**, associates username/IP/protocol with that session, and decides whether the connection is deceptive or eligible for backend routing.
4. The client sends `SELECT * FROM employees`. For a deceptive route, the proxy asks the **FastAPI engine** for an answer. The engine uses fake business schema + Redis state to produce appropriate columns and rows; the proxy encodes these as PostgreSQL/MySQL protocol responses.
5. Query/auth/session events go to **Redpanda**; this does **not** block on slow downstream analysis. Session analysis reconstructs query history and evidence storage records actions.
6. **Prometheus** tracks rates/latency/errors; **Loki** stores searchable logs; **Grafana** displays them. On disconnect/timeout, the system finalizes the session.

**Key distinction:** The deception engine produces *logical results*; the proxy produces *protocol-correct packets*. Redpanda is for *asynchronous telemetry*, not for returning the SQL result to the client.

## 3. “Why not ...?” — the questions you are especially likely to get

| Interviewer asks | Answer to say |
|---|---|
| **Why not Wireshark/tcpdump?** | “They capture packets, which is useful for network forensics, but they do not create an interactive decoy, maintain fake query state, or automatically create session-level attacker profiles. TLS can also hide SQL payloads from passive capture. Wireshark is complementary for debugging the wire protocol.” |
| **Why not normal DB audit/query logs?** | “Audit logs capture actions on a real DB, but I do not want adversaries experimenting on production. A honeypot deliberately offers false assets and can record exploration without risking real records. Native logs can also be excellent telemetry, but they don’t provide the deception layer.” |
| **Why not just open a PostgreSQL DB with fake rows?** | “That is a valid simpler, high-interaction honeypot. It provides authentic SQL semantics, but running a real DB introduces a larger attack surface. Our proxy/engine route gives more control over what is exposed and what gets logged. Trade-off: emulating SQL faithfully is harder; a real DB may actually be more believable.” |
| **Why proxy at all?** | “A normal `psql`/`mysql` client expects a native database handshake and response packets. The proxy can preserve that interface, enforce routing and isolation, and capture context without requiring clients to use an HTTP API.” |
| **Why Go instead of Python/Node?** | “The proxy works with raw TCP, binary packets and many concurrent connections. Go offers goroutines, straightforward network I/O and memory safety with good performance. Python or Node could work; this was a trade-off for the protocol hot path.” |
| **Why FastAPI instead of Flask/Django?** | “This is a focused decision API. FastAPI provides typed request validation via Pydantic, async endpoints and OpenAPI docs. Flask can also implement it; Django’s larger web framework is unnecessary here.” |
| **Why Redis instead of PostgreSQL?** | “Session lookups and virtual mutations are frequent, short-lived keyed operations. Redis has fast hashes/sets/TTL and can share state between the proxy and engine. A relational DB could work but adds schema/query overhead for hot session state. Redis persistence still needs deliberate configuration.” |
| **Why Redpanda instead of REST calls?** | “Session scoring and evidence capture do not need to finish before the client sees its result. Events buffer work and separate producer from consumers. A synchronous chain would couple query latency to every analytics service.” |
| **Why Redpanda rather than Kafka?** | “Redpanda implements the Kafka API while being convenient for a single-machine container environment. Kafka itself is also a sound choice; I chose lower operational overhead, not because Kafka cannot do the job.” |
| **Why not RabbitMQ?** | “RabbitMQ is excellent for task delivery. Here I want retained ordered streams, independent consumer groups and replay. RabbitMQ can also support streams, so the main reason is the Kafka-style event-log model, not a claim that RabbitMQ only consumes messages once.” |
| **Why both HTTP and Redpanda?** | “Synchronous HTTP delivers the result the proxy needs *now*. Asynchronous events handle evidence/profiling *later*. Different latency requirements require different communication patterns.” |
| **Why microservices rather than a monolith?** | “The proxy, response generation, evidence ingestion and analytics have different workloads and failure requirements. Separation lets the proxy keep running if an analyzer lags. But 20+ services are costly to operate; a modular monolith would be simpler for a small deployment.” |
| **Why Prometheus/Grafana/Loki instead of just print logs?** | “Metrics show trends and alerts; structured logs answer exactly what happened in one session. Grafana correlates both. Plain logs alone make it much harder to track latency, error rate and consumer backlog.” |
| **Why Loki instead of ELK?** | “Loki is comparatively lightweight and indexes labels rather than every log field. ELK offers powerful full-text search, but at higher operational cost for our Docker deployment.” |
| **Why Docker Compose, not Kubernetes?** | “This is a single-host deployment; Compose is reproducible and easy to debug. Kubernetes is justified by multi-node availability, automated rollouts and horizontal orchestration, not automatically by having many containers.” |

## 4. Deep technical follow-ups (say the short answer; expand only when asked)

### A. Proxy, TCP, protocol and authentication

**Q: What exactly is a wire protocol?** A defined format and sequence of messages sent between DB client and server over TCP: startup, authentication, query command, typed results, and errors. TCP delivers an **ordered byte stream**, *not message boundaries*, so code must read exact lengths and handle partial reads.

**Q: PostgreSQL vs MySQL protocol?** Both speak TCP but have different startup flows, framing, authentication, commands and result packets. PostgreSQL uses client startup messages and e.g. `Q` for simple query; MySQL begins with a server handshake and uses commands such as `COM_QUERY`. Their packets are **not interchangeable**.

**Q: How does SCRAM-SHA-256 work?** Client and server exchange nonces and a salt; the client derives a salted secret via PBKDF2 and sends a proof based on HMAC. Server verifies the proof using stored derived keys; server also returns a signature. **The password itself is not sent as plaintext in the SCRAM proof.** Know: salt, nonce, 4096 iterations in your notes, StoredKey, ServerKey.

**Q: Why fake database accounts (“deceptive principals”)?** They can be authenticated virtually without adding identities to a real DB. Their use is a strong signal that someone touched a planted credential, and a repeat use can be correlated across sessions. **Do not say IP changes prove credential sharing**; NAT, VPNs and shared hosts complicate attribution.

**Q: What if the engine is down?** “A deceptive request must not silently fall through to the real backend. Our documented design includes timeout/fallback behavior and a circuit breaker; suspicious requests receive a controlled error or safe fallback.” The notes describe a **3-failures/10-second** breaker; verify in code before quoting it.

**Q: Can the proxy handle prepared statements?** PostgreSQL's **extended query** flow includes Parse, Bind and Execute; MySQL has `COM_STMT_PREPARE` and `COM_STMT_EXECUTE`. A proxy must track prepared SQL/parameters and preserve protocol state. Claim complete support only if tested with real clients.

**Q: How are query results sent back?** The engine might respond with JSON columns/rows. The proxy serializes database-native **column metadata, data rows, completion and ready/error** messages. Returning raw JSON over a PostgreSQL socket would break `psql`.

### B. Deception and database semantics

**Q: What is deterministic data?** The same defined input and state produce repeatable outputs. The uploaded design describes seeding a pseudorandom generator from session ID + normalized SQL. **Caveat:** stable data across *different* queries to the same table needs a stable underlying synthetic dataset or stored row state; query-seeded randomness alone does not provide relational consistency.

**Q: What is session consistency?** If someone `CREATE TABLE`, `INSERT`, then `SELECT`, they should see their own changes; `ROLLBACK` should undo uncommitted mutations. The notes describe per-session Redis mutations and snapshots. This is harder than returning a static random response.

**Q: How do you process arbitrary SQL and JOINs?** “We support a bounded subset needed for believable interaction; SQL classification is not a complete SQL execution engine. Complex joins, functions, schema metadata and transaction semantics are the hard fidelity limit.” Avoid claiming arbitrary SQL correctness.

**Q: What is progressive exposure/honeytoken?** Begin with plausible tables (`employees`, `departments`, `sessions`), reveal sensitive-looking ones as exploration deepens, then decoy/trap tables. Access to a trap records high-interest behavior. A honeytoken is **fake bait**, not a real secret.

**Q: If no SQL executes, is it automatically secure?** No. Protocol parsing bugs, unbounded inputs, HTTP injection into internal services, container escapes, unsafe file access and data leakage via a misrouted real session still matter. The notes' “zero risk” phrasing is too strong.

### C. Events, Redis and analytics

**Q: What is in an event?** Event type, timestamp, session ID, protocol, SQL or normalized fingerprint, authentication/query outcome and response route. `session_id` allows all actions in a session to be correlated.

**Q: Topic vs partition vs consumer group?** Topic is a named stream; partitions provide scaling and ordered append logs; a consumer group shares partitions across its members. **Different groups each independently receive the stream.** Using session ID as a partition key maintains *per-partition* ordering for a session within a topic, not universal cross-topic ordering.

**Q: What if a consumer crashes?** It can resume from its last committed offset while retained records exist. This is generally **at-least-once** processing, so an event might be handled twice; deduplicate by event ID or use idempotent updates. Do not promise zero loss or exactly-once without configuration and tests.

**Q: What is backpressure?** Producers are faster than consumers, so lag grows. Monitor consumer lag, enforce bounded buffers, scale consumers if partitions permit, and choose what to do when the broker is unavailable. Redis and Redpanda durability depend on persistence/replication settings.

**Q: Why two Redis instances?** Uploaded design separates **hot deceptive auth/session state** from **analytics/evidence state**, reducing resource contention and failure coupling. The deception Redis is described as AOF-persisted, so planted identities can survive a restart.

**Q: What does session analysis calculate?** Query count and rate, session duration, distinct tables touched, failed authentications, query diversity and exploration depth. The notes describe DBSCAN clustering on seven numeric features, with heuristics when data are sparse.

**Q: Why DBSCAN, not K-means?** DBSCAN does not require preselecting K, can flag noise and works with irregularly shaped groups. Its weaknesses are sensitivity to `eps`/`min_samples`, distance scaling and varying-density data. **Clustering does not itself label 'attacker types'**: mapping a cluster to `scanner`/`targeted` needs separate interpretation or rules.

**Q: Why use entropy?** Shannon entropy measures diversity of query fingerprints, `H = -Σ pᵢ log₂(pᵢ)`. Low values mean repeated query patterns; high values mean diverse patterns. It's one feature, **not a reliable attacker detector by itself**.

**Q: Is the scaling agent actually autoscaling Docker?** The notes describe scoring, desired replica counts and a KEDA-compatible endpoint. That is **a scaling recommendation/control interface**, not proof of production Kubernetes autoscaling. Explain the current deployment boundary clearly.

### D. Observability and operations

**Q: Counter vs gauge vs histogram?** Counter: cumulative events (`queries_total`). Gauge: current value (`active_connections`). Histogram: a distribution (`query_duration_seconds`) used for latency percentiles.

**Q: Why not use session ID in Prometheus labels?** High-cardinality labels create an exploding number of time series. Store per-session detail in events/logs/evidence instead; label metrics with bounded categories such as protocol and query type.

**Q: How do you investigate a suspicious session?** Locate the relevant time window in Grafana; check connection/auth rates in Prometheus; filter Loki logs by service and a correlated session identifier; retrieve the ordered query timeline and evidence events.

**Q: How do you contain a compromised honeypot?** Expose only the intended proxy ports; segment real backend and replay services; run least-privileged users, restrict egress, limit resources and queries, redact secrets, and rebuild compromised containers. **Docker networks alone are not a complete security boundary.**

**Q: How do you scale?** Load-balance TCP proxies while preserving per-connection affinity; run more deception engine instances behind HTTP load balancing; keep common state in Redis; increase topic partitions/consumer replicas; eventually use replicated brokers and a more durable evidence backend. First measure bottlenecks and run load tests.

## 5. Five harder counterquestions

1. **“Couldn't you just use audit logs and a firewall?”** “Yes for detection and enforcement; my aim is also to learn post-connection intent without granting access to real data. These tools complement rather than replace each other.”
2. **“Isn't a proxy easier to fingerprint than genuine PostgreSQL?”** “Yes, incomplete query semantics and mismatched server behavior are weaknesses. Native protocol support and consistent synthetic state reduce obvious signals, but I would test against different CLI/drivers rather than claim it is indistinguishable.”
3. **“What if 10,000 connections arrive?”** “Goroutines help with concurrent I/O, but maximum capacity depends on file descriptors, memory, authentication cost, engine latency and broker throughput. I'd load-test, rate-limit and bound concurrent work before stating an achieved number.”
4. **“What if events are duplicated/out of order?”** “Use immutable event IDs, session IDs, timestamps, deduplication and per-session ordering where supported. Treat consumer delivery as at-least-once and distinguish ordering inside one topic partition from ordering across topics.”
5. **“What would you improve?”** “First measure query-fidelity gaps with real clients, strengthen service TLS/network isolation, define versioned event schemas, and load-test the proxy-to-engine path. These address security and correctness before extra ML features.”

## 6. Behavioral answers (keep to ~30 seconds)

**Biggest challenge:** “Native DB authentication and packet serialization. A small mistake in message framing makes standard clients fail. I broke the handshake into steps and tested protocol responses and client interoperability.” *Only mention specific tests you personally executed.*

**Most important trade-off:** “I chose synchronous HTTP for a query decision because the client is waiting, and asynchronous Redpanda events for analysis because analytics should not delay query responses. This adds infrastructure, but isolates latency and failures.”

**What makes it different from a CRUD project?** “It deals with protocol compatibility, session state, adversarial inputs, event processing and observability rather than only exposing an API over a database.”

**Project status:** “The notes describe a deterministic core with optional AI/MITRE components disabled. I distinguish the tested core from future adaptive intelligence or Kubernetes deployment; I don't present planned modules as operational.”

## 7. Facts to **verify**, because the uploaded notes disagree

- **Ports:** Older overview says external proxy ports `5433/3307`; detailed architecture and infrastructure say proxy **5432/3306**, backend admin ports **5433/3307**. Prefer the newer detailed infrastructure only if it matches actual Compose.
- **Endpoint:** Older deception deep-dive says `POST /query`; interview/cheatsheet say `POST /decide`; auth/proxy paths have additional endpoints. Check route definitions before memorizing.
- **Service inventory:** Older overview says 18+; newer materials say **21 active, 5 dormant, plus an ephemeral init**. Do not claim the number if runtime has changed.
- **Implementation language:** An older table calls scaling-agent Python; detailed infrastructure identifies it as **Go**.
- **Data generation:** Some notes claim `hash(session_id + query)`, others use SHA-256; Python's built-in `hash()` is not necessarily stable across processes. Say “stable deterministic seed” and verify the exact function.
- **ML / auto-scaling / full SQL and transaction support:** Notes contain stronger claims than a design overview can prove. Explain intended mechanics, but only state implemented/tested behavior that you can show in code.
- **Validation counts and performance:** Avoid repeating “339 tests,” “10K connections,” guaranteed auth compatibility, or “indistinguishable from a real database” unless verified.

## Final 30-second recap

**Problem:** observe malicious database exploration without risking sensitive data.  
**Core:** native-protocol Go proxy → controlled FastAPI deception → Redis session consistency.  
**Telemetry:** Redpanda streams → session analysis/evidence → Prometheus/Loki/Grafana.  
**Main why:** more interactive and semantically useful than passive packet capture or real DB audit logs; more controllable than an exposed real database.  
**Main limitation:** SQL/protocol fidelity, safe isolation and high-load reliability require ongoing testing.
