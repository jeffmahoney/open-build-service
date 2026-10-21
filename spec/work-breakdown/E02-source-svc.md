# E02 — source-svc

Milestone M1. Inputs: `services/source-svc.md`, `04` § 2.2, `05` § 2.1,
`proto/source.proto`, analysis S (all sections).

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E02-T01 | Store layer: project/package/blob/tree/revision/meta_revision/upload_tree tables, migrations, repositories | `internal/source/store`, `pkg/store` | migration applies; CRUD tests; `rev`/`vrev` assignment under concurrency (S § 1.5, § 2.2) | E03-T01 | M |
| E02-T02 | Blob and tree store: `PutBlob` (sha256+md5, collision error), `WriteTree` (write-if-absent, pseudo entries), `ReadTree`, package_tree witness | `internal/source/blobs` | S § 1.3-1.4 semantics; `nosharedtrees` witness enforced (`lsfiles` dies without it) | T01, E03-T02 | M |
| E02-T03 | Project RPCs: List/Get/Put/Delete/Undelete/Config/Keys | `internal/source/project` | parity for `/source`, `_meta`, `_config`, `_pubkey`, `_keyinfo` routes (S § 3.1); critical-change detection for `obs.source.project.changed` | T01 | M |
| E02-T04 | Package RPCs + history + delete/undelete generations | `internal/source/package` | S § 2.9, § 3.2 parity | T01 | M |
| E02-T05 | `getrev` resolution incl. deleted srcmd5, project links, `missingok`, `_pattern`, retrofit | `internal/source/rev` | S § 2.1 table | T04 | M |
| E02-T06 | Commit path: `addrev`, `getcommitinfo` (recipe parse), servicemark, `_product` hook, notifications, events | `internal/source/commit` | S § 2.3 steps 1-12; `SRCSRV_*` params exact | T05, E01-T04, E01-T08 | L |
| E02-T07 | `PutFile`/`DeleteFile`/`Commit`/`CommitFileList` handshake, upload tree, `keeplink` integration | `internal/source/files` | `osc commit` e2e; `commitfilelist` missing/validate semantics | T06 | M |
| E02-T08 | Link expansion: `handlelinks`, `applylink` (simple), `xsrcmd5` chain, cache, `_linkerror` | `internal/source/link` | S § 2.5; corpus link chains reproduce `xsrcmd5`/`verifymd5` | T05, E13-T03 (tool sandbox for patch/diff3) | L |
| E02-T09 | `applylink` patch/branch/add/topadd/apply via tool sandbox; `patchspec` placement rules; `bs_mergechanges` port | `internal/source/link`, `pkg/mergechanges` | corpus patch links and branch merges (incl. conflicts) identical | T08 | L |
| E02-T10 | `keeplink`, `integratelink`, `findlastworkinglink`, `linkinfo_addtarget`, `update_link_in_service` | `internal/source/link` | S § 2.5 | T09 | M |
| E02-T11 | Project links: `getrev_projlink`, `findpackages_projlink`, `readpackage_projlink`, frozenlinks, `getnewvrev`, maintenance_release hiding | `internal/source/projlink` | S § 2.6 | T05 | M |
| E02-T12 | Copy/branch/linktobranch/copyproject/move with every option and origin write-backs | `internal/source/copy` | S § 2.10; request-accept e2e via Rails | T10, T11 | L |
| E02-T13 | Services client side: servicemark, run state, dispatch to queue, `handleservice` lazy start, `waitservice`, `mergeservice`, `getprojectservices`, `CommitServiceResult`, `servicemark_noservice` | `internal/source/service` | S § 6.1; F § C | T06, E13-T01 | L |
| E02-T14 | obs-scm/scmsync: constant servicemark, run ids, `CommitObsScm`, `sync_project`, scmsync index, `TriggerScmSync` | `internal/source/scmsync` | S § 6.2 | T13 | M |
| E02-T15 | `_multibuild` cache, `_product` expansion via tool sandbox, `_aggregate`/`_patchinfo`/`_channel`/`_constraints`/`_buildenv`/`_modulemd` extraction | `internal/source/special` | S § 2.8 | T06, E13-T03 | M |
| E02-T16 | `GetProjPack` streaming + delta; flags evaluation (`BSUtil::enabled`), errors, remotemap with shard ids, quota | `internal/source/projpack` | output equals Perl `getprojpack` (Storable→JSON dump) on corpus for all parameter combos the scheduler/publisher use | T05, T11, T15, E01-T08 | L |
| E02-T17 | `GetConfig` (`concatconfigs`): reverse-order concat, macro injection, `FromScratch`, `extraconfig`, `%vendor` | `internal/source/config` | equals Perl on corpus paths incl. remote | T16, E01-T07 | S |
| E02-T18 | Remote projects: `remoteprojid`, `maptoremote`, remote getrev with materialisation, meta/config fetch, batching, binary proxies for `getbinaries*` | `internal/source/remote` | S § 3.9 against a Perl peer in CI | T05 | L |
| E02-T19 | Access checks at the documented edges; `lastevents` suppression | `internal/source/access` | S § 5.2 403/404 parity | T08, T11 | S |
| E02-T20 | Diff/linkdiff/servicediff/blame with cache, `withissues` issue trackers | `internal/source/diff` | corpus diffs identical for plain/unified/xml views | T08, E13-T03 | M |
| E02-T21 | `lastevents` table writer/reader, `NewEvent`, configuration/issue_trackers store, `_repoid` | `internal/source/streams` | S § 4.2 semantics incl. `sync=lost`, pruning, heartbeat | T01 | M |
| E02-T22 | Source publishing export | `internal/source/sourcepublish` | P § 1.9 | T05 | S |
| E02-T23 | Signing-key metadata ops delegated to sign-svc; hierarchy lookup; auto-extend; `forceprojectkeys` | `internal/source/keys` | P § 2.2 | T03, E09-T01 | S |
