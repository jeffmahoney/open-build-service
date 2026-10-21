# 06 — Compatibility Edge API

Depends on: `02`, `03`, `services/*`. Route-level inventory:
`06a-legacy-route-catalog.md`. Frontend usage analysis:
`01-legacy-system-analysis.md` § F.

## 1. What the edge must be

The Rails application talks to exactly one backend endpoint, the source
server (`source_host:source_port`, default 5352), over plain HTTP with
XML bodies and no authentication. The source server internally proxies
`/build`, `/published`, `/worker`, `/registry`, `/search/published` to
repository servers. `osc` reaches the backend only through Rails.

`compat-edge` therefore exposes **one HTTP endpoint that behaves exactly like
`bs_srcserver` on port 5352** (plus the worker-facing routes that
`bs_repserver`/`bs_srcserver` serve to workers, for `osc build`'s
`_buildinfo`/`getbinaries` style access through Rails), and maps every route
to internal gRPC calls. It also serves `/published` and `/registry` to
the public web tier where an instance exposes them directly.

## 2. Wire-level obligations

| Aspect | Rule |
|---|---|
| Transport | HTTP/1.1 and HTTP/2; keep-alive; chunked responses for streams. |
| Query parsing | `BSDispatch` semantics: positional `$params`, typed validation (`BSVerify` rules in `02` § 1), unknown parameters → `400`, form-encoded POST bodies merged into the query, repeated keys as lists, bare keys as flags. |
| Errors | `<status code="N"><summary>…</summary><details>…</details></status>` with the exact legacy texts for the cases Rails/osc match on (`404` not found variants, `403 source access denied`, `400 … is locked`, `service in progress`, `conflict in file`, …). gRPC `LegacyError` detail → HTTP status + body. |
| XML | Rendered by `pkg/bsxml`, a Go implementation of every `XML::Structured` schema in `BSXML.pm` (103 schemas; element order, attribute order, and empty-element forms are preserved; see `01` § F.D for the DSL). Output must validate against `docs/api/api/*.rng|xsd` and be byte-compatible with the Perl output on the golden corpus. |
| Storable | `view=storable` (used by schedulers and the publisher's published-DB feeder) is not needed by Rails; the edge accepts the parameter and answers XML; internal services never use Storable. |
| Headers | honour `X-Frontend-Start` (queue-wait accounting), `x-request-id` (propagate as trace context), `X-Peer`/`trusted_peerip_header`, `Timeout`. |
| Long-poll | `/lastnotifications?block=1`, `/lastevents` (blocking default), `/build/<p>/_result?oldstate=`, `cmd=waitservice`, `_log` streaming (`nostream`, `start`, `end`, `last`, `view=entry`) keep their blocking semantics with heartbeats (100 events / 5 min for lastevents). |
| Authorization classes | `ipaccess`/`subjectdnaccess` are replaced by mTLS identity → class (`rw`, `worker`, `public`). The dispatch defaults (`!rw`, `!- GET`) are reproduced per route. |
| Offload | `X-Forward`/`X-Rewrite-URI`/`X-Accel-Redirect` flows keep working: the edge is the target of those fetches. |

## 3. Route mapping

Grouped by legacy namespace. `→` names the internal RPC. Full list with
parameters in `06a`.

### 3.1 `/source` (source-svc)

| Legacy | Internal |
|---|---|
| `GET /source[?deleted]` | `Source.ListProjects` |
| `GET /source/$p[?view=info|expand|deleted|noorigins]` | `ListPackages` / `GetSourceInfo` (batched) |
| `GET/PUT/DELETE /source/$p/_meta`, `_config`, `_pubkey`, `_keyinfo`, `_history` | `GetProject`/`PutProject`/…; `_keyinfo` → `Sign.ResolveKey` + `GetPublicKey`/`GetCertificate` |
| `POST /source/$p?cmd=createkey|extendkey|recreatesslcert|undelete|copy|move|freezelink|notifypackagechange` | corresponding RPCs; `notifypackagechange` publishes `obs.source.package.changed` |
| `GET /source/$p/$pkg[?rev|linkrev|expand|emptylink|deleted|view|extension|lastworking|withlinked|meta|product]` | `GetFileList`; `view=cpio` → streamed bundle; `view=info` → `GetSourceInfo`; `view=getmultibuild` |
| `GET/PUT/DELETE /source/$p/$pkg/$file` | `GetFile` (stream or presigned redirect when the client is not Rails), `PutFile`, `DeleteFile`; `view=blame` |
| `GET/PUT /source/$p/$pkg/_meta`, `_history` | `GetPackage`/`PutPackage`/`GetPackageHistory` |
| `POST /source/$p/$pkg?cmd=…` (31 commands) | `Commit`, `CommitFileList`, `CommitObsScm`, `Copy`, `Branch`, `LinkToBranch`, `CollectBuildEnv`, `Diff`/`LinkDiff`/`ServiceDiff`, `RunService`, `WaitService`, `MergeService`, `GetProjectServices`, `Lock`/`Unlock`, `Undelete`, `DeleteUploadRev`, `orderkiwirepos` |
| `POST /source?cmd=triggerscmsync` | `TriggerScmSync` |

### 3.2 `/build` (scheduler, artifact-svc, dispatcher, log-svc)

| Legacy | Internal |
|---|---|
| `GET /build`, `/build/$p`, `/build/$p/$r` | from project meta (`Source.GetProject`) |
| `GET /build/_workerstatus` | `Dispatcher.WorkerStatus` + scheduler/publisher/dispatcher daemon states (rendered as the legacy `workerstatus` XML incl. `<scheduler>`/`<partition>` elements) |
| `GET/PUT /build/_dispatchprios` | `Dispatcher.Get/SetDispatchPrios` |
| `GET /build/_result?scmrepository=` | `Source.ScmSyncIndex` + `Scheduler.Result` |
| `GET /build/$p/_result[?view|package|repository|arch|code|lastbuild|multibuild|locallink|oldstate]` | `Scheduler.Result` (package_status tables; `oldstate` long-poll on LISTEN/NOTIFY; `summary` computed by the edge) |
| `POST /build/$p?cmd=rebuild|abort|wipe|restartbuild|unpublish|sendsysrq|killbuild|suspendproject|resumeproject|wipepublishedlocked` | `Scheduler.Command` / `Dispatcher.Kill` / `Publisher.Unpublish` |
| `GET /build/$p/$r/_buildconfig` | `Source.GetConfig` |
| `GET /build/$p/$r/$a[?view=…]` | `Artifact.ListPackages`/`BinaryVersions`/… |
| `_builddepinfo`, `_jobhistory`, `_repository` views, `cmd=copy` (copybuild), `uploadbuild`, `getbinarylist`, `_buildinfo` (GET/POST), `_jobstatus`, `_log`, `_reason`, `_status`, `_history`, `_buildstats`, `getbinary` (`view=fileinfo|fileinfo_ext|publishedpath|names|cpio|binaryversions|…`), `putbinary`, `delbinary`, `_availablebinaries`, `cmd=missingdodresources`, `match:` | `Scheduler.Depends`, `Artifact.JobHistory`, `Artifact.Repository*`, `Artifact.CopyBuild`, `Artifact.UploadBuild`, `Artifact.BuildInfo` (the `_buildinfo` generator = BuildInfo.pm port), `Dispatcher.JobStatus`, `Log.Read`/`Follow`, `Artifact.Reason/Status/History/Stats`, `Artifact.GetBinary` (presigned when possible), `Artifact.PutBinary/DeleteBinary`, `Artifact.AvailableBinaries`, `Dod.MissingResources` |

### 3.3 `/published`, `/search`, `/registry`, `/slsa`, `/sigstore`

| Legacy | Internal |
|---|---|
| `GET /published[/$p[/$r[/$a[/$file[/$sub]]]]][?view=ymp|fileinfo|status|publishedpath|collection]` | `Publisher.ListPublished`/`GetPublishedFile` (redirect to published-gw), `Index.Collection`, `Publisher.Status`, `Publisher.PublishedPath` |
| `/search/project`, `/search/package[/id]`, `/search …` | `Index.SearchPackages` (XPath subset: the queries Rails issues, plus a generic XPath evaluator over the indexed fields for osc) |
| `/search/published/{binary,pattern,repoinfo}/{id,name}`, `POST /search/published cmd=updatedb` | `Index.SearchPublished`; `updatedb` is internal only |
| `/registry/*`, `/v2/*`, `/ownregistryrepo`, `/disownregistryrepo`, `/sigstore/*` | `registry-svc` (proxy/redirect) |
| `/slsa …` | `Artifact.SlsaAddRefs` / `GetSlsaMaterial` |

### 3.4 Streams, config, misc

| Legacy | Internal |
|---|---|
| `/lastevents`, `/public/lastevents`, `/lasteventsproxy` | `Source.LastEvents` (table-backed, filters `package/P/K`, `project/P`, `repository/P/R[/A]`, `start`, `sync=lost`); proxy variant kept for interconnect peers |
| `/lastnotifications`, `/notificationpayload/$key` | `Notifier.LastNotifications`, payload object fetch |
| `POST /notify/$type`, `/notify_plugins/$type`, `POST /event` | internal publishers; `notify_plugins` kept for API-triggered plugin fan-out via notifier |
| `PUT/GET /configuration`, `/issue_trackers`, `cmd=issues` | `Source.Configuration*`, `IssueTrackers*`, `ExtractIssues` |
| `/service` | `ServiceRunner.ListServices` |
| `/worker/$id`, `POST /worker cmd=checkconstraints`, `/jobs/$arch` | `Dispatcher.WorkerInfo`, `CheckConstraints`, `ListJobs` |
| `/relsync` | `Scheduler.Relsync` |
| `/getprojpack` (`nopackages&withrepos&expandedrepos` only from Rails) | `Source.GetProjPack` |
| `/request/*`, `/ajaxstatus`, `/serverstatus` | `/request` returns the legacy 404-ish behaviour; status routes report edge health |

### 3.5 Worker-facing routes (served on a second listener, class `worker`)

`/getsources`, `/getconfig`, `/getbinaries`, `/getbinaryversions`,
`/getobsgendiffdata`, `/getmodulemd`, `/getsslcert`, `/getsignkey`,
`/getpubkey`, `/build/$p/$r/$a/_repository?view=cache|solvstate|names|cpio`,
`/build/…/_preinstallimage`, `/registry/...` for base containers. The Go
worker-agent uses gRPC + presigned URLs instead, but these remain for
`osc build` (through Rails) and interconnect peers (`view=solvstate`,
`view=gbininfocode`, `view=binaryversionscode`, `cmd=missingdodresources`).

## 4. Interconnect (remote OBS instances)

Everything an OBS instance fetches from another goes through the remote's
`/public/*` Rails routes, which proxy to that backend with `interconnect=1`.
Our edge must therefore serve the `/public`-reachable subset identically
(meta without `scmsync`, package meta without roles/flags, `lastevents`,
`_repository?view=solvstate|cache`, `getbinaries` cpio, `view=info`,
`_result`, published and binary listings), and `source-svc`'s remote client
must consume the same from peers running either implementation.

## 5. Caching at the edge

Short-TTL (5 s) response cache for read-heavy routes that Rails polls
(`_result`, `_workerstatus`, `view=status` build ids, `_jobhistory`),
invalidated by `obs.build.*` events. ETags derived from row versions.

## 6. Parity testing

For every route in `06a` the golden corpus (`10-testing-strategy.md`)
contains request/response pairs captured from the Perl backend; the edge
test suite replays them against the Go stack and diffs canonicalised XML
(whitespace-insensitive, attribute-order-insensitive unless the schema is
order-sensitive, which `XML::Structured` output is: element order is
compared strictly).

## 7. Tasks

`work-breakdown/epics.md` E1 (bsxml + verify), E11 (edge routes).
