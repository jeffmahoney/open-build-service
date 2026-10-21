# Service: artifact-svc

Replaces: the repository tree side of `bs_repserver` and
`BSRepServer/{Bininfo,BuildInfo,Checker,Containerinfo,Containertar,DoD
(serving side),ProjPacks,Remote,SLSA,YMP}.pm`, `BSSched/{BuildRepo,
BuildResult,Bininfo,Blobstore}.pm` (pool maintenance, executed on behalf of
the scheduler), `BSRepServer/Registry.pm` build-time pulls (delegated to
registry-svc), the `_repository` views, `bs_check_consistency` for the
build tree.

Contract: `proto/artifact.proto`. Storage: `04` § 1.1 (`bins/`, `meta/`,
`pools/`, `jobs/`), § 2.3 (`package_result`, `bininfo`, `full_pool`,
`full_index`, `repo_pool`, `repo_info`, `preinstallimage`, `jobhistory`).

## 1. Responsibilities

1. **Result integration** on behalf of the scheduler (`IntegrateResult`):
   the `update_dst_full` algorithm (B § 3.11, § 4.5): unpack the result
   bundle into `bins/` objects, write `package_result`, `build_meta`,
   `.meta.success`, `bininfo` (per package + global), `.nouseforbuild`,
   `.nosourceaccess`, preinstall image registration (bitstring), container
   `obsbinlnk` records, `exportfilter` evaluation → `obs.build.import` jobs,
   `:full` membership update with the precedence rules (`orderpackids`,
   maintenance incident ordering, `_volatile`, imports last, `volatile_cmp`),
   `useforbuild` gating, solv index rebuild (incremental `updatefrombins`
   equivalent via `pkg/solv`), `full_version` bump, `obs.build.pool.changed`
   (full vs bininfo-only).
2. **Pool reads**: `GetPoolIndex(prpa, version)` → solv object key +
   repostate (not-ready set) + modules; `GetFull(prpa)` membership;
   `view=solvstate|cache|names|binaryversions|binaryversionscode|
   gbininfocode|fileinfo|fileinfo_ext|cpio|cpioheaderchksums|names|
   publishedpath` as RPCs for compat-edge and interconnect; DoD placeholder
   handling (`d0d0d0d0…` hdrmd5, `id=dod`) and `dodfetch` downloads
   on demand.
3. **Binary access** for workers and API: presigned URLs by digest;
   `getbinaries` cpio fallback for legacy consumers; `getbinaryversions`;
   `getpackagebinaryversionlist` (old versions still in package dirs);
   `_buildenv`, `_statistics`, `rpmlint.log`, `_reason`, `_status`,
   `_history` (succeeded history BSFileDB equivalent), `.stats`,
   `_jobhistory` queries with `limit/endtime/code/package` filters.
4. **`_buildinfo` generation** (`BuildInfo.pm` port): runs the scheduler's
   job-creation path in "repserver mode" (`isreposerver`, `dobuildinfo`)
   against pools and gbininfo, including POST variants with uploaded spec/
   buildenv/deps/_service, `add=`, `debug=1`, `internal=1`, `fixupbuildinfo`
   (strip internal fields, public `url`s, `downloadurl`, preinstall image,
   modularity yaml). Implemented as a library shared with the scheduler
   (`internal/scheduler/buildjob`) to guarantee identical results.
5. **Repo pool for publishing**: `SetRepoPool(prpa, entries, repoinfo)` from
   the scheduler, versioned; `GetRepoPool` for the publisher; `unpublish`
   edits with the `.finishedlock` semantics replaced by `repo_info.version`
   CAS.
6. `copybuild` (`cmd=copy` with `setupdateinfoid`, `resign`, `setrelease`,
   `multibuild`), `uploadbuild`, `putbinary`/`delbinary` on `_repository`,
   `wipe`, `scanrepo`, `scanprjbinaries`, `useforbuild` refresh, SLSA
   `addrefs`/material store (`slsa/<prpa>/<file>/<digest>` refs table and
   objects kept alive beyond binary GC), YMP generation, `_availablebinaries`.
7. Cross-shard reads for the scheduler: `GetBininfo(prpa)`,
   `GetPackStatus(prpa)` (proxy to the owning shard's tables), remote OBS
   fetches with the `remotecache` (solv + bininfo cookies, 24 h/1 h TTLs,
   serialised per prpa, `lastevents`-driven invalidation).
8. GC: mark-and-sweep over `bin_ref` from `package_result`, `full_pool`,
   `repo_pool`, published versions, SLSA refs.

## 2. Consistency

All integration for a prpa is serialised by the owning scheduler (single
caller), which makes `full_pool` updates per prpa linearizable. Readers see
versioned snapshots (`full_version`). The legacy self-healing
(`fctx_rebuild_full`, `rebuild_gbininfo`) becomes a `Verify(prpa)` RPC that
recomputes `full_pool` from `package_result` + `useforbuild` and reports or
repairs drift; `obs-admin check-consistency` drives it.

## 3. Scaling and locality

Stateless replicas; solv rebuilds are CPU-bound and cached by
`(prpa, full_version)`; shards pin their prpas' objects to a storage zone
(`shard_map.storage_zone`) so same-zone workers get same-zone reads.

## 4. Tasks

`work-breakdown/epics.md` E6.
