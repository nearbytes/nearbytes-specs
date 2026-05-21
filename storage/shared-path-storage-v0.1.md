# Shared-path storage backend (historical)

**Status:** superseded for clean-code packages  
**Superseded by:** [`log-api-v1.md`](log-api-v1.md)

The `StorageBackend` VFS interface described in earlier drafts is **not** used by `nearbytes-log`, `nearbytes-skeleton`, or `nearbytes-files` as of log-api v1. Persistence variability belongs in `Log` implementations (`createFilesystemLog`, `createInMemoryLog`, …).

`nearbytes-app` may retain multi-root filesystem routing until it migrates to app-specific `Log` implementations.
