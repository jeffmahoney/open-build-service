# 08 — Observability and Operations

## 1. Telemetry

- OpenTelemetry traces on every RPC, queue hop (trace context in message
  headers), and object-storage call. The legacy `x-request-id` from Rails
  becomes the trace id.
- Metrics (Prometheus via OTel): per service RED metrics; domain metrics:
  scheduler queue depths by level and shard, check duration histogram,
  `buildavg`, event lag, pool cache hit rate; dispatcher ready jobs per
  arch/class, idle workers, claim latency, badhost counts; worker build
  duration, cache hit ratio, download bytes; publisher duration per prp and
  repotype, failures by class; signer ops/s; notifier cursor lag;
  edge per-route latency and error rate.
- Structured logs (JSON) with the legacy log lines that operators grep for
  preserved as messages (`+ path`/`! path`/`- path` in publishes,
  `publish statistics:`, `servicedispatch statistics:`, scheduler check
  summaries).
- `schedulerinfo`, `workerstatus`, `/serverstatus`, `/ajaxstatus` reproduced
  for legacy tooling.

## 2. SLOs and alerts (shipped with the chart)

| SLO | Alert |
|---|---|
| edge availability 99.9 % | error rate > 1 % 5 min |
| source commit p99 < 2 s | |
| dispatch latency p99 < 5 s with idle capacity | ready jobs > 0 and idle workers > 0 for 2 min |
| scheduler event lag < 60 s | lag > 300 s |
| publish backlog age < 15 min | oldest request > 1 h |
| notifier cursor lag < 30 s | |
| shard lease age < 30 s | missing owner > 60 s |

## 3. Runbooks (to be written with the implementation; required set)

Shard rebalancing; stuck publish with `details`; worker pool drain and
image roll; signer backend outage; object storage degradation; PostgreSQL
failover; NATS stream repair; importer re-run; consistency verification
(`obs-admin check-consistency`) and repair; GC pause/resume.

## 4. Upgrades

Rolling, service by service, with proto compatibility guaranteed from M2
on; schema migrations are additive and run before the rollout; the operator
blocks rollouts whose migration is pending.

## 5. Capacity

Per `00` targets: PostgreSQL ~1 TB with partitioning; NATS streams sized
by retention; worker pools sized by queue depth; object storage unbounded.
