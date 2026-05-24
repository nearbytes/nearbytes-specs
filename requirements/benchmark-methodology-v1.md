# Benchmark methodology v1 (friend carriage / Implementation 0)

Requirements for reproducible performance numbers reported in `paper-nearbytes-hypercore`.

## Profiles

| Profile | Env | Purpose |
|---------|-----|---------|
| `quick` | `NEARBYTES_BENCH_QUICK=1` | CI smoke (≤30s) |
| `latency-only` | `NEARBYTES_BENCH_PROFILE=latency-only` | Fast latency sweep |
| `full` | default | Legacy 6-size × 5 repeat + 12×1 MiB batch |
| `paper` | `NEARBYTES_BENCH_PROFILE=paper` | Publication: warmup discard, 10×/size, 32 MiB stream |

## Metrics

### Latency (`oneWayLatencyMs`)

- **Start:** sender `file-published` marker (`sync/activity.log`).
- **End:** receiver first `inbound-stored` block matching payload size (±512 B framing).
- **Excluded:** trials named `bench-lat-warm-*` (warmup).
- **Statistics:** per size — n, min, p50, p95, mean, 95% CI of mean (Student-t).

### Throughput (goodput)

- **Payload:** single `bench-tp-stream.bin` (paper: 32 MiB) or batch files (full/quick).
- **Receiver completion:** `bench-phase-throughput-complete.txt`, ≥95% of nominal bytes in `inbound-stored` chunks (≥1 MiB), or `listFiles` sees the stream file.
- **Progress logs:** show inbound bytes/chunks (not only `listFiles 0/1`, which lags behind sync).
- **Goodput (merge):** prefers sender `throughput-phase-start` wall time; falls back to receiver phase-4 start. `8 × bytes / (t_last − t_first)` over large inbound blocks.
- **Timeouts:** separate `NEARBYTES_BENCH_LATENCY_TIMEOUT_MS` and `NEARBYTES_BENCH_THROUGHPUT_TIMEOUT_MS` (paper defaults: 120s / 180s).
- Includes encryption, volume journal, sync framing — not wire-line iperf.

### Phases

- **Boot:** config + skeleton + sync start.
- **Friend session:** until `friend-session-attached` (handshake + mDNS/Hyperswarm).
- **Not included in latency:** discovery warmup and friend-session setup.

## Topology

Report topology explicitly in merge (`--topology`). Do not mix localhost and WAN in one table.

## Artifacts

- Per role: `benchmark-result.json`, `trial-manifest.json` (sender).
- Merged: `bench-report.json` via `merge-benchmark-results.mjs`.
- LaTeX: `render-benchmark-figures.mjs` → `paper-nearbytes-hypercore/figures/`.

## Limitations (must state in paper)

- Application-level goodput, not isolated link capacity.
- Cross-host latency assumes NTP within a few ms or uses receiver-only markers.
- Single friend pair; no churn, overlap, or hub-peer experiments in v0 harness.
