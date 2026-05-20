# Nearbytes Specs

This repository contains specifications for the **clean-code layer** of Nearbytes: the crypto, storage, log, and file-protocol packages. Application-layer, identity, transport, and reference specs live in the app repo (`nearbytes-app/docs/specs/`).

## Active specifications

**Storage**

- `storage/data-correctness-v0.2.md` — on-disk correctness and hash integrity rules
- `storage/meta-storage-v0.3.md` — storage root layout (blocks/, channels/, opaque event format)
- `storage/meta-storage-v2.md` — storage root discovery marker (Nearbytes.html), multi-root routing
- `storage/shared-path-storage-v0.1.md` — shared-path storage backend

**Registry**

- `registry/protocol-registry.md` — protocol identifier naming and versioning rules

## Directory hygiene

1. Only specs that are implemented (or actively being implemented) in the clean-code packages belong here.
2. Application-layer semantics (file events, chat, identity, transport) live in `nearbytes-app/docs/specs/`.
3. Keep filenames concise and family-scoped; prefer one concern per file.
