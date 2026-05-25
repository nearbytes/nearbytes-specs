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
| DISC-10 | Friend carriage MUST join $\topic(\mathsf{profile}(f))$ per configured friend $f$. |
| DISC-11 | Topic derivation MUST be `H("nearbytes:sync:v1" \|\| canonical(subject))`. |

## 3. mDNS / DNS-SD

| Rule | Requirement |
|------|-------------|
| DISC-20 | LAN discovery MUST publish `_nearbytes._udp` with `alpn=nearbytes-sync-v1` and `syncPort`. |
| DISC-21 | Implementations MAY emit UDP multicast fallback to `239.255.40.41:40441`. |
| DISC-22 | After LAN discovery, peers connect via TCP and speak the same framed sync protocol as Hyperswarm duplex streams. |
| DISC-23 | LAN TXT MUST include `prof`, the lower-case hex profile public key of the advertiser (130 hex chars). |
| DISC-24 | LAN discovery MUST NOT open sync to advertisers whose `prof` is not in `config.friends` (and MUST ignore the local `prof`). |
| DISC-25 | See `sync-protocol-v1.md` for association, handshake, and anti-entropy rules. |

## 4. Boot integration

| Rule | Requirement |
|------|-------------|
| DISC-30 | `nearbytes-skeleton` MUST call `start(log, config.friends)` on boot. |
| DISC-31 | Each `start` MUST append a marker line via `log.sync.appendMarker`. |
| DISC-32 | `NearbytesConfig.friends` MUST always be present (possibly empty). |
