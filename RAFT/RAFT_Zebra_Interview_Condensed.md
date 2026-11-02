# Distributed Drawing Board with Mini-RAFT — Zebra Interview Quick Revision

**Read time: ~15–20 minutes | Priority: interview-ready answers, not a Raft textbook.**  
Based on your **resume** and **`LEARN_THE_CODE.md`**. **Built** = supported by your implementation guide; **Concept/Future** = know the idea, but do not claim you implemented it.

## 1. Your 30-second introduction (memorize the flow)

> “I built a real-time collaborative drawing board with a WebSocket gateway and three Node.js replicas implementing a simplified Raft protocol. The challenge was giving all users one consistent ordering of strokes even if a replica or leader fails. The client shows strokes locally immediately, but the leader replicates each operation and commits it only with a majority. The gateway then broadcasts committed updates. I implemented leader elections, log replication, persistence, retries with operation IDs, follower catch-up, and replay after reconnection.”

**Architecture:** `Browser (local preview) → WebSocket gateway → active Raft leader → 2-of-3 commit → committed-log read → gateway broadcast → all browsers`.

**One operation:** pointer events → segment batch (`opId`, `strokeId`, `segmentIndex`) → gateway validates → leader appends **and persists** → followers validate/persist `AppendEntries` → leader confirms current-term majority → apply in log order → gateway polls safely committed history → WebSocket fan-out → browser removes preview *after* painting confirmed stroke.

---

## 2. Consensus and architecture decisions — FIRST PRIORITY

**Q1. What is distributed consensus?**  
Several independent machines **agree on one decision/order** despite delays or failures. In a replicated state machine, all healthy replicas eventually apply the **same committed commands in the same order**, so they reach the same state.

**Q2. Why do we need consensus in this drawing board?**  
A broadcast-only server can lose history on failure; naive copying can leave replicas with **different orders or conflicting tails**. Consensus decides **which operations are authoritative and in what order**, even after leader failure. *Important honesty:* a normal drawing app could use one durable database; I deliberately used Raft to explore fault-tolerant replication.

**Q3. Consensus vs replication vs synchronization?**  
**Replication** copies data; **consensus** agrees which copy/order is committed; **synchronization** delivers committed changes to browsers. Having three copies without agreement is **not** consensus.

**Q4. Why Raft? Why not Paxos?**  
Both provide crash-fault-tolerant consensus. **Raft** explicitly organizes the design into leader election, log replication, and safety, making it easier to implement, explain, and test for a small replicated log. **Paxos/Multi-Paxos** can provide similar guarantees but was less approachable for this educational implementation; **not** because Paxos is inherently unsafe or slower.

**Q5. What other approaches exist? Why not them?**  
- **Single PostgreSQL server + WebSockets:** simplest practical design, but a single authority unless DB HA is added. Good answer for a typical product MVP.
- **Redis Pub/Sub:** fast fan-out, but Pub/Sub alone does **not** durably agree on an ordered committed log or recover missed messages.
- **Primary/backup with asynchronous replication:** simpler, but a failed primary can lose acknowledged writes unless safe failover/commit rules are added.
- **Paxos/Multi-Paxos:** viable consensus alternative; more complex to teach/implement than Raft in this project.
- **Zab (ZooKeeper Atomic Broadcast):** leader-based total-order broadcast and recovery, designed for ZooKeeper's coordination model.
- **CRDTs:** useful when allowing independently accepted edits and later conflict-free convergence; a different trade-off from demanding **one quorum-committed operation order**.
- **PBFT / Byzantine consensus:** considers arbitrary/malicious node behavior; overkill here, which assumes crash/communication failures and non-malicious peers.

**Q6. Why *three* nodes? Why does majority work?**  
A 3-voter cluster needs **2** to elect or commit; any two majorities overlap in at least one replica. It tolerates **one unavailable voter**. With only 2 voters, both are required; with 5 voters, quorum is 3 and it tolerates 2 failures. More voters **do not automatically improve write throughput**.

**Q7. What are safety and liveness?**  
**Safety:** never mark conflicting histories as committed (even during failures). **Liveness:** eventually make progress when a stable leader and majority connectivity exist. With 2 of 3 voters down, stop committing rather than violate safety.

**Q8. Is your system strongly consistent? What about CAP?**  
The **committed Raft log** is ordered by a single leader and protected by quorum rules; the browser view can lag because WebSocket delivery is asynchronous. Under a partition, the minority cannot safely accept commits: this chooses consistency of confirmed writes over write availability in that partition. Do **not** claim formal end-to-end linearizability from this demo alone.

**Q9. Why a separate WebSocket gateway?**  
It isolates browser clients from leader changes, validates commands, discovers the active leader, forwards operations, caches a **confirmed** prefix, replays history, and fans out updates. **Raft nodes decide; gateway distributes.** It is a single point of service availability in this version.

**Q10. Why not connect browsers directly to all replicas?**  
Clients would need to handle elections, leader discovery, history reconciliation, and multiple connections. A gateway centralizes that complexity; the trade-off is an additional hop and no gateway high availability.

**Q11. Why not send every mouse movement immediately?**  
The browser batches points **every 80 ms or at 64 points**, reducing socket traffic and log entries. `strokeId` identifies the stroke, `segmentIndex` keeps its pieces ordered, and each command gets a stable `opId` for deduplication. One in-flight command per tab keeps that tab's order; operations from different tabs may interleave.

---

## 3. Raft internals — MUST KNOW

**Q12. What are follower, candidate, and leader?**  
**Follower:** receives heartbeats, votes, and log entries. **Candidate:** starts election after timeout. **Leader:** accepts client commands and replicates logs. Nodes restart as followers, not as remembered leaders.

**Q13. What is a *term*? Why does it matter?**  
A monotonically increasing logical election epoch. Terms distinguish newer authority from stale leaders; a node seeing a higher term **steps down and persists it**. Terms are **not wall-clock timestamps**.

**Q14. How does leader election work?**  
A follower times out → increments term, votes for itself and persists → sends `RequestVote(lastLogTerm, lastLogIndex)` → needs **self + one peer** in this three-node cluster → becomes leader. Failed elections retry with randomized timeouts; heartbeats prevent unnecessary elections.

**Q15. Why random election timeouts?**  
If all followers time out together, they may vote for themselves and split the vote. Random deadlines reduce simultaneous candidacies; they cannot promise a fixed recovery time.

**Q16. Can a node vote for two candidates? Why persist `votedFor`?**  
At most **one vote per term**. Persisting `currentTerm` and `votedFor` before answering prevents a crash/restart from accidentally voting twice in that term.

**Q17. Which candidate wins if one has more log entries?**  
Check **last log term first**, then **last log index** if terms tie. Example: term 7/index 3 is more up-to-date than term 6/index 100. This prevents a node with a stale log from being elected just because its log is longer.

**Q18. What is a heartbeat? Is it enough to prove a live leader?**  
An empty `AppendEntries` maintains contact and conveys leader/commit information. **An old heartbeat is not proof of current authority**: for authoritative history reads, your leader sends **fresh same-term probes** and requires another successful voter response.

**Q19. What is stored in a log entry?**  
`{index, term, command}`; commands represent drawing segments or a `NO_OP`. Index is one-based; index 0/term 0 is an implicit sentinel. The log is an **ordered command history**, not canvas pixels.

**Q20. How does `AppendEntries` work?**  
Leader sends `prevLogIndex`, `prevLogTerm`, entries, and `leaderCommit`. Follower first verifies the matching prefix, then replaces a **conflicting uncommitted suffix**, persists before acknowledging, and later applies committed entries in order. A failed prefix check makes leader back up `nextIndex`.

**Q21. What do `nextIndex`, `matchIndex`, `commitIndex`, `lastApplied` mean?**  
`nextIndex`: next entry leader should send to a follower; `matchIndex`: highest index confirmed replicated there; `commitIndex`: highest index known committed; `lastApplied`: highest committed index already executed by the state machine. **Replicated ≠ committed ≠ applied.**

**Q22. When is a write committed?**  
Leader counts its **durable** copy plus followers' acknowledged copies. In 3 nodes, 2 suffice **for an entry from the leader's current term**. Committing that entry also commits its earlier log prefix. Do not acknowledge merely because one node appended it.

**Q23. Why the special current-term rule?**  
A newly elected leader must not directly declare an **older-term** entry committed only because it sees two copies; old uncommitted tails can be tricky across leadership changes. Committing a **current-term** entry by majority safely commits all preceding entries as well.

**Q24. Why does the new leader append `NO_OP`?**  
It creates a **current-term commit barrier** without drawing pixels. Your leader becomes *ready* only after that barrier commits, establishing a safe committed prefix before new operations/authoritative reads. `NO_OP` takes a log index but doesn't count as a stroke.

**Q25. What is the Log Matching Property?**  
If two logs have an entry with the **same index and term**, their prefixes through that entry match. `prevLogIndex` + `prevLogTerm` checks and conflict replacement help preserve this property.

**Q26. How does a follower catch up after being offline?**  
Leader sends missing entries from `nextIndex`. On mismatch it backs up and finds a common prefix, then replaces the follower's **uncommitted** divergent tail. This project uses **entry replay**, not snapshots; one-step backtracking may be slow.

**Q27. What is a deterministic state machine, and why use one?**  
Every replica runs the same ordered committed commands and must produce the same drawing-operation state. No random or node-specific side effects during application; committed commands are applied **in index order**.

**Q28. What is the most important Raft guarantee here?**  
Once an entry is **safely committed**, future eligible leaders retain it; a failover may discard **uncommitted** tail entries. This is why you cannot equate “written locally” with “successfully submitted.”

---

## 4. Failure scenarios and tricky follow-ups

**Q29. Leader crashes *before* majority replication?**  
Write is **not known committed**. It may be discarded by a later leader. Client keeps local preview and retries the *same* `opId` after leader discovery.

**Q30. Leader crashes *after* majority replication, but before client receives `200`?**  
Client outcome is **ambiguous**: the write may be committed even though response was lost. Retrying the **same `opId`** avoids a double drawing effect; never assume a timeout means failure.

**Q31. One follower crashes? Both followers crash?**  
One follower down: leader + other follower make quorum, so writes continue. Both down: no quorum; no new safe commit. Existing confirmed canvas/history can remain visible.

**Q32. Network partition: A alone, B and C together?**  
B+C can elect/retain a leader and commit. A cannot gain a majority and must not authorize new commits or fresh authoritative reads. On reunion, stale terms and log conflict repair reconcile A.

**Q33. Can there be two leaders?**  
A stale leader may *believe* it is still leader, and different terms can briefly have leader claimants. But **two conflicting majorities cannot both commit histories** under Raft's voting/log rules. Don't say “two leaders can never exist at any instant.”

**Q34. What if a follower ACK is lost?**  
Leader may retry replication; matching entries are recognized instead of incorrectly appended as new commands. Network delivery need not be exactly once.

**Q35. What if gateway crashes, or browser reconnects?**  
Gateway has **no authority** over committed writes; after restart it rebuilds its cache by reading Raft's committed prefix. Browser reconnects and receives a full ordered replay; the project has **one gateway**, so service is interrupted during its outage.

**Q36. What if the browser tab closes before commit?**  
Pending operations live **only in browser memory**; they can be lost on tab close. Confirmed log entries persist. An ambiguous operation may still become visible if it had already reached the leader.

**Q37. What if a stale or corrupted replica returns?**  
Higher term demotes stale leadership; AppendEntries repairs an **uncommitted** conflicting suffix. Corrupted saved state is rejected, **not silently reset**. Do **not** replace a lost voter volume with an empty voter: dynamic membership/recovery from lost identity is not implemented.

**Q38. Can the gateway display a drawing late even after it commits?**  
Yes. **Commit** is an agreement event; **broadcast/display** is a later asynchronous event. Your gateway polls committed history separately; clients verify index continuity and repaint after replay if they detect a gap.

---

## 5. Your code: likely “which file/function does that?” drilling

**Q39. What files own the main responsibilities?**  
`client/app.js`: batching, socket, pending queue, replay. `gateway/server.js`: WebSocket and validation. `gateway/raft-client.js`: leader discovery/HTTP. `gateway/sync.js`: committed polling/replay/fan-out. `raft/raft-node.js`: coordination and serialized mutations. `raft/election.js`: votes. `raft/replication.js`: AppendEntries and commits. `raft/storage.js`: persistence. `raft/state-machine.js`: deterministic application.

**Q40. How is data persisted safely?**  
Replica snapshots state to JSON, writes a **temporary file**, flushes/closes, **renames** to `state.json`, and attempts a directory flush. Each voter has an **independent Docker volume**. It's safety-conscious demo persistence, **not a benchmarked production WAL or absolute power-loss guarantee**.

**Q41. What state is persistent vs volatile?**  
Persisted: `currentTerm`, `votedFor`, `log`, plus this implementation's `commitIndex` and `nodeId`. Volatile: current role, leader ID, deadlines, `nextIndex`/`matchIndex`; rebuild applied state from the stored committed prefix on restart.

**Q42. What are `opId`, `strokeId`, and `segmentIndex`?**  
`strokeId` groups all batches of one continuous stroke; `segmentIndex` orders its pieces; `opId` uniquely identifies **one submitted operation for idempotent retry**. Retried payload with same ID is reused; *same ID, different data* is rejected with **409**.

**Q43. Is the application exactly-once?**  
**Not exactly-once transport.** Retries can deliver duplicates; application-level **deduplication by stable `opId`** makes repeated identical operations produce one drawing effect. A new ID for the same physical stroke is a different logical operation.

**Q44. Why do you keep two canvas layers?**  
Preview layer paints **unconfirmed** strokes immediately for responsive UX; authoritative layer paints **committed** strokes. On committed replay, paint confirmed content **first**, then remove matching preview in the same JS turn to avoid flicker/double lines.

**Q45. What exactly happens after a submit `200`, `503`, or `409`?**  
`200`: the Raft node confirms commit (browser still uses authoritative replay for shared painting). `503`: commit/leader status may be uncertain, so retry **same ID**. `409`: reused ID conflicts with different payload; don't blindly retry changed content under the same ID.

**Q46. How do gateway reads avoid accepting a stale leader?**  
Discovery checks all three status endpoints, prefers the **highest observed term** and requires a committed `NO_OP` barrier. The leader then performs a **fresh majority heartbeat check** for the committed-history read, rather than trusting an old leadership flag or cached lease.

**Q47. How do you handle late RPC responses and concurrent state changes?**  
Node.js `await` yields: two handlers can interleave around disk I/O. A Promise-based **mutation queue** serializes term/vote/log updates; HTTP/network waits happen **outside** it. Replication responses are accepted only for relevant leadership terms, with at most one regular in-flight replication request per peer.

**Q48. How did you test it?**  
Implementation guide references `tests/raft.test.js` for consensus/durability/timing contracts and `tests/gateway.test.js` for discovery, stale leaders, replay, and socket behavior. Say **simulated/invariant tests**; don't claim full network chaos, production load tests, or measured latency numbers unless you actually ran them.

---

## 6. Node.js, networking, Docker — high-yield cross-questions

**Q49. Why Node.js?**  
Good for asynchronous socket/HTTP I/O and fast iteration in a browser-based prototype. It **does not** guarantee race-free state: awaited operations can interleave. CPU-intensive work or heavy synchronous I/O would block the event loop.

**Q50. WebSocket vs HTTP vs Socket.IO?**  
**WebSocket:** persistent, bidirectional low-overhead updates between browser and gateway. **HTTP:** request/response for gateway-to-Raft RPCs in this project. **Socket.IO:** higher-level library with conveniences such as reconnection and fallback; it is **not identical** to raw WebSocket. Don't claim Socket.IO was used in this project.

**Q51. WebSocket vs polling vs WebRTC?**  
Polling repeats requests and can increase overhead/latency; WebSocket maintains a two-way connection. WebRTC suits direct peer media/data but introduces signaling and peer mesh complexity; here the server must broadcast one authoritative committed order. **Note:** the gateway itself *polls Raft* internally; browsers receive WebSockets.

**Q52. If TCP is reliable, why retries and idempotency?**  
TCP reliably transports bytes **while a connection works**; it cannot tell the client whether an application committed a write just before a disconnect. Timeouts and duplicate submissions are still possible, so operations need stable IDs.

**Q53. What is the Node.js event loop?**  
JavaScript callbacks execute on the main event-loop thread; asynchronous I/O completes via the runtime and callbacks/microtasks resume execution. `await` **does not lock shared state**. Your queue ensures logical serialization for consensus mutations.

**Q54. Why Docker Compose? How do volumes help?**  
Compose starts **one gateway and three Raft replica processes** with repeatable networking and configuration. **Separate named volumes** keep independent term/vote/log state across container restarts. Restarting a container is **not** the same as wiping its volume or changing cluster membership.

**Q55. What if 1,000 users draw simultaneously?**  
Their commands interleave into **one leader-ordered log**; one in-flight operation **per tab** is not a global lock. Likely pressure points: single gateway fan-out, serialized durable writes, JSON state replacement, replay size and slow-client backpressure. These are **design hypotheses**, not measured benchmarks.

---

## 7. Honest limitations, improvements, and final traps

**Q56. What is the biggest limitation?**  
One gateway, three **fixed** Raft voters, no snapshots/log compaction, no dynamic membership, memory-only client pending queue, and simple JSON-file persistence. A sufficiently long log makes replay/repair expensive; gateway outage temporarily disconnects clients.

**Q57. How would you make this production-ready?**  
Use a mature Raft implementation or durable DB where appropriate; add snapshots/compaction, tested membership changes, replicated gateways, browser outbox, authentication/rate limits, bounded queues/backpressure, observability, chaos tests, and p95/p99 latency measurements. **These are future improvements, not built features.**

**Q58. Could you use PostgreSQL instead of Raft? Which would you actually choose?**  
For a normal drawing product, **PostgreSQL plus WebSocket fan-out** is usually much simpler. This project's learning objective was to **build the replication/consensus mechanism itself**, not to claim every collaborative canvas needs custom Raft.

**Q59. Why not CRDTs for collaborative drawing?**  
CRDTs can merge concurrent edits while replicas accept operations independently, often giving better offline editing. My architecture intentionally enforces a **central, majority-committed global order**; CRDTs would be a different consistency model and could fit a production canvas better depending on requirements.

**Q60. Your resume says a node can be “restarted or replaced.” What is accurate?**  
A **replica process/container** can restart **with its original persistent volume** and catch up. Replacing a voter whose identity/log are lost with a blank node is **not safely supported** by this fixed-membership implementation. Say this clearly if challenged.

### Last-minute rapid recall (30 seconds)

| Interview trap | Correct one-line answer |
|---|---|
| Replication vs commit | A stored entry can still be uncommitted. |
| Majority size in 3 voters | **2**; 1 unavailable voter tolerated. |
| Which candidate has newer log? | Compare **last term**, then index. |
| Can a stale leader respond? | Its old belief is insufficient; fresh majority validation matters. |
| Can old-term entries be directly committed by counting? | **No**; current-term rule and `NO_OP` barrier. |
| Are retries safe automatically? | **No**; reuse the same `opId` and deduplicate. |
| Does Node's single thread prevent races? | **No**; `await` can interleave state changes. |
| Is every on-disk entry committed? | **No**; apply only through `commitIndex`. |
| Can the minority keep writing? | **No**, quorum is required. |
| Are snapshots/cluster resizing supported? | **No**, describe as future improvements. |

**If only 5 minutes remain:** memorize your pitch, Q1–Q6, Q14, Q17, Q20–Q24, Q29–Q33, Q42–Q47, and Q60. Know the **data flow** well enough to draw it without this file.

### Sources / evidence boundaries

- **Your implementation:** uploaded `LEARN_THE_CODE.md` and `resume_v3(3).pdf` (source-of-truth for *built* features; this guide is not a line-by-line audit of the actual repository).
- **Raft authors:** [Raft overview](https://raft.github.io/) and [Ongaro & Ousterhout, USENIX ATC 2014](https://www.usenix.org/conference/atc14/technical-sessions/presentation/ongaro).
- **Alternative consensus context:** [Apache ZooKeeper — Zab vs. Paxos](https://cwiki.apache.org/confluence/spaces/ZOOKEEPER/pages/27842690/Zab+vs.+Paxos).
- **Zebra candidate report (Aug 5, 2026):** [Glassdoor intern reports](https://www.glassdoor.co.in/Interview/Zebra-Technologies-Interview-Questions-E2089.htm?filter.jobTitleExact=Software+Engineer%28Internship%29) mention OOP, projects, Docker/Kubernetes and DSA. Candidate reports are anecdotes, **not confirmed questions for your panel**.
