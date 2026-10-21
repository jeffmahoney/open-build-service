# E17 — importer, migration, parity harness, e2e

Milestone M4. Inputs: `services/importer.md`, `09`, `10`, analysis S § 1,
B § 1.5, P § 0.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E17-T01 | Importer framework: read-only tree walker, `BSConfig.pm` reader, progress table, parallelism, resumability | `cmd/obs-import`, `internal/importer` | re-run idempotent | E03 | M |
| E17-T02 | Phases 1-3: blobs (delta expand), trees, projects/packages/revisions/keys | `internal/importer/source` | imported corpus tree equals live `GetFileList` outputs | T01, E02 | L |
| E17-T03 | Phase 4-5: build trees and jobs | `internal/importer/build` | `_result` parity after import; jobs re-dispatch | T01, E04, E06, E07 | L |
| E17-T04 | Phase 6-7: published trees, registry, streams continuation | `internal/importer/publish` | `/published` parity; Rails cursor continues | T01, E08, E10, E12 | M |
| E17-T05 | Verification/parity report tool (Perl vs Go side by side) | `internal/importer/verify`, `test/parity` | report format; CI job | T02-T04 | M |
| E17-T06 | `obs-export` rollback tool (sources/meta back to Perl layout) | `cmd/obs-export` | round-trip test | T02 | M |
| E17-T07 | E2E suite: Rails + osc scenarios per `10` § 1; interconnect with a Perl peer; chaos scenarios | `test/e2e` | nightly green | all | L |
| E17-T08 | Scheduler dry-run decision diff harness (M1 Replay) | `internal/scheduler/dryrun`, `test/parity/sched` | diff report on corpus and on a snapshot instance | E04 | M |
| E17-T09 | Performance suite against `00` targets | `test/perf` | report | all | M |
