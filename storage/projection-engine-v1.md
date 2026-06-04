# Projection engine (normative)

**Status:** normative
**Version:** 1.0
**Implementation:** `nearbytes-log` (cores), `nearbytes-files` / `nearbytes-chat`
(projectors), `nearbytes-engine` (wiring)

---

## 1. Scope

This document defines how replayed events are projected into persistent,
incrementally-maintained application state. It defines two reusable cores in
`nearbytes-log` — a **log router** and an **order-agnostic projection engine** —
and the **projector** contract that application protocols implement.

It supersedes the in-memory replay-cache description in
`application/file-events-v0.5.md` §11 and the full-reload chat replay in
`application/chat-v1.md` §5. It settles the "ordering/replay layer" left
non-final in `registry/protocol-registry.md` §5 and resolves the concerns in
`WIP.md` §§2–8.

This document does not change the on-disk event/block formats
(`storage/log-api-v1.md`, `storage/data-correctness-v0.2.md`) nor `blockRefs`
semantics (`application/blockrefs-v0.1.md`).

---

## 2. Design principles

1. **Push, not poll.** New events are pushed to consumers as they are persisted.
   Implementations MUST NOT rebuild state by periodically re-listing a channel
   directory.
2. **Ordering belongs to the application protocol.** The log router and the
   projection engine MUST be order-agnostic: they MUST NOT sort events and MUST
   NOT assume a total order exists. Any ordering — including a total order — is a
   projector policy expressed through `reorder` (§4).
3. **Always incremental.** A new event MUST be folded onto live state in time
   proportional to the affected suffix, never the whole channel, except on first
   materialization of a channel with no persisted state (§6.4).
4. **Persistent.** Materialized state, the order index, and snapshots MUST persist
   across process restarts through a `MaterializedStore` (§5).

---

## 3. Log router (PROJ-1)

`EventLogApi` (see `storage/log-api-v1.md`) gains a subscription:

```ts
subscribe(
  filter: { channel?: PublicKeyHex; protocols?: ReadonlySet<string> },
  sink: (entries: EventLogEntry[]) => void,
): () => void;   // returns an unsubscribe handle
```

- **PROJ-1.1** After a successful `storeEvent`, and after the `nearbytes-sync`
  receive path persists a channel event, the log MUST notify every matching
  `sink` with the newly observed entry/entries (batched).
- **PROJ-1.2** `filter.channel` selects one channel by lowercase-hex public key;
  `filter.protocols`, when present, selects inner payload protocol IDs. A `sink`
  whose filter does not match an event MUST NOT be invoked for it.
- **PROJ-1.3** The router is key-agnostic: it MUST NOT require channel secrets and
  MUST push raw `EventLogEntry` values. Payload hydration/decryption is the
  projection engine's responsibility (§6).
- **PROJ-1.4** Routing MUST be O(1) per event per subscriber; it MUST NOT scan the
  channel directory.

---

## 4. Projector contract (PROJ-2)

An application protocol projects events into a state value `TState` using a
compact, serializable order key `TKey`:

```ts
interface Projector<TState, TKey> {
  readonly id: string;                         // registry protocol id, e.g. 'nb.files.v0.5'
  initial(): TState;
  serialize(state: TState): Uint8Array;
  deserialize(bytes: Uint8Array): TState;
  key(entry: EventLogEntry): TKey;
  reorder(prevKeys: readonly TKey[], newKeys: readonly TKey[]):
    { keys: TKey[]; insertAt: number };
  reduce(base: TState, orderedTail: readonly EventLogEntry[]): TState;
}
```

- **PROJ-2.1** `key` MUST extract only the information the projector needs to
  order events (e.g. event hash, causal parent, timestamp), small enough to
  persist for every event without the payload.
- **PROJ-2.2** `reorder` returns the projector's canonical sequence of keys for
  the union of `prevKeys` and `newKeys`, and `insertAt`: the smallest index whose
  key differs from the previously persisted sequence. A **commutative /
  append-only** projector MUST return `{ keys: [...prevKeys, ...newKeys],
  insertAt: prevKeys.length }` (the `appendOrder` default). An **ordered**
  projector MAY return a smaller `insertAt` when a new event sorts before existing
  ones.
- **PROJ-2.3** `reduce(base, orderedTail)` MUST fold a contiguous run of ordered
  entries onto `base` and MUST be deterministic. `reduce(initial(), fullOrder)`
  MUST equal the result of incremental folding (`reduce` is a fold over the same
  order).
- **PROJ-2.4** `serialize`/`deserialize` MUST round-trip `TState` for live and
  snapshot persistence.
- **PROJ-2.5** A projector MUST NOT perform routing decisions; it never chooses
  which other protocols receive an event.

---

## 5. MaterializedStore (PROJ-3)

Persistence is a pluggable interface, per `(projectorId, channelHex)` namespace:

| Concern | Semantics |
|--------|-----------|
| order index | persist/load the projector's `TKey[]` sequence |
| live state | persist/load the serialized live `TState` and its order position |
| snapshots | put/list/load/delete serialized `TState` keyed by `{ position, createdAt }` |
| meta KV | small string KV (e.g. chat last-seen event hash, reply target) |

- **PROJ-3.1** Implementations MUST provide `createInMemoryMaterializedStore()`
  for tests and browser builds.
- **PROJ-3.2** The reference durable implementation is
  `createSqliteMaterializedStore(path)` over the built-in `node:sqlite`. It is
  Node-only and MUST NOT be referenced from a package's browser entry point.
- **PROJ-3.3** SQLite tables are namespaced by `(projectorId, channelHex)`:
  `order_index`, `live_state`, `snapshots`, `meta`.
- **PROJ-3.4** Files-domain state persists to `files.sqlite3`, chat-domain state
  to `chat.sqlite3`, under `dataDir/.nearbytes/` (see `meta-storage-v0.3.md`).
- **PROJ-3.5** The store interface is the only persistence dependency of the
  engine; a future revision MAY back it with an internal NearBytes volume
  protocol without changing projectors.

---

## 6. Projection engine (PROJ-4)

`createProjection(log, channel, crypto, projector, store)` yields an instance
with `ingest(entries)`, `state()`, and a change subscription.

- **PROJ-4.1 Load.** On creation it MUST load the persisted order index, live
  state, and snapshot metadata from the store, and subscribe to the router
  (§3) for `channel`.
- **PROJ-4.2 Ingest.** `ingest(entries)` is invoked by the router sink and by the
  boot path (§7). It MUST:
  1. drop entries whose hash is already in the order index;
  2. compute `{ keys, insertAt } = projector.reorder(prevKeys, newKeys)`;
  3. if `insertAt === liveVersion` (the live state's order position), **forward
     fold**: `live = reduce(live, hydrate(tail))` where `tail` is the ordered
     suffix from `liveVersion`;
  4. otherwise, select the nearest snapshot with `position ≤ insertAt` (or
     `initial()` if none), hydrate the ordered suffix from that position, and
     `reduce` over it;
  5. persist the new order index and live state; conditionally write a snapshot
     (§6.3) and prune (§6.3); emit a change event.
- **PROJ-4.3 Bounded hydration.** Only the ordered suffix actually folded MUST be
  hydrated (decrypted). Implementations MUST NOT hydrate the whole channel on an
  incremental ingest.
- **PROJ-4.4 Warm reads.** `state()` MUST return live state without re-reading the
  log when the engine is warm.

### 6.3 Snapshot ladder (PROJ-5)

Snapshots are checkpoints of serialized `TState` at an order position, tagged with
wall-clock `createdAt`.

- **PROJ-5.1** Retention MUST be logarithmic: at most one snapshot per **day** for
  recent days, collapsing to at most one per **week** further back and one per
  **month** oldest, always retaining the most recent snapshot.
- **PROJ-5.2** Nearest-snapshot selection for replay MUST return the snapshot with
  the greatest `position ≤ target`.
- **PROJ-5.3** Snapshot maintenance (write + prune) MUST be internal to the engine
  and store; callers MUST NOT manage snapshots.

### 6.4 First materialization

When no persisted state exists for a channel, the engine MUST perform one full
materialization (`reduce(initial(), fullOrder)`), then persist live state and an
initial snapshot. This is the only non-incremental path.

---

## 7. Boot and bucketing (PROJ-6)

- **PROJ-6.1** On boot, a host MUST read only events newer than each channel's
  persisted order index, bucket them by channel, and `ingest` them
  volume-by-volume. It MUST NOT push events one at a time across volumes.
- **PROJ-6.2** A single channel's event log MAY feed multiple projectors (e.g.
  FILES and CHAT share one hub channel). Each projector maintains its own order
  index, live state, and snapshots under its own `projectorId` namespace.

---

## 8. Incremental guarantees (PROJ-7)

| Trigger | Behavior |
|--------|----------|
| local write | `storeEvent` → router → forward fold → persist |
| sync inbound | reception persists → router → forward fold, or snapshot-bounded replay only when `insertAt < liveVersion` |
| boot / warm read | live state loaded from store; only newer events ingested |
| cold / new volume | one full materialization (§6.4), then snapshot + persist |

Implementations MUST NOT gate materialization on the local presence of
`blockRefs` dependencies. Verified events are folded as they arrive; the live view
MAY be partial and becomes complete as more events/blocks arrive. Byte reads of
file content MAY still fail until the content block exists locally
(`application/blockrefs-v0.1.md` §4).

---

## 9. Compatibility

The projector ids registered for v1 are `nb.files.v0.5` and `nb.chat.v1`
(`registry/protocol-registry.md`). The on-disk event/block format is unchanged;
removing a `MaterializedStore` database MUST cause a transparent full
re-materialization (§6.4) on next access with no data loss.
