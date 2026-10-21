# Service: source-svc

Replaces: `bs_srcserver` (source side), `BSSrcrep.pm`, `BSRevision.pm`,
`BSFileDB.pm`, `BSSrcServer/{Link,Projlink,Service (client side),Scmsync,
ScmsyncDB,Remote (source side),Access,Multibuild,Product,Signkey (metadata
side),LinkinfoDB,Srcdiff,Blame}.pm`, `bs_deltastore`, `bs_mergechanges`
(as a library), `bs_sourcepublish` (as a module).

Reference analysis: `01-legacy-system-analysis.md` § S.
Contracts: `proto/source.proto`. Storage: `04-storage.md` § 2.2.

## 1. Responsibilities

1. Projects and packages: meta revisions, config, flags, existence, listing,
   project links, deletion and undeletion with history.
2. Source revisions: blobs, trees, `rev`/`vrev`, upload tree, commit
   handshake, history.
3. `_link` expansion, `keeplink`, `integratelink`, branch/copy/copyproject
   with every option in S § 2.10, `_frozenlinks`.
4. Special files: `_multibuild` cache, `_product` expansion
   (`bs_productconvert` invoked in a sandbox), `_aggregate`/`_patchinfo`/
   `_channel`/`_constraints`/`_modulemd.yaml`/`_buildenv` validation and
   extraction for the scheduler.
5. Service orchestration: servicemark computation, run state, dispatch to
   `service-runner`, result commit, `waitservice`, `mergeservice`,
   obs-scm (`scmsync`) runs and project sync.
6. Remote projects (interconnect): name resolution, remote getrev with local
   materialisation, remote meta/config fetch, batching.
7. The scheduler contract: `GetProjPack` streaming view (see § 4).
8. Signing-key metadata: `_pubkey`/`_signkey`/`_sslcert` storage with
   hierarchy lookup; the private key handling itself is delegated to
   `signer` (`services/signer.md`).
9. Diff and blame rendering with cache.
10. Source publishing export (`sourcepublish`).
11. Emitting `obs.source.*` events and `obs.notify.SRCSRV_*` notifications.

Not responsible for: XML rendering (compat-edge), build routing (compat-edge
→ artifact-svc/scheduler), search over published data (index-svc),
authorization of users (Rails).

## 2. gRPC API (summary; authoritative in `proto/source.proto`)

```
service Source {
  // projects
  rpc ListProjects(ListProjectsRequest) returns (stream ProjectRef);
  rpc GetProject(GetProjectRequest) returns (Project);            // meta at mrev or current; withconfig
  rpc PutProject(PutProjectRequest) returns (Project);            // idempotency_key, user, comment, requestid, lowprio
  rpc DeleteProject / UndeleteProject / CopyProject / MoveProject / FreezeProjectLink
  rpc GetProjectConfig / PutProjectConfig / DeleteProjectConfig
  rpc GetKeyInfo / CreateKey / ExtendKey / RecreateSslCert / DeleteKey
  // packages
  rpc ListPackages(ListPackagesRequest) returns (stream PackageRef); // expand (project links), deleted, noorigins
  rpc GetPackage / PutPackage / DeletePackage / UndeletePackage / LockPackage / UnlockPackage
  rpc GetPackageHistory(...) returns (stream Revision);           // rev, meta, deleted, limit, startbefore
  // files and revisions
  rpc GetFileList(GetFileListRequest) returns (FileList);          // rev, linkrev, expand, emptylink, deleted, lastworking, withlinked, meta, product
  rpc GetFile(GetFileRequest) returns (stream Chunk);              // or GetFileURL -> presigned
  rpc PutFile(stream PutFileRequest) returns (Revision);           // rev=upload default, keeplink, force, meta
  rpc DeleteFile(...) returns (Revision);
  rpc CommitFileList(CommitFileListRequest) returns (CommitFileListResponse); // missing list or revision
  rpc Commit(CommitRequest) returns (Revision);                    // cmd=commit
  rpc Branch / Copy / LinkToBranch / Diff / LinkDiff / ServiceDiff / Blame
  rpc GetSourcesCpio(GetSourcesRequest) returns (stream Chunk);    // /getsources for workers; also presigned bundle
  rpc GetSourceInfo(...) returns (SourceInfo);                     // view=info, parse recipe
  rpc ParseUploadedRecipe(...)                                     // POST view=info
  // services
  rpc RunService / WaitService / MergeService / GetProjectServices
  rpc CommitServiceResult(...)   // from service-runner
  rpc CommitObsScm(...)          // from service-runner (obs-scm)
  rpc TriggerScmSync(...)
  // scheduler contract
  rpc GetProjPack(GetProjPackRequest) returns (stream ProjPackChunk);
  rpc GetConfig(GetConfigRequest) returns (BuildConfig);           // concatenated prjconf along a path
  rpc GetModulemd(...)
  rpc CollectBuildEnv(...)
  // interconnect
  rpc RemoteGetRev / RemoteFetchProject / RemoteFetchConfig       // internal helpers used by compat-edge for remote proxies
  // streams
  rpc LastEvents(LastEventsRequest) returns (stream Event);        // numbered, filterable, long-poll
  rpc LastNotifications(...) returns (stream Notification);
  rpc NewEvent(...)                                                // POST /event from artifact-svc/scheduler for interconnect visibility
}
```

Every mutating RPC takes `Actor {user, comment, requestid}` and
`idempotency_key`. Errors use gRPC status with a `LegacyError` detail that
carries the exact legacy HTTP status and message text (`403 source access
denied`, `404 project 'X' does not exist`, `400 ... is locked`, and so on) so
compat-edge can reproduce them.

## 3. Algorithms (normative references)

| Function | Spec | Notes |
|---|---|---|
| `calcsrcmd5`, tree write/read | `02` § 2.5 | trees immutable; write-if-absent |
| `addrev` | `02` § 3.1, S § 2.3 | one transaction |
| `getrev` resolution | S § 2.1 | including deleted srcmd5 lookup, project links, remote, `missingok` |
| `getcommitinfo` (version/vrev) | S § 2.2 | recipe parsing uses `pkg/recipe` (Go port of `Build::parse_typed` for version/release only) |
| `handlelinks` / `applylink` | `02` § 3.3-3.4 | `patch`, `diff3`, `bs_mergechanges` run in a sandboxed helper container with the exact flags |
| `keeplink`, `integratelink`, `findlastworkinglink`, `linkinfo_addtarget` | S § 2.5 | |
| project links, frozenlinks, `getnewvrev` | S § 2.6 | |
| services | S § 6.1-6.2 | dispatch always via queue (no in-process fork); reads still trigger runs |
| `copy`, `branch`, `copyproject`, `move` | S § 2.10 | origin write-backs synchronous; `copyproject` bracketed by suspend/resume events |
| delete/undelete | S § 2.9 | |
| `_product` | S § 2.8 | `bs_productconvert` runs in a job container with the expanded sources; outputs become `_product:<x>` packages |
| remote | S § 3.9 | materialise remote trees as local trees in the consuming project; batch `view=info` prefetch |
| access checks | S § 5.2 | |
| `sourcediff` | S § 3.2 | views plain/unified/xml with issues; cache keyed as today |

## 4. The `GetProjPack` contract

The scheduler's bulk view. Semantics identical to `getprojpack` (S § 3.3,
`BSXML::projpack`): for each selected project, the project meta subset,
repositories (optionally expanded path with `base`), config, packages with
`srcmd5` (servicemark or xsrcmd5), `verifymd5`, `rev`, `versrel`, `linked`,
`originproject`, per-repository `info` (recipe name, deps, prereqs, path,
containerpath, extrasource, imagetype, buildtimeservice, error), aggregates,
patchinfo, channel data, modulemd, flags evaluated per repo/arch, errors
(`disabled`, `locked`, `excluded`, `broken link …`, `source update running`,
`no source uploaded`, `quota exceeded`), `remotemap`, `repoid`.

Differences from legacy:
- Streamed per project (chunked), never one giant document.
- Filtered by **shard** instead of partition: the scheduler asks for the
  projects of its shard; projects referenced from other shards come back in
  `remotemap` with `shard_id` so the scheduler uses published pool indexes
  (artifact-svc) rather than its own tables.
- Incremental: `GetProjPackDelta(since_cursor)` returns only projects/packages
  changed since a cursor (driven by `obs.source.*` events).
- Recipe parsing (`Build::parse` for deps/prereqs/exclarch/path) is done by
  `pkg/recipe`, a Go port of the parsers in `obs-build`'s `Build/*.pm`
  (spec, dsc, kiwi, Dockerfile, PKGBUILD, productcompose, helm, mkosi,
  flatpak, snapcraft, appimage, livebuild, simpleimage, collax, apk,
  preinstallimage). Parity tests compare output with `obs-build` on the
  golden corpus.

## 5. Service runs

- `RunService` computes the servicemark, writes `service_run(pending)`,
  publishes `obs.service.run`.
- `service-runner` executes and calls `CommitServiceResult` with the result
  files (already uploaded to blobs) or an error. `source-svc` writes the
  result tree with `/LSERVICE`/`/LXSERVICE`, clears the marker, emits events.
- `waitservice` is a server-streaming RPC that completes when the run leaves
  `running`.
- Per-project concurrency and retry (`due +10 min`, transient error
  detection) are the runner's job; `source-svc` only records state.

## 6. Scaling and locality

- Stateless replicas behind gRPC load balancing. Package-level mutual
  exclusion is a `SELECT … FOR UPDATE` on the `package` row.
- Hot read caches (project meta, package meta, latest revision, multibuild)
  are per replica with event-driven invalidation.
- Blob upload/download for large files uses presigned URLs; the
  `commitfilelist` handshake stays RPC-sized.

## 7. Compatibility obligations

- All ids (`srcmd5`, `xsrcmd5`, `verifymd5`, `servicemark`, `vrev`) are
  bit-identical to the Perl implementation on the golden corpus.
- Listing order, `<linkinfo>` and `<serviceinfo>` attributes, `error`
  strings in `sourceinfo`, and history record contents match.
- `_repoid` is stable and exposed in `GetProjPack`.

## 8. Tasks

See `work-breakdown/epics.md` E2.

## 9. Open questions

- `old_style_services` (service results committed as revisions by user
  `_service`) is not carried forward unless the importer finds instances
  using it.
- Whether `_product` expansion should move to `service-runner` entirely
  (it is a sandboxed tool invocation). Default: yes, via the same queue with
  a synchronous wait, since commits of `_product` are rare.
