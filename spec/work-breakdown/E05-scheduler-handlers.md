# E05 — Scheduler handlers

Milestone M2. Inputs: analysis B scheduler § 4.4 and the cited
`BSSched/BuildJob/*.pm`; `services/scheduler.md` § 5.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E05-T01 | Handler interface + registry; `metacheck` generic; `fakejobfinished` path | `internal/scheduler/handler` | unit tests | E04-T06 | S |
| E05-T02 | PreInstallImage + Image (mkosi/appimage/livebuild/simpleimage) | `handler/preinstallimage`, `handler/image` | corpus | T01 | M |
| E05-T03 | KiwiImage + Docker/fissile (expandkiwipath, container pools, annotation path mapping, `append_info_path`, `filterbasecontainerpkgs`, unordered repos) | `handler/kiwiimage`, `handler/docker` | corpus kiwi/docker projects | T01 | L |
| E05-T04 | KiwiProduct + ProductCompose (build arch selection, waiting markers, gbininfo walks, `kiwiproductcheck`, versioned deps, modulemd/updateinfo inclusion) | `handler/kiwiproduct`, `handler/productcompose` | corpus products | T01, E06-T06 | L |
| E05-T05 | Aggregate (in-process build, remote cpio, containers/helm/provenance, resign via signer, `_repository` sources) | `handler/aggregate` | corpus aggregates | T01, E09-T02 | M |
| E05-T06 | Channel | `handler/channel` | corpus maintenance | T01 | M |
| E05-T07 | Patchinfo incl. ptf job, updateinfo generation, issue trackers | `handler/patchinfo` | corpus maintenance | T01 | M |
| E05-T08 | DeltaRpm job orchestration (`makedeltas`, batching limits) | `handler/deltarpm` | corpus | T01, E04-T12 | S |
| E05-T09 | Import (exportfilter) + Upload | `handler/import`, `handler/upload` | corpus baselibs | T01, E06-T04 | S |
| E05-T10 | Reproduciblecheck (two-job replay/compare) | `handler/reproducible` | corpus | T01 | M |
| E05-T11 | BuildEnv (`_buildinfo` with `_buildenv`) + Unknown | `handler/buildenv` | buildinfo parity | T01 | S |
