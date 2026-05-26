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
| SYNC-00a | Each sync association MUST authenticate exactly one local profile from the served set against exactly one remote profile from the **authorized set** $= \text{served} \cup \text{friends}$; an association MUST NOT swap identities mid-stream. A sibling association (`remote profile` $\in \text{served}$) is allowed and MUST be treated identically to a friend association by the anti-entropy layer; only `sync-discovery-v1.md` DISC-27 loopback filtering distinguishes a sibling node from the local node itself. |

The terms *profile* and *identity* are not synonyms in this stack: a **profile** is the keypair / public-key noun used here, on the wire, and on disk; an **identity** is the social/display record (`nb.identity.record.v1`) signed *by* a profile. This spec governs the profile layer.

## 1. Transport association

| Rule | Requirement |
|------|-------------|
| SYNC-01 | Each association MUST serialize framed writes and message handling so frames are never interleaved on one duplex. |
| SYNC-02 | Every association (friend or sibling) MUST perform a `hello` exchange before any `delta`, `subscribe`, `have`, `want`, or `data` message. |
| SYNC-03 | `hello.senderProfile` MUST be the lower-case hex profile public key of the sender (equal to the local profile chosen for this association from SYNC-00). `hello.senderPeerId` MUST be a 32-char lower-case hex string identifying the sending **node** (per `sync-discovery-v1.md` DISC-27, this is the per-`dataDir` stable identifier and MUST be persisted there). The pair $(\text{senderProfile}, \text{senderPeerId})$ uniquely identifies the remote endpoint of the association. |
| SYNC-04 | The receiver MUST close the association if `senderProfile` is not in the authorized set (friend ∪ served) or policy denies the subject. The receiver MUST additionally close the association as a loopback (DISC-27) when `senderProfile == localProfile` **and** `senderPeerId == localPeerId`. |
| SYNC-05 | The receiver MUST reject a duplicate `sessionNonce` on the same association. |
| SYNC-06 | At most one active sync association per ordered triple $(\text{local profile}, \text{remote profile}, \text{remote peerId})$; a new association for the same triple MUST replace the previous one. The remote `peerId` MUST be part of the key so two sibling nodes sharing $\text{remote profile}$ each get a live session. |
| SYNC-07 | On a friend association, `subject` MUST be `profile(localProfilePublicKey)` — i.e. the profile being *targeted* on the receiver — and both sides MUST agree on `subject` for the association to proceed. The receiver MUST select its local profile by matching `subject.profile` against its served set; if no match exists the receiver MUST close the association. |

## 2. Anti-entropy

| Rule | Requirement |
|------|-------------|
| SYNC-10 | After local `acceptData`, objects enter the reception journal; each append MUST push `have` to all open friend associations (no timer-driven polling). |
| SYNC-11 | `want` is receiver-driven; senders MUST NOT send `data` without a prior `want`. |
| SYNC-12 | When both blocks and events are missing from a `have`, the receiver MUST issue `want` for blocks before `want` for events (blocks-first ordering). |
| SYNC-13 | `have` for events SHOULD include `blockRefs` when known so receivers can prioritize block fetches. |

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

## 5. Concurrency model

| Rule | Requirement |
|------|-------------|
| SYNC-50 | Block fetches MUST be allowed to run in parallel across distinct peers/friends/siblings and across distinct blocks. The number of in-flight block streams per association is bounded by the implementation; the number of in-flight associations is bounded by the authorized set (friend ∪ sibling). |
| SYNC-51 | Per-block hashing is single-thread SHA-256 (hardware-accelerated where available). Cross-block parallelism MUST come from concurrent in-flight blocks, not from intra-block tree hashing, as long as the file-level encoding splits files into individually addressable blocks below the single-thread SHA-256 budget (see `application/file-events-v0.3.md`). |
| SYNC-52 | Senders MAY pump multiple block streams concurrently as long as each duplex serialises framed writes per SYNC-01; receivers MAY hash and persist them in parallel by routing each stream to an independent `BlockStoreApi.storeAlreadyVerified` call. |
