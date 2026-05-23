# Portability requirements v1

Status: normative for clean-code repositories.

## 1. Scope

Applies to `nearbytes-crypto`, `nearbytes-log`, `nearbytes-sync`, `nearbytes-files` (library surface), and shared types. Does not apply to `nearbytes-app`.

## 2. Runtime targets

Implementations MUST support:

1. **Browser** — Vite/renderer and Web Crypto where applicable.
2. **Node.js** — CLI and filesystem-backed storage (≥ 18).
3. **Pear / embedded** — Node-compatible modules without Electron-only APIs.

## 3. Module boundaries

| Rule | Requirement |
|------|-------------|
| PORT-01 | Core protocol logic MUST NOT import `node:fs`, `node:path`, `node:net`, or `node:dgram`. |
| PORT-02 | Filesystem and LAN discovery MUST live in explicit entry points (`nearbytes-log` root, `nearbytes-sync/node`). |
| PORT-03 | Browser entry points (`*/browser` exports) MUST be free of Node built-ins. |
| PORT-04 | Use `Uint8Array`, `TextEncoder`, and `TextDecoder` for binary/text I/O in shared code. |
| PORT-05 | Optional Node APIs (e.g. `Buffer`) MAY be used only behind runtime checks in transport adapters, not in core codecs. |

## 4. Dependencies

| Rule | Requirement |
|------|-------------|
| PORT-10 | Shared dependencies MUST be browser-compatible (e.g. `@noble/curves`). |
| PORT-11 | Node-only packages (`hyperswarm`, `bonjour-service`) MUST NOT be imported from browser export paths. |

## 5. Storage

| Rule | Requirement |
|------|-------------|
| PORT-20 | `nearbytes-log` MUST expose `createLogFromIo(LogIo)` for non-filesystem backends. |
| PORT-21 | `createFilesystemLog` remains a Node convenience wrapper over `LogIo`. |
