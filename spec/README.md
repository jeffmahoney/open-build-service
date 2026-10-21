# OBS Next-Generation Backend Specification

This directory specifies a re-architecture of the Open Build Service (OBS)
backend (`src/backend`, Perl) as a set of Go microservices designed for
Kubernetes, horizontal autoscaling, sharding, and data locality across a
distributed storage substrate.

The specification is written to be implemented by many independent agents
working in parallel. Every document states its scope, its dependencies on other
documents, and the invariants an implementer must preserve. The work breakdown
in `work-breakdown/` turns the specification into bounded tasks with
acceptance criteria.

## Decisions that frame this specification

| Topic | Decision |
|---|---|
| Scope | Backend only. The Rails API/WebUI (`src/api`) is retained. Phase 1 is wire-compatible with the backend HTTP/XML API that the Rails application and `osc` already use. |
| Compatibility | `osc`, the public XML API, and every existing client keep working unchanged. Internal storage is greenfield. A one-time importer migrates an existing `/srv/obs` tree. |
| Platform | Kubernetes-native: Deployments with HPA/KEDA, a Go operator with CRDs for shards and worker pools, KVM-capable node pools for build workers. |
| Storage | S3-compatible object storage for all content (sources, binaries, published repositories, logs) and PostgreSQL for metadata and state. Locality via shard affinity and worker/node caches. |
| Build tooling | `obs-build` (the shell/Perl scripts that run inside the build VM) is reused unchanged. Only the worker daemon (`bs_worker`) is rewritten in Go. |
| Build formats | Full parity: every recipe type and repository type supported by OBS today. |
| Plumbing | gRPC + protobuf between services; HTTP/XML only at the edge; NATS JetStream for events and work queues; artifacts move through presigned S3 URLs, never through RPC; OpenTelemetry; Helm charts plus a Go operator. |
| Language | Go for all services. Shell is acceptable where it wraps external tooling (createrepo_c, dpkg tools, cosign). Perl only inside `obs-build`. |

## Document map

Read in this order.

| # | Document | What it covers |
|---|---|---|
| 0 | [00-goals-and-requirements.md](00-goals-and-requirements.md) | Goals, non-goals, non-functional requirements, scale targets, glossary |
| 1 | [01-legacy-system-analysis.md](01-legacy-system-analysis.md) | How the current Perl backend works: components, on-disk formats, protocols, algorithms. The ground truth for parity |
| 2 | [02-domain-model.md](02-domain-model.md) | Projects, packages, revisions, links, repositories, build states, jobs, workers, published repos. Invariants |
| 3 | [03-architecture.md](03-architecture.md) | Service decomposition, sharding, data locality, request and event flows, Kubernetes topology |
| 4 | [04-storage.md](04-storage.md) | Object store layout (content-addressed), PostgreSQL schema, caches, consistency rules |
| 5 | [05-events-and-queues.md](05-events-and-queues.md) | NATS subjects, event catalog, work queues, delivery semantics |
| 6 | [06-compat-edge-api.md](06-compat-edge-api.md) | The legacy backend HTTP/XML API that the edge must serve, route by route, mapped to internal services |
| 7 | [07-security.md](07-security.md) | Trust boundaries, authn/authz between services, worker isolation, signing keys, secrets |
| 8 | [08-observability-and-operations.md](08-observability-and-operations.md) | Metrics, tracing, logging, SLOs, runbooks, upgrade and rollback |
| 9 | [09-migration-and-import.md](09-migration-and-import.md) | Importer from `/srv/obs`, cutover plan, parity test harness |
| 10 | [10-testing-strategy.md](10-testing-strategy.md) | Unit, contract, integration, end-to-end and parity testing; the golden corpus |
| 6a | [06a-legacy-route-catalog.md](06a-legacy-route-catalog.md) | All 279 legacy backend routes, extracted mechanically from the dispatch tables |
| A | [analysis/](analysis/) | The five engineering reports on the Perl backend (≈9 200 lines, `file:line` cited) that `01` indexes |
| S | [services/](services/) | One specification per service: compat-edge, source-svc, scheduler, dispatcher, worker-agent, artifact-svc, log-svc, publisher, signer, registry-svc, service-runner, notifier, index-svc, dod-svc, operator, importer |
| P | [proto/](proto/) | Protobuf IDL for every internal gRPC API (15 files, compile with `protoc`) and the service authorization matrix |
| W | [work-breakdown/](work-breakdown/) | 17 epics, 5 milestones, dependency graph, and ~150 agent-sized tasks with scope, inputs, acceptance criteria and dependencies |

## Conventions for implementing agents

- **Repository layout.** Implementation lives in a new Go module `obs-ng` (separate repository or `src/backend-go/`, decided at kickoff). Layout: `cmd/<service>/`, `internal/<service>/`, `pkg/` for shared libraries, `api/proto/` for IDL, `deploy/` for Helm and operator manifests.
- **Ownership.** Each service in `services/` is independently buildable, testable and deployable. A task may touch only the service it is assigned to plus `pkg/` libraries listed in the task. Cross-service changes require a contract change in `proto/` first.
- **Contracts first.** The `.proto` files in `proto/` are the source of truth for internal APIs. Change the proto, regenerate, then implement. Breaking changes are not allowed after a service reaches milestone M2; add fields, never renumber.
- **Compatibility is a test.** Every legacy route listed in `06-compat-edge-api.md` has a parity test against the golden corpus (`10-testing-strategy.md`). A task that touches a route is not done until its parity test passes.
- **Perl is reference, not design.** When this specification is silent, read the Perl code cited in `01-legacy-system-analysis.md` to determine behaviour, and reproduce the behaviour, not the structure.
- **Names.** Use the legacy vocabulary where users see it (`prp`, `prpa`, `srcmd5`, `:full`, `bininfo`, `buildinfo`) and plain Go names internally.
- **Diagrams** are Mermaid in Markdown so they render on GitHub without tooling.
- **Commits** follow `CONTRIBUTING.md`; AI-assisted commits carry an `Assisted-by:` trailer.

## Status

This is the initial version of the specification, produced from an analysis of
the Perl backend at commit `db93ae665b` and the `obs-build` submodule at
`1fe25db5f6`. Open questions are collected at the end of each document under
"Open questions".
