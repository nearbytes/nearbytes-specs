# WIP — Replay, engine layering, and recent repo work

Notes from investigation starting at: *"great! so I'm worried about the replay strategy, is it always incremental for chat and files"* (2026-06-04). No code changes in this section — documentation only.

---

## 1. Work completed just before that message (context)

Two separate tracks were finished and pushed to `main` on all affected repos:

### A. Dev bootstrap (`yarn dev`)

- Added `scripts/dev-bootstrap.mjs` in consumer repos: runs `yarn install`, then `yarn update` (if script exists), then `yarn refresh` (if script exists).
- Wired into `yarn dev` (and app `build` / `package`) where `dev` already existed: **cli**, **app**, **engine**, **components**, **widgets**, **files**.
- Also committed standalone `dev-bootstrap.mjs` in **benchmarks**, **chat**, **log**, **skeleton**, **sync** (no `dev` script there).

### B. Remove duplicate CLI from `nearbytes-files`

- Deleted `src/cli/`, `src/dev/`, `src/webdav/`, `bin.nbf`, `scripts/webdav-serve.mjs`.
- Added `src/probeRuntime.ts` + export `nearbytes-files/probe-runtime` (integration-test runtime).
- Kept deprecated alias `nearbytes-files/cli/context` → same module.
- Probes and **benchmarks** updated to import `probe-runtime`.
- `yarn dev` for files is now `dev-bootstrap && tsc --watch` (library only; canonical `nbf` is **nearbytes-cli**).

---

## 2. Replay strategy — files vs chat

### Files — incremental by design (with fallback)

**Where:** `nearbytes-files` (`FileService`, `fileEmit.ts`, `fileMaterializer.ts`, `fileLogEntries.ts`). Spec: `file-events-v0.5.md` §11.

**In-memory cache per volume secret (`FileReplayContext`):**

| Field | Role |
|-------|------|
| `fs` | `MaterializedFileSystem` — live directory tree snapshot |
| `orderedEntries` | Hydrated channel log in **causal replay order** |
| `liveEncryptedKeys` | path → wrapped key for decrypting blobs |
| `observedHead` | Causal head for emitting new events |

**Incremental paths:**

1. **Warm read** — `getReplayContext` returns cache if not stale.
2. **Local write** — `commitReplayEntry` → `extendFileReplayContext` (no disk rescan).
3. **Inbound fast path** — `applyInboundEvent(secret, eventHash)` hydrates **one** event, extends cache.
4. **Stale refresh** — `markReplayStale` + reload with **seed**: `newHashes = listEvents(disk) − cache`; hydrate only new hashes; `orderEventLogEntries`; if **cache prefix** holds → `materializeIncremental` on tail only.
5. **Timeline delta** — `getTimelineDelta` slices from cached `orderedEntries` (cursor API).

**Full replay fallback when:**

- No cache / cold start.
- `hasOrderedPrefix` fails (reorder, gap, repair — cached ordered list is not identical to the first N entries of the newly computed order).
- Historical `throughEventHash` views (prefix slice + rematerialize).

**Important:** Replay range is **not** inferred from FILES payload paths (e.g. “this DELETE touches `/foo`”). It is always: hash diff vs cache → causal reorder → materialize suffix (or whole log).

Chat/identity/`OTHER` events in the same hub channel are loaded in the log but materializer treats them as `OTHER` — they do not change `fs.files` / `fs.directories`.

### Chat — not incrementally cached for reads

**Where:** `nearbytes-chat` (`readChatTimeline` → `loadEventLog` + `projectChatTimeline`).

- Every bulk read lists **all** event hashes, retrieves and hydrates **each** event (`nearbytes-log` `loadEventLog`).
- **No** replay cache like `FileService`.
- `engine.chatRead()`, `chatSay` refresh, REPL `chat` command → full timeline rebuild.

**Incremental only for live delivery:**

- `nearbytes-engine` `attachChatPush`: `seen` set per channel; `event-received` → one-event hydrate; `volumeRefreshHooks` → scan listed hashes for unseen; `seed()` → one full `readChatTimeline` to baseline `seen`.
- REPL live bubbles use push; bulk reads do not.

### Same hub channel

Files materialization and chat projection share one event log per hub secret. File replay cache still walks the full channel when extending; chat reads always walk the full channel on `readChatTimeline`.

---

## 3. Materialized state (files)

**Type:** `MaterializedFileSystem` (`fileMaterializer.ts`) — plain JS object, not on-disk FS.

```ts
{
  files: Map<string, FileMetadata>,           // live files
  directories: Map<string, DirectoryMetadata>,
  fileOrigins: Map<string, string>,           // path → CREATE tiebreak (event hash)
  entryHeads: Map<string, string>,            // path → last writer event hash
  shadows: Array<{ tiebreak, reject }>,       // rejected/conflict replay rows
}
```

**UI layer:** `ReactiveVolume` projects to `VolumeFileSystemState` via `volumeStateFromMaterialized` (Svelte-store-shaped, no Svelte dep).

Blobs stay in log/block store; materialized state is paths + metadata + `blobHash` pointers.

---

## 4. Incoming sync event — how files react

### Readiness (“waits”)

**Not** a blocking wait or retry queue. `inboundEventReadyToMaterialize` (in **probeRuntime** only) returns **false** → handler **skips** that notification. Later `block-received` or another event may trigger reload.

Checks `blockRefs`: parent event(s) and content blocks must exist locally (log or `dataDir` block path). If deps never arrive (lost data, bad refs), materialized state does not update until repair/full resync.

### “Apply fails”

`applyInboundEvent` returns **`undefined`** when:

- No warm replay cache (`state.context` missing).
- `retrieveEvent` fails or envelope does not match channel key.
- Caller then uses `reloadVolumeFromDisk` (stale + seed path).

### “Cache prefix”

`hasOrderedPrefix(previous, next)`: for all `i < previous.length`, `previous[i].eventHash === next[i].eventHash`.

Meaning: cached causal order is unchanged at the front; only a **tail** was appended. Then materialize `orderedEntries.slice(previous.length)` via `materializeIncremental`.

If any middle hash or order changes → full `loadEventLog` + full `runMaterialization`.

### Production vs probes (three behaviors)

| Consumer | On sync `event-received` / `block-received` |
|----------|-----------------------------------------------|
| **CLI REPL** | `nearbytes-files` `attachSyncInboundRefresh` (via engine): channel-scoped, `applyInboundEvent` first, open volumes only. |
| **Desktop app** | `NearbytesEngine.boot()` registers `sync.onEvent` → `refreshActive()` only for **active hub**: `markReplayStale` + `listFiles` + full `readChatTimeline`. Does **not** call `attachSyncInboundRefresh`. |
| **Probes / benchmarks** | `nearbytes-files` `probeRuntime.attachSyncInboundRefresh`: `block-received` → reload open volumes in `volumeRegistry`; `event-received` → readiness check → per-channel `applyInboundEvent` then fallback reload. |

**Conclusion:** CLI and app both use engine but **different sync policies**; neither uses probeRuntime’s smart hook. Integration tests use the smarter path in **files**.

---

## 5. Where replay lives (package map)

| Package | Responsibility |
|---------|----------------|
| **nearbytes-log** | Generic `loadEventLog`, `verifyEventLog`, hydrate/retrieve (channel-agnostic). |
| **nearbytes-files** | File replay cache, causal order, materializer, `FileService`, `probeRuntime`. |
| **nearbytes-engine** | Orchestration: runtime wiring, blunt sync hook, `NearbytesEngine` API, `chatPush`. |
| **nearbytes-cli** | REPL/UI; re-exports engine runtime; `attachChatPush` for live chat. |
| **nearbytes-chat** | `readChatTimeline`, `publishChatMessage`, codecs (no replay cache). |
| **nearbytes-skeleton** | Config, skeleton (crypto+log+sync), FS watchers. |

Replay **logic** is not in engine; engine calls `fileService.getReplayContext` / `markReplayStale`.

---

## 6. `nearbytes-engine` — role and critique

**Not a dumb proxy today.** ~4 source files:

- **`runtime.ts`** — Boot skeleton + `FileService`, volume/watcher maps, `reloadVolumeFromDisk`, **blunt** `attachSyncInboundRefresh`.
- **`engine.ts`** — Large shell API: profile/friend/hub CRUD, config persist, file ops delegation, full chat refresh, identity publish, `EngineEvent` bus, **`ui-state.json`**.
- **`chatPush.ts`** — Live chat push (`seen`, per-event ingest, hook on volume reload).

**CRUD** — Create/Read/Update/Delete for profiles, friends, hubs in `NearbytesEngine`.

**`ui-state.json`** — `{ activeHub }` under `dataDir/.nearbytes/`; restored on `boot()` so app/CLI can re-`hubUse` last hub. Session UX, not a protocol type.

**Blunt sync hook concern:** O(open volumes) × stale reload on **any** sync traffic, including unrelated channels; ignores `applyInboundEvent` and readiness; duplicates smarter logic in `probeRuntime.ts`.

### Agreed target architecture (from discussion)

Engine should ideally only **wire** protocol instances after boot:

- **files** — replay, materialization, inbound refresh (single implementation).
- **chat** — timeline, publish, push.
- **skeleton** — config, profiles, friends, sync.
- **crypto / log** — primitives.

Shells (CLI, app) call protocols directly or through a thin coordinator — not a second `NearbytesEngine` that re-implements `readChatTimeline` and divergent sync policies.

**Action items:**

1. ~~Move smart `attachSyncInboundRefresh` into **files**; delete blunt engine version~~ **Done** (`syncInboundRefresh.ts`; engine delegates).
2. Unify CLI and app on one inbound-sync + chat refresh policy (app still uses `refreshActive` on every sync event).
3. Add chat replay cache (optional) mirroring files seed/tail pattern.
4. Shrink or remove `NearbytesEngine` monolith; keep `ui-state` in shell or skeleton.

**2026-06-04 follow-up:** `nearbytes-files/src/syncInboundRefresh.ts` is canonical; engine `attachSyncInboundRefresh` delegates (blunt reload-all removed). Push **files** before **engine** so `yarn refresh` resolves the new export.

---

## 7. Quick reference — inbound files (10-line version)

1. Sync emits `event-received` / `block-received`.
2. Inbound path today skips until `blockRefs` deps look local (**target: remove this** — see §8).
3. Fast: `applyInboundEvent` — one hash, extend cache, incremental materialize if prefix OK.
4. Slow: `markReplayStale` + `getReplayContext` — `newHashes` on disk minus cache.
5. Re-sort merged log (`orderEventLogEntries` over `blockRefs` DAG).
6. Prefix OK → materialize tail only; else full channel replay.
7. CLI: channel-scoped inbound refresh via `syncInboundRefresh.ts` (since c0dd6d4).
8. App: `refreshActive` on **active hub** only + full chat reread.
9. Chat bulk reads: always full `loadEventLog`; push path is per-event.
10. Replay implementation: **nearbytes-files**; engine orchestrates only.

---

## 8. `blockRefs`, inbound gating, and materialization philosophy

### Design direction (target behaviour)

**We should not wait for causal dependencies before materializing.** Apply and materialize every verified event as soon as we have it. The live view may be **partial** (missing paths, stale ordering on unrelated paths, unreadable blobs until sync catches up) and can become **more complete** as more events and blocks arrive. Deferral is not a first-class user-visible state — progress is.

Read paths (`getFile`, decrypt) may still fail until content blocks exist; **tree / timeline / metadata** should not block on blobs or on unrelated channel events.

---

### Typical `blockRefs` on a FILES `CREATE_FILE` (`buildBlockRefs` in `fileEmit.ts`)

| Ref | Typical hash | Role when **emitting** |
|-----|----------------|------------------------|
| `[0]` | **observedHead** | Last FILES event on the channel (any path) — causal “I built on this head”. |
| `[1]` | **supersededEvent** (optional) | Prior event that last touched **this** path. |
| `[2]` | **lastBlock** (optional) | Prior **content blob** at this path (overwrite lineage). |
| last | **Y** | New encrypted blob (`introducedBlocks`; also in payload `content.blockHash`). |

`MKDIR` / `DELETE` / `RENAME`: no new blob — usually `[observedHead, supersededEvent?, lastBlock?]` only.

Only **`blockRefs[0]`** is used as the **parent link** in `orderEventLogEntries`. Other refs are lineage / carriage on the envelope; the materializer reads the **payload** (path, content ref, verbs), not those refs directly.

---

### What the code does today (`syncInboundRefresh.ts`)

**Open volumes only** — replay where a `ReactiveVolume` is live (REPL / active hub).

**`block-received`:** full stale reload of open volumes (conservative catch-all).

**`event-received`:** `inboundEventReadyToMaterialize` requires:

- `blockRefs[0]` in the channel event list (when multiple refs), or block-on-disk if sole ref; and  
- every other ref not in the event list must exist as a **block** on disk.

If not → skip (no queue; retry on a later sync notification).

---

### Why much of this waiting is wrong

**Content blobs (`lastBlock`, new Y):** Materialized tree state comes from the **event payload**, not from reading blob bytes. Waiting for **Y** (or old **lastBlock**) before `ls` / timeline has no FILES-semantics benefit; only **`getFile`** should care about **Y**.

**`supersededEvent`:** Lineage on emit. If already in the log, the checker skips it; if missing, the code incorrectly treats unknown **event hashes** like blocks (`blockReady`). Not a materialization prerequisite for the new path state.

**`observedHead`:** Parent **event** on the channel — often about a **different file** (e.g. “file 1 = X” then “file 2 = Y”). For **reading file 2**, you do not need file 1’s **content**; orphan in the **global DAG** does **not** mean “no semantics” for file 2’s payload.

**When global order actually matters:** same path (overwrite/delete/rename chain), directory cascades, latest-wins on overlapping namespace. **When it mostly doesn’t:** independent paths — `/a` and `/b` are self-contained in their payloads.

Today’s engine **couples** unrelated paths by refusing to merge until `observedHead` exists, so `orderedEntries` and incremental prefix stay globally consistent. That is an **implementation choice**, not a logical requirement for showing file 2 once its event is verified.

**“Wait for parent so we don’t insert before parent exists”** — precise meaning: without the parent **event record** in the hydrated log, `orderEventLogEntries` / `hasOrderedPrefix` can mis-order or force full reload. It is **not** “wait to avoid not waiting”; it is deferring merge until a **global ordering** constraint is satisfied — a constraint we **want to relax** per design direction above.

---

### Likely fix direction (not implemented)

- Remove `inboundEventReadyToMaterialize` gating (or limit to signature verify + event on disk).
- Materialize each verified FILES event into state **immediately**; accept partial channel views; reconcile when more events arrive (re-materialize or per-path merge).
- Gate **byte reads** on blob presence only.
- Revisit `block-received` full reload vs retry pending channels only.

---

## 9. Related commits (pushed `main`)

| Repo | Notes |
|------|--------|
| nearbytes-files | CLI removal, `probeRuntime`, `syncInboundRefresh.ts` (c0dd6d4) |
| nearbytes-engine | Delegates inbound sync to files; blunt reload-all removed |
| nearbytes-cli, app, engine, components, widgets | dev-bootstrap wired to `dev` |
| nearbytes-benchmarks | `probe-runtime` imports + dev-bootstrap |
| nearbytes-specs | This WIP |
| chat, log, skeleton, sync | dev-bootstrap script only |

---

*Last updated: 2026-06-04 — materialization philosophy: apply events as we have them; partial → total over time; no causal wait.*

---

## 10. Resolution — projection engine (2026-06-04)

§§2–8 are superseded by the normative `storage/projection-engine-v1.md` and the
design note `nearbytes-design/design/projection-engine-v1.md`. Replay becomes a
**push** architecture with two shared cores in `nearbytes-log` — a log router
(`subscribe(filter, sink)`) and an **order-agnostic** projection engine — plus
per-protocol **projectors** (`reorder` + `reduce` + key + state codec) in
`nearbytes-files` (`nb.files.v0.5`, ordered/causal) and `nearbytes-chat`
(`nb.chat.v1`). Ordering is the protocol's concern, never the log's/engine's: a
total order is one projector's policy. Materialized state, the order index, and a
**logarithmic snapshot ladder** (day/week/month) persist via a `MaterializedStore`
(`files.sqlite3` / `chat.sqlite3` under `dataDir/.nearbytes/`, `node:sqlite`),
deletable and rebuildable by full re-materialization. `nearbytes-engine` becomes
dumb wiring that exposes the files/chat APIs and boot-buckets new events per
volume; the blunt `attachSyncInboundRefresh` reload-all, the app `refreshActive`
full reread, and `chatPush.ts` are removed in favour of one router-driven push
policy. Full replay happens only on first access of a channel with no persisted
state.
