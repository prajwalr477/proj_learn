# Learn the code

This is a guide to the actual implementation. Start with a stroke, then follow
the two independent paths: immediate local preview and majority-confirmed shared
ink. The beginner-friendly comments in each module explain the guards next to
the code that enforces them.

## 1. One stroke from pointer to every browser

1. `client/index.html` loads `client/app.js`. `startStroke` captures one pointer
   and fixes its color/width. `canvas.js:position` scales CSS coordinates into
   the fixed 1000 × 600 drawing space. `moveStroke` updates the preview immediately.
2. `app.js:flushBatch` runs every 80 ms or at 64 points. It creates a stable
   `opId`, keeps one `strokeId`, increments `segmentIndex`, and carries the final
   point into the next batch so the line stays connected. A tap becomes one dot.
3. `sendNext` sends the oldest pending command over the gateway WebSocket. One
   command is in flight per tab, preserving that tab's segment order. Retries
   reuse the same command/ID after 1800 ms; different tabs can interleave.
4. `gateway/server.js:submitMessage` uses `shared/commands.js:validateCommand`
   to bound/normalize the command. `RaftClient.submit` discovers a ready leader
   and sends `POST /operations`. Gateway validation and replica validation both
   run: the replica must also protect direct HTTP submissions.
5. `raft/server.js:route` calls `RaftNode.submit`, which calls `propose` inside
   the serialized mutation queue. A new command is appended and flushed before
   any replication is attempted. An identical opId reuses its existing index;
   a different payload for that ID receives 409.
6. `RaftNode.replicatePeer` snapshots `makeAppendRequest`, sends it outside the
   mutation queue, then queues `receiveAppendResponse`. A follower's
   `handleAppendEntries` verifies the prefix and persists before acknowledging.
7. `maybeAdvanceCommitIndex` counts the leader's durable copy plus follower
   match indexes. Two copies suffice, but the chosen entry **must belong to the
   current term**. The entire preceding prefix becomes committed too.
8. `applyCommitted` calls `DrawingStateMachine.apply` in index order. Committed
   entries deduplicate by opId. `submit` returns 200 only after its index commits;
   it may instead time out with 503, in which case retrying is essential.
9. Independently, `gateway/sync.js:poll` calls `RaftClient.readCommitted`. The
   leader's `RaftNode.committed` requires a fresh majority heartbeat response.
   The gateway validates contiguous indexes and broadcasts `entries` by
   iterating over open sockets. Submit responses themselves never broadcast ink.
10. `app.js:acceptEntries` paints the authoritative canvas first, then removes
    matching previews in the same JavaScript turn. `renderPreview` rebuilds only
    pending/unfinished ink. This prevents confirmed lines remaining on both layers.

## 2. Why use Raft here?

A gateway could keep an array and broadcast immediately; that was the temporary
Phase 1 implementation. If that process disappeared, its history disappeared.
Simply copying writes to three servers also leaves unresolved questions: which
copy wins, which write order survives a leader failure, and which replies prove
commitment? Raft supplies an elected leader and a majority-agreed ordered prefix.

Phase 2 added the actual replica implementation. Phase 3 removed the temporary
memory authority from the gateway. Its present `history` array is just a cache
of a Raft-confirmed prefix and can be rebuilt after a gateway restart. There is
no standalone memory-log mode in the final project.

## 3. Raft state, indexes, and serialization

`raft/raft-node.js` is the orchestration class. `raft/server.js` constructs it
with a fixed node identity, two peers, and its own `FileStorage`. Tests inject
clock/transport/storage without replacing the consensus algorithms.

| State | Lifetime / meaning |
| --- | --- |
| `currentTerm`, `votedFor`, `log` | Persisted before relevant RPC success |
| `commitIndex` | Also persisted in this demo; locally known safe prefix |
| `nodeId` | Persisted to reject accidentally attaching another node's volume |
| `role`, `leaderId`, deadlines | Volatile; restart begins as follower |
| `barrierIndex` | Volatile index of this election's NO_OP |
| `nextIndex`, `matchIndex` | Leader-only per-peer replication progress |
| `machine.lastApplied`, `operations` | Rebuilt from persisted committed prefix |

Real entries are `{index,term,command}` with one-based indexes. Array offset is
index minus one. Index 0 is an **implicit sentinel** with term 0, not a real
persisted entry. `termAt(0)` handles that convention. Thus an empty log can
match a leader's first AppendEntries without inventing a drawing operation.

JavaScript's event loop does not make async disk operations atomic. Without
serialization, two vote handlers might both examine an old `votedFor`, then
both grant votes while their writes overlap. `enqueue` chains state operations
through a Promise queue. Helpers in `election.js` and `replication.js` run inside
it; network waits run outside it. Holding the queue while awaiting another node
could prevent incoming heartbeats/votes and cause unnecessary deadlock/timeouts.

`persist` makes storage errors fatal for the running instance. It refuses later
state requests; it cannot keep voting after forgetting whether a disk write
succeeded. Ordinary invalid client input does not poison the queue.

## 4. Elections and the NO_OP barrier

`RaftNode.start` schedules `tick`. A follower/candidate whose randomized deadline
has passed calls `election.js:startElection`:

1. Increment the term, become candidate, vote for self, randomize the next retry.
2. Persist that state before sending RequestVote.
3. `requestVotes` asks both peers concurrently. `handleRequestVote` first observes
   higher terms, then checks whether this candidate's `(lastLogTerm,lastLogIndex)`
   is at least as up to date. The last term takes priority over log length.
4. `receiveVote` ignores obsolete election responses and counts distinct voters.
   Self plus one peer wins. The new leader initializes replication progress and
   appends a current-term `NO_OP` durably.

Why the NO_OP? Counting copies of an old-term entry is not enough to commit it
under a new leader. Replicating the new leader's own term establishes a safe
prefix, including earlier entries. The leader stays unready until this barrier
commits. `NO_OP` occupies a log index, but produces no pixels and no drawing count.

`observeTerm` clears the old vote, steps down, and persists whenever a higher
term arrives in a request **or response**. Valid AppendEntries resets the election
deadline, even when the prefix needs repair. Lower-term messages do not reset it.
Randomized deadlines reduce split votes; they are not a promise of fixed failover
latency. No pre-vote extension or dynamic membership exists in this demo.

## 5. Replication, conflict repair, and commitment

`replication.js:makeAppendRequest` starts at the follower's `nextIndex`, with
the previous index/term and up to 64 entries. Empty entries are heartbeats.

`handleAppendEntries` follows this order:

1. Observe higher terms; reject stale leaders. Become follower on a valid
   current-term leader contact and reset the election timer.
2. Verify `prevLogIndex` exists and `prevLogTerm` matches. A mismatch fails
   without rewriting the log. `receiveAppendResponse` backs `nextIndex` up one
   position, and later attempts find a matching prefix.
3. Compare entries at their exact positions. Conflicting **uncommitted** entries
   cause suffix replacement. Never overwrite a locally committed position.
   Matching entries remain. Empty heartbeats do not blindly truncate tails.
4. Persist the replacement before acknowledging. Advance commit to at most the
   prefix actually matched by this RPC. A short heartbeat must not accidentally
   commit an unrelated stale local suffix just because its length is large.
5. Apply the newly committed prefix in order.

`receiveAppendResponse` credits only responses from the current leadership term.
At most one regular replication request per peer is in flight, preventing an
older rejection from undoing newer progress. `maybeAdvanceCommitIndex` scans
backward for the highest index in the current term with two durable copies.
It persists the new committed prefix before application/confirmation. A later
heartbeat teaches followers the leader's commit index, so their status can lag
briefly even after the leader safely confirms a command.

Catch-up uses exactly the same mechanism. A restarted follower keeps its old
term/vote/log and committed prefix. The leader sends missing entries or backs up
to repair an uncommitted divergence. No special full-state copy or snapshot is
implemented. One-step backtracking is intentionally readable and slow for long
divergent histories.

## 6. Durability and deterministic application

`raft/storage.js:save` snapshots the state into JSON before enqueuing a write.
`replaceFile` writes a temporary file in the same directory, flushes/closes it,
renames it to `state.json`, then attempts a directory flush. Unsupported directory
flush operations on some filesystems are tolerated; actual other I/O failures
fail closed. A leftover temporary file is not authoritative on restart.

`load` validates identity, terms, indexes, log command shapes, and conflicting
opIds. Corrupt state is not silently replaced with an empty log. Named Docker
volumes are independent and not inside watched source directories.

On restart, `initialize` applies only entries through the stored `commitIndex`.
A follower may know less than the true global committed prefix, and later
AppendEntries advances it. It must never infer that every on-disk entry committed.
Persisting commitIndex is an extra conservative choice for this demo, beyond
Raft's usual minimum persistent term/vote/log state.

`DrawingStateMachine.apply` requires contiguous indexes. The first committed
opId records its command; an identical duplicate consumes its log position
without applying the drawing again. A different payload is rejected. The leader
also checks opIds in its uncommitted log to avoid redundant appends. This does
**not** authorize immediate commit acknowledgement: `submit` still checks the
index, and a newly elected leader still needs its current-term barrier.

There is no durable client outbox. The official client retains one stable command
until committed replay/fan-out removes it. A timed-out attempt can have succeeded;
the right recovery is the same opId, not drawing the same pixels under a new ID.

## 7. Discovery, authoritative reads, and replay

`gateway/raft-client.js:discover` queries all three statuses with bounded timeouts
and tracks the highest term it has seen. It chooses a leader only in that term
whose NO_OP barrier completed. This avoids favoring an older isolated leader just
because it responds fastest. The policy can conservatively pause reads/writes
until the cluster catches up to an observed newer term.

A status flag is still not sufficient. `RaftNode.committed` captures a probe after
the read begins and sends **new** AppendEntries heartbeat requests to both peers.
It needs one same-term successful response, while remaining the ready leader in
that term. It does not reuse an old heartbeat success or a cached lease. These
read probes do not alter nextIndex/matchIndex; regular replication owns progress.
An isolated former leader therefore cannot serve a newly authorized replay.

`CommittedSync.poll` has one poll in flight. It validates a full page before
advancing the cursor, including NO_OP entries. On initial gateway startup it
collects pages until a returned committed head is reached, then sends one full
`replay`. Afterward it broadcasts `entries` to live sockets.

`add`/`replay` sends the cached prefix and marks the socket live in one synchronous
turn, without `await`. A polling continuation cannot interleave between them,
so a newly connected client gets replay then live updates without a gap. During
an election the cache and old browser pixels remain; the cache contains only a
previously confirmed prefix. A gateway restart rebuilds that cache from Raft.

Browser `acceptEntries` deduplicates drawing IDs and verifies contiguous indexes.
It removes a pending preview only after rendering its committed command. During
reconnection, `connect` leaves old pixels visible until a full replay arrives,
then clears/rebuilds in one JS turn. A detected gap closes the socket for replay.
The overlay only paints uncommitted/unfinished work, preventing doubled lines.

## 8. File map and supporting boundaries

| File | Responsibility and main caller |
| --- | --- |
| `client/index.html`, `styles.css` | Layered fixed-resolution drawing UI served by the gateway |
| `client/app.js` | Pointer batching, WebSocket lifecycle, pending/replay cursor |
| `client/canvas.js` | Coordinate mapping and committed/preview rendering called by app.js |
| `gateway/server.js` | Static allowlist, socket lifecycle, command validation/routing |
| `gateway/raft-client.js` | Leader discovery and bounded submit/history HTTP |
| `gateway/sync.js` | One ordered committed cache, coherent replay and WS fan-out |
| `raft/server.js` | Environment configuration, HTTP/RPC validation, process startup |
| `raft/raft-node.js` | Queue, lifecycle, network orchestration, submission/read barriers |
| `raft/election.js` | Deadlines, terms, voting, leader NO_OP |
| `raft/replication.js` | Prefix/suffix rules, next/match indexes, safe majority commit |
| `raft/storage.js` | Independent durable state replacement/reload |
| `raft/state-machine.js` | Log validation and deterministic opId application |
| `shared/commands.js` | Command normalization/comparison at both server boundaries |
| `shared/http.js` | Bounded JSON fetch/body handling and explicit HTTP responses |
| `tests/raft.test.js` | Direct consensus, disk, timing, and gateway-to-core contract tests |
| `tests/gateway.test.js` | Discovery, stale authority, paged replay, cursor and socket limits |
| `scripts/check-syntax.js` | Zero-install syntax checks of project JavaScript |
| `Dockerfile`, `compose.yaml` | Four processes, source mounts, three volumes, development watches |

## 9. Interview walkthrough and failure cases

- **Why not acknowledge on append?** A lone leader can store a command that a later
  majority never sees. Persisted is not necessarily committed. The client waits
  for the gateway's ordered committed broadcast.
- **Why majority and not all three?** Two voters intersect every other possible
  majority of three. One follower can fail without blocking agreement. Requiring
  all three would defeat that availability benefit.
- **What happens if a leader dies mid-write?** The command may or may not have
  committed. The browser retries the same opId. The new leader has the safe prefix
  and commits its barrier before serving new operations or authoritative history.
- **What if two voters stop?** No new commits. Local previews remain, pending
  counts stay nonzero, and the gateway shows waiting for quorum. Restore a second
  voter and the official client retries in order.
- **What if a stale leader returns?** A higher term demotes it; prefix matching
  repairs its uncommitted tail. Its already committed prefix cannot be overwritten.
- **What if the gateway stops?** Sockets close. Browsers reconnect and replay after
  it restarts. Replica history persists, but there is no gateway HA.
- **What if a tab closes?** Its memory-only pending queue is lost. Already committed
  history survives. An ambiguous attempt may still appear after reload if it commits.
- **Why not replace a lost volume with a blank voter?** That erases remembered
  terms/votes/logs and breaks the fixed-voter recovery assumptions. Ordinary
  catch-up means restarting the same persisted voter, not creating a new identity.

The tests demonstrate small, explicit invariants and simulated failures. They
do not claim real network chaos coverage or production power-loss guarantees.
Follow the PowerShell/browser acceptance checklist in README for the remaining
manual evidence, and consult STATUS for exactly what has already run.
