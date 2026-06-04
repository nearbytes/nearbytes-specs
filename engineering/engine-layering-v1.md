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
  the reactive-volume cache, filesystem watchers, and the high-level profile /
  hub / friend / file / chat / status operations. Replay, materialization,
  persistence, and inbound-event projection are NOT engine logic — they live in
  the projection engine (`storage/projection-engine-v1.md`) and the per-protocol
  projectors in `nearbytes-files` / `nearbytes-chat`. The engine only **wires**
  protocol instances and **exposes** their APIs; it MUST NOT re-implement chat
  timeline reads, replay caches, or per-shell inbound-sync policies.
- **ENG-1.2** `nearbytes-engine` MUST NOT depend on terminal, Electron, DOM, or
  any rendering concern. It depends only on the protocol packages
  (`nearbytes-skeleton`, `nearbytes-files`, `nearbytes-chat`, `nearbytes-crypto`,
  `nearbytes-log`, `nearbytes-sync`).
- **ENG-1.3** Both shells MUST consume `nearbytes-engine` and MUST NOT
  re-implement core logic. Sync behaviour MUST be identical across shells
  because it is the same code.

## Surface (ENG-2)

- **Runtime core**: `createEngineRuntime`, `openAndWatch`, `refreshIfOpen`,
  `closeVolume`, and the `EngineRuntime` type. The runtime opens the
  `MaterializedStore` databases (`files.sqlite3`, `chat.sqlite3`), builds the
  files engine + chat service over them, and wires the log router so both
  receive pushes. Inbound-sync refresh is NOT engine code; new events flow
  through the log router into the projectors (`projection-engine-v1.md` §3, §7).
- **Operations**: the `NearbytesEngine` class with profile/hub/friend/file/chat
  methods and a change stream `on(listener)` emitting `status` / `volume` /
  `chat` events. Its file/chat methods MUST delegate to (expose) the files/chat
  APIs, not re-derive timelines or replay. The CLI renders these as text; the app
  pushes them to the renderer over IPC.

## Sync and projection (ENG-5)

- **ENG-5.1** There MUST be exactly one inbound-sync policy across shells: the log
  router pushes newly persisted events to the files/chat projectors. Shells MUST
  NOT add a second policy (no app-only `refreshActive` full reread, no blunt
  reload-all of open volumes).
- **ENG-5.2** On boot the engine MUST bucket new events by channel and ingest them
  volume-by-volume (`projection-engine-v1.md` §7), not one event at a time across
  volumes.

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
