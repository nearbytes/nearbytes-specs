# Nearbytes Event `blockRefs` — v0.1

Status: draft normative.

This spec defines the meaning of the visible `blockRefs` field on a signed
Nearbytes event envelope. It does not change the envelope schema: `blockRefs`
remains an ordered list of lower-case SHA-256 hashes with no cleartext type
tags.

## 1. Scope

`blockRefs` are cleartext application-level dependency references. They answer
this question:

> Which already-addressed data must be available for the current event to have
> an unambiguous meaning at the application level?

The referenced data MAY be:

1. signed event payloads, named by event hash;
2. ciphertext blocks or manifest blocks, named by block hash.

This document defines the generic discipline. Each application protocol that
uses `blockRefs` MUST define its own role layout and replay rules.

## 2. Design Rules

1. **Application-level, not storage-level typing.** The envelope does not say
   whether a hash names an event or a block. Role and ordering semantics are
   defined by the application protocol that owns the encrypted payload.
2. **Visible only when needed.** A producer SHOULD put a hash in cleartext
   `blockRefs` only when sync, backup, retention, or application replay needs
   that object without decrypting unrelated state.
3. **No diagnostic spill.** Refs that only enrich debugging, UI narration, or
   non-semantic audit SHOULD be placed inside the encrypted payload instead.
4. **Direct dependencies only.** `blockRefs` MUST NOT contain transitive closure
   expansion. In particular, an operation on a directory/prefix MUST NOT list
   every descendant file block or event merely because the materializer will
   cascade over that subtree.
5. **No global replay rule.** Generic log replay MUST NOT infer ordering from
   `blockRefs`. An application protocol MAY nominate a subset of its visible
   refs as ordering parents.
6. **No global block rule.** Generic storage/sync MUST NOT assume that every
   `blockRefs` entry is a block. It MAY use application knowledge, local object
   existence, or opportunistic probing to prioritize event or block fetches.
7. **Stable order, no duplicate semantics.** Producers SHOULD preserve the
   application-defined order of refs, but applications MUST NOT rely on duplicate
   occurrences. Log implementations MAY deduplicate while preserving first
   occurrence order.

## 3. Cleartext Versus Ciphertext

Cleartext `blockRefs` are appropriate for:

1. event payload references that an application uses for causal replay;
2. previous event payload references that the current event directly updates and
   that the application uses for version browsing, conflict tooling, or reverse
   replay;
3. current content blocks or manifest blocks required to materialize the event;
4. previous content blocks or manifest blocks when the application uses them as
   version/conflict dependencies.

Encrypted payload fields are appropriate for:

1. typed or verbose explanations of what each clear ref means;
2. client-observed validators such as WebDAV `If-Match` when they do not drive
   replay ordering;
3. UI-only provenance, comments, or conflict-resolution hints;
4. any dependency whose visibility would leak more than the application needs
   for sync, backup, retention, or deterministic replay.

If a clear hash must be interpreted with a role, the role MUST be recoverable
from the application protocol and decrypted payload. The clear envelope SHOULD
not grow typed substructures solely for application convenience.

## 4. Missing Dependencies

An event whose envelope and signature are valid remains a valid storage object
even if some `blockRefs` are missing locally.

Application projections MAY treat missing dependencies as:

1. incomplete materialization;
2. a fetch priority;
3. a reason to delay replay of that application protocol;
4. a recoverable integrity/availability error.

They MUST NOT reinterpret the event as a different operation because a ref is
missing.

## 5. Storage And Sync Interpretation

Storage and sync layers operate on opaque objects:

1. event objects are addressed as `(channel public key, event hash)`;
2. block objects are addressed as `(block hash)`.

When a visible `blockRefs` hash is known to be a block dependency, durable
storage policy SHOULD retain that block with the event. When it is known to be
an event dependency, durable storage policy SHOULD ensure that the referenced
event is retained under the same channel policy. If the generic layer cannot
classify a hash, it MUST NOT assume that it is a block.

Sync MAY include `blockRefs` in event announcements to let receivers prioritize
fetches, but the receiver remains responsible for validating every fetched
object by its own address and type.

## 6. Application Protocol Obligations

Any application protocol that assigns semantics to `blockRefs` MUST specify:

1. which visible refs are ordering parents, if any;
2. whether ordering parents are channel-local or may cross channels;
3. which visible refs are content dependencies;
4. how missing refs affect replay;
5. the deterministic tiebreak used when more than one event is eligible under
   the parent rule;
6. why the clear refs are direct dependencies rather than transitive closure.

## 7. FILES v0.5 Summary

`application/file-events-v0.5.md` uses `blockRefs` as follows:

1. every filesystem v0.5 event on a non-empty channel carries one channel-local
   event parent: the last FILES event observed in the channel's canonical replay
   order at emit time;
2. only that event parent participates in topological replay;
3. direct predecessor event refs support reverse replay/versioning but do not
   create replay edges;
4. content block refs, including previous-content refs, do not create replay
   edges;
5. directory cascades do not expand into descendant refs.
