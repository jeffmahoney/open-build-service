# 00 — Goals, Non-Goals, and Requirements

## 1. Purpose

Replace the OBS Perl backend with Go services that preserve every user-visible
behaviour of OBS while removing the architectural limits of the current design:

- a single large POSIX filesystem (`/srv/obs`) shared by source server,
  repository server, schedulers, dispatcher, publisher and signer;
- one scheduler process per architecture holding the state of every project in
  memory;
- fixed, hand-provisioned worker hosts registered against one repository
  server;
- Perl idioms (forking servers, pipe/fifo wakeups, `Storable` caches,
  `XML::Structured`) that limit who can maintain the code.

## 2. Goals

1. **Behavioural parity.** Every operation reachable through the Rails API,
   the WebUI, `osc`, and the worker protocol produces the same observable
   result: same XML, same status values, same build decisions, same published
   repository layout and metadata, same signatures.
2. **Horizontal scalability.** Every service scales out by adding replicas.
   No service holds global state in process memory that cannot be rebuilt
   from PostgreSQL and object storage.
3. **Sharding.** Scheduling state and binary pools are partitioned by project
   (configurable down to repository/architecture). Shards can be rebalanced
   online.
4. **Autoscaling workers.** Build capacity scales on queue depth per
   architecture and worker class, from zero, on Kubernetes node pools or
   cloud instances, with no manual registration.
5. **Data locality.** The dispatcher prefers workers whose caches already hold
   a job's build dependencies and whose zone matches the shard that owns the
   job's binaries. Published repositories and binary pools live in object
   storage and are served through CDN-friendly, immutable, content-addressed
   paths.
6. **Operability.** Everything is observable via OpenTelemetry; every service
   is stateless or has an explicit, documented, persisted state machine; a
   Kubernetes operator owns topology.
7. **Maintainability.** Idiomatic Go, typed contracts, a test suite with a
   golden parity corpus derived from the Perl implementation.

## 3. Non-goals

- Replacing the Rails API/WebUI, the MariaDB frontend database, or the public
  OBS API. The Rails application is a client of the new backend.
- Rewriting `obs-build`. It is reused unchanged inside the build VM.
- Rewriting external tools the publisher shells out to (`createrepo_c`,
  `modifyrepo_c`, `dpkg-scanpackages`-equivalents, `cosign`, `rsync`).
  They are wrapped and containerised.
- Live, mixed-version operation of Perl and Go components against the same
  `/srv/obs` tree. Migration is an import followed by a cutover
  (`09-migration-and-import.md`).
- Changing the semantics of project configuration (`prjconf`), `_link`,
  `_aggregate`, `_multibuild`, `_service`, `_patchinfo`, `_product`,
  `_channel`, `_constraints`. They are specified for parity.

## 4. Non-functional requirements

| Area | Requirement |
|---|---|
| Durability | No acknowledged source commit, build result, or publish is lost on loss of any single node, zone, or service replica. Every write to object storage is fsynced-equivalent (S3 PUT returns) before the corresponding PostgreSQL row is committed. |
| Consistency | Source revisions are linearizable per package. Build state is eventually consistent across shards with bounded staleness (configurable, default 30 s) for cross-project dependency indexes. Published repositories switch atomically per repository. |
| Availability | Edge API, source service, and artifact download: 99.9 % monthly. Scheduling and publishing are asynchronous and may lag without counting as unavailability. |
| Latency | Edge read requests served from PostgreSQL or cache: p99 < 200 ms excluding payload transfer. Source commit (small package): p99 < 2 s. Job dispatch after a worker becomes idle: p99 < 5 s. |
| Scale targets (design, not measured) | 500 k packages across 100 k projects; 10 M source revisions; 50 M stored binaries; 5 000 concurrent workers across 10 architectures; 100 k events per minute during a full Factory rebuild; 500 TB published data. Each must be reachable by adding replicas, shards, or storage, not by code change. |
| Security | Build VMs are the isolation boundary for untrusted code. Services authenticate with mTLS (SPIFFE/SPIRE or cert-manager). Signing keys never leave the signer service or its HSM/KMS. The edge trusts the Rails application by mTLS identity, not IP. |
| Compatibility | All XML documents validate against the RelaxNG/XSD schemas in `docs/api/api` and against the `XML::Structured` definitions in `BSXML.pm` (`01-legacy-system-analysis.md`). |
| Portability | Any S3-compatible store (AWS S3, Ceph RGW, MinIO, GCS via S3 interop), PostgreSQL 15+, NATS 2.10+, Kubernetes 1.29+. |
| Observability | Every RPC and queue hop carries trace context. Every state transition of a package build state emits a structured event. Dashboards and alerts are shipped with the Helm chart. |

## 5. Glossary

| Term | Meaning |
|---|---|
| project (`projid`) | Namespace with meta XML, project configuration (`prjconf`), repositories, flags, roles. |
| package (`packid`) | Source package inside a project; has revisions, meta XML, flags. |
| `prp` | `project/repository` pair. A repository is a named build target with path references to other `prp`s and a list of architectures. |
| `prpa` | `project/repository/arch`. The unit of scheduling state and of the binary pool. |
| `srcmd5` | MD5 over the sorted list of `md5  filename` lines of a package revision's files. Identifies source content. The "expanded" srcmd5 is the srcmd5 after `_link` expansion. |
| `:full` | The pool of binaries of a `prpa` that are usable as build dependencies (one version per package name, subject to `useforbuild`). |
| `:repo` | The complete set of binaries of a `prpa` intended for publishing. |
| `bininfo` | Per-package index of built binaries (name, version, arch, provides, hdrmd5). |
| `buildinfo` | The fully resolved description of a build job: sources, dependencies, project config, constraints. |
| `prjconf` | Project configuration text (`Macros:`, `Prefer:`, `Ignore:`, `Required:`, `Support:`, `Keep:`, `Substitute:`, `Repotype:`, `Type:`, …). |
| shard | A set of `prpa`s whose scheduling state and binary pool are owned by one scheduler replica at a time. |
| worker class | A worker pool with a given architecture, sandbox type (kvm, docker, …), and hardware profile. |
| DoD | Download-on-demand repository: an external repository mirrored and indexed as a build dependency source. |
| interconnect / remote | A project in another OBS instance referenced by `<remoteurl>` and used for sources or binaries. |

## 6. Open questions

- Scale targets above are design assumptions; replace with measured numbers
  from the target instance before capacity planning.
- Whether a managed PostgreSQL with logical sharding (Citus) or CockroachDB is
  used for metadata is an operator choice; the schema in `04-storage.md` is
  written to be portable to both.
