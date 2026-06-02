# Engine layering v1 (normative)

## Motivation

The `nbf` CLI historically owned both the NearBytes runtime/operation logic and
its terminal presentation. The desktop app (`nearbytes-app`) needs the *same*
logic. To avoid divergence, the shared core is extracted into a dedicated
package, **`nearbytes-engine`**, and both front-ends become thin shells.

## Layering (ENG-1)

```
nearbytes-engine   ← shared core: runtime, sync, config, file & chat operations
   ├── nearbytes-cli   ← terminal shell: arg parsing, REPL, text rendering
   └── nearbytes-app   ← UI shell: Electron main/preload + Svelte renderer
```

- **ENG-1.1** `nearbytes-engine` MUST contain all logic that is not
  argument parsing, command bookkeeping, or result rendering: skeleton boot,
  the reactive-volume cache, filesystem watchers, sync inbound-refresh, and the
  high-level profile / hub / friend / file / chat / status operations.
- **ENG-1.2** `nearbytes-engine` MUST NOT depend on terminal, Electron, DOM, or
  any rendering concern. It depends only on the protocol packages
  (`nearbytes-skeleton`, `nearbytes-files`, `nearbytes-chat`, `nearbytes-crypto`,
  `nearbytes-log`, `nearbytes-sync`).
- **ENG-1.3** Both shells MUST consume `nearbytes-engine` and MUST NOT
  re-implement core logic. Sync behaviour MUST be identical across shells
  because it is the same code.

## Surface (ENG-2)

- **Runtime core** (ported verbatim from the CLI's former `cli/context.ts`):
  `createEngineRuntime`, `openAndWatch`, `reloadVolumeFromDisk`, `refreshIfOpen`,
  `closeVolume`, `attachSyncInboundRefresh`, and the `EngineRuntime` type.
- **Operations**: the `NearbytesEngine` class with profile/hub/friend/file/chat
  methods and a change stream `on(listener)` emitting `status` / `volume` /
  `chat` events. The CLI renders these as text; the app pushes them to the
  renderer over IPC.

## Consumption (ENG-3)

- **ENG-3.1 CLI** — `cli/context.ts` builds its `Context` on
  `createEngineRuntime` and re-exports the runtime helpers under their historical
  names, so existing command modules import them unchanged. `Context extends
  EngineRuntime`.
- **ENG-3.2 App** — `src/main/service.ts` wraps `NearbytesEngine`, bridging its
  change stream onto Electron IPC and adding only the OS-open concern
  (`shell.openPath`). No domain logic lives in the app.

## Packaging (ENG-4)

`nearbytes-engine` is consumed from GitHub exactly like the other protocol
packages (`github:nearbytes/nearbytes-engine`), with a `prepare` build so a git
install produces `dist/` automatically. No sibling-folder references.
