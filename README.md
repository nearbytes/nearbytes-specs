# Nearbytes Specs

Normative specifications for the clean-code packages (`nearbytes-crypto`, `nearbytes-log`, `nearbytes-sync`, `nearbytes-skeleton`, `nearbytes-files`, `nearbytes-chat`, `nearbytes-cli`).

Application-layer specs of the `nearbytes-app` UI live in `nearbytes-app/docs/specs/`.

## Requirements

- [`requirements/README.md`](requirements/README.md) — index of normative requirement documents
- [`requirements/portability-v1.md`](requirements/portability-v1.md) — browser / Node / Pear runtime portability
- [`requirements/sync-discovery-v1.md`](requirements/sync-discovery-v1.md) — friend & sibling carriage, dataDir-anchored node identity, singleton-sync/plural-writers split (DISC-26, DISC-27.1–27.4)
- [`requirements/sync-protocol-v1.md`](requirements/sync-protocol-v1.md) — `nearbytes.sync.v1` framed message protocol
- [`requirements/sync-observability-v1.md`](requirements/sync-observability-v1.md) — state beacon, event bus, transport labels, CLI monitor, operational modes
- [`requirements/benchmark-methodology-v1.md`](requirements/benchmark-methodology-v1.md) — methodology for `nearbytes-benchmarks`

## Engineering

- [`engineering/typescript-conventions-v1.md`](engineering/typescript-conventions-v1.md) — TypeScript style and package rules (normative)
- [`engineering/hash-evolution-v1.md`](engineering/hash-evolution-v1.md) — policy governing the SHA-256 content-address and conditions for evolving it (normative)

## Storage and log

- [`storage/log-api-v1.md`](storage/log-api-v1.md) — `Log` / `EventLogApi` / `BlockStoreApi` (normative)
- [`storage/data-correctness-v0.2.md`](storage/data-correctness-v0.2.md) — hash integrity on disk
- [`storage/meta-storage-v0.3.md`](storage/meta-storage-v0.3.md) — root layout
- [`storage/meta-storage-v2.md`](storage/meta-storage-v2.md) — discovery marker, multi-root (app layer)
- [`storage/shared-path-storage-v0.1.md`](storage/shared-path-storage-v0.1.md) — historical; superseded by log-api-v1 for clean packages

## Application

- [`application/blockrefs-v0.1.md`](application/blockrefs-v0.1.md) — cleartext event `blockRefs` as application-level dependency references
- [`application/chat-v1.md`](application/chat-v1.md) — hub-scoped chat app records (`nb.chat.message.v1`)
- [`application/file-events-v0.4.md`](application/file-events-v0.4.md) — file-volume event protocol (`CREATE_FILE`, `MKDIR`, `DELETE`, `RENAME`) and materializer cascade semantics
- [`application/file-events-v0.5.md`](application/file-events-v0.5.md) — FILES semantic `blockRefs` and topological replay over observed log heads
- [`application/webdav-v1.md`](application/webdav-v1.md) — local HTTPS WebDAV projection for FILES volumes (in-memory replay, PROPFIND sizes)
- [`application/webdav-v2.md`](application/webdav-v2.md) — single-root multi-volume WebDAV mount and timeline projection
- [`application/volume-session-v1.md`](application/volume-session-v1.md) — registered hub/volume session state for `nbf`

## Registry

- [`registry/protocol-registry.md`](registry/protocol-registry.md)
