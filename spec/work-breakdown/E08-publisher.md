# E08 — publisher

Milestone M3. Inputs: `services/publisher.md`, analysis P § 1, § 3, § 4-6,
`04` § 1.1, `proto/publish.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E08-T01 | Orchestrator: queue consumer, per-prp lease, coalescing, retry classes/back-off, penalty multiplier, `publish_job` state, `REPO_PUBLISH_STATE` | `internal/publisher/orchestrator` | P § 1.1 semantics tested | E03, E06-T05 | M |
| E08-T02 | Plan builder: gather repo pools, projpack/config, repotype parsing, flags, `BSUrlmapper` (`get_extrep`, redirects, stage servers), archsync wait, disappearing repos | `internal/publisher/plan` | corpus plans equal | T01, E02-T16, E01-T07 | M |
| E08-T03 | Path classification (full table P § 1.3.6), archorder/last-wins, nocontentchange, splitdebug recursion, packagesubdir, slepool/sleimage renames, provenance pairing, deltas/dseq, containers decision, `.registry.txt` | `internal/publisher/classify` | published file lists equal corpus | T02 | L |
| E08-T04 | State hash, `unchanged` exit, publisher `:repoinfo` doc (incl. `publishid`, trackercache), versioned published tree writer + pointer flip + version GC | `internal/publisher/tree` | atomic switch test; status parity | T03, E03-T02 | M |
| E08-T05 | Generator sandbox runner (per repotype image, local object cache, tool invocation) | `internal/publisher/gen` | runs createrepo_c in kind | T04 | M |
| E08-T06 | rpm-md generator (all options), updateinfo/appdata/deltainfo/prestodelta/modulemd/SBOM/repoview/zchunk, `.repo` file, signing | `internal/publisher/gen/rpmmd` | metadata equals corpus (normalised) | T05, E09-T03 | L |
| E08-T07 | debian, arch, apk generators (Go-native) with signatures | `gen/debian`, `gen/arch`, `gen/apk`, `pkg/deb`… | corpus equality | T05, E09-T03 | M |
| E08-T08 | susetags, hdlist2 (external), vagrant, staticlinks (redirect entries), zyppservice, helm index, checksumsfile, virtbuilder, patterns ymp/rpm-md/comps, `directory.yast` | `gen/*` | corpus equality | T05 | M |
| E08-T09 | Container publishing: tag mapping, conflict resolution, repostate queries, compare, push via `pkg/regpush`, cosign/bundle/rekor, deletion safeguard, notary optional, manifestinfos, artifacthub, diffnotify callbacks | `internal/publisher/containers`, `pkg/regpush` | push to a local registry + external registry mock equals Perl behaviour | T04, E09-T04, E10-T02 | L |
| E08-T10 | Mirror sync (rsync exporter two-pass + trigger files, `script:`, rekor://), hooks (`publishedhook`/`unpublishedhook`/`publishprogram` adapters), markfileorigins | `internal/publisher/sync` | integration with rsync container | T04 | M |
| E08-T11 | Notifications (`REPO_PUBLISHED`, `PACKTRACK` payload, `sourcepublish`, diffnotify JSON), index update, delete/unpublish/createempty flows | `internal/publisher/post` | Rails ingests PACKTRACK e2e | T04, E12-T01, E14-T01 | M |
| E08-T12 | Publisher API: `Status`, listings, `PublishedPath`, `Unpublish`, versions/rollback | `internal/publisher/api` | `/published` parity | T04 | S |
