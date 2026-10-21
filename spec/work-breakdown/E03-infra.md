# E03 — Storage, events, infrastructure libraries

Milestone M0. Inputs: `04`, `05`, `03` § 7.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E03-T01 | `pkg/store`: pgx pool, migrations framework (goose/atlas), partition helpers, idempotency table, advisory lock helpers, LISTEN/NOTIFY helper | `pkg/store` | integration tests with testcontainers | — | M |
| E03-T02 | `pkg/objstore`: S3 client (multipart, streaming sha256, presign GET/PUT, server-side copy), key layout helpers for `04` § 1.1, GC primitives (mark/sweep with grace) | `pkg/objstore` | MinIO integration tests; 5 GB streaming upload test | — | M |
| E03-T03 | `pkg/events`: JetStream streams/consumers per `05` § 1, envelope codec, subject encoding, durable consumer helpers, outbox pattern (publish after commit) | `pkg/events` | tests incl. redelivery and ordering | — | M |
| E03-T04 | Schema v1: all tables of `04` § 2.2-2.3 as migrations with indexes and partitioning | `migrations/` | applies on PostgreSQL 15/16; load test for `full_pool` inserts | T01 | M |
| E03-T05 | Dev stack: `kind` + MinIO + PostgreSQL + NATS + Rails (test mode) + stub services; `make dev-up`; Perl backend container for parity | `deploy/dev`, `test/parity` | one command boots everything | — | M |
| E03-T06 | Observability baseline: OTel setup, gRPC interceptors (tracing, metrics, auth matrix), structured logging, health/readiness endpoints | `pkg/telemetry`, `pkg/grpcx` | every service template uses it | — | S |
| E03-T07 | mTLS identity: cert-manager/SPIRE integration, identity → class mapping, interceptor | `pkg/grpcx/auth` | matrix tests | T06 | S |
| E03-T08 | Service skeleton generator (`cmd/<svc>` template with config, health, telemetry, graceful shutdown) | `tools/svcgen` | all services generated from it | T06 | S |
