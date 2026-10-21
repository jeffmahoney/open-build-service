# E04 — Scheduler core

Milestone M2. Inputs: `services/scheduler.md`, analysis B scheduler (all),
`04` § 2.3, `05`, `proto/scheduler.proto`, `proto/artifact.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E04-T01 | Shard lease manager, working-set load from `GetProjPack` (shard filter) and tables, graceful release | `internal/scheduler/shard` | failover test: kill owner, new owner resumes with no full pass | E02-T16, E03 | M |
| E04-T02 | Projpacks working set: `update_projpacks` semantics, `setup_projects` (search path expansion rule, `prpdeps`/`rprpdeps`, related, `alllocked`, `expandedprojlink`, `projpacks_linked`), incremental postprocess, `bigdeps` interning | `internal/scheduler/projpacks` | derived structures equal Perl dumps on corpus | T01, E01-T10 | L |
| E04-T03 | Event intake: subscriptions, batch ordering, delayed fetch, link following, upload/import deferral, suspension | `internal/scheduler/events` | B § 2.2-2.4 behaviours in unit tests | T02 | M |
| E04-T04 | Lookat priority queues with persistence (`sched_queue`), back-off, starvation, topological low series, `extend_lookat_next` | `internal/scheduler/lookat` | B § 2.5 property tests | T01 | M |
| E04-T05 | Check pipeline: `setup` outcomes table, `wipeobsolete`, `preparepool` (pool from artifact indexes, `preparehashes`, `notready` value 2 rule), `expandandsort` (types, expansion, genbuildreqs, depsort/SCC, `:depends`) | `internal/scheduler/check` | B § 3.2-3.4 parity on corpus | T02, E06-T03 | L |
| E04-T06 | `checkpkgs` 21-rule decision, handler result mapping, post-processing (prpnotready, repostate, unblocked events, killunwantedjobs, packstatus write, lastcheck persistence, repo state) | `internal/scheduler/check` | corpus decision diff = 0 | T05 | L |
| E04-T07 | Default package handler (`Package.pm`): blocked computation, lastcheck fast path, meta regeneration, cycle `cyclevel`, rebuild/block modes, relsync trigger, `expand_sysroot` | `internal/scheduler/handler/pkg` | B § 3.8 cases as table tests; corpus | T05, E01-T09 | L |
| E04-T08 | Cycle handling three-pass | `internal/scheduler/check` | B § 3.9 cases | T06 | M |
| E04-T09 | Job creation (`create`, `create_jobdata`, `writejob`): paths with server URLs, sysdeps/bdeps/pdeps/vmdeps, DoD check, bdep flags, `bcnt`/release, `needed`, constraints, SLSA/ccache/reprorepo fields; `SubmitJob` | `internal/scheduler/buildjob` | buildinfo JSON equals Perl job XML (masked fields) on corpus | T06, E06-T02, E07-T01 | L |
| E04-T10 | Result integration driver (`jobfinished` dispatch, validation, code mapping, fake meta, patchpackstatus, jobhistory, relsync, buildavg) calling `artifact.IntegrateResult` | `internal/scheduler/results` | B § 4.5 | T09, E06-T04 | M |
| E04-T11 | Relsync (`:relsync`, `relsync_max` max-merge, trigger calc, nextbcnt) | `internal/scheduler/relsync` | B § 3.12 | T06 | S |
| E04-T12 | Publish hand-off (`repodone` hash, filters, naming, deltas jobs, archsync, `SetRepoPool`, event) | `internal/scheduler/publish` | B § 5.5 / P § 1.2 outputs equal on corpus | T06, E06-T05 | M |
| E04-T13 | Remote repositories and watches (`addrepo_remote` via artifact-svc remote cache, `remote-watch` translating `lastevents`, resync) | `internal/scheduler/remote` | against Perl peer in CI | T05, E06-T07 | M |
| E04-T14 | DoD glue (`init_doddata` → `Dod.SyncRepos`, placeholder handling, `dodfetch`, `checkdodresources`) | `internal/scheduler/dod` | B § 5.3 | T05, E06-T08 | S |
| E04-T15 | Access checks (`checkprpaccess`, roles) | `internal/scheduler/access` | B § 6.5 | T02 | S |
| E04-T16 | API RPCs: `Result` (incl. summary, multibuild, locallink, oldstate wait), `Depends`, `Command`, `RepoState`, admin RPCs, schedulerinfo metrics | `internal/scheduler/api` | `/build/<p>/_result` parity | T06 | M |
| E04-T17 | Incremental expansion memo (deviation intended, perf): cache `(config hash, pool key, deps hash) → edeps` | `internal/scheduler/check` | no decision change on corpus; check time reduction measured | T05 | M |
