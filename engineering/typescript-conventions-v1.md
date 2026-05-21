# TypeScript engineering conventions (normative)

**Status:** normative for all packages under `nearbytes/nearbytes-*` except `nearbytes-app`.  
**Version:** 1.0  
**Applies to:** `nearbytes-crypto`, `nearbytes-log`, `nearbytes-files`, `nearbytes-skeleton`.

---

## 1. Purpose

These rules define how clean-code packages are written so they stay minimal, typed, and composable without enterprise OOP ceremony.

---

## 2. Language and module system

| Rule | Requirement |
|------|-------------|
| TS-01 | TypeScript `strict: true` in every package `tsconfig.json`. |
| TS-02 | ESM only: `"type": "module"`, `.js` extensions in relative imports. |
| TS-03 | `moduleResolution: "bundler"` (or equivalent) with `target` ≥ ES2020. |
| TS-04 | No `any` in published APIs. Use `unknown` + narrowing at boundaries. |
| TS-05 | Prefer `readonly` on interface fields and immutable data shapes. |

---

## 3. Style: functional, not classical

| Rule | Requirement |
|------|-------------|
| FN-01 | **No `class` for domain or protocol logic.** Use interfaces + plain functions + factories. |
| FN-02 | Exception: `Error` subclasses in `nearbytes-crypto` typed error hierarchy only. |
| FN-03 | **No design-pattern naming** (Factory, Manager, Builder, Strategy) in public APIs. |
| FN-04 | **No inheritance** beyond the error types in FN-02. |
| FN-05 | Prefer small pure functions; side effects live at the edges (I/O, factories). |
| FN-06 | Dependency injection = pass interfaces as arguments (`createFileService({ log, crypto })`), not constructors on classes. |

---

## 4. Documentation

| Rule | Requirement |
|------|-------------|
| DOC-01 | Public exports MUST have TSDoc (`/** … */`) describing purpose, parameters, throws, and invariants when non-obvious. |
| DOC-02 | **No line comments** (`//` or `/* */`) except inside TSDoc or to disable a linter rule with justification. |
| DOC-03 | README per package: install, minimal example, module map (file names, one line each). |

---

## 5. File and package layout

| Rule | Requirement |
|------|-------------|
| LAY-01 | One primary concern per file; target **&lt; 200 lines**; split before 300. |
| LAY-02 | `src/index.ts` is a thin barrel: re-exports only, no logic. |
| LAY-03 | Implementation details under `src/internal/` MUST NOT be exported from the package root. |
| LAY-04 | Environment-specific code (Node `fs`) lives under `src/impl/` or `src/internal/`, not in protocol modules. |
| LAY-05 | Package dependency direction (acyclic): `crypto` ← `log` ← `files` / `skeleton`; `storage` is path-only and must not depend on `log`. |

---

## 6. Interfaces and APIs

| Rule | Requirement |
|------|-------------|
| API-01 | Public persistence contract is **`Log`** (`EventLogApi` + `BlockStoreApi`) in `nearbytes-log`, not a generic file VFS. |
| API-02 | Protocol codecs (`serializeEventEnvelope`, `validateEventBytes`, …) are shared and implementation-agnostic. |
| API-03 | Factories name the mechanism: `createFilesystemLog`, `createInMemoryLog`, `createFilesystemSkeleton`. |
| API-04 | Avoid optional parameters when two call sites need different behaviour; use separate functions or options objects. |
| API-05 | Use `type` aliases for unions; use `interface` for object contracts that may be implemented by plain objects. |

---

## 7. Errors and async

| Rule | Requirement |
|------|-------------|
| ERR-01 | Domain failures use typed errors from `nearbytes-crypto` (`StorageError`, `CryptoError`, …). |
| ERR-02 | `async` functions that perform I/O return `Promise<…>`; do not mix callback APIs. |
| ERR-03 | Corrupt on-disk protocol data: validate, delete offending blob, throw `StorageError` (see log spec). |

---

## 8. Naming

| Rule | Requirement |
|------|-------------|
| NAM-01 | Functions: verb phrase (`createFilesystemLog`, `validateEventBytes`). |
| NAM-02 | Types/interfaces: noun phrase (`Log`, `EventLogApi`, `IntegrityValidationResult`). |
| NAM-03 | No Hungarian notation; no `I` prefix on interfaces. |
| NAM-04 | File names: camelCase for modules (`eventApi.ts`, `fsIo.ts`). |

---

## 9. Testing (when present)

| Rule | Requirement |
|------|-------------|
| TST-01 | Prefer `createInMemoryLog()` for unit tests; filesystem tests only when layout matters. |
| TST-02 | No test-only exports from package roots; test helpers live beside tests or under `internal`. |

---

## 10. Compliance

A package is **conformant** when a reviewer can verify:

1. `tsc` / `yarn build` passes with no `strict` escapes.  
2. No `class` outside allowed error types.  
3. Root export surface matches README and this document.  
4. No generic path→bytes `StorageBackend` VFS in `log`, `skeleton`, or `files`; persistence is `nearbytes-log` only. Path defaults live in app config or skeleton `config.ts`.

Non-conformant code MUST be fixed or removed; do not add compatibility shims in clean packages for `nearbytes-app` (app migrates separately).
