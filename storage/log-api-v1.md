# Log API (normative)

**Status:** normative  
**Version:** 1.0  
**Implementation:** `nearbytes-log`

---

## 1. Scope

Defines the **only** persistence contract for the clean-code layer. Callers MUST NOT use a generic path→bytes VFS as the public abstraction.

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
| `store(hash, data, skipIfExists?)` | Write `blocks/<hash>.bin`; skip write if `skipIfExists` and file exists. |
| `retrieve(hash)` | Read and `validateBlockBytes`; delete and throw on mismatch. |
| `has(hash)` | Existence check for canonical block path. |

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
