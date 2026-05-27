# Sync discovery requirements v1

Status: normative for `nearbytes-sync`.

## 1. Unified discovery layer

| Rule | Requirement |
|------|-------------|
| DISC-01 | All discovery backends MUST implement `PeerDiscovery`: `start`, `onPeer`, `stop`. |
| DISC-02 | `onPeer` MUST emit normalized `DiscoveredPeer` values (`duplex` or `tcp`). |
| DISC-03 | A composite discovery MUST fan-in multiple backends without protocol changes. |

## 2. Hyperswarm (DHT)

| Rule | Requirement |
|------|-------------|
| DISC-10 | A sync node MUST join $\topic(\mathsf{profile}(p))$ for every served local profile $p$ (the set from `sync-protocol-v1.md` SYNC-00) **and** for every configured friend $f$. The topic set is the union $\{\,\topic(\mathsf{profile}(p)) : p \in \text{served}\,\} \cup \{\,\topic(\mathsf{profile}(f)) : f \in \text{friends}\,\}$. |
| DISC-11 | Topic derivation MUST be `H("nearbytes:sync:v1" \|\| canonical(subject))`. |
| DISC-12 | On an incoming Hyperswarm connection, the implementation MUST consult the connection's topic set (e.g. `peerInfo.topics` in the JS hyperswarm library). It MUST pick the local profile for the association as the first served profile $p$ whose $\topic(\mathsf{profile}(p))$ appears in that set; if no served profile matches (i.e. we are the *follower* on this association), the implementation MUST pick the **active** served profile (`NearbytesConfig.activeProfile`) and sign the handshake with it. |
| DISC-13 | Hyperswarm `DiscoveredPeer.label` values MUST follow the transport-label grammar in `sync-observability-v1.md` OBS-30–OBS-33. Implementations MUST resolve `dht:<host>:<port>` from the underlying socket when possible rather than advertising only an opaque swarm public-key prefix. |

## 3. mDNS / DNS-SD

| Rule | Requirement |
|------|-------------|
| DISC-20 | LAN discovery MUST publish one `_nearbytes._udp` record per served local profile, each with `alpn=nearbytes-sync-v1` and its own `syncPort`. |
| DISC-21 | Implementations MAY emit a UDP multicast fallback to `239.255.40.41:40441`; if used, the announcement MUST be repeated once per served local profile so the TXT `prof` field uniquely identifies one profile per announcement. |
| DISC-22 | After LAN discovery, peers connect via TCP and speak the same framed sync protocol as Hyperswarm duplex streams. |
| DISC-23 | LAN TXT MUST include `prof`, the lower-case hex profile public key of the advertised profile (130 hex chars). Implementations serving $K \ge 2$ local profiles MUST publish $K$ records with distinct `prof` values; each record's `syncPort` MUST be bound to a dedicated TCP listener so an inbound socket on `syncPort` unambiguously identifies the targeted local profile. |
| DISC-24 | LAN discovery MUST NOT open sync to advertisers whose `prof` is neither in `config.friends` nor in the set of served local profiles, MUST ignore announcements whose `peerId` equals the local `peerId` (process-level loopback), and MUST initiate the TCP dial as the served local profile that the advertised `prof` is **dialed from**, i.e. the implementation's outbound socket targets the advertiser's announced `syncPort` while the local handshake signs with the **active** served profile (the served profile chosen by the user via `profile use`). |
| DISC-25 | See `sync-protocol-v1.md` for association, handshake, and anti-entropy rules. |
| DISC-26 | **Sibling carriage.** A served local profile $p$ MAY be carried by two or more nodes (a single identity present on multiple devices). On LAN, when two siblings discover each other (matching `prof`, distinct `peerId`), exactly one side initiates the TCP dial: the node whose `peerId` is **lexicographically smaller** in lower-case hex. On Hyperswarm, both sides accept the connection and the upper protocol identifies siblings by `peerId` (handshake field) so two simultaneous sibling sessions never collide in the friend-session registry. |
| DISC-27 | **Node identity is anchored in `dataDir`, not in the OS process.** Implementations MUST treat `peerId` as a per-`dataDir` stable random identifier (created on first start, persisted under the storage root) so that two processes sharing the same on-disk log evaluate to the **same** `peerId` and are filtered as loopback by DISC-24, never as siblings. Purely in-memory deployments (no `dataDir`) MAY use a per-process random `peerId`. |
| DISC-27.1 | **Singleton sync, plural writers.** The exclusive `dataDir` lock protects the *sync engine* (discovery joins, peer-loop, outbound block pump), **not** the writer slot. At most one process MAY run the sync engine against a given `dataDir`; any number of processes MAY append events or blocks to the same `dataDir`. Implementations MUST take an exclusive advisory file lock at sync start and release it at sync stop. A second sync start against a locked `dataDir` MUST fail loudly with a typed error naming the holder. Implementations MUST also expose a side-effect-free `probe()` API that reports the lock state without acquiring it, so writer-only consumers can detect an active sync daemon and choose to become writer-only clients rather than competing for the slot. |
| DISC-27.2 | **Crash-safe lock.** The lock file MUST encode the holder's pid; on contention, implementations MUST check that the holder is alive (via `process.kill(pid, 0)` or equivalent) before reporting "already running", and MUST silently replace the lock if the holder pid is dead. Implementations SHOULD also encode a creation timestamp so probe results can surface "held since" without parsing the OS process table. |
| DISC-27.3 | **Concurrent writes are CRDT-trivial.** Content-addressed publishing (block and event filenames are SHA-256 of contents) makes the writer slot pluralisable without coordination. Implementations MUST publish each file via "write to `<final>.<rand>.tmp` → `link(2)` to `<final>` → `unlink(tmp)`": the random per-write tmp suffix isolates scratch spaces, and `link(2)` is the only atomic verb that fails on existing target — the loser observes `EEXIST` and treats it as success (their bytes ≡ the published bytes by definition). Implementations on filesystems without hardlink support (FAT, some NFS variants) MAY fall back to `rename(2)` since content-addressed naming preserves correctness either way. Implementations MUST reap orphaned `*.tmp` scratch files on storage-root initialisation, using a safety window long enough to never disturb a concurrent in-flight write. |
| DISC-27.4 | **Cross-process write propagation.** When a sync engine is running, it MUST also watch the `dataDir` for new event and block files appearing from other writer processes (e.g. via inotify / FSEvents / ReadDirectoryChangesW or an equivalent abstraction such as `chokidar`). For each newly-observed file, the sync engine MUST append a reception entry to its in-process log so the standard "appendReception → broadcast `have`" path fires (SYNC-10), making cross-process writes indistinguishable on the wire from writes authored by the sync engine itself. |

## 4. Boot integration

| Rule | Requirement |
|------|-------------|
| DISC-30 | `nearbytes-skeleton` MUST call `start(log, config.friends, { serveProfilePublicKeys, activeProfilePublicKey })` on boot, where `serveProfilePublicKeys` is the lower-case hex public key set derived from `config.profiles` and `activeProfilePublicKey` is the public key of `config.activeProfile`. |
| DISC-31 | Each `start` MUST append a marker line via `log.sync.appendMarker`. |
| DISC-32 | `NearbytesConfig.friends` MUST always be present (possibly empty). |
| DISC-33 | `NearbytesConfig.profiles` MUST always be present as an array (possibly empty) of `{ name, secret }`; names MUST be unique within the array. `NearbytesConfig.activeProfile` MUST be either `null` (no profile selected, sync inert per SYNC-00) or equal to one of the profile names. Implementations encountering a legacy `profileSecret: string` field MUST upgrade it in-place to `profiles: [{ name: "default", secret: profileSecret }], activeProfile: "default"`. |
