# Service: scheduler

Replaces: `bs_sched`, `BSSched/*` (Checker, BuildJob and BuildJob/*,
BuildRepo, BuildResult, ProjPacks, PublishRepo, Remote, RepoCache, DoD glue,
EventHandler, EventQueue, EventSource/*, Lookat, RPC, Access, Bininfo,
Modulemd), `BSBuild.pm`, the scheduler's use of `BSSolv`.

Reference analysis: `01-legacy-system-analysis.md` § B (scheduler).
Contracts: `proto/scheduler.proto`, `proto/artifact.proto`.
Storage: `04-storage.md` § 2.3.

## 1. Responsibilities

Per shard (a set of projects, § 3 of `03`), for every `prpa`:

1. Maintain the working set of project/package data (projpacks) from
   `source-svc`, incrementally updated from `obs.source.*` events.
2. Run the check pipeline (`setup → wipeobsolete → preparepool →
   expandandsort → calcrelsynctrigger → checkpkgs → publish`) with the exact
   decision rules of B § 3.
3. Create build jobs (buildinfo) and hand them to `dispatcher`; kill obsolete
   jobs.
4. Integrate finished jobs (`jobfinished`) through `artifact-svc`: results,
   meta, `bininfo`, `:full`, `:repo`, relsync, jobhistory.
5. Build in-process "jobs": aggregates, channels, patchinfos (non-ptf),
   imports, uploads, deltarpm orchestration, reproducible-check.
6. Cross-arch and cross-prp coordination: unblocked events, waiting markers,
   relsync, exportfilter imports, `prpnotready` publication.
7. Remote repositories (interconnect) and DoD interaction.
8. Repository state, dirty flag, `REPO_BUILD_STARTED/FINISHED`, package
   status publication.
9. Publish hand-off (`:repo` snapshot + `obs.publish.requested`).

## 2. Process model

- One binary, N replicas. Each replica acquires shard leases (`shard_lease`)
  according to the operator's assignment and runs one **shard loop** per
  lease, each with its own goroutine pool. Within a shard, prpa checks are
  serialised per prpa but may run in parallel across prpas that do not
  depend on each other (the prp dependency DAG from `prpdeps` decides;
  default parallelism 4 per shard, configurable).
- The legacy process-per-architecture split disappears: a shard holds all
  architectures of its projects. The per-arch state is still separate per
  prpa; cross-arch events that used to go through files become in-memory
  channel sends when both prpas are in the same shard, and NATS events
  otherwise.
- The working set is loaded on lease acquisition from `GetProjPack`
  (shard-filtered, streamed) and from the shard's tables; it is **not**
  dumped on exit. A replica that loses a lease stops its loop; the next owner
  reloads. Anything that was in-memory only in Perl (`changed_*`, `lookat_*`,
  `prpnotready`, `repounchanged`, `retryevents`, `delayedfetchprojpacks`)
  is persisted in small tables (`prpa.dirty`, `prpa.next_med/next_high`,
  `sched_queue(prpa_id, level, changelevel, enqueued_at)`,
  `prp_notready(prpa_id, srcname)`, `sched_retry(prpa_id, due, kind)`) so a
  handover costs no full pass.

## 3. Event intake

Subscriptions per shard (durable consumer named by shard id):

| Event | Handling (B § 2.2 semantics) |
|---|---|
| `obs.source.package.changed` | fetch package data (delayed until the prp is looked at, as `delayedfetchprojpacks`), follow source links (`linkinfo`), enqueue high |
| `obs.source.project.changed` | cheap meta update; critical change → full refetch; enqueue high (level 2) or low for `lowprio` |
| `obs.source.config.changed` | as critical project change; dependents via `rprpdeps` to next series |
| `obs.build.job.finished` | `jobfinished` integration (batched per prpa, `dstcache` semantics) then enqueue med |
| `obs.build.pool.changed` (other shard/remote) | drop solv cache for that prpa; enqueue med level 2 |
| `obs.build.unblocked` / `relsync` | enqueue med / low |
| `obs.build.admin.*` | rebuild/recheck/admincheck/wipe/useforbuild/scanrepo/scanprjbinaries |
| `obs.source.project.suspend/resume` | suspension list with 8 h timeout |
| `obs.remote.*` | translated remote events (watch filters as `setup_watches`) |

Events are batched per loop iteration and ordered as `EventQueue::order`
(exit-class first, suspend, normal, upload/import last). Upload/import
events are deferred while a projpack fetch for the same project is pending.

## 4. Priority scheduler (Lookat)

Implements B § 2.5 exactly: `high`, `med`, `low`, `next` series; starvation
counters (`notlow > 10`, `notmed > 2`); back-off after expensive checks
(`nextmed = now + min(4, 1+|med|/100)*10*delay`, `nexthigh = now +
min(4, 1+|high|/10)*delay`); the low series is walked in global topological
prp order; `extend_lookat_next` transitive closure over `rprpdeps` chunked
by 100. The queue lives in `sched_queue` and a mirrored in-memory heap.

## 5. Check pipeline

All rules in B § 3 are normative. Implementation notes:

- **Pool construction** uses `pkg/solv`, a Go binding to libsolv (cgo) with
  the same operations the Perl `BSSolv` XS exposes: `pool.createwhatprovides`,
  `repofromfile/repofrombins/updatefrombins`, `expander` with the prjconf
  flags (`prefer`, `ignore`, `conflict`, `expandflags`), `depsort2` with SCC
  mode, `gen_meta`/`add_meta`/`diffdepth` (`genmetaalgo` 0/1),
  `kiwiproductcheck`, `obscpio` helpers are in `pkg/obscpio` instead.
  A pure-Go fallback for `expand` (port of `Build::Expand`) is used only for
  tests.
- **Pools for other prpas** come from `artifact-svc.GetPoolIndex(prpa,
  version)` which returns the solv object key and the `repostate`
  (not-ready set). The scheduler caches solv files on local disk keyed by
  `(prpa, full_version)`; same-shard prpas may be read directly from the
  shard's tables.
- **Build config** = `source-svc.GetConfig(project, repository, path)`
  parsed by `pkg/prjconf` (same semantics as `Build::read_config` for the
  scheduler-relevant directives; macros stripped).
- **`lastcheck`** 128-byte decision cache persisted in `build_meta.lastcheck`.
- **Cycle handling**: three-pass `handlecycle` as B § 3.9.
- **Status overlay**: `finished`/`dispatching`/`signing` and dispatch details
  are written to `package_status.overlay_*` by `dispatcher`/`signer`
  callbacks, pruned by the next check.
- **Per-type handlers**: one Go package per recipe type under
  `internal/scheduler/handler/` with the interface
  `Expand / Check / Build / JobFinished` and the behaviours in B § 4.4:
  `package` (default: spec, dsc, arch, helm, flatpak, snapcraft, collax,
  apk, buildenv), `preinstallimage`, `image` (mkosi, appimage, livebuild,
  simpleimage), `kiwiimage`, `docker` (also fissile), `kiwiproduct`,
  `productcompose`, `aggregate`, `channel`, `patchinfo` (incl. ptf job),
  `deltarpm`, `import`, `upload`, `reproduciblecheck`, `unknown`.
- **Job creation** (`create`, B § 4.2): `buildinfo` is produced as protobuf
  and rendered to the legacy XML by compat-edge/worker-agent; field semantics
  (`bdep` flags `preinstall/vminstall/runscripts/notmeta/installonly/
  noinstall/sysroot`, `path` with `server`, `syspath`, `needed`, `readytime`,
  `bcnt`, `release`, `disturl`, `genbuildreqs`, `constraintsmd5`, …) are
  preserved. `path[].server` points at compat-edge (worker-facing URL) so the
  Go worker-agent and any legacy tooling resolve binaries the same way.
- **Kill**: `kill_otherjobs`, `killunwantedjobs` → `dispatcher.KillJob`.

## 6. Result integration

`jobfinished` (B § 4.5) becomes:

1. `dispatcher` publishes `obs.build.job.finished` after the worker's result
   bundle is in `jobs/<id>/result.tar` and (if signing applies) after
   `signer` finished.
2. Scheduler validates (`status.job == this job`, project/package exist),
   maps the result code, and calls `artifact-svc.IntegrateResult(job,
   kind)` which performs the `update_dst_full` logic (`package_result`,
   `build_meta`, `bininfo`, `full_pool` selection with `useforbuild`,
   `exportfilter` → import jobs for other archs, `_volatile`, solv index
   rebuild, `full_version` bump, `obs.build.pool.changed`).
3. Scheduler updates `package_status` overlay, `jobhistory`, `relsync`,
   `buildavg`, and enqueues the prpa at med (level 2 if `:full` changed).

`unchanged` and `failed` results follow B § 4.5 steps 6-7 (meta and log
only). `genbuildreqs` stores the generated requires and re-queues.

## 7. Publish hand-off

`Checker::publish` semantics (B § 5.5, P § 1.2): compute `repodone` state
hash; if changed, write the `:repo` snapshot via
`artifact-svc.SetRepoPool(prpa, entries, repoinfo)` (filters: `publishfilter`,
`excludepublish/onlypublish`, `singleexport`, `noorphanedsrcrpms`,
`nosourceaccess`; naming rules `<packid>::` for debian/updateinfo/modulemd;
deltas; provenance; conflicts; `.archsync`), bump `repo_info.version`, and
publish `obs.publish.requested`. Delta generation creates `_delta` jobs
and keeps the prpa `building` until they finish.

## 8. Cross-arch and cross-shard

| Legacy mechanism | New |
|---|---|
| `events/<otherarch>/unblocked::P::R` | in-shard channel or `obs.build.unblocked` |
| `.waiting_for_<arch>` marker files | `package_status.overlay_details` + a `waiting_for` jsonb on `package_status` |
| other arch's `:packstatus`, `:logfiles.*`, `:bininfo` read from disk | `package_status`/`bininfo` tables for same shard; `artifact-svc.GetBininfo/GetPackStatus` for other shards |
| `relsync` files and events | `relsync`, `relsync_max` tables + event |
| `exportfilter` import jobs in another arch's job dir | `obs.build.import` with result object refs |
| `prpnotready` in memory | `prp_notready` table + included in `GetPoolIndex` response (`repostate`) |
| remote `lastevents` watchers | `remote-watch` module |

## 9. Observability

Metrics per shard: queue lengths by level, check duration histogram per prpa,
`buildavg`, event lag, lease age, pool cache hit rate, solv memory. The
`schedulerinfo` XML (`writeschedulerinfo`) is reproduced by compat-edge
from these metrics for `/build/_workerstatus` parity.

## 10. Tasks

`work-breakdown/epics.md` E4 (scheduler core), E5 (handlers), E6
(artifact integration).

## 11. Open questions

- Whether to allow a shard to be split by architecture for very large
  projects (shard_override at prpa granularity) in phase 1. The schema allows
  it; the scheduler must then treat sibling archs as "other shard" for the
  mechanisms in § 8, which is the general path anyway.
- libsolv via cgo vs. a pure-Go solver: cgo is the default for parity and
  performance; a pure-Go expander is a stretch goal.
