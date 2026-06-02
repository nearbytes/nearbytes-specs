# chat-v1 — hub-scoped chat records

Status: normative target for `nearbytes-chat` and `nearbytes-cli`.

## 1. Scope

This document defines the v1 chat application record used by Nearbytes hubs.
A hub is a Nearbytes log channel derived from a shared channel secret; current
CLI user-facing commands may call the same channel a volume.

Chat is scoped to the hub/volume channel where the record is stored. Profiles
are sync/social identities and are not chat containers in v1.

## 2. Protocol Identifier

The chat record protocol id is:

```text
nb.chat.message.v1
```

It is carried inside an `APP_RECORD` event payload:

```json
{
  "type": "APP_RECORD",
  "protocol": "nb.chat.message.v1",
  "authorPublicKey": "<hub-public-key>",
  "record": "{...canonical chat JSON...}",
  "publishedAt": 1710000000000
}
```

## 3. Chat Message Record

The canonical record shape is:

```json
{
  "p": "nb.chat.message.v1",
  "k": "<hub-public-key>",
  "ts": 1710000000000,
  "body": "hello",
  "sig": "<base64url-signature>"
}
```

Fields:

| Field | Requirement |
|-------|-------------|
| `p` | MUST equal `nb.chat.message.v1`. |
| `k` | MUST be the lower-case hex public key of the hub/channel keypair. |
| `ts` | MUST be a non-negative safe integer Unix epoch in milliseconds. |
| `body` | MUST be a non-empty string after trimming. Producers MUST trim leading/trailing whitespace. |
| `sig` | MUST be a base64url signature over the canonical unsigned record. |

## 4. Signing

The unsigned object is:

```json
{
  "p": "nb.chat.message.v1",
  "k": "<hub-public-key>",
  "ts": 1710000000000,
  "body": "hello"
}
```

Producers MUST canonicalize the unsigned object using canonical JSON and sign
the UTF-8 bytes with the hub/channel private key. Readers MUST verify `sig`
against `k` before treating the message as verified.

In v1 the hub key is both the channel key and the message signing key. A future
revision MAY add profile-authored messages, but that would be a new protocol
version or a separately specified envelope.

## 5. Replay

Readers build a hub chat timeline by:

1. loading the hub channel event log;
2. selecting `APP_RECORD` payloads whose `protocol` is `nb.chat.message.v1`;
3. parsing the nested `record`;
4. verifying the nested record signature;
5. sorting by `publishedAt`, then event hash as deterministic tiebreak.

Malformed or unsupported records MUST NOT mutate any file state and SHOULD be
ignored or surfaced as invalid application records by user interfaces.

## 6. CLI Requirements

The reference CLI (`nearbytes-cli`, binary `nbf`) SHOULD expose:

| Command | Behavior |
|---------|----------|
| `say <message> [-s <secret>]` | Append a chat message to the selected hub. Without `-s`, use the active volume/hub. |
| `chat [limit] [-s <secret>]` | Print recent chat messages for the selected hub. |

One-shot `say` is a mutating command and follows `sync-observability-v1.md`
OBS-54 flush behavior.

## 7. Compatibility

Readers MUST fail closed for records whose `p` is not `nb.chat.message.v1`.
There is no compatibility with future major versions unless those versions
explicitly define a downgrade path.
