# Sync protocol requirements v1 (friend carriage)

Status: normative for `nearbytes-sync` Impl. v0 friend carriage.

References: paper §friend-v0, §handshake, `sync-discovery-v1.md`.

## 1. Transport association

| Rule | Requirement |
|------|-------------|
| SYNC-01 | Each association MUST serialize framed writes and message handling so frames are never interleaved on one duplex. |
| SYNC-02 | Friend carriage MUST perform a `hello` exchange before any `delta`, `subscribe`, `have`, `want`, or `data` message. |
| SYNC-03 | `hello.senderProfile` MUST be the lower-case hex profile public key of the sender. |
| SYNC-04 | The receiver MUST close the association if `senderProfile` is not a configured friend (friend carriage) or policy denies the subject. |
| SYNC-05 | The receiver MUST reject a duplicate `sessionNonce` on the same association. |
| SYNC-06 | At most one active sync association per remote friend profile public key; a new association for the same friend MUST replace the previous one. |
| SYNC-07 | On a friend association, `subject` MUST be `profile(remoteFriendPublicKey)` (the remote party's profile subject). |

## 2. Anti-entropy

| Rule | Requirement |
|------|-------------|
| SYNC-10 | After local `acceptData`, objects enter the reception journal; each append MUST push `have` to all open friend associations (no timer-driven polling). |
| SYNC-11 | `want` is receiver-driven; senders MUST NOT send `data` without a prior `want`. |
| SYNC-12 | When both blocks and events are missing from a `have`, the receiver MUST issue `want` for blocks before `want` for events (blocks-first ordering). |
| SYNC-13 | `have` for events SHOULD include `blockRefs` when known so receivers can prioritize block fetches. |

## 3. Boot

| Rule | Requirement |
|------|-------------|
| SYNC-20 | `patchLogForReactiveHave` MUST apply per `Log` instance (not process-global). |
