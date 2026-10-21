# proto/ — Internal gRPC contracts

Package root: `obs.v1`. Go package: `github.com/openSUSE/obs-ng/api/proto/obs/v1`.
Tooling: `buf` (lint `DEFAULT`, breaking `FILE`), `protoc-gen-go`,
`protoc-gen-go-grpc`. Every file here is normative; implementers regenerate
and never hand-edit generated code.

| File | Service(s) | Owner service |
|---|---|---|
| `common.proto` | shared messages: `Actor`, `Prpa`, `LegacyError`, `Digest`, `PresignedURL`, `Flags`, `Pagination` | pkg |
| `events.proto` | all NATS message payloads (`05`) | pkg |
| `source.proto` | `Source` | source-svc |
| `scheduler.proto` | `Scheduler`, `SchedulerAdmin` | scheduler |
| `artifact.proto` | `Artifact` | artifact-svc |
| `dispatcher.proto` | `Dispatcher` (API-facing + scheduler-facing) | dispatcher |
| `worker.proto` | `WorkerGateway` (worker-facing) | dispatcher |
| `log.proto` | `BuildLog` | log-svc |
| `publish.proto` | `Publisher` | publisher |
| `sign.proto` | `Sign` | signer |
| `registry.proto` | `Registry` (internal push/ownership) | registry-svc |
| `notifier.proto` | `Notifier` | notifier |
| `index.proto` | `Index` | index-svc |
| `dod.proto` | `Dod` | dod-svc |
| `servicerunner.proto` | `ServiceRunner` | service-runner |

## Conventions

- Every mutating RPC: `Actor actor = 1; string idempotency_key = 2;` first.
- Names: legacy vocabulary in field names where users see it (`srcmd5`,
  `vrev`, `prpa`, `bininfo`), snake_case.
- Errors: `google.rpc.Status` with `LegacyError` detail (`http_status`,
  `summary`, `details`) for anything compat-edge must relay verbatim.
- Streams for anything > 1 MiB or unbounded (file lists of huge projects,
  logs, cpio).
- Large content never crosses RPC: `Digest` + `PresignedURL`.
- Versioning: additive only after milestone M2; reserved field numbers on
  removal.

## Authorization matrix (enforced by interceptors)

| Caller → Callee | Allowed RPCs |
|---|---|
| compat-edge → everything | read RPCs + the user-facing mutations listed per service |
| scheduler → source | `GetProjPack*`, `GetConfig`, `GetModulemd`, `NewEvent` |
| scheduler → artifact | all pool/integration RPCs |
| scheduler → dispatcher | `SubmitJob`, `KillJob`, `SetDispatchDetails` |
| dispatcher → artifact | `ValidateResult`, `RegisterResult` |
| dispatcher → scheduler | none (events only) |
| worker-agent → WorkerGateway, BuildLog | all |
| publisher → artifact, sign, registry, index, source(`GetProjPack nopackages`, `GetConfig`, patterns) | listed |
| sign-worker → sign, artifact(`SlsaAddRefs`) | listed |
| service-runner → source(`CommitServiceResult`, `CommitObsScm`) | listed |
| notifier → none | consumes NATS only |
| obs-admin → `*Admin` services | all |
