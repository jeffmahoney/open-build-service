# E07 — dispatcher, worker protocol, log-svc

Milestone M2. Inputs: `services/dispatcher.md`, `services/log-svc.md`,
analysis B build-path § 2.3-2.5, § 3, § 5; `proto/dispatcher.proto`,
`proto/worker.proto`, `proto/log.proto`; `05` § 3.1.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E07-T01 | Job table + state machine + `SubmitJob`/`KillJob`/`SetDispatchDetails`; job id md5 parity | `internal/dispatcher/jobs` | state transitions tested; CAS claim under contention | E03 | M |
| E07-T02 | Worker registry: `Register`, capability doc, host grouping, states, heartbeats, lease expiry (warden), orphan rules | `internal/dispatcher/workers` | B build-path § 3.6 timing rules | T01 | M |
| E07-T03 | Constraints: three-source merge, `list2struct`, `overwrite`, oracle, `getmbsize`, `nativeonly`, `CheckConstraints` feedback | `internal/dispatcher/constraints` | `--test-constraints` fixtures from Perl reproduce | — | M |
| E07-T04 | Ranking: load model (decay, finished accounting), prios/adjust scales, project smoothing, staleness, srcchange-first, needed/readytime, powerpkgs, preinstallimage-first, pass limit; publish to class subjects with hints | `internal/dispatcher/rank` | ordering equals Perl on recorded `dispatch.data` snapshots | T01 | L |
| E07-T05 | Class mapping from constraints and `ObsWorkerPool` definitions; locality hints from cache filters and zones | `internal/dispatcher/classes` | unit tests; hint hit-rate metric | T03, T04 | M |
| E07-T06 | `WorkerGateway`: claim with oracle/badhost checks, `JobSpec` assembly (bdep digests, config, preinstall image selection server-side, image digest), heartbeat stream + control, presigned URLs, `UploadResult` handshake, badhost/time-skew, signing decision, events | `internal/dispatcher/gateway` | protocol tests with a fake agent; B build-path § 5.4 recovery scenarios | T01-T05, E06-T03 | L |
| E07-T07 | Badhost table, `nobadhost`, quarantine expiry, `checkconstraints` auto-fail | `internal/dispatcher/badhost` | rules tested | T06 | S |
| E07-T08 | API RPCs: `WorkerStatus` (legacy document incl. daemon states from operator status), `WorkerInfo`, `JobStatus`, `ListJobs`, dispatchprios, `SendSysrq` | `internal/dispatcher/api` | `/build/_workerstatus`, `/worker/<id>` parity | T02 | M |
| E07-T09 | log-svc: append stream, tail cache, spill, `Read`/`Follow`/`Size`, promote, retention | `cmd/log-svc`, `internal/log` | live log e2e in Rails UI; `_log` parameter parity | E03 | M |
| E07-T10 | Load/perf: 5 000 simulated workers, 100 k ready jobs; claim p99 < 5 s | `test/perf/dispatcher` | numbers in report | T06 | M |
