# Nearbytes engineering requirements

Normative requirements for clean-code packages (`nearbytes-crypto`, `nearbytes-log`, `nearbytes-sync`, `nearbytes-skeleton`, `nearbytes-files`, `nearbytes-specs`).

| Document | Scope |
|----------|--------|
| [portability-v1.md](./portability-v1.md) | Browser, Node.js, and Pear runtime portability |
| [sync-discovery-v1.md](./sync-discovery-v1.md) | Sync discovery layer (Hyperswarm + mDNS), friend / sibling carriage, dataDir-anchored node identity, singleton-sync / plural-writers split |
| [sync-protocol-v1.md](./sync-protocol-v1.md) | `nearbytes.sync.v1` framed anti-entropy protocol (`have` / `want` / `data`, single-flight, hash-set delta, blocks-first ordering) |
| [sync-observability-v1.md](./sync-observability-v1.md) | State beacon, in-process event bus, transport labels, handshake failure UX, reference CLI (`nbf peers` / `monitor` / `whoami`) |
| [benchmark-methodology-v1.md](./benchmark-methodology-v1.md) | Methodology for `nearbytes-benchmarks` (warm-cache loopback, $n{=}10$ medians, etc.) |

## Highlights of recent rule evolution

- **DISC-26** — `dataDir`-anchored node identity (per-storage-root random `peerId`), so two processes pointing at the same `dataDir` resolve to the same node and are correctly filtered as loopback, while two devices sharing one profile pass as siblings.
- **DISC-27.1** — *Singleton sync, plural writers.* The exclusive `dataDir` lock protects the *sync engine*, not the writer slot; a side-effect-free `probeSyncLock(dataDir)` API lets writer-only consumers (the file CLI, scripts) detect an active daemon and downgrade to writer-only mode instead of contending.
- **DISC-27.2** — *Crash-safe lock.* Lock files encode the holder pid + creation timestamp; stale locks held by dead pids are silently reclaimed.
- **DISC-27.3** — *Concurrent writes are CRDT-trivial.* Content-addressed publish uses `write→link(2)→unlink(tmp)` with a unique per-write tmp suffix; concurrent writers see `EEXIST` and treat it as success (bytes identical by content-address argument). Filesystems without hardlink support fall back to `rename(2)`.
- **DISC-27.4** — *Cross-process write propagation.* A running sync engine watches `dataDir` (inotify / FSEvents / ReadDirectoryChangesW or an equivalent abstraction) and appends a reception entry for each newly-observed file, so cross-process writes are indistinguishable on the wire from sync-engine-authored writes.
- **OBS-01–OBS-15** — *Observability.* State beacon (`.nearbytes-sync.state.json`), LIVE/DAEMON/WRITER-ONLY modes, `nbsync status` / `probeSyncLock`.
- **OBS-20–OBS-43** — *Event bus + handshake UX.* Wire events (`peer-connected`, `peer-connect-failed`, transfers), DHT `host:port` labels, classified handshake retries without stderr stack traces.
- **OBS-50–OBS-54** — *Reference CLI.* `nbf peers`, `nbf monitor`, `nbf whoami`, flush budgets for one-shot writers when friends are configured.
- **SYNC-15** — *Mutual anti-entropy.* Every live association runs identical subscribe/delta/have/want/data logic on both sides; transport dial direction does not assign roles.
- **SYNC-16–SYNC-17** — *Causal dependency repair.* After storing events/blocks and after each complete inbound `have` page, scan local orphans and issue explicit `want`s for missing parent events and content blocks named in visible `blockRefs`; serialized on the inbound queue, no timers.
- **SYNC-60–SYNC-62** — *Open recovery design.* New-machine bootstrap and set-reconciliation coupling called out explicitly; operators should not assume fresh `dataDir` auto-heals without a reachable peer. SYNC-16–SYNC-17 mitigate out-of-order child-before-parent delivery but not full historical bootstrap.
