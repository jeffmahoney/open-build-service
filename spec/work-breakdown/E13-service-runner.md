# E13 — service-runner

Milestone M1. Inputs: `services/service-runner.md`, analysis F § C,
S § 6, `proto/servicerunner.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E13-T01 | Queue consumer, per-project semaphore, retry/transient classes, input assembly (sources bundle, `.old/`, `_serviceproject`), result upload and `CommitServiceResult` | `cmd/service-runner`, `internal/servicerunner` | corpus service runs reproduce `_service:*` outputs (tar_scm fixtures) | E03, E02-T13 | M |
| E13-T02 | Sandbox executor: image, rootless podman/crun or K8s Job, volumes, env, timeout, network policy table, SCM cache volumes, exit-code contract, output renaming/noprefix, mode skipping, chaining | `internal/servicerunner/sandbox` | F § C.2 step tests; network policy verified | T01 | L |
| E13-T03 | `RunTool` sandbox for `bs_productconvert`, `patch`/`diff3`/`bs_mergechanges` link application, `diff` rendering | `internal/servicerunner/tools` | used by E02-T09/T15/T20 | T02 | M |
| E13-T04 | obs-scm bridge runs (`obs_scm_bridge` definition, project mode cpio, `CommitObsScm`) | `internal/servicerunner/obsscm` | scmsync project sync e2e | T02, E02-T14 | M |
| E13-T05 | `ListServices` from image; configuration fan-in | `internal/servicerunner/api` | `GET /service` parity | T01 | S |
