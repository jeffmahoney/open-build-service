# E11 — compat-edge

Milestones M1 (`/source`, streams), M2 (`/build`, worker routes),
M3 (`/published`, `/search/published`, `/registry`). Inputs: `06`, `06a`,
`services/compat-edge.md`, analysis F § A, F § D.1-D.6.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E11-T01 | Server skeleton: listeners/classes, route table from DSL, form merge, limits/timeouts, error rendering, headers (`X-Frontend-Start`, request id), `/serverstatus`, `/ajaxstatus` | `cmd/compat-edge`, `internal/edge` | E01-T06 routes mounted; health | E01-T02/T05/T06, E03-T06 | M |
| E11-T02 | `/source` project and package routes (meta, config, keys, lists, history, files, commands) | `internal/edge/handlers/source` | parity for all `/source` routes in `06a` | T01, E02 | L |
| E11-T03 | Streams and long-poll: `/lastevents`, `/lasteventsproxy`, `/lastnotifications`, `/notificationpayload`, `cmd=waitservice`, `view=cpio`, `/getsources` | `internal/edge/longpoll`, `stream` | heartbeat/timing parity; Rails clock job runs against it | T01, E02-T21, E12-T01 | M |
| E11-T04 | `/build` routes: lists, `_result` (+summary, oldstate), commands, `_buildconfig`, `_builddepinfo`, `_jobhistory`, `_buildinfo`, `_jobstatus`, `_log`, `_reason`, `_status`, `_history`, `_buildstats`, binaries and views, `_availablebinaries`, `_workerstatus`, `_dispatchprios`, `scmrepository` results | `internal/edge/handlers/build` | parity; `osc build`, `osc results`, `osc remotebuildlog` e2e | T01, E04-T16, E06-T06/T09/T10, E07-T08, E07-T09 | L |
| E11-T05 | Worker/interconnect listener routes (`getconfig`, `getbinaries`, `getbinaryversions`, `getobsgendiffdata`, `getmodulemd`, `getsslcert`, `getsignkey`, `getpubkey`, `_repository?view=solvstate|cache`, `cmd=missingdodresources`, `/jobs/<arch>`, `/worker`) | `internal/edge/handlers/worker` | Perl peer interconnect tests | T01, E06, E07 | M |
| E11-T06 | `/published` + published-gw (pointer resolution, listings, `view=ymp|fileinfo|status|publishedpath|collection`, redirects, `directory.yast`) | `internal/edge/handlers/published`, `internal/edge/published-gw` | zypper/apt/pacman/apk clients install from the gateway; parity | T01, E08-T12, E14 | M |
| E11-T07 | `/search`, `/search/project|package`, `/search/published/*` | `internal/edge/handlers/search` | Rails search callers pass | T01, E14 | S |
| E11-T08 | `/registry`, `/v2`, `/sigstore`, `/slsa`, own/disown | `internal/edge/handlers/registry` | proxy/redirect tests | T01, E10 | S |
| E11-T09 | Config/misc: `/configuration`, `/issue_trackers`, `cmd=issues`, `/service`, `/relsync`, `/event`, `/notify*`, `/request/*`, `/getprojpack` | `internal/edge/handlers/misc` | parity | T01 | S |
| E11-T10 | Edge response cache + ETags + event invalidation | `internal/edge/cache` | hit-rate metric; no stale `_result` beyond 5 s | T04 | S |
