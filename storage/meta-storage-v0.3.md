# Nearbytes Meta-Storage v0.3

Status: draft normative specification.

This document defines how Nearbytes stores, discovers, retains, and prunes encrypted events and blocks across configured storage roots after the move to opaque event payloads.

## 1. Scope

This specification defines:

1. source-centric block discovery;
2. volume-centric event and block placement;
3. storage-level dependency tracking using only visible event metadata;
4. retention and pruning policy.

This specification does not define:

1. cryptographic primitives;
2. decrypted application semantics;
3. secret handling.

## 2. Design Rule

All routing, retention, and pruning decisions MUST be made from:

1. configured source directories;
2. configured volume public keys;
3. canonical storage paths;
4. the visible outer event envelope, especially the cleartext `blockRefs` list.

The storage layer MUST NOT require decryption of event payloads to decide:

1. where events are stored;
2. which blocks are copied;
3. which blocks are retained;
4. which blocks may be pruned.

## 3. Visible Dependency Rule

An event's visible `blockRefs` envelope field contains application-level
dependency hashes. A dependency hash can name either an event or a ciphertext
block, according to the application protocol that owns the encrypted payload.

Rules:

1. retention decisions may use only visible `blockRefs` and canonical storage paths;
2. the storage layer MUST NOT inspect decrypted event commands to infer block ownership;
3. the storage layer MUST NOT assume every `blockRefs` entry is a block;
4. if a dependency hash is known to be a block, it contributes block retention obligations;
5. if a dependency hash is known to be an event, event retention follows the channel policy;
6. if an event references zero known blocks, it contributes no block retention obligations by itself.

## 4. Source and Volume Model

The source-centric and volume-centric model from v2 remains, with one change:

1. block attribution is now derived from visible outer `blockRefs` entries that are known to name blocks, rather than cleartext command fields.

Canonical source layout remains:

```text
<source>/
  blocks/<block-hash>.bin
  channels/<volumeId>/<eventHash>.bin
```

## 5. Durable Volume Guarantee

For every configured volume, at least one enabled writable destination MUST durably retain:

1. that volume's event files;
2. every ciphertext block named by those events' visible `blockRefs` lists when the application protocol or local storage identifies the hash as a block dependency.

## 6. Sync and Backfill Rule

Any sync, mirror, or backfill process operating at the storage layer MUST:

1. discover event files for the volume;
2. read visible `blockRefs` from those events;
3. fetch or preserve referenced ciphertext blocks when a ref is known or expected to be a block dependency;
4. fetch or preserve referenced events according to channel policy when a ref is known or expected to be an event dependency.

It MUST NOT depend on decrypting file/chat/identity/app payloads.
