# Real-Time File Sharing Platform — Zebra Interview Master Guide

> **Priority:** Read sections 1–6, 9, 11, and 14 first. Designed for fast spoken interview revision, not a production architecture textbook.
>
> **Accuracy convention:** **Built** = described in the supplied `LEARNING_GUIDE.md`. **Proposed** = improvement you can discuss, **not** something you built. Don't confuse your earlier resume description with the current guide if the actual running version differs.

## 1. My 60-second explanation (memorize)

**Q: Explain your project.**

I built a browser-based file-sharing platform supporting two workflows. **Stored sharing** uploads files through an Express API using multipart/form-data and Multer, keeps the actual bytes on private server disk, and stores metadata and access-control token hashes in MongoDB. The owner can create, replace, revoke, or expire a shareable link, and the server verifies permission again whenever a recipient downloads. **Live sharing** uses Socket.IO only for room management and WebRTC signaling; the actual file travels as binary chunks over an encrypted WebRTC DataChannel between browsers. I implemented receiver acceptance, chunking, backpressure, completion acknowledgment, and cleanup. I designed the two modes separately because asynchronous sharing needs durable storage, whereas live sharing benefits from avoiding persistent server-side file storage.

**Q: What did you personally implement?**

Discuss concrete code you can show: Express endpoints, Multer staging, Mongoose model, file storage adapter, token permissions, React flows, Socket.IO room handling, SDP/ICE exchange, DataChannel chunk protocol, test cases. Do not claim cloud deployment, TURN, multi-node scaling, or resumable transfer as built.

**Q: What was the hardest part?**

A defensible example is correctly sequencing asynchronous WebRTC signaling (ICE candidates sometimes arrive before remote SDP) and handling DataChannel backpressure and transfer success semantics. Explain how queued ICE candidates, `bufferedAmount` thresholds, and a receiver acknowledgment address those issues. If asked about a *real bug*, identify one you actually reproduced and fixed rather than inventing an incident.

## 2. Architecture and technology decisions

**Q: Why two file-sharing modes?** Stored mode lets someone retrieve a file later when the sender is offline. Live mode requires both peers online but avoids persistence on the application server. They optimize different user needs.

**Q: Why React?** Component-based UI, state management for upload/transfer progress, lifecycle cleanup with hooks, and reusable controls. Plain JavaScript could work, but stateful asynchronous UI becomes harder to maintain.

**Q: Why Node.js?** Event-driven, nonblocking I/O suits concurrent HTTP uploads/download streams and Socket.IO connections; JavaScript can be shared across client/server. CPU-heavy transformations would need workers/separate services.

**Q: Why Express?** Lightweight routing and middleware for REST endpoints, Multer, validation and centralized errors. Express itself is not a database, file store, or upload parser.

**Q: Why Socket.IO rather than WebSockets directly?** Event-based signaling, acknowledgments, rooms, convenient client API and transport/reconnection abstractions. For this design, reconnect is deliberately disabled because a resumed transport would not restore the old live transfer. Socket.IO is built over WebSocket or HTTP long-polling, not synonymous with bare WebSocket.

**Q: Why WebRTC instead of WebSocket for file bytes?** A WebSocket generally sends application bytes through the backend. WebRTC DataChannel can exchange encrypted bytes between browsers directly after connectivity negotiation, reducing application-server bandwidth for direct connections. TURN relay may still be required on difficult networks. WebSockets could be a simpler server-relayed fallback, but that increases backend bandwidth.

**Q: Why MongoDB?** The stored-file metadata is a compact, document-shaped record (name, size, key, token hashes, expiry, permissions). Mongoose gives schema validation and convenient indexed lookups. The current app has few joins and simple per-file updates. **It's a pragmatic choice, not proof MongoDB is universally faster.** PostgreSQL would be equally valid and potentially preferable with relational users, shared ownership, billing or complex transactional requirements.

**Q: Why not store file bytes in MongoDB?** Storing big payloads alongside metadata can complicate document access, storage and backups. Here, MongoDB stores indexed metadata; filesystem streams handle binary bytes. MongoDB GridFS is an alternative for large files; cloud object storage is usually better for scaled binary storage.

**Q: Why disk now, not S3?** A single-instance local prototype is easy to implement/test without cloud infrastructure. The adapter separates the storage API from its implementation. **Proposed:** S3-compatible object storage provides independent scaling, durability options and presigned uploads. The existing code does **not** implement S3.

**Q: Why Vite?** Fast frontend development server and production build tooling. `VITE_` variables are client-public: never put server secrets in them.

**Q: Why not microservices?** For this scope, a modular monolith avoids deployment, observability and consistency overhead. Split signaling, metadata and object delivery into separate services only if scaling/organizational needs justify it.

## 3. Stored upload: know every byte

**Q: Describe upload end to end.**

1. Browser reads a `File` selected by the user and adds it under field `file` in `FormData`.
2. `fetch` POSTs it as `multipart/form-data`; browser supplies boundary and file-part headers.
3. Express routes the request; Multer's disk storage parses multipart data and stages the bytes under a random filename in a private temp directory.
4. `req.file` provides temp path, uploader filename, reported MIME and size. Service checks nonempty file and sanitizes its display name.
5. Service generates random **storage key** and **owner capability**, moves the staged file to permanent private disk (rename or cross-device copy+unlink), then creates MongoDB metadata containing the SHA-256 hash of the owner secret, **not raw file bytes**.
6. API returns file metadata plus one-time raw owner secret; frontend retains ownership locally and requests a share link separately. If link creation fails, upload can still have succeeded.

**Q: What is multipart/form-data?** An HTTP request-body encoding with boundary-delimited parts, each with its own content disposition and optionally content type. It allows binary files and form fields. It is **not** multiple HTTP requests.

**Q: Multipart upload vs chunked upload vs S3 multipart upload?**

| Term | Meaning | Used here? |
|---|---|---|
| HTTP `multipart/form-data` | One HTTP body with one or more parts | **Yes**, stored upload |
| HTTP chunked transfer encoding / streamed request | Transport sends body incrementally without requiring full upfront buffering | Distinct HTTP framing detail; not the same as form multipart |
| Application-level chunk/resumable upload | Multiple logical chunks/requests with offsets and resume state | **No**, stored mode |
| S3 Multipart Upload | S3 API uploads numbered parts then completes a multipart object | **No** |
| WebRTC file chunks | `File.slice` pieces sent as ArrayBuffers through DataChannel | **Yes**, live mode |

**Q: Why Multer, not `express.json()`?** JSON middleware is for JSON; Multer parses multipart form bodies. Disk storage avoids holding an entire uploaded file in Node's application memory.

**Q: Why temp staging?** An incomplete/invalid upload should not become a visible permanent file. Temporary staging separates parsing from application validation and permanent placement. Handled errors trigger cleanup; crashes may still leave temp/orphan files.

**Q: How is the filename stored?** User filename is only a sanitized display label. A random storage key determines disk location. Prevents accidental overwrite, path traversal and name collisions. MIME/extension are untrusted; content scanning is not implemented.

**Q: What happens on file-size limit?** Multer rejects oversized uploads and the error layer maps that to HTTP **413 Payload Too Large**; limits also restrict count and unexpected fields. No automatic resumability.

**Q: Where do bytes live?** In private server disk for stored mode, **not** MongoDB, localStorage or a public static folder. LocalStorage only keeps per-file owner capability entries.

**Q: Is upload atomic?** No. Filesystem and MongoDB aren't one transaction. If DB insertion fails after saving bytes, service attempts compensating deletion. Process crashes can leave orphans; reconciliation would be a production improvement.

**Q: What if the client times out after the server saved the file?** The browser may not know whether the upload completed. Blind retry could create a duplicate. **Proposed:** client-generated idempotency key plus unique upload record/status endpoint; do not claim implemented.

## 4. Stored download, share links and security

**Q: How does a share link work?** Owner sends `X-Owner-Token` to create/replace link with expiry. Backend hashes a newly generated random share token, records `access=link` and expiry, and returns the raw token for the URL `/share/<token>`. Recipient info and download requests hash the supplied token, look up the document and check permission and clock on **every request**.

**Q: Owner token vs share token?** Owner token is a secret capability to manage the file; share token only grants recipient operations allowed by link. A file ID by itself is not authorization. Replacing/revoking a share invalidates old access.

**Q: Why hash tokens?** The DB stores fingerprints rather than immediately usable bearer secrets. These tokens are generated from 32 cryptographically random bytes; SHA-256 is suitable for high-entropy tokens. Human passwords need slow, salted password hashing such as Argon2, not just SHA-256.

**Q: How do you prevent timing attacks?** Owner token verification uses constant-time comparison (`timingSafeEqual`) after validating expected lengths. Hash lookup for share tokens identifies a record, followed by permission/expiry checks.

**Q: What if someone steals the share URL?** It is a bearer capability: possession grants access while valid. Mitigations include HTTPS, short expiry, revocation, no-referrer/no-store headers and not logging sensitive URLs. There is no recipient identity authentication.

**Q: What if localStorage is cleared or XSS runs?** Owner control can be lost or exposed. This implementation has no account-based recovery. **Proposed:** authenticated accounts and safer session controls; localStorage is not a secret vault.

**Q: Does expiry delete the file?** No. It blocks *new authorization requests* at/after expiry. Original bytes and record remain until owner deletion. Already downloaded bytes cannot be revoked; a download begun before expiry is not retroactively stopped by periodic expiry checks.

**Q: How is downloading memory-efficient?** Express uses `fs.createReadStream` and pipes to HTTP response, giving bounded server-side buffering/backpressure. However, the current frontend uses `fetch` → `Blob`, buffering the download in **browser** memory before creating an object URL. **Proposed:** streaming browser download, direct navigation or signed object-store download.

**Q: Why `Content-Disposition: attachment` and octet-stream?** Browser downloads rather than executing untrusted uploaded content. A content-disposition library safely formats filenames. Never trust uploader-supplied MIME as proof of safe file content.

**Q: How do you secure routes?** Only owner-capability holders may mutate/manage; share holders may access authorized metadata/download; private disk paths are not public routes. CORS restricts which browser origins may read responses but **CORS is not authentication** and non-browser clients can still call a public endpoint.

**Q: How would you prevent malicious uploads?** **Proposed:** authentication, per-user quotas, rate limits, antivirus/content scanning, MIME signature checks, file-size restrictions, isolated object storage, abuse reporting and download security headers. Existing safeguards include file-size/form limits, private disk, safe filename and forced attachment.

## 5. Live WebRTC: signaling to confirmation

**Q: Explain live transfer.** Sender creates room through Socket.IO; receiver joins. Sender creates `RTCPeerConnection` and ordered reliable DataChannel; offer/answer SDP and ICE candidates go through Socket.IO. Peers establish connectivity, then sender sends JSON metadata offer. Receiver accepts. Sender reads `File.slice(...).arrayBuffer()` sequentially, sends binary chunks subject to backpressure. Receiver validates and accumulates chunks, constructs Blob after `finish`, and sends `complete` acknowledgment; only then sender reports completion.

**Q: What is signaling?** Out-of-band exchange of session descriptions and network candidates so WebRTC peers can connect. WebRTC does not prescribe your signaling transport; this app uses Socket.IO.

**Q: What is SDP?** Session Description Protocol describes capabilities, transport/security parameters and negotiation. It doesn't carry your file contents. Offerer creates offer; answerer sets remote offer and replies with answer.

**Q: What are ICE, STUN and TURN?** ICE tests candidate network paths. STUN helps a peer learn network-reflexive addresses/ports. TURN relays encrypted traffic when direct connectivity fails. **Built:** STUN configuration and ICE exchange. **Not built:** TURN server. NAT/firewalls mean direct P2P is not guaranteed.

**Q: Why queue ICE candidates?** A candidate can reach the app before the associated remote description is set. Queueing/serializing SDP-candidate processing prevents avoidable `addIceCandidate()` failures.

**Q: Why DataChannel rather than sending bytes with Socket.IO?** Once negotiated, it sends bytes across an encrypted peer transport, usually avoiding app-server file relay. Socket.IO is only for signaling and room controls in this design.

**Q: Is WebRTC encrypted?** RTCDataChannel transport uses DTLS security (over SCTP/DTLS/ICE); avoid claiming a specific peer identity is authenticated, since signaling rooms are bearer-invite based. A TURN relay can forward packets without receiving application plaintext.

**Q: TCP or UDP?** Typical WebRTC DataChannels use SCTP over DTLS over UDP; ICE can select supported relay paths. Ordered/reliable delivery is a DataChannel property; not 'WebRTC is TCP'.

**Q: Why 16 KiB chunks?** Keeps application messages modest, limits per-chunk allocation and handles a predictable max-message size choice. Chunks can be smaller at the end. Max live file **100 MiB**: up to **6400** full-sized chunk buffers (plus control), bounding receiver allocation count.

**Q: What is backpressure?** Sender can enqueue data faster than network drains. It pauses when `RTCDataChannel.bufferedAmount` exceeds **256 KiB** and resumes on `bufferedamountlow`, limiting sender-side queued data. This does not cap receiver's total assembled file memory.

**Q: Why does receiver need Accept?** To avoid surprise file receipt/memory use. File's name/type/size are sent first as a JSON control message; binary starts only after explicit acceptance.

**Q: How do you know transfer succeeded?** Sender finishing its loop means chunks were **queued**, not delivered. On `finish`, receiver checks exact byte count, creates Blob and replies `complete`. This confirms assembly, **not** that the person clicked Save or that the disk write succeeded. There is no file-wide cryptographic hash check currently.

**Q: What if a peer disconnects?** Room is torn down; the in-progress transfer fails and cannot resume. An already completed Blob may stay available until user leaves; leaving revokes object URLs and releases resources.

**Q: How are rooms secure?** Unpredictable IDs, expiration, at most one sender and receiver, role/membership checking and allowed signaling message fields. A room invite is not real user authentication; anyone with a valid invite may join if room available.

**Q: Why no retransmission logic?** The DataChannel is configured for ordered reliable delivery; SCTP handles transport retransmissions. Application-level resume across total disconnects is **not** implemented.

## 6. MongoDB, schema, queries and DBMS drilling

**Q: What does the File collection contain?** `_id`, safe `originalName`, random `storageKey`, user-reported `mimeType`, byte `size`, `createdAt`, `access`, `ownerTokenHash`, optional `shareTokenHash` and `linkExpiresAt`. The schema does not contain actual file bytes, absolute filesystem path or recoverable owner secret.

**Q: Which indexes?** MongoDB indexes `_id` by default; this schema also indexes the share-token hash to support lookups. Unique storage key is a data invariant; confirm actual index declaration in source before claiming a unique index. A lookup indexed by token hash is roughly O(log N) B-tree lookup instead of full scan in broad terms (actual index lookup costs vary).

**Q: Why no TTL index for expiry?** TTL could delete metadata while leaving disk bytes orphaned. Here expiry is checked at authorization time; deleting physical bytes is a different lifecycle policy.

**Q: Is MongoDB schema-less?** It is flexible-schema; Mongoose enforces application schema but other writers may bypass it. Database indexes/constraints must be considered separately.

**Q: What are atomic operations?** MongoDB single-document updates are atomic. `$set` of current share hash/expiry avoids partially updating a document. MongoDB document atomicity does not extend to filesystem operations.

**Q: MongoDB vs PostgreSQL?** MongoDB excels at flexible document-oriented metadata, but PostgreSQL would support normalized user/file/share relationships, strong constraints, joins and transactions. This app's small flat metadata record and simple access patterns motivated MongoDB, not performance benchmark results.

**Q: How would user accounts change your schema?** Add `users`, ownership/ACL relationships, potentially a separate `shares` collection supporting multiple simultaneous recipient links; index `ownerId`, file IDs, share hash, expiry. Use least privilege and avoid placing secrets in logs.

**Q: What is an index tradeoff?** Speeds reads by maintaining a lookup structure but uses storage and increases write/update cost. Index actual predicates, not every field.

**Q: What about consistency?** Metadata is the authorization source, while disk is the byte source. No cross-resource transaction; compensation for handled failures, reconciliation needed for crash recovery.

## 7. REST, networking and JavaScript fundamentals

**Q: REST vs Socket.IO vs WebRTC?** REST/HTTP for upload, ownership, links and stored download; Socket.IO for ephemeral room and SDP/ICE messages; WebRTC DataChannel for live binary payload.

**Q: HTTP verbs/statuses?** POST creates uploads/links; GET reads metadata/download; DELETE removes/revokes. 201 created, 400 bad input, 401/403 authentication/permission (depending endpoint policy), 404 unknown/unavailable, 413 too large, 5xx server failure. Know your actual routes and codes; don't invent specific unverified ones.

**Q: What is Express middleware?** A function in the request pipeline that can inspect/transform request, send response or forward error. Order matters: origins/security headers, parsers, routes, 404 and final error handler.

**Q: What is async/await?** Syntax over Promises; the function yields while asynchronous I/O is pending, not blocking the whole Node event loop. CPU-bound loops can still block.

**Q: What is a stream?** A producer/consumer interface allowing progressive data processing rather than loading all content at once. `pipe` coordinates flow and backpressure. Stream errors after response headers require connection termination, not a second JSON body.

**Q: Why event-driven Node for concurrency?** Many connections wait on I/O, while the event loop switches between completions. Expensive CPU-bound hashing/transcoding could stall it; offload to worker threads/processes where needed. Here SHA-256 is on small token values, not full files.

**Q: What is a Blob and object URL?** Blob represents binary data in browser-managed storage; `URL.createObjectURL(blob)` makes a temporary local browser URL. It isn't a public share link and should be revoked on cleanup.

**Q: HTTP vs WebSocket?** HTTP request-response suits stateless file API actions. WebSocket is a persistent bidirectional connection suited to signaling notifications. Neither replaces WebRTC's P2P negotiation.

**Q: What are CORS and same-origin?** Browsers restrict cross-origin access; CORS tells browsers which origins are allowed. It doesn't stop curl or authenticate requesters. WebSocket handshakes need explicit origin policy too.

## 8. Scaling: what if traffic increases?

**Q: Can your current app handle 1,000 simultaneous uploads?** Not with guaranteed performance. Each upload consumes socket/file descriptors, disk bandwidth, temporary space and some memory; a single disk becomes the bottleneck. Need load testing before giving a capacity number.

**Q: How would you scale stored uploads?**

1. Add authentication, rate limiting, per-user quotas and max size/concurrency.
2. Move bytes from local instance disk to durable **object storage** (e.g., S3).
3. For larger uploads, issue short-lived scoped **presigned URLs** so browser uploads directly to storage; backend creates pending metadata record and marks complete only after verifying uploaded object/size/checksum.
4. Add load balancer and multiple stateless API nodes; use managed/replicated metadata DB with correct indexes.
5. Use lifecycle cleanup for abandoned multipart uploads, metrics, backups, virus scanning and CDN/signed download URLs where appropriate.

**Q: What if a 1 GB file arrives?** Current live mode rejects files over **100 MiB** and stored mode has configured Multer file-size limits; do not pretend either already supports 1 GB. **Proposed:** object-store multipart/chunked resumable upload, per-part checksums, upload IDs, retry only failed parts, final commit and orphan cleanup.

**Q: Explain S3 multipart upload.** Initiate multipart operation → upload numbered parts (possibly in parallel) → complete with part identifiers/ETags → abort failed/abandoned operations. Appropriate S3 part size and maximum count rules must be observed. The exact S3 ETag is not always a full-object MD5 checksum. Not used in baseline.

**Q: How do you make resumable upload?** Assign upload session ID; record total size and verified chunk positions/checksums; send missing ranges after reconnect; finalize atomically from client's perspective, validate total size/hash. Apply idempotency and expiration cleanup. Not implemented.

**Q: How would you scale downloads?** Serve through object store + CDN and short-lived presigned downloads, use HTTP range requests for resume where suitable; let app authorize before issuing access. The existing Express implementation streams from one server's disk.

**Q: How would you scale MongoDB?** First proper indexes/connection pooling and profiling; then replica sets for availability/read scaling where suitable, partition/shard only when dataset or write throughput actually requires it. Object storage handles byte scale; metadata DB handles metadata scale.

**Q: How would you scale Socket.IO rooms?** Current rooms live in one Node process Map, so a second node would not know membership or offers. **Proposed:** shared ephemeral coordination (e.g., Redis), Socket.IO compatible adapter for cross-node emission, careful atomic room admission/expiry, correct WebSocket routing. Sticky sessions may be required when HTTP long-polling is enabled; WebSocket-only deployment changes this constraint. Still need distributed room ownership semantics beyond just an adapter.

**Q: How many P2P transfers can server support?** With successful direct P2P, server mainly handles signaling, reducing file-byte bandwidth compared with relaying. But actual socket count, CPU, signaling volume, and especially TURN relay bandwidth constrain capacity; benchmark instead of inventing a number.

**Q: What if 1,000 transfers require TURN?** TURN relay egress/ingress becomes a major operating cost and scaling bottleneck. Deploy multiple TURN endpoints, bandwidth capacity planning, rate limits and TURN credentials. The app currently has **no** TURN integration.

**Q: CDN for live transfers?** A CDN doesn't accelerate a direct peer-to-peer channel like a static file. CDN makes sense for stored downloads, not direct DataChannel chunks.

## 9. Failure scenarios and debugging

| Situation | Accurate explanation / remedy |
|---|---|
| Disk fills during upload | I/O error, failed upload, staging cleanup where possible; monitor space/quotas |
| MongoDB unavailable at startup | Service does not start accepting required metadata operations |
| DB insert fails after disk save | Attempt compensating permanent-file removal; crashes still can orphan |
| Share link revoked while info page open | Download rechecks permission and fails |
| Expiry occurs during download | Authorization checked when new download begins; not continuously mid-stream |
| Original disk bytes missing | Metadata exists but streaming fails with a visible error |
| Client cancels HTTP upload | Handle disconnect and clean staged state; hard crash may leave temp data |
| ICE candidate arrives early | Queue until remote description is applied |
| Two senders attempt to offer | Sender-only offer role and room permissions prevent glare scenario |
| Third receiver joins a full room | Room admission rejects it |
| Receiver refuses live file | No binary file chunks should be sent |
| Channel buffer grows | Pause sender at backpressure threshold; resume on drain |
| Sender says 100% queued, receiver incomplete | Don't report complete until receiver's `complete` ACK |
| Peer network drops | Transfer fails, new room/restart required; no resume |
| Client refreshes after completion | In-memory received bytes/object URL aren't durable |
| Uploaded PDF has malicious script | Force attachment, don't trust MIME; scanning is future protection |

**Q: How would you debug a failed transfer?** Separate phases: Socket.IO connection → room admission → SDP negotiation → ICE gathering/connection → DataChannel open → metadata offer/accept → chunk counts and backpressure → finish/ACK. Record nonsecret state transitions and error codes; avoid logging owner/share bearer tokens or whole SDP unnecessarily. Inspect browser WebRTC internals and network traces where appropriate.

**Q: How would you test it?** Unit-test token generation/permissions, filename normalization, storage adapter, room membership and transfer protocol; integration-test HTTP/Multer/disk/Mongo and Socket.IO signaling; manual browser-to-browser connectivity and actual file hash comparisons. Test failure paths (expiry, revoke, oversize, wrong token, disconnect). A skipped unit test or simulated channel is not proof of live network connectivity.

## 10. Security deep-dive questions

**Q: Authentication vs authorization here?** There is no account/login authentication. Owner/share *bearer capabilities* authorize certain actions. The system verifies possession of a secret, not human identity.

**Q: Can someone enumerate files by ID?** Knowing an ID does not grant access to owner actions or shared download; permission checks protect data. Public API abuse still needs rate limits.

**Q: Is URL sharing secure?** It depends on random token entropy, HTTPS, exposure/logging, expiry/revoke and client security. Tokens in URLs may leak through screenshots/history/referrers, so no-referrer headers and short expiry matter.

**Q: Is HTTPS needed for WebRTC?** Secure contexts are generally required for full browser WebRTC functionality in deployed environments; localhost is a special development case. HTTPS/WSS also protect signaling and HTTP capabilities in transit.

**Q: What if many bots upload junk files?** Current unauthenticated uploads are vulnerable to abuse beyond existing size limits. Proposed controls: account verification, quotas, rate limiting, IP/device signals, scanning and storage-cost protection.

**Q: Can malicious code be hidden in file names or MIME?** Yes, treat both as untrusted. Sanitize display names, use safe response headers, avoid evaluating file contents and don't infer safety from extension.

## 11. System design variations interviewers may ask

**Q: Design a service like WeTransfer.** Client initiates authorized upload → direct object-storage multipart upload → completion verification → metadata in DB → share tokens and expiry → CDN/object download with authorization. Include quotas, malware scan, lifecycle cleanup, observability, monitoring and billing safeguards.

**Q: Design a service like AirDrop in browser.** Room discovery/signaling, peer negotiation (ICE/STUN/TURN), encrypted DataChannel transport, receiver consent, segmented chunks with backpressure, validation, integrity and reconnect/resume as required. Browsers can't simply access LAN discovery like native AirDrop.

**Q: What happens if two recipients use the same stored link?** Multiple authorized stored downloads can be served while link remains valid, subject to server capacity. Live mode allows only one receiver per room. Distinguish these semantics.

**Q: Would you use Redis or Kafka?** Redis is appropriate for ephemeral shared room state/rate limits, if needed. Kafka is unjustified for basic signaling, but could serve asynchronous audit/scan pipelines at higher scale. Never add a distributed system without a clear requirement.

**Q: Why not just transfer via HTTP?** HTTP supports stored upload/download and could relay a live file through a server; the WebRTC route targets interactive online P2P sharing without app-server byte storage. HTTP is still central to stored mode.

**Q: What are likely bottlenecks?** Stored: disk throughput, network egress, temp capacity, DB query rate, server connection counts. Live direct: peer uplink/downlink, NAT success and receiver memory; TURN changes relay cost substantially.

**Q: What metrics would you monitor?** Upload latency/failure by reason, temp-space growth, storage errors, API p95/p99, disk/egress usage, room admission errors, ICE connection success, TURN usage, transfer completion/abort rates and memory per client.

## 12. Key code/API recognition (be able to explain, not necessarily write all from memory)

```js
// Stored upload: browser; do not set Content-Type manually for FormData.
const body = new FormData();
body.append('file', selectedFile);
await fetch('/api/files', { method: 'POST', body });
```

```js
// Express/Multer concept: middleware parses multipart before handler.
router.post('/files', upload.single('file'), async (req, res) => {
  // req.file.path => private temporary disk path
  // validate, save durable bytes, persist metadata
});
```

```js
// DataChannel chunking concept (abbreviated).
for (let offset = 0; offset < file.size; offset += 16 * 1024) {
  const chunk = await file.slice(offset, offset + 16 * 1024).arrayBuffer();
  // In real implementation, WAIT for buffer to drain before send.
  channel.send(chunk);
}
// Sending all chunks is NOT equivalent to receiver-confirmed completion.
```

**What important Node APIs mean:** `fs.rename` moves same-filesystem entries, `fs.copyFile` handles cross-device fallback, `fs.createReadStream` reads incrementally, `stream.pipe` forwards data, `crypto.randomBytes` creates hard-to-guess capabilities, `createHash('sha256')` fingerprints tokens, `timingSafeEqual` compares suitable buffers, `mongoose.connect` connects metadata DB, `socket.join` joins a signaling routing room, `RTCPeerConnection` negotiates peer transport.

## 13. Honest limitations and proposed next steps

**Built in source guide:** React + Express + Mongoose/MongoDB + Multer + private local disk; hashed owner/share tokens and expiry; Socket.IO one-sender/one-receiver rooms; STUN-based ICE; ordered reliable DataChannel; 16 KiB chunking; 256 KiB buffered threshold; 100 MiB live limit; receiver confirmation; unit and integration test files (some gates require installed dependencies and manual verification).

**Not built (don't overclaim):** S3 or GridFS persistence, object-store presigned URLs, full upload/download resume, user accounts/login, distributed room state, TURN infrastructure, CDN, virus scanning, file-level live checksum, end-to-end browser integration automation, production throughput numbers, guaranteed cross-network connectivity. Tests existing in repository do not prove they all ran successfully.

**Q: What would you improve first?** For a public service: authentication/quotas/abuse controls and durable object storage; for live reliability: TURN fallback plus resumable/integrity-checked transfers; for scale: stateless HTTP nodes and shared room state. Prioritize by product requirement, not buzzwords.

## 14. Final rapid-fire — 25 answers you should deliver in one breath

1. **MERN?** MongoDB, Express, React, Node.js; WebRTC and Socket.IO add live communication.
2. **Where are stored bytes?** Private local disk; MongoDB only metadata and hashes.
3. **Where are live bytes?** Sender browser → WebRTC channel → receiver memory/download; not Express storage.
4. **Why DB?** Persistent indexed metadata and permission state.
5. **Why Mongo?** Simple document-shaped per-file data; not a universal performance win.
6. **Why Express?** Structured HTTP APIs and middleware.
7. **Why Multer?** Parse HTTP multipart file upload and stage to disk.
8. **Does multipart mean resumable?** No.
9. **Why stream?** Avoid loading entire file into server RAM.
10. **Does browser download also stream efficiently?** Current `fetch`/Blob buffers it.
11. **Why Socket.IO?** Room coordination and SDP/ICE signaling, not live bytes.
12. **Why WebRTC?** Encrypted peer DataChannel avoids app relay on direct paths.
13. **STUN?** Finds candidate reachable public-facing address/port.
14. **TURN?** Relays when direct connectivity fails; not implemented.
15. **ICE?** Determines viable connectivity path from candidates.
16. **SDP?** Negotiates session/transport parameters, not file payload.
17. **Reliable delivery?** Configured ordered/reliable DataChannel using SCTP.
18. **Chunk size?** 16 KiB.
19. **Buffer threshold?** 256 KiB for sender backpressure.
20. **Live file limit?** 100 MiB.
21. **Why ACK?** Queued bytes do not prove receiver assembled file.
22. **Can resume?** No; start a new room after disconnect.
23. **Why hash tokens?** Avoid storing usable bearer credentials in DB.
24. **What does expiry do?** Blocks subsequent authorized requests; doesn't auto-delete original file.
25. **How to scale?** Object store/direct multipart uploads + stateless API and DB indexes; Redis-backed room coordination and TURN capacity where needed.

## 15. Zebra interview-specific closing prompts

Candidate-report themes include **resume explanations and ownership**, **OOP**, **DBMS/SQL**, **JavaScript/fundamentals**, and, for some software roles, **Docker/CI/CD and DSA**. These are not a prediction of your particular interview. For this project, be ready to answer:

- Draw the whole architecture on paper and trace a byte from sender to recipient for **each mode**.
- Why exactly these technologies? What alternative would you choose and when?
- Which part did **you** build? Show one tricky failure and its fix.
- Describe one MongoDB schema and lookup, and compare to PostgreSQL.
- What if storage is full, DB fails, network drops or the link expires?
- How would you serve 10× traffic? Why does one-process Map fail across nodes?
- What's the difference between HTTP multipart body, upload chunks, and S3 multipart?
- Can you guarantee direct P2P? What role does TURN play?
- How do you prove a transfer finished, and what do you **not** guarantee?
- How would you secure a public-facing version without claiming current features?

### Research references (outside the uploaded code guide)

- Zebra Software Engineer interview candidate reports: https://www.interviewquery.com/guides/zebra-technologies-software-engineer
- Zebra Software Developer / university hire accounts: https://www2.glassdoor.co.in/Interview/Zebra-Technologies-Software-Developer-Interview-Questions-EI_IE2089.0%2C18_KO19%2C37.htm
- Zebra internship report mentioning project discussions, Docker and CI/CD: https://www.glassdoor.co.in/Interview/Zebra-Technologies-Interview-Questions-E2089.htm

**Source basis:** Architecture, implementation details and limitations are derived from your uploaded `LEARNING_GUIDE.md`; scaling and system design are marked as proposed; Zebra themes are from public candidate-report pages, not guaranteed questions.
