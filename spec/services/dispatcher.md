# Service: dispatcher

Replaces: `bs_dispatch`, `BSDispatcher/Constraints.pm`, `bs_warden`, the
worker-registration and job endpoints of `bs_repserver` (`/worker`,
`/putjob`, `/jobs/*`, `/workerstatus`, `/getpreinstallimageinfos`,
`/getworkercode`, `/getbuildcode`), `BSCando.pm` (as `pkg/arch`).

Reference analysis: `01-legacy-system-analysis.md` § B.3-B.5.
Contracts: `proto/dispatcher.proto`, `proto/worker.proto`.
Storage: `04` § 2.3 (`job`, `worker`), queues `05` § 3.1.

## 1. Responsibilities

1. Own the `job` table lifecycle after the scheduler creates a job:
   `scheduled → dispatching → building → finished|signing → integrated`,
   plus `killed`, `badhost` re-queue, `failed` by dispatcher decision.
2. Worker registry: registration with capability document, heartbeats,
   states `idle|building|down|away|dead|disabled`, physical-host grouping,
   cache summaries, zone and class.
3. Routing and prioritisation: compute per-arch/class candidate sets and
   publish jobs to `obs.jobs.<arch>.<class>.<prio>` with locality hints;
   workers pull. The ordering rules of `bs_dispatch` § 3.2 are preserved as
   the *ranking* that decides the order and priority subject in which jobs
   are published and re-published.
4. Constraints: three-source merge (prjconf `Constraint:`, package
   `_constraints` with `overwrite`, recipe hints), oracle semantics
   (`hostlabel`, `sandbox` incl. `secure` class, `linux` flavor/version
   range, `hardware` processors/jobs/disk/memory(+swap)/physicalmemory/
   cpu flags, `memoryperjob`), `nativeonly`, `powerpkgs`/`powerhosts`,
   `dispatch_adjust` and `dispatchprios` scales, site hook equivalent
   (`dispatch_constraint` as a WASM/plugin or webhook).
5. Badhost quarantine (24 h TTL, `prj/pkg/arch/worker`, `arch/job`,
   `arch/job/worker` keys, `>5+20/nhosts` attempts → `nobadhost`),
   `checkconstraints` feedback (dispatch details "waiting for N compliant
   workers (M down) (K bad)"; fail with hint when none comply),
   `CheckConstraints` RPC for the API.
6. Liveness (warden): lease expiry on missed heartbeats; orphan cleanup
   rules (status not building > 24 h → restart; stale dirs).
7. Result ingestion handshake (`putjob`): validate `(job, jobid, workerid,
   starttime)` claim, time-skew check, accept the result bundle object,
   run `artifact-svc.ValidateResult` (meta line format, forbidden files,
   container normalisation check, JSON size limits), append load accounting,
   decide `signing` vs `finished`, publish `obs.sign.job` or
   `obs.build.job.finished`, emit `BUILD_SUCCESS|FAIL|UNCHANGED`.
8. Preinstall image selection **server-side**: given a job's bdep hdrmd5
   set, choose the best image from `preinstallimage` rows (bitstring subset
   test, most packages, not own package, at least one new package for
   `_preinstallimage` builds, disabled for SLSA and `needsbinariesforbuild`)
   and attach it to the job sent to the worker.
9. Load model: per-prpa exponential decay (`log(.5)/4h`), project
   smoothing, staleness/source-change bias, `needed` ordering,
   `_preinstallimage` and `powerpkgs` first, 50-assignments-per-60 s
   fairness.
10. Master/slave dispatch across partitions is dropped; sharded dispatchers
    by arch replace it.

## 2. Pull model and ranking

```mermaid
sequenceDiagram
  participant S as scheduler
  participant D as dispatcher
  participant N as NATS
  participant W as worker-agent
  S->>D: SubmitJob(buildinfo, constraints, prio inputs)
  D->>D: rank: scale*load, srcchange-first, needed, readytime
  D->>N: publish obs.jobs.<arch>.<class>.<hi|med|lo> {job_id, hints}
  W->>N: pull (hinted first, then any)
  W->>D: ClaimJob(job_id, worker_id, capabilities) -> lease or reject
  D->>D: oracle(constraints, worker) ; badhost ; nativeonly ; CAS state=dispatching
  D-->>W: JobSpec (buildinfo + presigned inputs + preinstall image)
  loop heartbeat
    W->>D: Heartbeat(lease, progress, cache filter)
  end
  W->>D: UploadResult(result object, code)
  D->>D: validate, ingest, signing?
  D->>N: obs.sign.job | obs.build.job.finished
```

- `class` is derived from constraints: `sandbox` (secure vs container),
  hardware tiers, hostlabels, hostarch (for personality/cross jobs). The
  operator defines classes in `ObsWorkerPool` CRs; the dispatcher maps a
  job to the smallest class set that can satisfy its constraints and
  publishes to each (first claim wins; duplicates are purged).
- Ranking is recomputed every pass over ready jobs; changed priority moves a
  message between `hi/med/lo` subjects.
- Hints: top-k workers by cache overlap (cuckoo filter) and zone.
- `ClaimJob` is the atomic claim (compare-and-swap on `job.state`), the
  replacement for `lockcreatexml`.

## 3. Worker protocol (`proto/worker.proto`)

```
service WorkerGateway {
  rpc Register(WorkerInfo) returns (RegisterResponse);            // capabilities (hostarch, archs via cando, sandbox, hardware, linux, hostlabel, nativeonly, zone, class, version)
  rpc ClaimJob(ClaimRequest) returns (JobSpec);                    // lease
  rpc Heartbeat(stream HeartbeatMsg) returns (stream ControlMsg);  // lease extension, progress, cache filter; control: kill, discard, sysrq, logtail request
  rpc UploadResult(ResultManifest) returns (UploadAck);            // result object key, code, statistics; presigned PUT obtained via GetUploadURL
  rpc GetUploadURL / GetDownloadURLs                               // presigned URLs for bundle and inputs by digest
  rpc SetState(WorkerState)                                        // idle/away/down/dead
}
```

`JobSpec` = legacy `buildinfo` fields + per-bdep `hdrmd5`/`metamd5` and
object digests (so the worker fetches by content hash), the concatenated
`.build.config` text, the preinstall image selection, the obs-build image
digest (`buildcode` equivalent), log limits, `nobadhost`, `jobid` (md5 of
the canonical buildinfo XML, kept for parity in logs and `_statistics`).

## 4. API-facing RPCs (via compat-edge)

`WorkerStatus` (legacy `workerstatus` XML: idle/building workers with
job/starttime, waiting/blocked counts per arch, scheduler/dispatcher/
publisher/signer daemon states), `WorkerInfo(id)`, `JobStatus(job)`,
`ListJobs(arch)`, `Get/SetDispatchPrios`, `KillJob`, `RestartBuild`,
`SendSysrq`, `CheckConstraints(project, package, repository, arch,
constraints xml)`.

## 5. Scaling

Sharded by architecture (consistent with the legacy per-arch design);
replicas per shard with the CAS claim; HPA on ready-job count. The pass
limit (50/60 s) applies per replica.

## 6. Tasks

`work-breakdown/epics.md` E7.
