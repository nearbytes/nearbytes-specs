# Hash Evolution Policy v1

**Status:** normative
**Scope:** governs how Nearbytes evolves the cryptographic hash function used as the content-address of blocks and events.

References: `storage/log-api-v1.md` (§2.3), `storage/data-correctness-v0.2.md` (§3), `paper-nearbytes-hypercore` Appendix on parallel SHA-256.

## 1. Current state

The block content-address and the event content-address are both **SHA-256** of the canonical bytes, encoded as lower-case 64-hex. SHA-256 is FIPS 180-4 and benefits from ARMv8 SHA / Intel SHA-NI hardware acceleration on every Tier-1 client platform.

## 2. Why we do not parallelise inside one digest

Two design choices remove any pressure to switch to a tree hash *inside* a single block:

1. **The log owns the address.** The log computes the digest from the bytes it persists (`BlockStoreApi.store(data)`); only the sync streaming receiver may assert a pre-computed digest (`storeAlreadyVerified`). There is no scatter-gather across packages.
2. **Files are split into blocks below the single-thread SHA-256 budget.** The `file-events` family already supports manifest-of-blocks (`nb.content.manifest.v1`). When file sizes can exceed a single-thread hash budget (currently ${\approx}27$ Gb/s of SHA-256 on Apple Silicon), the file layer MUST split into multiple blocks. Cross-block parallelism is then a property of `nearbytes-sync` (concurrent streams) and of the per-block hash worker (one core per block).

Consequently, RFC 6962 §2.1 Merkle Tree Hash over SHA-256 is **not** required as a content-address scheme. It remains documented in `paper-nearbytes-hypercore` (Appendix on parallel SHA-256) as a fallback should a future protocol revision need to hash a *single* block that exceeds the single-thread SHA-256 budget; in that case it would be introduced under a new major version of the content-address.

## 3. Compatibility boundary

Any change to the block or event hash function is a **major version** event affecting every content-address ever issued.

Normative rules:

1. Mixed-algorithm content-addresses are PROHIBITED within one volume.
2. A hash-algorithm migration MUST publish a new namespace (e.g. `blocks_v2/<hash>.bin`) and MUST NOT overwrite `blocks/` in place.
3. Sync protocol negotiations MUST advertise the supported hash algorithm and refuse to fetch blocks whose announced algorithm is unknown.
4. The current namespace `blocks/<hash>.bin` with `<hash>` lower-case 64-hex is unambiguously SHA-256.

## 4. Implementation requirements

1. `nearbytes-crypto` MUST expose both `computeHash` (SHA-256) and `computeMerkleHash` (RFC 6962 §2.1 over SHA-256). Only `computeHash` is on the production content-address path today; `computeMerkleHash` is reserved for the future fallback above and for diagnostic benchmarks.
2. `nearbytes-log` MUST use `computeHash` internally in `BlockStoreApi.store`. Callers MUST NOT bypass `nearbytes-log` to pick a different primitive.
3. `nearbytes-sync` MUST use SHA-256 incrementally in its streaming receiver. It MUST NOT compute a different digest than the log would compute on the same bytes.

## 5. Test obligations

1. `nearbytes-log` MUST include unit tests proving that `BlockStoreApi.store(data)` returns the same hash as the byte-equivalent `nearbytes-sync` streaming receive of the same data.
2. `nearbytes-crypto` MUST include test vectors:
   - `computeHash("")` $= \texttt{e3b0c442\dots b855}$.
   - `computeMerkleHash` of any single-leaf input equals `SHA-256(0x00 || data)` per RFC 6962 §2.1.
