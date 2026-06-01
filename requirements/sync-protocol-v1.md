# Sync protocol requirements v1 (friend + sibling carriage)

Status: normative for `nearbytes-sync` Impl. v0.

Two relations share this protocol:

  - **Friend carriage**: a node serves profile $P_L$ and accepts a remote
    profile $P_R \ne P_L$ that is listed in `config.friends`.
  - **Sibling carriage**: the same identity carried by two or more nodes;
    the remote profile $P_R$ equals one of the locally served profiles
    ($P_R \in \text{servedSet}$). See `sync-discovery-v1.md` DISC-26.

The two relations differ only in the membership check (friend set vs
served set); message formats, association state machine, and anti-
entropy rules below are identical.

References: `sync-discovery-v1.md`.

## 0. Profiles served by a node

| Rule | Requirement |
|------|-------------|
| SYNC-00 | A sync node MAY serve $K \ge 0$ local profile public keys simultaneously. The set MUST be lower-case hex and SHOULD be derived from `NearbytesConfig.profiles`. When $K = 0$ the node MUST NOT start sync (no topic to join, no peer to authenticate against). |
| SYNC-00a | Each sync association MUST authenticate exactly one local profile from the served set against exactly one remote profile from the **authorized set** $= \text{served} \cup \text{friends}$; an association MUST NOT swap identities mid-stream. A sibling association (`remote profile` $\in \text{served}$) is allowed and MUST be treated identically to a friend association by the anti-entropy layer; only `sync-discovery-v1.md` DISC-27 loopback filtering distinguishes a sibling instance from the local instance itself. |

The terms *profile* and *identity* are not synonyms in this stack: a **profile** is the keypair / public-key noun used here, on the wire, and on disk; an **identity** is the social/display record (`nb.identity.record.v1`) signed *by* a profile. This spec governs the profile layer.

## 1. Transport association

| Rule | Requirement |
|------|-------------|
| SYNC-01 | Each association MUST serialize framed writes and message handling so frames are never interleaved on one duplex. |
| SYNC-02 | Every association (friend or sibling) MUST perform a `hello` exchange before any `delta`, `subscribe`, `have`, `want`, or `data` message. |
| SYNC-03 | `hello.senderProfile` MUST be the lower-case hex profile public key of the sender (equal to the local profile chosen for this association from SYNC-00). `hello.senderPeerId` MUST be the sender's short stable per-`dataDir` peer id (DISC-27). `hello.senderInstancePublicKey` MUST be the lower-case hex public key of the sender's per-`dataDir` instance keypair (DISC-27). The pair $(\text{senderProfile}, \text{senderInstancePublicKey})$ uniquely identifies the remote endpoint of the association; `senderPeerId` is diagnostic/transport identity and MUST NOT be used as the cursor key. |
| SYNC-04 | The receiver MUST close the association if `senderProfile` is not in the authorized set (friend ∪ served) or policy denies the subject. The receiver MUST additionally close the association as a loopback (DISC-27) when `senderProfile == localProfile` **and** `senderInstancePublicKey == localInstancePublicKey`. |
| SYNC-05 | The receiver MUST reject a duplicate `sessionNonce` on the same association. |
| SYNC-06 | At most one active sync association per ordered triple $(\text{local profile}, \text{remote profile}, \text{remote instance public key})$; a new association for the same triple MUST replace the previous one. The remote instance public key MUST be part of the key so two sibling instances sharing $\text{remote profile}$ each get a live session. |
| SYNC-07 | On a friend association, `subject` MUST be `profile(localProfilePublicKey)` — i.e. the profile being *targeted* on the receiver — and both sides MUST agree on `subject` for the association to proceed. The receiver MUST select its local profile by matching `subject.profile` against its served set; if no match exists the receiver MUST close the association. |

## 2. Anti-entropy

| Rule | Requirement |
|------|-------------|
| SYNC-10 | After local `acceptData`, objects enter the reception journal; each append MUST push `have` to all open friend associations (no timer-driven polling). |
| SYNC-10a | The reception journal MUST be a durable per-instance append-only JSONL log at `sync/reception.jsonl`. Each line MUST be one JSON object containing at least `{ "seq": <number>, "ref": <ObjectRef> }`, where `ref` MAY name either a block or an event. `seq` MUST be strictly monotonically increasing within that instance and defines the instance-local total order of accepted sync objects. |
| SYNC-10b | `delta { mode: "global", cursor }` pagination MUST enumerate the remote instance's reception journal in ascending `seq` order. The cursor returned as `have.nextCursor` MUST identify the last remote `seq` fully covered by that page. Receivers MUST treat the cursor as opaque, but the responder's ordering semantics are the `seq` total order from SYNC-10a. |
| SYNC-11 | `want` is receiver-driven; senders MUST NOT send `data` without a prior `want`. |
| SYNC-12 | When both blocks and events are missing from a `have`, the receiver MUST issue `want` for blocks before `want` for events (blocks-first ordering). |
| SYNC-13 | `have` for events SHOULD include `blockRefs` when known so receivers can prioritize dependency fetches. Receivers MUST NOT assume every `blockRefs` hash is a block; application protocols may use visible refs for event parents as well as content blocks. |
| SYNC-14 | At most one outstanding `want(H)` per block hash MUST exist across all associations sharing a local `Log`. On `have(H)` with `H` already in-flight (either reserved or actively streaming on some association), the receiver MUST suppress the duplicate `want`. The reservation MUST be released when the corresponding incoming stream finishes (stored, hash-mismatch, or already-local discard) or when the association holding the reservation closes; on release, a future `have(H)` from any peer is again eligible to `want`. This rule eliminates redundant bandwidth (the same content-addressed block arriving twice over disjoint transports) and the disk-commit race that follows from concurrent deliveries. |
| SYNC-15 | Anti-entropy is **mutual** on every live association. After `hello`, each peer MUST run the same session logic: send `subscribe` plus an initial bounded `global` `delta`, respond to inbound `delta`/`subscribe` with `have`, answer inbound `have` with receiver-driven `want`, and answer inbound `want` with `data`/block streams. There is no fixed client/server role beyond who opened the transport. |
| SYNC-16 | When an inbound event is stored, the receiver MUST inspect visible `blockRefs` and issue `want` for any direct dependency it still lacks locally: if `blockRefs.length > 1`, the first ref names a missing parent **event** in the same channel; if `blockRefs.length === 1`, the sole ref names a missing **content block**; remaining refs name missing content blocks. This is in addition to ordinary `have`-driven fetch and MUST obey SYNC-12. |
| SYNC-17 | When local storage already contains **orphaned** events whose observed-log-head parent (per the application protocol, e.g. FILES v0.5) is absent, reception-journal pagination alone MAY never re-offer that parent because the orphan is already local. Implementations MUST therefore scan stored channel events for missing parents and issue explicit `want`s. The scan MUST run (a) after each stored inbound event, (b) after each stored inbound block, and (c) after processing an inbound `have` page whose `more` flag is false. Scans MUST be serialized behind the association's inbound message queue so repair never races a half-handled frame; timer-based repair is PROHIBITED. |
| SYNC-18 | Before serving a requested block, the sender MUST verify that the block exists locally via `BlockStoreApi.has` or equivalent. If absent, the sender MUST NOT emit a block stream and MUST NOT log at error level or include a stack trace. Missing blocks are normal for partial replicas; implementations MAY emit debug/trace telemetry. Application-layer reads (`file get`, replay, WebDAV GET, etc.) remain responsible for surfacing user-visible failures when required content blocks are unavailable. |
| SYNC-19 | A receiver that sends `delta { mode: "global", cursor }` and consumes the resulting `have` pages MUST persist fetch progress on disk per remote endpoint. The cursor key MUST be `(remote profile public key, remote instance public key)`, not merely profile and not transport session id. |
| SYNC-21 | On session attach, the initial global `delta` SHOULD use the stored cursor for that remote endpoint. After a complete `have` page is processed, the receiver MUST checkpoint fetch progress to `have.nextCursor` when present; when `have.more === false`, the receiver MUST mark the remote endpoint caught up at the last returned cursor. Checkpointing MUST happen only after outstanding `want`/`data` work for that page has finished (including orphan-dependency repair). |
| SYNC-21a | Responders MUST send a terminal `have { more: false }` even when the requested `delta` page is empty so the receiver can finish pagination. |
| SYNC-21b | If the receiver holds a persisted fetch cursor `C` greater than the highest `seq` in its own `sync/reception.jsonl` and the next terminal `have` page is empty, the receiver MUST treat `C` as stale and restart global `delta` from the beginning (no cursor) once per session attach. |
| SYNC-22 | Fetch cursors index the **remote** peer's reception stream and therefore the remote instance-local `seq` total order from SYNC-10a. They are independent of the local `sync/reception.jsonl`, which records objects accepted by this node. Cursor values are opaque strings; receivers MUST NOT derive ordering except by handing the value back to the same remote endpoint in a later `delta`. |

## 3. Boot

| Rule | Requirement |
|------|-------------|
| SYNC-20 | `patchLogForReactiveHave` MUST apply per `Log` instance (not process-global). |

## 4. Wire efficiency (block transfer)

| Rule | Requirement |
|------|-------------|
| SYNC-30 | Block bytes MUST be sent as a **continuous raw stream** after stream-begin (see SYNC-33). Base64, base64url, JSON-in-JSON, or any other **content encoding of block bytes is PROHIBITED**. |
| SYNC-31 | Implementations MUST maximize throughput and minimize copies: decode paths SHOULD expose chunk payloads as views (`subarray`) over the receive buffer where safe; avoid re-encoding ciphertext or plaintext for transport. |
| SYNC-32 | Control messages (`hello`, `delta`, `subscribe`, `have`, `want`, `error`) MAY use compact UTF-8 JSON **without** embedded binary blobs. Event `data` MAY use a separate binary event frame (channel + hash + raw bytes). |
| SYNC-33 | After `want` for a block, the sender MUST emit one **stream-begin** frame (hash + total length) then **`total` contiguous raw bytes** on the duplex — no per-chunk length prefixes, no splitting the file into framed segments. |
| SYNC-34 | Receivers MUST read the byte stream directly into the destination buffer (one allocation per block); implementations MUST NOT re-chunk large blocks for transport. |
| SYNC-35 | Pump writes MAY use `subarray` slices over the source `ArrayBuffer`; avoid base64/JSON/text encodings of block ciphertext (SYNC-30). |
| SYNC-36 | Receivers MUST compute `SHA-256` incrementally as bytes arrive (single-thread per block), MUST verify the resulting digest against the expected block hash from `stream-begin`, and on success MUST persist via `BlockStoreApi.storeAlreadyVerified(hash, data)` (see `storage/log-api-v1.md` §2.3). Callers other than the sync receiver MUST NOT use `storeAlreadyVerified`. |
| SYNC-37 | Block-on-disk commit MUST be idempotent under concurrent delivery. Implementations using a tmp-then-rename pattern MUST give each in-flight stream a unique scratch path (so concurrent writers never clobber each other), MUST treat a populated final path at commit time as success (drop the verified scratch — the bytes ARE the hash, so the merge is identity), and MUST tolerate `ENOENT`/`EEXIST` on the atomic rename when the final path exists. No file locks, no copies, no coordination: content-addressed storage is CRDT-trivial. |

## 5. Concurrency model

| Rule | Requirement |
|------|-------------|
| SYNC-50 | Block fetches MUST be allowed to run in parallel across distinct peers/friends/siblings and across distinct blocks, subject to SYNC-14 (per-hash single-flight). The number of in-flight block streams per association is bounded by the implementation; the number of in-flight associations is bounded by the authorized set (friend ∪ sibling). |
| SYNC-51 | Per-block hashing is single-thread SHA-256 (hardware-accelerated where available). Cross-block parallelism MUST come from concurrent in-flight blocks, not from intra-block tree hashing, as long as the file-level encoding splits files into individually addressable blocks below the single-thread SHA-256 budget (see `application/file-events-v0.3.md`). |
| SYNC-52 | Senders MAY pump multiple block streams concurrently as long as each duplex serialises framed writes per SYNC-01; receivers MAY hash and persist them in parallel by routing each stream to an independent `BlockStoreApi.storeAlreadyVerified` call. |

## 6. Handshake robustness

| Rule | Requirement |
|------|-------------|
| SYNC-55 | Handshake failures classified as expected (timeout, race, policy) MUST be surfaced per `sync-observability-v1.md` OBS-40–OBS-43 and MUST NOT abort unrelated associations. |

## 7. Bootstrap and recovery

The steady-state and bootstrap path is reactive `have`/`want` driven by each instance's durable reception journal (SYNC-10). Persistent fetch cursors (SYNC-19--SYNC-22) make interrupted pagination efficient across reconnects.

| Rule | Requirement |
|------|-------------|
| SYNC-60 | **New-machine recovery.** A fresh `dataDir` that serves a profile and reaches at least one online friend or sibling whose reception journal contains the relevant history MUST converge to the objects advertised by that remote instance without manual reindex steps. The fresh receiver has no stored cursor for that `(remote profile, remote instance)` pair, so its initial `delta` starts at the beginning of that remote instance's durable reception stream. |
| SYNC-61 | **Set reconciliation vs steady-state.** Initial bulk catch-up MAY later use hash-set, Bloom-filter, IBLT, Merkle-prefix, or other set-reconciliation optimisations, but steady-state sync MUST remain the same receiver-driven `have`/`want` semantics as SYNC-10. Any optimisation MUST be equivalent to advertising stored object refs and fetching missing hashes. |
| SYNC-62 | Operators moving to a new device SHOULD still keep a reachable peer online until `have.more === false`, all emitted `want`s have drained, and application replay succeeds. Peer presence alone is not a proof of convergence; empty inflight queues plus successful file/timeline reads are. |
