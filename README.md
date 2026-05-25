# Nearbytes Specs

Normative specifications for the clean-code packages (`nearbytes-crypto`, `nearbytes-log`, `nearbytes-files`, `nearbytes-skeleton`).

Application-layer specs remain in `nearbytes-app/docs/specs/`.

## Requirements

- [`requirements/README.md`](requirements/README.md) — portability and sync discovery rules for clean packages

## Engineering

- [`engineering/typescript-conventions-v1.md`](engineering/typescript-conventions-v1.md) — TypeScript style and package rules (normative)
- [`engineering/hash-evolution-v1.md`](engineering/hash-evolution-v1.md) — policy governing the SHA-256 content-address and conditions for evolving it (normative)

## Storage and log

- [`storage/log-api-v1.md`](storage/log-api-v1.md) — `Log` / `EventLogApi` / `BlockStoreApi` (normative)
- [`storage/data-correctness-v0.2.md`](storage/data-correctness-v0.2.md) — hash integrity on disk
- [`storage/meta-storage-v0.3.md`](storage/meta-storage-v0.3.md) — root layout
- [`storage/meta-storage-v2.md`](storage/meta-storage-v2.md) — discovery marker, multi-root (app layer)
- [`storage/shared-path-storage-v0.1.md`](storage/shared-path-storage-v0.1.md) — historical; superseded by log-api-v1 for clean packages

## Registry

- [`registry/protocol-registry.md`](registry/protocol-registry.md)
