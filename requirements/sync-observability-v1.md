# Sync observability and operations v1

Status: normative for `nearbytes-sync`, `nearbytes-skeleton`, and the reference file CLI (`nearbytes-files` / `nbf`).

This document specifies how operators observe a running sync engine, how a writer-only CLI coexists with a daemon, and how connection failures are surfaced without stalling the wire protocol.

It complements `sync-discovery-v1.md` (DISC-27) and `sync-protocol-v1.md` (wire messages). It does **not** redefine anti-entropy semantics.

## 1. Modes of operation

| Mode | Who holds the lock | Network | How operators observe peers |
|------|------------------|---------|-----------------------------|
| **LIVE** | This process (`start()` / REPL without daemon) | This process | In-process `SyncHandle` (`snapshot`, `peers`, `onEvent`, `stats`) |
| **DAEMON** | Another process (`nbsync daemon`) | The daemon | Poll `<dataDir>/.nearbytes-sync.state.json` (state beacon) |
| **WRITER-ONLY** | Another process (daemon) | None in this process | Beacon only; local writes propagate via DISC-27.4 |

| Rule | Requirement |
|------|-------------|
| OBS-01 | Implementations MUST expose whether the current consumer is LIVE, DAEMON (beacon reader), or WRITER-ONLY in user-facing tooling (`nbf peers`, `nbf monitor`, `nbf whoami`). |
| OBS-02 | WRITER-ONLY consumers MUST NOT open discovery backends or register peer-loop handlers. They MAY append to `dataDir` and MUST rely on the active sync engine to propagate new objects (DISC-27.4). |
| OBS-03 | For reliable wide-area sync, at least one side of a friendship SHOULD run a long-lived sync engine (daemon or interactive REPL left open). Short-lived one-shot CLI processes alone are insufficient when peer discovery exceeds a few seconds (typical on WAN). |

## 2. State beacon

| Rule | Requirement |
|------|-------------|
| OBS-10 | A running sync engine MUST publish `<dataDir>/.nearbytes-sync.state.json` at a fixed cadence (reference: 500 ms). The file MUST be written atomically (`*.tmp` then `rename`). |
| OBS-11 | Beacon payload MUST be versioned JSON (`version: 1`). Readers MUST ignore beacons with unknown `version`. |
| OBS-12 | Beacon fields MUST include at minimum: `pid`, `dataDir`, `updatedAt` (ISO-8601 UTC), `snapshot` (connected peer count, inflight counters), and `peers[]` (remote profile, remote instance public key, transport label, role, connectedAt). |
| OBS-13 | Beacon SHOULD include `peerId`, `instancePublicKey`, `activeProfilePublicKey`, `events[]` (recent wire events, oldest-first), and `stats` (throughput counters). Absence of optional fields MUST be treated as empty/zero, not as error for stale readers. |
| OBS-14 | On graceful `stop()`, the sync engine MUST remove the beacon file. If the process crashes, the file MAY remain; readers MUST treat `updatedAt` older than a staleness threshold (reference: 5 s) as "beacon stale / daemon probably dead". |
| OBS-15 | `nbsync status` and `nbf whoami` MUST report lock holder pid and dataDir when probing the lock, without acquiring it. |

## 3. In-process event bus

| Rule | Requirement |
|------|-------------|
| OBS-20 | `start()` MUST own a `SyncEventBus` emitting wire-level observability events. Subscribers via `SyncHandle.onEvent()` MUST be notified without blocking the protocol (slow handlers MUST NOT stall sends/receives). |
| OBS-21 | Event kinds (closed union, reference implementation): `peer-connected`, `peer-disconnected`, `peer-connect-failed`, `block-sent`, `block-received`, `event-received`. Each event carries `at` as epoch milliseconds. |
| OBS-22 | A bounded ring buffer (reference: 256 events) MUST retain recent events for `SyncHandle.recentEvents()` and for mirroring into the state beacon. |
| OBS-23 | `peer-connected` / `peer-disconnected` MUST include `remoteProfilePublicKey`, `remoteInstancePublicKey`, `transportLabel`, and `role` (`sibling` \| `friend`). |
| OBS-24 | `block-sent`, `block-received`, and `event-received` MUST include byte counts and object identifiers sufficient for throughput aggregation (`stats`). |

## 4. Transport labels

Discovery backends MUST normalise endpoints into stable, parseable `transportLabel` strings for logs, beacons, and CLI tables.

| Rule | Requirement |
|------|-------------|
| OBS-30 | Hyperswarm associations SHOULD use `dht:<host>:<port>` once the remote endpoint is known, by unwrapping the underlying TCP or UDX socket from the Noise secret-stream wrapper. Until resolved, `dht:unknown` is permitted. |
| OBS-31 | LAN TCP associations MUST use `mdns-tcp:<host>:<port>` with an optional `-><profile-prefix>` suffix when the targeted served profile is known from the listener. |
| OBS-32 | Direct TCP associations MAY use `tcp:<host>:<port>`. |
| OBS-33 | IPv6 hosts MUST be bracketed in labels: `dht:[fe80::1]:53432`. |

## 5. Handshake failures

| Rule | Requirement |
|------|-------------|
| OBS-40 | Expected handshake failures (timeout, unsupported protocol, duplicate nonce, unauthorized profile, process loopback) MUST be classified as `SyncHandshakeError` with `retryable: boolean` and MUST NOT print stack traces to stderr by default. |
| OBS-41 | Implementations SHOULD retry retryable handshake failures with bounded backoff (reference: up to 3 attempts). |
| OBS-42 | After retries are exhausted, implementations MUST emit `peer-connect-failed` on the event bus with `reason` (machine tag, e.g. `handshake-timeout`), `attempts`, and best-effort remote identity fields. |
| OBS-43 | Reference CLI (`nbf monitor`) MUST render `peer-connect-failed` in the activity log instead of dumping an exception. `--debug` MAY enable stack traces for diagnostics. |

## 6. Reference CLI (`nbf`)

| Rule | Requirement |
|------|-------------|
| OBS-50 | `nbf peers` MUST print a snapshot table: role, route (LAN / DHT / local), age, transport endpoint, peer id, and optional wide instance public key. |
| OBS-51 | `nbf monitor` (alias `top`) MUST render a live panel when stdout is a TTY; in non-TTY environments it MUST fall back to a single `peers` snapshot. In REPL mode, monitor MUST use a sticky overlay so the command line remains usable. |
| OBS-52 | `nbf whoami` MUST print local peer id, local instance public key, active profile public key, friend count, served profile count, sync mode (LIVE / DAEMON / WRITER-ONLY), and dataDir. |
| OBS-53 | Global flags `-m` / `--monitor` and `--debug` MUST be documented in `nbf --help` and REPL `help`. |
| OBS-54 | One-shot mutating commands (`file add`, `file remove`, `profile publish`) that run without an active daemon MUST wait for at least one connected peer (when `friends` is non-empty) before exiting, with a WAN-friendly timeout (reference: 60 s when friends configured, 15 s otherwise). They MUST warn when no peer was found within the budget; local writes remain durable and sync later. |

## 7. Throughput statistics

| Rule | Requirement |
|------|-------------|
| OBS-60 | `SyncHandle.stats()` MUST expose cumulative and windowed counters for blocks/events sent and received (bytes and counts). |
| OBS-61 | `nbf monitor` SHOULD display a throughput summary row derived from `stats`. |
