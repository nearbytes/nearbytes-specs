# Log API (normative)

**Status:** normative  
**Version:** 1.1  
**Implementation:** `nearbytes-log`

---

## 1. Scope

Defines the **only** persistence contract for the clean-code layer. Callers MUST NOT use a generic path→bytes VFS as the public abstraction.

The log is the **sole authority of the content-address namespace**: callers MUST NOT compute a block's hash and pass it to the log; the log computes the hash from the bytes it persists and returns it. The single exception (§2.3) is the streaming receiver path used by `nearbytes-sync`, which computes the hash incrementally as bytes arrive over the wire and MAY assert it back to the log via a separate method.

---

## 2. Types

### 2.1 `Log`

```ts
interface Log {
  readonly events: EventLogApi;
  readonly blocks: BlockStoreApi;
}
```

### 2.2 `EventLogApi`

| Method | Semantics |
|--------|-----------|
| `storeEvent(publicKey, signedEvent)` | Persist event; return content-address `SHA-256(serializeEventEnvelope(envelope))`. Idempotent on same hash. |
| `retrieveEvent(publicKey, eventHash)` | Load, `validateEventBytes`, verify signature; delete file and throw on failure. |
| `listEvents(publicKey)` | List event hashes in channel directory; ignore non-`<64-hex>.bin` names. |

### 2.3 `BlockStoreApi`

| Method | Semantics |
|--------|-----------|
| `store(data, skipIfExists?)` | Compute `hash = SHA-256(data)`, write `blocks/<hash>.bin`, return `hash`. Skip write if `skipIfExists` and file exists. The hash returned is authoritative; callers MUST use it (and only it) to reference the block. Idempotent on same hash, including under concurrent writers (the file IS the hash, so the merge is identity). |
| `storeAlreadyVerified(hash, data, skipIfExists?)` | Streaming-receiver fast path: the caller asserts that `hash == SHA-256(data)` has already been verified incrementally. The log MUST NOT recompute the digest. Use only from `nearbytes-sync`'s receive sink; misuse silently corrupts the content-address namespace. Idempotent on same hash, including under concurrent writers. |
| `retrieve(hash, { verifyIntegrity? = true })` | Read; if `verifyIntegrity`, run `validateBlockBytes`; delete and throw on mismatch. |
| `has(hash)` | Existence check for canonical block path. |

Normative rules:

1. The block hash algorithm is **SHA-256** of the encrypted block bytes (lower-case 64-hex). The protocol reserves the right to evolve this primitive (e.g. to RFC 6962 §2.1 Merkle Tree Hash over SHA-256) at a future major version; see `engineering/hash-evolution-v1.md`.
2. `store(data, …)` MUST NOT accept a hash parameter. The log is the sole producer of the address.
3. `storeAlreadyVerified` MUST be exposed but is **not** the default. It exists exclusively to avoid re-hashing during the `nearbytes-sync` streaming receive path; consumers of the log other than `nearbytes-sync` MUST use `store`.
4. `retrieve` with `verifyIntegrity: true` (default) MUST recompute the digest and reject mismatches. Sync senders MAY pass `verifyIntegrity: false` when streaming a block to a peer because the receiver re-verifies on arrival.
5. Block-on-disk commit MUST be safe under concurrent writers for the same hash. Implementations using a tmp-then-rename pattern MUST give each in-flight write a stream-unique scratch path (so two writers never share a tmp), MUST treat a populated `blocks/<hash>.bin` at commit time as success (drop the verified scratch), and MUST tolerate `ENOENT`/`EEXIST` on the atomic rename when the final path exists. Content-addressed bytes are a CRDT under identity merge: no file locks, no copies, no coordination are required. This rule is the storage realisation of `sync-protocol-v1.md` SYNC-37.

### 2.4 `ChannelPathMapper`

`(publicKey: Uint8Array) => string` — default: `channels/<130-hex-lowercase>`.

### 2.5 Channel identity and replay

| Type / function | Semantics |
|-----------------|-----------|
| `Channel` | `{ publicKey, secret }` — deterministic identity from a seed |
| `openChannel(secret, crypto)` | Derive keys; no I/O |
| `loadEventLog(channel, log, crypto)` | List + retrieve + hydrate all channel events; sort by `eventHash` |
| `verifyEventLog(entries, channel, crypto)` | Verify envelope signatures against `channel.publicKey` |

Replay is **generic event sourcing**: it does not interpret inner payloads. Domain packages (e.g. `nearbytes-files`) project replayed entries into file state, chat timelines, etc.

---

## 3. Implementations

| Factory | Environment |
|---------|-------------|
| `createFilesystemLog(dataDir, pathMapper?)` | Node.js, single root on disk |
| `createInMemoryLog(options?)` | Tests, embedded runtimes |

Additional implementations (IndexedDB, multi-root) MUST implement the same `Log` interface without extending a shared VFS type.

---

## 4. Protocol modules (implementation-agnostic)

These modules MUST be used by every `Log` implementation:

- `serialization` — envelope and inner payload codecs  
- `integrity` — `validateEventBytes`, `validateBlockBytes`, canonical path parsing  
- `eventEnvelope` — `createSignedEvent`, `hydrateSignedEvent`, …

---

## 5. On-disk layout

Filesystem implementations MUST use paths from §2 under `dataDir` as documented in `meta-storage-v0.3.md` / `meta-storage-v2.md`.

---

## 6. Removed API

`StorageBackend`, `createLog(storage)`, `EventLog` class, `BlockStore` class, and `MultiRootStorageLike` duck typing are **not** part of v1. `nearbytes-app` must migrate to `Log` factories.
