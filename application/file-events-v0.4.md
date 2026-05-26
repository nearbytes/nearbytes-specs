# Nearbytes File Events — v0.4

Status: draft normative.

This spec defines the **file** event family carried inside Nearbytes log entries. It supersedes v0.3. Older event types are retired and MUST NOT be emitted; readers MAY treat them as unknown verbs (warning + skip; see §7).

It defines:

1. the four file event types and their canonical payload shape (§2),
2. how the file system is **materialized** from a stream of those events (§3),
3. the conflict-resolution rules used by the materializer (§4),
4. the path grammar and normalization (§5),
5. CLI / FileService obligations that follow from this spec (§6),
6. forward-compat rules for unknown verbs (§7).

It does **not** define how events are signed, hashed, or transported — those rules live in the cryptographic-events spec, the log-API spec, and the sync-protocol spec respectively.

## 1. Design principles

1. **Events are pure syntax.** Each event is one atomic action. Events MUST NOT imply other events on the log. There is no "delete-then-recreate" pair, no "rename = delete+create", no implicit MKDIR.
2. **All semantics live in the materializer.** Cascading deletion of a directory's contents, renaming every file under a renamed directory, the appearance of implicit ancestor directories — none of these are extra events. They are computed by replay.
3. **Names are paths, not filenames.** A volume is one filesystem-shaped namespace. A name is a slash-separated path with no leading or trailing `/`, no `.` or `..` segments, and no empty segments.
4. **A name is either a file XOR a directory at any given point in replay.** The replay-order winner of a name conflict establishes the kind; later conflicting events become **shadow events** (recorded for audit, not applied to live state).
5. **Implicit directories are first-class.** A directory exists if it is `MKDIR`'d (explicit) **or** if any live file is nested under it (implicit). Implicit directories disappear when their last file leaves; explicit ones survive empty.
6. **CRDT-trivial.** Two replicas observing the same set of events in the same total order MUST produce the same materialized state. The total order is defined externally (timestamp → log position → primary path → event hash) and is the same one used by the timeline.

## 2. Event types

Each event payload is a `nb.*`-style record carried inside a signed log entry. The four file verbs are:

| `type`        | Required fields                       | Meaning                                    |
| ------------- | ------------------------------------- | ------------------------------------------ |
| `CREATE_FILE` | `path`, `blobHash`, `encryptedKey`, `contentType`, `size`, `mimeType?`, `createdAt` | Place file content at `path`. Overwrites if a file is already at `path`. |
| `MKDIR`       | `path`, `createdAt`                   | Mark `path` as an explicit directory.       |
| `DELETE`      | `path`, `deletedAt`                   | Remove whatever lives at `path`.            |
| `RENAME`      | `fromPath`, `toPath`, `renamedAt`     | Move whatever lives at `fromPath` to `toPath`. |

Field rules:

1. `path`, `fromPath`, `toPath` MUST satisfy the path grammar of §5.
2. `blobHash` is the cleartext SHA-256 of the file content (see meta-storage spec).
3. `encryptedKey` is the wrapped per-file key.
4. `*At` are millisecond UTC timestamps; see §2.5 for the clock model (wall-clock-sourced, ties broken by §6 of the log-api spec).
5. `opts` is reserved: future versions MAY add an optional `opts: { ... }` map to any event for non-breaking extensions. v0.4 readers MUST tolerate an unknown `opts` map (ignore unknown keys, fail closed only on unknown verb).

Removed in v0.4 (do not emit, do not honor):

- `DELETE_FILE` — replaced by `DELETE`.
- `RENAME_FILE` — replaced by `RENAME`.
- `CREATE_FILE.filename` — replaced by `CREATE_FILE.path`.

### 2.5 Clock model

Timestamps in v0.4 events are scalars on the wire but wall-clock-sourced at production. The split matters and is normative:

1. **Encoding (universal).** `createdAt`, `deletedAt`, `renamedAt` MUST be integer milliseconds since the Unix epoch in UTC (i.e. the value of an unmodified `Date.now()` on a JavaScript host, or `time.time_ns() // 1_000_000` on Python). They MUST NOT carry a timezone, a UTC offset, an ISO-8601 string, or any other form; they are pure numbers and any two implementations agree on what a given integer means.
2. **Source (local, uncorrected).** The producer MUST mint the timestamp from its local wall clock at the moment of authoring the event. Implementations MUST NOT silently rewrite, clamp, "fix up", or NTP-correct timestamps before signing; the signed value is exactly what the producer saw.
3. **Total order is wall-clock-based.** The materializer total order is `(timestamp, log-position, eventHash)` per §6 of the log-api spec. This is **not** a causal order. Two producers whose clocks disagree by Δ can produce events whose `(timestamp, eventHash)` order does not match true wall-clock causality across machines, even when both producers observe the same local sequence of operations.
4. **Determinism survives skew.** Any clock skew, however large, is absorbed by the deterministic total order: every replica that has observed the same set of events reconstructs the same `MaterializedFileSystem`. Skew can therefore reorder events relative to a human observer's intent but it cannot break convergence, the file-XOR-dir invariant, or the shadow-event audit trail. Cross-device disagreements caused by skew surface as ordinary §4 shadows on the timeline, not as undefined behavior.
5. **No clock attestation.** The protocol does not include a signed clock anchor, a Lamport counter, or a Hybrid Logical Clock. Receivers MUST NOT reject events for "implausible" timestamps and MUST NOT use timestamp comparisons for liveness or freshness decisions; the timestamp is a sort key, nothing more.
6. **Producer obligations.** Implementations SHOULD use a monotonic-friendly source where possible (e.g. on JavaScript, `Date.now()` is sufficient; on POSIX, `clock_gettime(CLOCK_REALTIME)` is sufficient — `CLOCK_MONOTONIC` MUST NOT be used because its epoch is process-local). A single producer MUST ensure timestamps within one process are non-decreasing in the order events are appended to its local log (it is acceptable to bump a freshly-minted `Date.now()` up to `lastEmittedAt + 1` to maintain this).
7. **Future causal mode.** A future major version of the envelope MAY add a cleartext `parents: Hash[]` field; the file-events spec MAY then define a v0.5+ total order as "topological sort of the parent DAG, ties broken by event hash". The materializer and §4 conflict rules are written to be independent of how the total order is computed, so adopting causal order is a sort-key change rather than a semantic change. See `engineering/hash-evolution-v1.md` for the major-version migration policy.

## 3. Materializer

The materializer consumes events in canonical replay order and produces a snapshot:

```
MaterializedFileSystem {
  files:        Map<path, FileMetadata>     // live files
  directories:  Map<path, DirectoryMetadata> // live dirs (explicit ∪ implicit)
  fileOrigins:  Map<path, eventTiebreak>     // CREATE_FILE that produced live content at `path`
  shadows:      [{ tiebreak, reject: ... }]  // events rejected by §4, in replay order
}
```

Each `DirectoryMetadata` carries an `explicit: boolean` flag and a `fileCount` (number of live files anywhere in its subtree).

### 3.1 `CREATE_FILE(path)`

1. If any live file or live directory occupies `path`, see §4.1 / §4.2.
2. Else: place the file at `path`. If a file already lived at `path` (overwrite), keep the directory bookkeeping unchanged; replace `fileOrigins[path]` with the new event tiebreak.
3. For every strict ancestor `a` of `path`: if `a` is not a live directory, create an **implicit** directory entry for `a`. Increment `a.fileCount` by 1 (only when the file is newly added, not on overwrite).

### 3.2 `MKDIR(path)`

1. If a live file occupies `path`, record a shadow `{ kind: 'file-where-dir', target: path }` and stop.
2. If `path` is already a live directory, mark it `explicit = true` (idempotent upgrade from implicit) and stop.
3. Otherwise create an explicit directory at `path`. Ensure all strict ancestors exist as implicit directories (do not change their `fileCount`).
4. If any strict ancestor of `path` is a live file, record `{ kind: 'ancestor-is-file', ancestor }` and stop.

### 3.3 `DELETE(path)`

1. If a live file is at `path`: remove it; for every strict ancestor `a` of `path`, decrement `a.fileCount` by 1, and if `a` is not explicit and `a.fileCount == 0`, drop `a`. Drop `fileOrigins[path]`. Stop.
2. If a live directory is at `path`: cascade — remove every live file `f` whose path equals `path` or starts with `path + '/'`, then remove every live directory whose path equals `path` or starts with `path + '/'`. After that, decrement strict-ancestor `fileCount`s by the number of files removed and prune now-empty implicit ancestors.
3. If `path` does not exist, record `{ kind: 'source-missing', source: path }` and stop. (No-op deletes are not silently ignored, but they are not errors either — the shadow exists for audit.)

### 3.4 `RENAME(fromPath, toPath)`

1. If `fromPath` is not a live file and not a live directory, record `{ kind: 'source-missing', source: fromPath }` and stop.
2. If `fromPath == toPath`, no-op.
3. Determine `fromKind` (`'file'` | `'dir'`) and inspect `toPath`:
   1. If `toPath` does not exist: proceed.
   2. If `toPath` is occupied by the **same kind** as `fromKind`:
      - file→file: overwrite (drop the old target file and `fileOrigins[toPath]`).
      - dir→dir: only if the target dir is empty (`fileCount == 0` and no descendants); otherwise record `{ kind: 'target-non-empty', target: toPath }` and stop. (Mirrors POSIX `rename(2)` for directories.)
   3. If `toPath` is occupied by the **opposite kind**, record `{ kind: 'target-kind-mismatch', target: toPath, expected: fromKind }` and stop.
4. Move:
   - file: relocate the file metadata to `toPath`; carry `fileOrigins[fromPath]` over to `toPath`. Decrement `fromPath`'s ancestors and increment `toPath`'s ancestors (creating implicit directories as needed).
   - dir: rewrite every live file path and every live directory path under `fromPath` to start with `toPath`. The `explicit` flag of the moved directory is preserved. `fileOrigins` are rewritten in lockstep. Bookkeep ancestors as for the file case, but with the count = number of files moved.

### 3.5 Idempotence

Replaying the same total-ordered event sequence twice yields the same `MaterializedFileSystem` (including the `shadows` list, in the same order). Replicas that have observed the same set of events in any topological extension of the canonical total order produce the same snapshot.

## 4. Conflict resolution

A name occupied by a file is not a directory and vice versa; the **first event in canonical replay order to claim a name wins**.

Recognized shadow kinds:

1. `file-where-dir` — `MKDIR(path)` arrived after a file already lived at `path`.
2. `dir-where-file` — `CREATE_FILE(path)` arrived after a directory already lived at `path`. (The file is rejected; the dir stays.)
3. `ancestor-is-file` — `MKDIR(a/b)` or `CREATE_FILE(a/b/c)` arrived while `a` is a live file.
4. `target-non-empty` — `RENAME(_, toPath)` where `toPath` is a non-empty directory.
5. `target-kind-mismatch` — `RENAME` between file and directory kinds.
6. `source-missing` — `DELETE(path)` or `RENAME(path, _)` where `path` doesn't exist.

Shadow events are not applied to `files` / `directories`, but are recorded on the timeline and SHOULD be surfaced to the user (the reference CLI marks them with `⚠`).

## 5. Path grammar

A volume path is a string matching the grammar:

```
path        ::= "" | segment ("/" segment)*
segment     ::= ?one or more of: any UTF-8 codepoint EXCEPT "/", NUL, "." (when alone), ".." (when alone)?
```

Rules:

1. `""` is the volume root. It MUST NOT appear as the `path` of `CREATE_FILE`, `MKDIR`, `DELETE`, `RENAME.fromPath` or `RENAME.toPath`.
2. Paths MUST NOT contain `.` or `..` segments, leading `/`, trailing `/`, repeated `/`, or NUL bytes.
3. Path comparison is bytewise on the canonical (NFC, no normalization performed) form of the path.
4. Implementations SHOULD reject malformed paths at the producer (CLI / FileService); receivers MUST treat malformed paths as a fatal log error and refuse to materialize that event (this is **not** a shadow — the log is corrupt).

## 6. CLI / FileService obligations

Producers (the FileService and the reference CLI) MUST:

1. Validate paths against §5 before constructing an event.
2. Emit exactly one event per user action — never synthesize "implied" `MKDIR` / `DELETE` events.
3. Surface materializer shadows to the user. The reference CLI uses `⚠` on the timeline and refuses to silently swallow conflicts.
4. Preserve `fileOrigins` lookups when exposing per-file metadata (encrypted key, mimeType, etc.). The materializer is the single source of truth for "which CREATE_FILE produced the bytes currently at path P".

Consumers (any reader that materializes a log) MUST follow §3 / §4 exactly. Two readers that disagree on the snapshot for the same total order are non-conformant.

## 7. Forward compatibility

1. A reader MUST fail closed on a structurally invalid event (bad path, missing required field).
2. A reader MUST tolerate unknown verbs by emitting a warning *once* per unknown verb name and skipping the event for materialization purposes (it remains in the timeline, marked as `unknown`).
3. v0.5+ MAY add new verbs and MAY introduce an optional `opts` map. Adding required fields or changing the meaning of existing verbs is a breaking change and increments the major version.

## 8. Test vectors

Conforming implementations SHOULD pass the following replay scenarios; the reference test fixtures live in `nearbytes-files/test/materializer/*`.

1. *Nested put + implicit dirs*: `CREATE_FILE notes/2026/file1.txt` ⇒ live `notes/`, `notes/2026/`, `notes/2026/file1.txt`; `notes/`, `notes/2026/` implicit.
2. *Implicit-to-explicit promotion*: `CREATE_FILE foo/bar.txt`, `MKDIR foo` ⇒ `foo/` is now explicit.
3. *Cascading delete*: `CREATE_FILE notes/2026/a.txt`, `CREATE_FILE notes/2025/b.txt`, `DELETE notes` ⇒ both files removed; `notes/` and its descendants gone.
4. *Cascading rename*: `CREATE_FILE notes/2026/a.txt`, `RENAME notes/2026 archived` ⇒ `archived/a.txt` live; `fileOrigins['archived/a.txt']` equals the original `CREATE_FILE`'s tiebreak.
5. *File-where-dir*: `CREATE_FILE foo`, `MKDIR foo` ⇒ shadow `file-where-dir`; `foo` stays a file.
6. *Dir-where-file*: `MKDIR bar`, `CREATE_FILE bar` ⇒ shadow `dir-where-file`; `bar` stays a directory.
7. *Ancestor-is-file*: `CREATE_FILE foo`, `CREATE_FILE foo/inner.txt` ⇒ shadow `ancestor-is-file`; `foo/inner.txt` not live.
8. *Empty-explicit-dir survives delete cascade*: `MKDIR empty`, `DELETE empty` ⇒ explicit dir is removed.
9. *Empty-explicit-dir survives content drain*: `MKDIR keep`, `CREATE_FILE keep/x`, `DELETE keep/x` ⇒ `keep/` survives empty.

## 9. Relationship to the registry

This spec replaces `application/file-events-v0.3.md` (which was reserved but never normative). The registry entry under §4 of the protocol registry MUST be updated to point at v0.4.

## 10. Cross-references

- Cryptographic event envelope and signing: `engineering/hash-evolution-v1.md`, the cryptographic events module (`nearbytes-crypto`).
- Log-API total order and rule numbering: `storage/log-api-v1.md`.
- Sync rules for transporting these events as blocks: `requirements/sync-protocol-v1.md` (SYNC-14, SYNC-37 in particular cover idempotent block delivery, which is orthogonal but relied upon for safe concurrent materialization).
- Storage of CREATE_FILE block hashes for GC: `storage/meta-storage-v2.md`.
