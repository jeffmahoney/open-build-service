# OBS frontend↔backend contract, public API inventory, source-service server, and backend infrastructure modules

Engineering report for the Go re-architecture of the Open Build Service backend. Repository: `/home/jeffm/src/open-build-service` (master, db93ae665b). All code citations are `file:line` ranges read from this checkout.

Contents:
- **A.** Frontend → backend contract (`Backend::Connection`, `Backend::Api::*`, `pass_to_backend`), complete route usage table, notification/event bridge, write-through data-authority table, polling jobs, trust model.
- **B.** Public OBS API / WebUI feature inventory with DB-vs-backend placement.
- **C.** Source service server (`bs_service`), `bs_servicedispatch`, `BSSrcServer::Service`, container wrappers, `obs-service-*` plugin contract, frontend `Service` model.
- **D.** Infra modules: D.1–D.11 BSServer/BSStdServer/BSRPC/BSHTTP/BSEvents/BSWatcher/BSHandoff/BSUtil/BSFileDB/BSFileQueue/BSRedis/BSBearer/BSSSL/BSPGP/BSAirBrake/BSNotify; then (separate parts) XML::Structured + every BSXML schema; BSVerify rules + BSDispatch syntax + BSConfiguration/BSConfig variables; bs_admin and admin tools, bs_setup, `dist/` packaging and on-disk layout.

---

# A. The contract between the Rails frontend (src/api) and the Perl backend

All paths below are relative to `/home/jeffm/src/open-build-service` unless absolute.
Route tables referenced as "bs_srcserver route table" are the extracted dispatch tables in
`spec/06a-legacy-route-catalog.md (bs_srcserver.txt` (179 lines),
`bs_repserver.txt` (88), `bs_service.txt` (8), `bs_getbinariesproxy.txt` (4).

## A.1 Topology: who talks to whom

```
  osc / browser / CI
        |  HTTPS (public API + WebUI)             [auth: Basic, session cookie, proxy headers, tokens]
        v
  Rails frontend (src/api) ------ MySQL/MariaDB (users, groups, roles, requests, reviews, attributes, flags, ...)
        |  plain HTTP, ONE host:port = CONFIG['source_host']:CONFIG['source_port'] (default localhost:5352)
        |  no authentication at all (backend trusts the network; see A.9)
        v
  bs_srcserver (Perl, :5352)  <--- the ONLY backend daemon the frontend talks to
        |  BSRPC to $BSConfig::reposerver(s) (:5252), $BSConfig::serviceserver (:5152), remote OBS instances
        v
  bs_repserver / bs_sched / bs_dispatch / bs_publish / bs_signer / bs_service / bs_servicedispatch / bs_warden / bs_dodup ...
        ^
        |  backend -> frontend direction is NOT a push: bs_srcserver appends to $bsdir/events/lastnotifications,
        |  the frontend POLLS GET /lastnotifications?start=N&block=1 every 17 s (clock.rb) and turns each
        |  notification into an Event::* row (see A.6).
```

Key facts:

* The frontend never contacts bs_repserver, bs_sched, bs_publish or bs_service directly. `grep -rn "reposerver\|5252" src/api/config` finds nothing; the only backend address is `source_host`/`source_port` (src/api/config/options.yml.example:19-26). Requests that logically belong to the repo server (`/build/...`, `/published/...`, `/worker/...`, `/getbinaries`...) are proxied internally by bs_srcserver via BSRPC/BSWatcher to the repo server(s) (e.g. `getworkercap` in src/backend/bs_srcserver:7590-7600 forwards `/worker/$workerid` to `$BSConfig::masterdispatcher || $BSConfig::reposerver`).
* Protocol: HTTP/1.1, XML bodies (XML::Structured output), occasionally `text/plain` (logs, pubkey), `application/json` (`/lastnotifications?view=json`, `/notificationpayload/*`), `application/x-cpio` (worker/service traffic only, not used by the frontend). Errors come back as `<status code="N"><summary>..</summary><details>..</details></status>` (BSStdServer::errreply, src/backend/BSStdServer.pm:65-75) which `Backend::Error#xml` parses (src/api/app/lib/backend/error.rb:490-512 of the concatenated read; file is 23 lines).
* Transport options for GET passthrough: the Rails process can either volley (download to tempfile, `send_file`) or emit `X-Forward` / `X-Rewrite-URI` / `X-Accel-Redirect` headers so Apache mod_xforward, lighttpd or nginx fetch the body from the backend directly (src/api/app/controllers/concerns/backend_proxy.rb:75-111; config keys `use_xforward`, `x_rewrite_host`, `use_nginx_redirect`).

## A.2 `Backend::Connection` — the low-level HTTP client (src/api/app/lib/backend/connection.rb)

| Aspect | Behaviour | Lines |
|---|---|---|
| Host/port/TLS | `CONFIG['source_host']`, `CONFIG['source_port']`, `use_ssl = CONFIG['source_protocol']=='https'`, `verify_mode` VERIFY_NONE iff `source_protocol_ssl_verification == false` | 4-23 |
| Headers added to every request | `X-Frontend-Start: <unix ts>` (used by backend `client_time_header` for queue-wait accounting, BSStdServer.pm:120-123); `Timeout` pseudo-header is stripped and used as `read_timeout`; `ConnectionHelper#calculate_headers` adds `x-request-id: Current.request_id` (connection_helper.rb:228-231), which the backend logs and propagates as `X-Request-ID` to its own RPCs (BSStdServer.pm:124-135) | 37-38, 67-69, 106-107 |
| Timeouts | GET/DELETE/PUT: 1000 s read timeout; POST: 100000 s ("POST requests can be quite complicate and take some time") | 38, 69, 121-126 |
| Body | `put_or_post` streams `body_stream` if data responds to `read` (Tempfile), else sets `body`; POST default Content-Type `application/octet-stream` | 58-63, 113-118 |
| Connection reuse | none: `Net::HTTP.start` per request (new TCP connection every call; the backend forks per connection, BSServer.pm:328-600) | 41, 70, 120 |
| Error mapping | 2xx/3xx -> response; 404 -> `Backend::NotFoundError`; everything else -> `Backend::Error` with body (XML `<status>`) as message; socket errors on PUT/POST re-raised as `Timeout::Error` | 129-131, 142-153 |
| Instrumentation | `Backend::Instrumentation` -> InfluxDB `connection.obs_backend` tagged with `backend_location` = `Module#method` captured by `Backend::RememberLocation` (singleton_method_added hook storing names in `Thread.current`) | instrumentation.rb, remember_location.rb |
| Query building | `build_query_from_hash(hash, key_list)` (connection.rb:81-99) is what every `pass_to_backend(path)` controller uses to whitelist which params are forwarded; nil values become bare flags, arrays repeat the key |

`Backend::ConnectionHelper` (connection_helper.rb:158-289) is the DSL used by all `Backend::Api::*` modules: endpoint is a string or `['/source/:project/:package', prj, pkg]` template (placeholders substituted with `ERB::Util.url_encode`), options `params:`, `defaults:`, `rename:`, `accepted:` (whitelist), `expand:` (repeat key for arrays without `[]`), `data:`, `headers:`. The URL is normalised with `Addressable::URI.parse(...).normalize`. All four verbs return the body as a UTF-8 `String`.

`Backend::File` (file.rb:290-419) is an ActiveModel wrapper for "a file that lives in the backend" (streams GET into a Tempfile, PUT from Tempfile or string, DELETE); subclasses `PackageFile` (`/source/:project/:package/:name`), `ProjectFile` (`/source/:project/_project/:name`), and the specialisations `ProjectConfigFile`, `ProjectMetaFile`, `PackageMetaFile`, `PackageServiceErrorFile`, `ProjectServiceFile` (src/api/app/models/{package_file,project_file,...}.rb).

`Backend::Test` (test.rb) starts a real backend (`script/start_test_backend`) for the test suite and forces `CONFIG['global_write_through'] = true`.

## A.3 `pass_to_backend` — verbatim proxying (src/api/app/controllers/concerns/backend_proxy.rb)

```
pass_to_backend(path = nil)        # path defaults to request.path_info + "?" + query (+ raw_post if form data)  [11-40, 115-125]
  GET/HEAD  -> forward_from_backend?(path) (X-Forward / X-Rewrite / X-Accel-Redirect) else volley_backend_path(path)
  POST      -> form data: Backend::Connection.post(path, '', x-www-form-urlencoded)   (query already carries the form)
               otherwise: body copied to Tempfile (download_request) and POSTed
  PUT       -> body to Tempfile then Backend::Connection.put
  DELETE    -> Backend::Connection.delete
  response body is send_data'd back with the backend's content-type; return value = body (used by commitfilelist)
volley_backend_path(path)          # raw Net::HTTP GET, read_timeout 1000, streamed into Tempfile, send_file  [46-73]
```

Note `@skip_validation = true` is set in the X-forward branches so `ApplicationController#validate_xml_response` (application_controller.rb:31,189) does not try to validate a body it never saw.

### Controllers that call `pass_to_backend`/`volley_backend_path` (frontend does auth/permission check, backend does the work)

| Frontend route (config/routes/api.rb) | Controller#action | Permission check done in Rails | Backend path forwarded (whitelisted params) |
|---|---|---|---|
| `GET /build/:project` ; `GET /build/_dispatchprios` | BuildController#project_index (build_controller.rb:28-89) | `Project.get_by_name` (visibility) ; POST requires `cmd` in `wipe restartbuild killbuild abortbuild rebuild unpublish sendsysrq`, project must be local, user needs project_change or per-package package_change; PUT requires admin | same path+query (`/build/$project cmd:...` → `docommand`; PUT → `putdispatchprios`) |
| `GET /build/:project/:repository`, `GET /build/:project/:repository/:arch`, `GET /build/:project/:repository/_buildconfig`, `GET|POST /build/:p/:r/:a/:package`, `.../_status`, `.../_history` | BuildController#index (6-26) | package/project visibility via `Package.get_by_project_and_name(use_source: false, follow_multibuild: true)`; POST (binary upload) admin only | same path+query |
| `GET|POST /build/:p/:r/:a/:package/_buildinfo` | BuildController#buildinfo (91-113) | visibility; adds `striphdrmd5` for osc < 0.175 | `/build/$p/$r/$a/$pkg/_buildinfo?<query>` |
| `GET|POST /build/:p/:r/:a/_builddepinfo` | BuildController#builddepinfo (116-124) | project visibility | same |
| `GET /build/:p/:r/:a/:pkg/_log` | BuildController#logfile (126-138) | visibility + `binarydownload` flag / `can_download_binaries?` | same |
| `GET /build/:project/_result` | BuildController#result (140-148) | project visibility; `?lastsuccess` is answered from Rails (`PackageBuildStatus`) | same |
| `GET /build/_result` | BuildController#scmresult (150-153) | `X-SCM_BRIDGE_COOKIE` must equal `CONFIG['scm_bridge_cookie']` (application_controller.rb:80-84) | `/build/_result?scmrepository=&scmbranch=...` |
| `GET /build/:p/:r/:a/:pkg/:filename` | Build::FileController#show (build/file_controller.rb:6-12) | `binarydownload` flag; `_buildenv`/`_statistics` are fetched via `Binaries.file` and rendered | `request.path_info?query` |
| `PUT .../_repository/:filename` | Build::FileController#update | admin only | same |
| `DELETE .../_repository/:filename` | Build::FileController#destroy | `project_change?`; only `_repository` package allowed | same |
| `GET /published/:project(/:repo(/:arch(/:binary)))` | PublishedController#index | `binarydownload` flag | same |
| `GET /service` | ServiceController#index | none | `/service` (list of installed source services, proxied by srcserver to service server) |
| `GET /worker/:worker`, `GET /worker/capability/:worker` | Worker::CapabilityController#show | login | `/worker/$workerid` (srcserver forwards to reposerver) |
| `GET /source?deleted=1` | SourceProjectController#index (source_project_controller.rb:150-166) | admin | `/source?deleted=1` (otherwise the project list comes from the DB!) |
| `GET /source/:project` (`deleted`, remote project, or `view=info`) | SourceProjectController#show (169-213) | deleted → `validate_read_access_of_deleted_project`; remote → `Project.remote_project?`; other views (`productlist`, `verboseproductlist`, `issues`, `versions`) and the plain package list are answered from the DB | same path+query |
| `GET /source/:project/_meta` (remote project only) | SourceProjectMetaController#show (20-31) | — | same; local projects render `prj.to_axml` from DB |
| `GET /source/:project/_config` | SourceProjectConfigController#show (245-259) | `Project.get_by_name` | `ProjectConfigFile#full_path` = `/source/$p/_project/_config?rev=` (X-forward or `Backend::File#content`) |
| `PUT /source/:project/_config` | SourceProjectConfigController#update (262-278) | `authorize @project, :update?` | `PUT /source/$p/_project/_config?user=&comment=` via `Backend::File#save` |
| `GET /source/:project/:package` | SourcePackageController#show (85-115) | `Package.get_by_project_and_name` (sourceaccess), deleted handling; `view=issues|versions` answered from DB | path + `rev linkrev emptylink expand view extension lastworking withlinked meta deleted parse arch repository product nofilename` |
| `GET /source/:project/:package/:filename`, `GET /source/:project/_project/:filename` | SourcePackageController#show_file (145-170) | sourceaccess; rewrites project/package to the link target for project links | `Package.source_path(...)` + `rev meta deleted limit expand view` |
| `PUT /source/:project/:package/:filename` | SourcePackageController#update_file (173-206) | `package_change?`; refuses scmsync'ed; `_attribute`/`_staging_workflow` for `_project` rejected; `Package.verify_file!` validates `_link`, `_service`, `_patchinfo`, `_channel`, `_constraints`, `_aggregate`...; after success `sources_changed` (sync for special files) | path + `user comment rev keeplink meta` |
| `GET /source/:project/:package/_meta` (`deleted`, `meta`, `rev`, `view` or remote) | SourcePackageMetaController#show (178-194) | sourceaccess / deleted check | path + `deleted meta rev view`; otherwise `pack.to_axml` from DB |
| `POST /source/:project/:package?cmd=` ... `getprojectservices`, `collectbuildenv`, `commit`, `commitfilelist`, `diff`, `linkdiff`, `servicediff`, `copy`, `waitservice`, `mergeservice`, `runservice`, `deleteuploadrev` | SourcePackageCommandController (source_package_command_controller.rb) | pundit `authorize @package, :update?` for mutating cmds; `diff`/`linkdiff`/`servicediff`/`waitservice`/`getprojectservices` read-only; `copy` creates the DB package first from the source `_meta` | path + per-command whitelist (e.g. diff: `cmd rev orev oproject opackage expand linkrev olinkrev unified missingok meta file filelimit tarlimit view withissues onlyissues cacheonly nodiff`; copy: `cmd rev user comment oproject opackage orev expand keeplink repairlink linkrev olinkrev requestid withvrev noservice dontupdatesource`); after `commit`/`commitfilelist`/`copy`/`mergeservice`/`runservice` → `@package.sources_changed` |
| `POST /source?cmd=orderkiwirepos` | SourceCommandController#global_command_orderkiwirepos | login | same |
| `POST /public/source?cmd=triggerscmsync` | SourceCommandController#global_command_triggerscmsync | scm bridge cookie | `/source?cmd=triggerscmsync&scmrepository=&scmbranch=&isdefaultbranch=` |
| `GET /public/build/...`, `GET /public/source/:project`, `/_meta`, `/_config`, `/_keyinfo`, `/_pubkey`, `/:package`, `/:package/_meta`, `/:package/:filename` | PublicController (public_controller.rb) | anonymous `_nobody_` user; visibility via `Project.get_by_name` / `check_package_access` (30-min cache) | `/public` prefix stripped; `_meta` requests get `?interconnect=1` (backend then hides `remoteurl`); `/public/source/:project` defaults to `?expand=1&noorigins=1` for OBS<2.4 compatibility; `view=info` forces `nofilename=1` |
| `GET /lastevents`, `POST /lastevents`, `GET|POST /public/lastevents` | SourceController#lastevents(_public) (source_controller.rb:10-22) | none for public | `/lastevents?...` / `/public/lastevents?...` (long-poll, `block`) — this is what *remote OBS instances* (interconnect) poll |
| `GET|POST /search`, `/search/published/binary/id`, `/search/published/repoinfo/id`, `/search/published/pattern/id` | `search#pass_to_backend` (the concern method is the action) | login | same path+query → backend's published-DB XPath search (`$extrepodb`) |

## A.4 `Backend::Api::*` — the typed client (src/api/app/lib/backend/api/**)

Every method, the backend route it hits, and *who calls it* (callers enumerated with `grep -rno 'Backend::Api::[A-Za-z:]+\.[a-z_?!]+' app lib`). "No callers" means the method exists but nothing in `app/` or `lib/` uses it.

### Backend::Api::Server (api/server.rb)
| Method | Backend route | Callers / purpose |
|---|---|---|
| `notification_payload(id)` | `GET /notificationpayload/:key` | `UpdateReleasedBinariesJob` (jobs/update_released_binaries_job.rb:16) — fetch JSON list of released binaries attached to a `PACKTRACK` notification |
| `delete_notification_payload(id)` | `DELETE /notificationpayload/:key` | same job (:22) after import |
| `write_configuration(xml)` | `PUT /configuration` | `Configuration#write_to_backend` (models/configuration.rb:80-85) via `ConfigurationWriteToBackendJob` (after_save) — pushes the frontend `<configuration>` (obs name, download_url, registration policy, ...) to the backend; bs_srcserver stores it at `$bsdir/configuration.xml` and re-PUTs it to every repo server and the service server (bs_srcserver:7260-7285) |
| `last_notifications(start)` | `GET /lastnotifications?start=N&block=1` | `UpdateNotificationEvents#perform` (models/update_notification_events.rb:41) — the event bridge, see A.6 |
| `notify_plugin(id, payload)` | `POST /notify_plugins/:plugin` (JSON body) | **no callers in app/** (dead; the backend side `BSSrcServer::Notify::notify_plugins` loads `plugins/notify_{hermes,rabbitmq}.pm`) |
| `root` | `GET /` | **no callers** (hello) |

### Backend::Api::IssueTrackers (api/issue_trackers.rb)
| `write_list(xml)` | `PUT /issue_trackers` | `IssueTrackerWriteToBackendJob` (after_save of IssueTracker, models/issue_tracker.rb:11,114) — backend uses the tracker regexes for `cmd=diff&withissues` |
| `list` | `GET /issue_trackers` | no callers |
| `parse(text)` | `POST /issue_trackers?cmd=issues` | `RequestController` — let the backend extract issue references from free text |

### Backend::Api::Request — `GET /request`, `GET /request/:id`, `GET /request/_lastid`: **no callers** (bs_srcserver:6445-6460 says "just required for migration into api"; requests have been DB-only since OBS 2.4).

### Backend::Api::Search (api/search.rb)
| `binary(projects, name)` | `GET /published?view=collection&name=&project=..` | `OwnerSearch::Assignee` (owner search by binary name) |
| `published_binaries_for_package(p,pkg)` | `GET /search/published/binary/id?match=` | `PublicController#binary_packages` (ymp / 1-click install page) |
| `packages_with_link(names)` | `GET /search/package/id?match=(@name=..)` | `UpdatePackageMetaJob#scan_links` (daily "refresh dirties") |
| `linking_packages(pkg,prj)` | `POST /search/package/id?match=(linkinfo/@package=..)` | `Package#linking_packages`-style lookups (package.rb) |
| `incident_packages(...)` | `POST /search/package/id` | `BranchPackage::LookupIncidentPackage` |
| `product_ids(project)` | `GET /search/package/id?match=@project=.. and starts-with(@name,'_product:')` | `Project#update_product_list`-ish (project.rb:1114) |
| `packages_for_project(project)` | `GET /search/package?match=@project=` | `Project.undelete` import (project.rb:163), `ProjectDoProjectCopyJob` |
| `projects(list)` | `GET /search/project?match=` | no callers |

Note: all other `/search/*` API routes (`project`, `package`, `person`, `request`, `issue`, `channel`, `released/binary`, `owner`, `missing_owner`, `repository/id`, `attribute`) are served from the **database** by `lib/xpath_engine.rb` (tables: attribs, packages, projects, users, repositories, issues, requests, channels; xpath_engine.rb:8-16) — the backend's XPath search is only used for *published* binaries.

### Backend::Api::Worker (api/worker.rb)
| `status` | `GET /build/_workerstatus` | clock.rb:25-29 every 30 s → `Rails.cache['workerstatus']` + `StatusHistory` rows; `WorkerStatus.hidden` (hides projects the user cannot see); `Worker::StatusController`; `WorkerMeasurementsJob` (metrics to RabbitMQ) |
| `check_constraints(params, xml)` | `POST /worker?cmd=checkconstraints&project&package&repository&arch` | `Worker::CommandController#run` |

### Backend::Api::Published (api/published.rb) — all `GET /published/:project/:repository`
| `download_url_for_repository` (`view=publishedpath`, cached) | `Repository#download_url` (repository.rb:297) |
| `build_id` (`view=status`, cached) | `Repository#build_id` (:313) |
| `published_repository_exist?` | `DownloadRepositoryLinkComponent` |

### Backend::Api::Build::Project (api/build/project.rb) — all `POST /build/:project?cmd=`
| `suspend_scheduler` (`cmd=suspendproject`) / `resume_scheduler` (`cmd=resumeproject`) | `Project#suspend_scheduler/resume_scheduler` (project.rb:1221-1227) — wrap multi-step meta changes atomically for the scheduler |
| `rebuild` (`cmd=rebuild&repository&arch`) | `Token::Rebuild` (project-wide) |
| `wipe_binaries` (`cmd=wipe&repository&arch&package`) | `Project` destroy of repositories, `Webui::Packages::BinariesController#destroy` |
| `abort_build` (`cmd=abortbuild`) | `Webui::Packages::TriggerController#abort_build` |
| `wipe_published_locked` (`cmd=wipepublishedlocked`) | `PublishedRepositoriesCleanupJob` (maintenance release repo lifetime) |
| `binarylist(project, package:)` | `GET /build/:project/_result?view=binarylist&package=` | `Webui::PatchinfoController` |

### Backend::Api::Build::Repository — `build_id` = `GET /build/:p/:r/:a?view=status` (cached) ← `RepositoryArchitecture#build_id`.

### Backend::Api::BuildResults::Binaries (api/build_results/binaries.rb)
| `files(p,r,a,pkg)` | `GET /build/:p/:r/:a/:pkg` | `BsRequestAction#check_maintenance_release` (:354), Webui binaries list |
| `history` | `GET .../:pkg/_history` | `BsRequestAction` (:362) |
| `copy(...)` | `POST /build/:p/:r/:a/:pkg?cmd=copy&oproject&opackage&orepository&setupdateinfoid&resign&setrelease&multibuild` | `MaintenanceHelper#copy_single_binary` (helpers/maintenance_helper.rb:236-247) — binary release (maintenance release / `cmd=release` / `Token::Release`) |
| `job_history(p,r,a)` (`code=lastfailures`) | `GET /build/:p/:r/:a/_jobhistory` | `ProjectStatus::Calculator` |
| `file(p,r,a,pkg,file)` | `GET /build/:p/:r/:a/:pkg/:file` | `Build::FileController#show` (`_buildenv`, `_statistics`), `RpmlintLogExtractor` (`_log`) |
| `publishedpath` / `download_url_for_file` (`view=publishedpath`) | `Webui::Packages::BinariesController` |
| `rpmlint_log` | `GET .../:pkg/rpmlint.log` | `RpmlintLogExtractor` |
| `fileinfo` / `fileinfo_ext` (`view=fileinfo[_ext]&withfilelist`) | `Webui::Packages::BinariesController` (binary details page) |
| `builddepinfo(p,r,a,pkg,view)` | `GET /build/:p/:r/:a/_builddepinfo?package&view` | `Package`, `Repository#...` (repository.rb:161, package '-'), `Webui::Projects::RebuildTimesController` |
| `build_dependency_info` (`view=pkgnames`) | same route | `PackageBuildStatus` (request "build results" check) |
| `available_in_project` / `available_in_repositories` | `GET /build/:project/_availablebinaries?url=&path=` | `Kiwi::Image` (kiwi editor package autocomplete) |

### Backend::Api::BuildResults::JobHistory
| `for_project` | `GET /build/:project/_jobhistory?limit&endtime_start&endtime_end&code&package` | `Project#jobhistory` (project.rb:429) |
| `for_package` / `for_repository_and_arch` | `GET /build/:p/:r/:a/_jobhistory?package&code&limit&endtime_*` | `Package#jobhistory` (package.rb:923), `Webui::Projects::RebuildTimesController` |

### Backend::Api::BuildResults::Status
| `log_chunk` (`_log?nostream=1&start&end`) / `build_log_size` (`_log?view=entry`) / `job_status` (`_jobstatus`) / `build_result` (`/_result?view=status&package&arch&repository`) | `BuildLogSupport` mixin (webui live build log) |
| `build_reason` (`_reason`) | `Package#build_reason` |
| `statistics` (`_statistics`) | `LocalBuildStatistic::ForPackage` |
| `result_swiss_knife(project, params)` (`/_result?view=&package=&repository=&arch=&code=&lastbuild=&multibuild=&locallink=`) | `Buildresult.find_hashed` (models/buildresult.rb:71-73) — used by nearly every webui build-status view, badge, request build results |
| `failed_results` (`code=failed,broken,unresolvable`) | `StagingProject#set_buildinfo` |
| `failed_results_summary` (`view=summary`) | `Project` (:1245) |
| `build_problems` (`view=status&code=failed..`) | `Project` (:453) |
| `version_releases` (`view=versrel`) | `Project` (:1353) |

### Backend::Api::Sources::File — `/source/:project/:package/:filename`
| `content(..., deleted expand meta rev view limit)` GET | `Package#source_file` (package.rb:503), `createSpecFileTemplate` |
| `blame` (`view=blame`) | `Webui::Packages::FilesController#blame` |
| `write(..., content, params)` PUT | `Package#save_file` (:1228), `Service#save` (`_service`), `MaintenanceHelper#import_channel` (`_channel`), `createSpecFileTemplate`, dev test data |
| `delete(..., user comment meta rev keeplink)` DELETE | `SourcePackageController#delete_file`, `Package#delete_file`, `Service#save` |

### Backend::Api::Sources::Package — `/source/:project/:package[...]`
| Method | Route | Callers |
|---|---|---|
| `attributes(p, pkg or '_project', rev meta view)` | `GET .../_attribute` | `SourceAttributeController#show` — **reads attributes from the backend copy, not the DB** |
| `write_attributes` | `PUT .../_attribute?user=` | `HasAttributes#write_attributes` (models/concerns/has_attributes.rb:5-16) — DB is authoritative, backend gets a copy for the scheduler (e.g. `OBS:...` attributes used in prjconf macros/`_constraints`) |
| `files(p,pkg,opts)` (`rev linkrev expand meta deleted view extension lastworking withlinked parse ...`) | `GET /source/:p/:pkg` | `Directory.hashed` → `Package#dir_hash` (used for kind detection, `update_backendinfo`, `linkinfo`, `rev`), `BranchPackage`, `BsRequestAction`, `BsRequestActionMaintenanceRelease`, `BsRequestPermissionCheck`, `BsRequestAction::Differ::QueryBuilder`, `Webui::PackageController`, `SyncLocalPackageVersionJob` (`view=info&parse=1`) |
| `products` (`view=products`) | same | `Package#update_product_list` |
| `revisions(... meta rev deleted limit startbefore)` | `GET .../_history` | `Package#commit`, `Webui::PackageController#revisions` |
| `meta(p,pkg, deleted meta rev view)` | `GET .../_meta` | `undelete`, `copy`, `BsRequestActionSubmit#execute_accept`, `ReadAccessOfDeleted`, consistency checker, `ProjectDoProjectCopyJob` |
| `write_meta(p,pkg,xml, user comment requestid)` | `PUT .../_meta` | `Package#write_to_backend` (after_save; package.rb:800-816) |
| `service` | `GET .../_service` | `Service#document` |
| `trigger_services(p,pkg,user,comment)` | `POST ?cmd=runservice` | `Token::Service`, `Webui::Packages::TriggerController#services`, `Service#save`, `Workflow::Step::TriggerServices` |
| `write_patchinfo` | `PUT .../_patchinfo?user&comment` | `Patchinfo` model, `Webui::PatchinfoController` |
| `wait_service` | `POST ?cmd=waitservice` | `Service#add_kiwi_import`, `Workflow::Step::BranchPackageStep/SubmitRequest` |
| `merge_service(user)` | `POST ?cmd=mergeservice` | `Service#add_kiwi_import` |
| `copy(target, source, user, orev keeplink expand comment requestid withacceptinfo dontupdatesource noservice instantiate vrevbump withvrev freezelink)` | `POST /source/:tp/:tpkg?cmd=copy&oproject&opackage` | `BsRequestActionSubmit#execute_accept` (request accept = backend copy with `withacceptinfo`), `BsRequestActionMaintenanceIncident`, `BranchPackage#create_branch_packages`, `MaintenanceHelper` (instantiate, release copy), `Package#...` |
| `branch(target, source, user, comment extendvrev keepcontent missingok noservice olinkrev orev requestid)` | `POST /source/:sp/:spkg?cmd=branch&oproject=&opackage=` (**note inverted direction: POSTed to the source, `oproject` is the target**) | `BsRequestActionSubmit`, `MaintenanceHelper#instantiate_container`, `Package#branch_from` |
| `linktobranch(user linkrev rev)` | `POST ?cmd=linktobranch` | `SourcePackageCommandController#linktobranch` |
| `link_info` / `write_link(user, content, rev ...)` | `GET`/`PUT .../_link` | `BranchPackage`, `MaintenanceHelper` (release relink), `Staging::StagedRequests#create_link`, `Workflow::Step::LinkPackageStep` |
| `source_diff(p,pkg, rev orev oproject opackage expand linkrev olinkrev unified missingok meta file filelimit tarlimit view withissues onlyissues nodiff cacheonly)` | `POST ?cmd=diff` | `Package#...`, `BsRequestAction::Differ::ForSource`, `BsRequestActionDelete`, `Webui::PackageController#rdiff` |
| `link_diff` | `POST ?cmd=linkdiff` | `Package` |
| `rebuild(p,pkg, repository arch)` | `POST /build/:project?cmd=rebuild&package=` | `SourcePackageCommandController#rebuild`, `Token::Rebuild`, `Webui::Packages::TriggerController`, `Workflow::Step::RebuildPackage` |
| `write_filelist(p,pkg,xml, user comment rev noservice requestid)` | `POST ?cmd=commitfilelist` (body = `<directory>` with `<entry name md5>`) | `FileService::Uploader` (webui multi-file upload), `MaintenanceHelper` (release) |
| `delete(p,pkg, user comment requestid)` | `DELETE /source/:p/:pkg` | `Package#delete_on_backend` (before_destroy; package.rb:818-844), consistency importer |
| `undelete(p,pkg, user comment time)` | `POST ?cmd=undelete` | `SourcePackageCommandController#undelete` |

### Backend::Api::Sources::Project — `/source/:project[...]`
| Method | Route | Callers |
|---|---|---|
| `attributes` / `write_attributes` | `GET`/`PUT /source/:p/_project/_attribute` | `HasAttributes` (write); read: none directly (SourceAttributeController uses `Package.attributes(p, nil)` which maps to `_project`) |
| `revisions` | `GET /source/:p/_project/_history?meta=1&deleted=1` | no callers |
| `meta(p, revision deleted)` | `GET /source/:p/_project/_meta?rev=` | `ReadAccessOfDeleted`, `SourceProjectCommandController#project_command_copy` (remote source), consistency checker/importer |
| `write_meta(p, xml, user comment requestid lowprio)` | `PUT /source/:p/_meta` | `Project#write_to_backend` (project.rb:680-703) |
| `configuration` / `write_configuration` | `GET`/`PUT /source/:p/_config` | `Project` (:937,1269 — e.g. DoD / `Repository` adds `%if`-free defaults), `Repository` (:283-286) |
| `key_info` | `GET /source/:p/_keyinfo?withsslcert=1&donotcreatecert=1` | `SourceProjectKeyinfoController`, `Project::SigningKeyGpg`, `Project::SigningKeySsl` |
| `pubkey(rev)` / `delete_pubkey(user comment meta)` | `GET`/`DELETE /source/:p/_pubkey` | `SourceProjectController#show_pubkey/#delete_pubkey` |
| `patchinfo` | `GET /source/:p/_patchinfo` | `UpdateReleasedBinariesJob` |
| `copy(p, user comment oproject withbinaries withhistory makeolder makeoriginolder noservice resign)` | `POST /source/:p?cmd=copy` | `ProjectDoProjectCopyJob` (`cmd=copy` on project is async in Rails) |
| `createkey` / `extendkey(days)` / `freezelink(requestid)` / `move(oproject)` / `undelete` | `POST /source/:p?cmd=...` | `SourceProjectCommandController#project_command_*`, `Project.undelete` |
| `commit(p, user, requestid rev comment)` | `POST /source/:p/_project?cmd=commit` | `BsRequestActionMaintenanceRelease#per_request_cleanup` (bump `_project` revision so the scheduler re-reads) |
| `packages(p, view parse)` | `GET /source/:p` | consistency checkers, `SyncLocalPackageVersionJob` (`view=info&parse=1`) |
| `list` | `GET /source` | `ConsistencyCheckJobService::ProjectConsistencyChecker` |
| `delete(p, user comment requestid)` | `DELETE /source/:p` | `Project#delete_on_backend` (after_destroy_commit; project.rb:1418-1438) |
| `repositories(p)` | `GET /getprojpack?project=&nopackages=1&withrepos=1&expandedrepos=1` | `Project` (:1398) — expanded repository paths (`expand_all_repositories`) |
| `staging_workflow` / `write_staging_workflow` | `GET`/`PUT /source/:p/_project/_staging_workflow?user=` | write: `Staging::Workflow#write_to_backend` (models/staging/workflow.rb:47-51); read: none |

## A.5 Complete table: backend route → frontend caller(s) → purpose

The route list is the bs_srcserver dispatch table (first 179 entries; the AJAX server sub-table starts at `/ajaxstatus`). ✔ = used by frontend, ∅ = not used by the frontend (only by workers/other daemons/remote OBS instances/admin tools), ◐ = reachable only via generic passthrough (osc can hit it through the frontend's verbatim proxy but no Rails code calls it on purpose).

| bs_srcserver route | Used | Frontend caller(s) | Purpose |
|---|---|---|---|
| `GET /` | ∅ (`Server.root` has no callers) | — | hello/version |
| `POST /source cmd=orderkiwirepos` | ✔ | SourceCommandController | order kiwi repos for image build |
| `POST /source cmd=triggerscmsync` | ✔ | SourceCommandController (scm bridge) | webhook from Gitea/GitHub bridge |
| `GET /source deleted:bool? project*` | ✔ (only `deleted=1`) | SourceProjectController#index | list deleted projects; non-deleted list comes from DB |
| `POST /source/$project cmd=createkey/extendkey` | ✔ | SourceProjectCommandController | signing key mgmt |
| `cmd=recreatesslcert` | ∅ | — | (bs_admin / manual) |
| `cmd=undelete` | ✔ | Project.undelete | |
| `cmd=copy` | ✔ | ProjectDoProjectCopyJob | project copy |
| `cmd=move` | ✔ | SourceProjectCommandController#project_command_move | rename project |
| `cmd=freezelink` | ✔ | SourceProjectCommandController | freeze project link |
| `cmd=notifypackagechange` | ∅ | — | bs_servicedispatch → srcserver internal |
| `GET /source/$project view=info ...` | ✔ | SourceProjectController#show (passthrough), SyncLocalPackageVersionJob, PublicController | project source info (srcmd5/verifymd5 per package) |
| `GET /source/$project deleted/expand/noorigins` | ✔ | passthrough for remote/deleted; PublicController; consistency check (`Project.packages`) | package list |
| `DELETE /source/$project` | ✔ | Project#delete_on_backend | |
| `GET /source/$project/_meta rev withconfig interconnect` | ✔ | remote passthrough, PublicController(`interconnect=1`), ReadAccessOfDeleted, consistency check | project meta (local projects answered from DB) |
| `PUT /source/$project/_meta user comment requestid lowprio` | ✔ | Project#write_to_backend | write-through |
| `GET/DELETE /source/$project/_pubkey` | ✔ | SourceProjectController | |
| `GET/PUT /source/$project/_config` | ✔ | ProjectConfigFile, Project/Repository | prjconf |
| `DELETE /source/$project/_config` | ∅ | — | |
| `GET /source/$project/_history` | ∅ | (`Sources::Project.revisions` unused) | project meta history |
| `GET /source/$project/_keyinfo` | ✔ | keyinfo controller, SigningKey* | gpg key + ssl cert |
| `POST /source/$p/$pkg view=info` | ∅ | — | (POST variant) |
| `cmd=diff` / `linkdiff` / `servicediff` | ✔ | passthrough + Sources::Package.source_diff/link_diff | diffs (request diff, rdiff) |
| `cmd=commit` | ✔ | passthrough (osc commit) | commit upload rev |
| `cmd=commitfilelist` | ✔ | passthrough (osc), write_filelist | commit by file list |
| `cmd=commitobsscm` | ∅ | — | bs_servicedispatch internal |
| `cmd=copy` | ✔ | passthrough + Sources::Package.copy (request accept, branch, release...) | |
| `cmd=collectbuildenv` | ✔ | passthrough | |
| `cmd=branch` | ✔ | Sources::Package.branch (frontend's own `cmd=branch` is BranchPackage, not a passthrough) | |
| `cmd=linktobranch` | ✔ | Sources::Package.linktobranch | |
| `cmd=deleteuploadrev` | ✔ | passthrough | |
| `cmd=undelete` | ✔ | Sources::Package.undelete | |
| `cmd=runservice` / `waitservice` / `mergeservice` / `getprojectservices` | ✔ | passthrough + Sources::Package.* | source services |
| `cmd=notifypackagechange` (package) | ∅ | — | servicedispatch internal |
| `cmd=lock` / `cmd=unlock` (package) | ∅ | — (frontend implements lock via `lock` flag in `_meta`; its `cmd=unlock` edits flags) | |
| `PUT /source/$p/$pkg cmd:` (obsolete) | ∅ | — | |
| `GET /source/$p/$pkg view=getmultibuild` | ◐ | passthrough (`view` is whitelisted in SourcePackageController#show) | |
| `GET /source/$p/$pkg view=info` | ✔ | Package#update_backendinfo (`view=info&withchangesmd5=1&nofilename=1`), SyncLocalPackageVersionJob | |
| `GET /source/$p/$pkg rev linkrev emptylink deleted expand view extension lastworking withlinked meta product` | ✔ | Directory.hashed (dozens of callers), passthrough | file list |
| `GET /source/$p/$pkg/_history` | ✔ | Package#commit, webui revisions | |
| `GET/PUT /source/$p/$pkg/_meta` | ✔ | passthrough(deleted/rev/meta/view), Package#write_to_backend, Package.meta | |
| `DELETE /source/$p/$pkg` | ✔ | Package#delete_on_backend | |
| `GET/PUT/DELETE /source/$p/$pkg/$filename` | ✔ | passthrough, Sources::File.* | source files incl. `_link`, `_service`, `_patchinfo`, `_attribute`(GET), `_channel`, `_constraints`, `_multibuild`, `_aggregate`, `_product`, `_staging_workflow` |
| `GET /published view=collection name project*` | ✔ | OwnerSearch::Assignee | |
| `GET /published`, `/published/$p`, `/published/$p/$r`, `/published/$p/$r/$arch[/$file[/$sub]]` | ✔ | PublishedController passthrough; `Published.*` (`view=publishedpath`, `view=status`) | browse published tree |
| `GET /getprojpack` | ✔ (only `nopackages=1&withrepos=1&expandedrepos=1`) | Sources::Project.repositories | expanded repo paths; everything else is scheduler/worker use |
| `POST/GET /relsync` | ∅ | — | bs_admin / release-number sync between schedulers |
| `GET /worker/$workerid` | ✔ | Worker::CapabilityController | |
| `POST /worker cmd=checkconstraints` | ✔ | Worker::CommandController | |
| `/getsources`, `/getconfig`, `/getsslcert`, `/getbinaries`, `/getbinaryversions`, `/getobsgendiffdata`, `/getmodulemd`, `/getsignkey`, `/getpubkey` | ∅ | — | worker & repo-server internal RPCs |
| `/lastevents`, `/public/lastevents` | ✔ | SourceController#lastevents (volley) | long-poll event stream for remote OBS (interconnect) / osc watchers |
| `/lasteventsproxy` | ∅ | — | srcserver → remote OBS interconnect |
| `GET /lastnotifications` | ✔ | UpdateNotificationEvents (clock, 17 s) | **the** backend→frontend event bridge |
| `GET/DELETE /notificationpayload/$key` | ✔ | UpdateReleasedBinariesJob | PACKTRACK payloads |
| `POST /event` | ∅ | — | bs_admin / daemons inject scheduler events |
| `GET /search in match return values` | ✔ | search#pass_to_backend | generic published XPath |
| `/search/project[/id]`, `/search/package[/id]` | ✔ (`package/id`, `package`) ; ∅ (`project*` via API module has no callers) | Search.* | link/incident/product lookups |
| `POST /search/published cmd` | ∅ | — | bs_publish updates published DB |
| `/search/published/binary/id`, `/repoinfo/id`, `/pattern/id` | ✔ | passthrough + Search.published_binaries_for_package | |
| `/search/published/binary/name` | ∅ | — | |
| `GET /service` | ✔ | ServiceController | installed services list |
| `PUT/GET /configuration` | ✔ PUT ; ∅ GET | Configuration#write_to_backend | |
| `PUT/GET /issue_trackers`, `POST cmd=issues` | ✔ PUT, POST ; ∅ GET | IssueTrackerWriteToBackendJob, RequestController | |
| `GET /build` | ∅ | (frontend answers `/build` from DB project list) | |
| `GET /build/_workerstatus` | ✔ | Worker.status (clock) | |
| `PUT/GET /build/_dispatchprios` | ✔ | BuildController#project_index (admin PUT; GET) | dispatch priorities |
| `GET /build/_result scmrepository...` | ✔ | BuildController#scmresult, PublicController#build | scm bridge CI status |
| `GET /build/$project/_availablebinaries` | ✔ | Kiwi::Image | |
| `POST /build/$project cmd:` | ✔ | BuildController#project_index passthrough (`wipe restartbuild killbuild abortbuild rebuild unpublish sendsysrq`), Build::Project.* (`suspendproject resumeproject rebuild wipe abortbuild wipepublishedlocked`) | |
| `GET /build/$project` | ✔ | passthrough | repo list |
| `GET /build/$project/_result` | ✔ | Buildresult.find_hashed and friends; passthrough | build results |
| `GET /build/$project/_jobhistory` | ✔ | JobHistory.for_project | |
| `GET /build/$p/$r`, `/_buildconfig`, `/$arch` | ✔ | passthrough | |
| `GET /build/$p/$r/$a/_builddepinfo` | ✔ | passthrough + Binaries.builddepinfo | |
| `GET /build/$p/$r/$a/_jobhistory` | ✔ | JobHistory.for_package/for_repository_and_arch, Binaries.job_history | |
| `POST /build/$p/$r/$a/_repository cmd=missingdodresources` / `match:` | ∅ | — | DoD / worker |
| `POST /build/$p/$r/$a/$pkg cmd=copy` | ✔ | Binaries.copy (release) | binary release |
| `POST /build/$p/$r/$a/$pkg` (upload) | ✔ (admin) | BuildController#index | |
| `GET /build/$p/$r/$a/$pkg_repository view binary nometa ...` | ✔ | passthrough, Binaries.files | binary list |
| `GET/POST .../_buildinfo` | ✔ | BuildController#buildinfo | osc build |
| `.../_jobstatus`, `/_log`, `/_reason`, `/_status`, `/_history` | ✔ | Status.*, passthrough | |
| `.../_buildstats` | ◐ | (not whitelisted anywhere; BuildController#index only matches `_status`/`_history`; so effectively ∅) | |
| `GET .../$pkg_repository/$filename view withfilelist module` | ✔ | Build::FileController, Binaries.file/fileinfo/fileinfo_ext/publishedpath/rpmlint_log | |
| `PUT/DELETE .../_repository/$filename` | ✔ (admin / project_change) | Build::FileController | |
| `GET /request/_lastid`, `/request/$id` | ∅ | (`Backend::Api::Request` has no callers) | 2.4 migration leftovers |
| `GET /jobs/$arch` | ∅ | — | masterdispatcher |
| `POST /notify/$type` | ∅ (backend internal: BSNotify from repserver/publisher/scheduler via bs_notifyforward) | — | |
| `POST /notify_plugins/$type` | ∅ (API method exists, no callers) | — | |
| `/ownregistryrepo`, `/disownregistryrepo`, `/registry/*`, `/sigstore/*`, `/slsa/*` | ∅ | — | container registry / notary / rekor / SLSA (served by srcserver + bs_repserver/BSRegistryServer, fronted by the web server directly) |
| `/ajaxstatus`, `/serverstatus` | ∅ | — | admin (bs_serverstatus) |

bs_repserver (`routes/bs_repserver.txt`, 88 routes) and bs_service (8 routes) and bs_getbinariesproxy (4) are **never** called by the frontend.

## A.6 Backend → frontend: the notification stream and event model

### Backend side
* Every backend daemon reports through **bs_srcserver only** ("Backend notifications are always routed through the source server", src/backend/BSNotify.pm:36-40). `BSNotify::notify($type, \%params, $payload)` does `POST $BSConfig::srcserver/notify/$type` with form-url-encoded params (payload, if any, as octet-stream body, 60 s + size-based timeout) (BSNotify.pm:41-71).
* Daemons that are not allowed to block (scheduler) write to the file queue `$bsdir/events/notifyforward/queue` via `BSRedisnotify::addforwardjob` (BSRedisnotify.pm:101-105) and `bs_notifyforward` drains it (src/backend/bs_notifyforward:116-161): non-`redis` entries are POSTed to `/notify/$type`; `redis` entries (build result updates `updateresult`, `deleteresult`, `updateoneresult`, `updatejobstatus`, BSRedisnotify.pm:107-138) are batched 32 at a time, serialised with Storable, and POSTed to `/notify/redis` with keep-alive (a dedicated forked keep-alive child in srcserver: bs_srcserver:7406-7437). `BSSched::Checker::notify` (BSSched/Checker.pm:119-130) uses the forwarder when `$BSConfig::notifyforward // $BSConfig::redisserver` is set, else calls BSNotify directly.
* srcserver `external_notification` (bs_srcserver:7527-7539): `redis` → `add_redis_notifications` (updates the Redis build-status cache used by… nothing in the frontend; `grep -ri redis src/api/app` finds no reader — Redis is consumed by the WebUI only indirectly through the backend `_result` route and by `bs_redis`); `sourcepublish` → writes `$eventdir/sourcepublish/*` events for `bs_sourcepublish`; everything else → `notify()` (bs_srcserver:165-181): params flattened to `key=value` strings, optional JSON payload stored under `$bsdir/notificationpayload/<time>.<md5>` (`PACKTRACK` only), then appended with `BSFileDB::fdb_add_i` to **`$bsdir/events/lastnotifications`** (layout `number type time data[]`, bs_srcserver:150).
* `GET /lastnotifications?start=N[&block=1][&view=json][&noprune=1]` (bs_srcserver:209-291): without `start` returns `{next: last+1, sync: 'lost'}`; otherwise returns all records with `number >= start` (max 1000; sets `limit_reached` and `next` accordingly); if `start` is older than the first record → `sync=lost` and an empty list; `block=1` hands the request off to the AJAX process which adds a file watcher on `lastnotifications` (long-poll). Auto-prunes records older than `start-500` once the file is 1000 ahead. Response schema `$BSXML::notifications` (BSXML.pm:2060): `<notifications next= sync= limit_reached=><notification type= time=><data key=>value</data>...`.

### Notification types actually emitted (grep of `notify(` sites)
| Type | Emitter | Params |
|---|---|---|
| `SRCSRV_CREATE_PROJECT` / `SRCSRV_UPDATE_PROJECT` | bs_srcserver:2254 (`putproject`) | project, sender |
| `SRCSRV_DELETE_PROJECT` / `SRCSRV_UNDELETE_PROJECT` | :2308, :2362 | project, comment, sender, requestid |
| `SRCSRV_CREATE_PACKAGE` / `SRCSRV_UPDATE_PACKAGE` | :2437 (`putpackage`) | project, package, sender |
| `SRCSRV_DELETE_PACKAGE` / `SRCSRV_UNDELETE_PACKAGE` | :2486, :2513, Scmsync.pm:154,163 | project, package, sender, comment, requestid |
| `SRCSRV_UPDATE_PROJECT_CONFIG` | :812, :2859, :2869, Scmsync.pm:229 | project, files, comment, sender |
| `SRCSRV_UPLOAD` | :728, :3273 (`putfile`) | project, package, filename, comment, user, requestid, target, meta |
| `SRCSRV_COMMIT` | :851 (`addrev`) | project, package, files, rev, user, comment, requestid |
| `SRCSRV_VERSION_CHANGE` | :854 | project, package, files, rev, newversion, oldversion, user, comment, requestid |
| `SRCSRV_SERVICE_SUCCESS` / `SRCSRV_SERVICE_FAIL` | BSSrcServer/Service.pm:122-141 | project, package, rev, user, comment, requestid, error |
| `BUILD_SUCCESS` / `BUILD_FAIL` / `BUILD_UNCHANGED` | bs_repserver:2300-2330 (`notify_jobresult`) | project package repository arch rev srcmd5 verifymd5 readytime reason versrel bcnt release starttime endtime workerid previouslyfailed buildtype successive_failcount |
| `REPO_BUILD_STARTED` / `REPO_BUILD_FINISHED` | BSSched/Checker.pm:164-190 | project, repo, arch, buildid |
| `REPO_PUBLISHED` / `REPO_PUBLISH_STATE` / `CONTAINER_PUBLISHED` | bs_publish:3671, 3836, 3208 | project, repo, buildid / state / container |
| `PACKTRACK` | bs_publish:2050, 3707 | project, repo + JSON payload (released binaries list) |

### Frontend side
* `config/clock.rb:18-23`: every 17 s (in a thread) `UpdateNotificationEvents.new.perform` (models/update_notification_events.rb). It keeps the cursor in the `backend_infos` table (`BackendInfo.lastnotification_nr`, models/backend_info.rb:13-21), calls `Backend::Api::Server.last_notifications(nr)` (`block=1` long-poll), and for each `<notification>` builds `Event::Factory.new_from_type(type, data)` (models/event/factory.rb: strips `SRCSRV_`, camel-cases → `Event::CreateProject`, `Event::BuildFail`, `Event::Packtrack`, `Event::RepoPublishState`, ...; unknown types are dropped). Loops while `limit_reached`. On `sync=lost` it just jumps the cursor (events are lost silently).
* `Event::Base` (models/event/base.rb): STI table `events`, `payload` JSON restricted to `payload_keys`; unknown keys raise (`check_left_attribs`, :108-118) — so **adding a param to a backend notification without adding it to the Ruby class breaks ingestion**. `after_create_commit :send_to_bus` publishes the raw payload to RabbitMQ (`RabbitmqBus.send_to_bus(routing_key, payload)`, lib/rabbitmq_bus.rb; routing keys `package.build_fail`, `repo.published`, `request.create`, ...) plus a `metrics` line; `after_create :create_project_log_entry_job` for project/package events. Jobs triggered by backend-originated events: `UpdateBackendInfosJob` (on `Event::Commit`/service events: refreshes `backend_packages` for the package and everything linking to it, jobs/update_backend_infos_job.rb), `UpdateReleasedBinariesJob` (on `Event::Packtrack`), `SendEventEmailsJob` (clock, 30 s), `ReportToScmJob` (build/repo events for SCM status reporting), `CreateProjectLogEntryJob`.
* Full payload key list per event class is in `app/models/event/*.rb` (e.g. `Event::Build` keys: `project package sender repository arch release readytime srcmd5 rev reason bcnt verifymd5 hostarch starttime endtime workerid versrel previouslyfailed successive_failcount buildtype`, build.rb:6-8).
* The frontend has **no** AMQP consumer and **no** Redis reader; RabbitMQ is publish-only (events + metrics, `CONFIG['amqp_options']`), InfluxDB is metrics-only.

## A.7 Write-through model and data authority

`CONFIG['global_write_through']` (options.yml `# if set to false, the API will only fake writes to backend`) gates every write (18 references). The DB is written first inside a transaction, then the backend; a backend failure raises and rolls back the Rails transaction (`Project#store` = `transaction { save!; write_to_backend }`, project.rb:705-711; `Package` does it in `after_save :write_to_backend`, package.rb:89). Commit metadata (`user`, `comment`, `requestid`, `lowprio`) is carried in `@commit_opts`.

| Data | Authoritative store | Backend copy? | Sync mechanism / notes |
|---|---|---|---|
| Users, groups, roles, static permissions, relationships (maintainer/bugowner/reviewer/reader/downloader per project/package/group), watchlists, tokens, notifications, event subscriptions, comments, reports/decisions/appeals, status messages, announcements, canned responses, labels, assignments, beta features, sessions | **DB only** | no | Backend knows nothing about users beyond the `user=` string in commits/_meta `<person>` entries. The backend does **no** authorization (A.9). |
| Project `_meta` (title, description, `<person>`/`<group>` roles, repositories, path elements, release targets, DoD, flags build/publish/debuginfo/useforbuild/binarydownload/sourceaccess/access/lock, kind, maintenance `<maintains>`, link, `scmsync`, `remoteurl`, `url`, `mountproject`) | **DB** (tables projects, repositories, path_elements, release_targets, architectures, repository_architectures, download_repositories, flags, relationships, linked_projects, maintained_projects) | yes — `PUT /source/:p/_meta` with `Project#to_axml` (cached render) | `Project::UpdateFromXmlCommand` parses incoming XML into DB rows, then `store` writes the *re-rendered* XML. The backend copy (`$bsdir/projects/<name>.xml`) is what the scheduler/publisher/worker read; the frontend reads its own DB copy for GET `_meta` (except remote/deleted/rev). Drift detector: `ConsistencyCheckJob` / `ProjectMetaChecker` (diff of hash of both), weekly `Old::ConsistencyCheckJob` (clock.rb:90). |
| Package `_meta` (title/description/url, persons/groups, flags, devel, `releasename`, `scmsync`, `bcntsynctag`) | **DB** (packages, flags, relationships) | yes — `PUT /source/:p/:pkg/_meta` | same pattern (`Package#update_from_xml` → `store`). Backend copy `$bsdir/projects/<prj>.pkg/<pkg>.xml`. |
| Attributes (namespaces, types, values, allowed values, modifiable_by) | **DB** (attrib_* tables) | yes — `PUT .../_attribute` (`HasAttributes#write_attributes`) | one-way; `GET /source/:p/:pkg/_attribute` API endpoint is answered **from the backend copy** (SourceAttributeController#show). |
| Project config (`_config`) | **backend** (`$bsdir/projects/<name>.conf` with history) | — | frontend only proxies / reads via `ProjectConfigFile`; `Repository` may rewrite it (repository.rb:283-286). |
| Sources: files, revisions, links, `_service`, `_patchinfo`, `_channel`, `_product`, `_constraints`, `_multibuild`, `_aggregate`, `_link`, `.changes`, md5 trees | **backend** (`$bsdir/sources`, `$bsdir/trees`, `$bsdir/projects/*.pkg/*.rev`) | — | frontend caches *derived* facts: `backend_packages` (srcmd5, verifymd5, changesmd5, expandedmd5, maxmtime, links_to, error — `Package#update_backendinfo` from `GET /source/:p/:pkg?view=info&withchangesmd5=1&nofilename=1` + plain filelist for linkinfo, package.rb:1053-1105), `package_kinds` (link/patchinfo/aggregate/channel/product/multibuild from the `<directory>` entry names, `Package.detect_package_kinds`), `package_issues` (from `_patchinfo`/`.changes` via backend diff `withissues`), `products`, `product_media`, `product_update_repositories` (parsed from `_product` via `view=products`), `channels`/`channel_binaries`/`channel_targets` (parsed `_channel`), `package_versions` (`view=info&parse=1`), `kiwi_images` (parsed `.kiwi` file). Invalidation: `Package#sources_changed` deletes the `backend_packages` row and enqueues `PackageUpdateIfDirtyJob`; `UpdatePackageMetaJob` (daily 23:00) rescans links via backend search; `BackendPackage.refresh_dirty`. Trigger paths: every passthrough write calls `sources_changed`, and backend-originated `SRCSRV_COMMIT`/service events trigger `UpdateBackendInfosJob`. |
| Build results, logs, binaries, job history, worker status, published repos, build ids, buildinfo, builddepinfo, rpmlint | **backend** (`$bsdir/build`, `$bsdir/jobs`, `$bsdir/repos`, `$bsdir/workers`) | — | always fetched live (some `Rails.cache` with no TTL for build ids/published paths, `workerstatus` every 30 s). `status_histories` (DB) are sampled from workerstatus for the monitor graphs. |
| Requests, actions, reviews, history elements, accept_info, request counters, staging workflows/projects/excluded requests, maintenance incidents (`maintenance_incidents` table, incident counters, `incident_updateinfo_counter_values`) | **DB** | partial: on accept the frontend performs backend copies/branches/links (`BsRequestActionSubmit#execute_accept`, `MaintenanceHelper#release_package`); staging workflow XML mirrored to `_project/_staging_workflow`; the backend records `requestid` on the resulting commits | The `/request/*` backend routes are dead. |
| Issue trackers, issues | **DB** | yes — `PUT /issue_trackers` (whole list) | backend needs them to recognise bug refs in diffs (`withissues`). |
| Configuration (`/configuration`: title, description, obs name, download_url, anonymous, registration, default_access_disabled, allow_user_to_create_home_project, disallow_group_creation, change_password, hide_private_options, gravatar, ymp_url, bugzilla_url, http_proxy, no_proxy, cleanup_*, enforce_project_keys, api_url, tos_url, code_of_conduct, admin_email...) | **DB** (singleton row) | yes — `PUT /configuration` (+ fan-out to repo servers and service server) | backend uses `obsname`, `download_url`, `enforce_project_keys`, `api_url`, `http_proxy`... (BSConfiguration pulls from `configuration.xml`). |
| Architectures | **DB** (`architectures.available`) | — | backend has its own list (`$BSConfig::schedulers`); the frontend validates repo archs against the DB. |
| Distributions (`/distributions`), image templates, package templates | **DB** | — | remote ones fetched hourly from `<remoteurl>/distributions.xml` of every remote project (FetchRemoteDistributionsJob, via `Project::RemoteURL` = open-uri **directly from Rails**, not via backend). |
| Signing keys / SSL certs | **backend** (`$bsdir/projects/<name>.pkg/_signkey`, `_sslcert`; bs_signer) | — | frontend proxies `_keyinfo`/`_pubkey`, `cmd=createkey/extendkey`. |
| Binary releases (`binary_releases`) — what was released where (maintenance tracking, `/search/released/binary`) | **DB** | — (derived) | imported from `PACKTRACK` payloads (UpdateReleasedBinariesJob). |
| Project log entries (`project_log_entries`), commit activity, events, status histories | **DB** (derived) | — | from the notification stream. |
| Interconnect / remote projects | **DB** row with `remoteurl`/`remoteproject` (projects table) + backend `_meta` copy | yes | The *backend* resolves remote packages/binaries itself through `BSSrcServer::Remote` (`<remoteurl>/public/...` with `$BSConfig::proxy`); the frontend resolves "which local project defines the remote instance" (`Project.find_remote_project`, project.rb:249-270) and then proxies to its own backend, which fetches from the remote `public` API. Only distributions.xml is fetched by Rails directly. |
| Source-service definitions (`/service`) | **service server** filesystem (`/usr/lib/obs/service/*.service`) | — | proxied. |

## A.8 Frontend jobs/polling that touch the backend (src/api/config/clock.rb + app/jobs)

| Schedule / trigger | Job | Backend interaction |
|---|---|---|
| every 17 s (thread) | `UpdateNotificationEvents` | `GET /lastnotifications?start&block=1` (long-poll) |
| every 30 s | inline | `GET /build/_workerstatus` → cache + `StatusHistory` |
| every 30 s | `SendEventEmailsJob` | none (DB) |
| every 5 min | `MeasurementsJob`, `WorkerMeasurementsJob` | reads cached workerstatus, publishes to RabbitMQ |
| every 49 min | `StatusHistoryRescalerJob` | DB |
| hourly | `IssueTrackerFetchIssuesJob`/`UpdateIssuesJob`, `BsRequestAutoAcceptJob`, `FetchRemoteDistributionsJob` (remote OBS HTTP) | accept → backend copies |
| daily 23:00 | `UpdatePackageMetaJob` + `BackendPackage.refresh_dirty` → `UpdatePackagesIfDirtyJob`/`PackageUpdateIfDirtyJob` | `GET /search/package/id` (links), `GET /source/:p/:pkg[?view=info]` |
| daily 06:00 | `ProjectCreateAutoCleanupRequestsJob` | DB (creates delete requests) |
| weekly Sun 03:00 | `Old::ConsistencyCheckJob` (and `ConsistencyCheckJob` + rake `check_project`/`fix_project`) | `GET /source`, `GET /source/:p`, `GET /source/:p/_meta`, `GET .../_meta`; with `fix:` deletes/imports projects & packages in the DB (`BackendProjectImporter`, `BackendPackageImporter`) or deletes stray backend objects (`Project.delete`, `Package.delete`) |
| event-triggered | `UpdateBackendInfosJob`, `UpdateReleasedBinariesJob`, `ConfigurationWriteToBackendJob`, `IssueTrackerWriteToBackendJob`, `ProjectDoProjectCopyJob`, `ProjectDoProjectReleaseJob`, `StagingProjectCopyJob`, `StagingProjectAcceptJob`, `PublishedRepositoriesCleanupJob`, `SyncLocalPackageVersionJob`, `BsRequestActionWebuiInfosJob` (diffs) | see A.4 |

## A.9 Trust, auth and failure semantics of the contract

* **No authentication from frontend to backend.** `BSStdServer::authorize` (BSStdServer.pm:77-106) only enforces optional `$BSConfig::ipaccess` / `$BSConfig::subjectdnaccess` regex→permission maps, and routes are marked `!rw`/`!-` in the dispatch table (e.g. `'!rw :' => undef`, `'!- GET:' => undef` in bs_service:333-335 meaning "GET/HEAD need no auth, everything else needs `rw`"). The `user=` query parameter is **informational** (recorded in revisions/notifications); the backend trusts it blindly. All authorization (pundit policies, `Package.get_by_project_and_name` access/sourceaccess flags, `can_modify?`) lives in Rails. A Go backend inherits this: it must stay network-isolated or add an auth layer.
* **Interconnect**: remote OBS instances are only ever accessed through `/public/*` (anonymous) of the *remote frontend*, which proxies to its backend with `interconnect=1`; backend-side `BSSrcServer::Remote` handles caching (`$bsdir/remotecache`) and `/lasteventsproxy` keeps remote caches fresh.
* **Request IDs**: `x-request-id` header is logged and propagated; `X-Frontend-Start` feeds the backend's slow-request log.
* **Idempotency / partial failure**: there are no transactions across the boundary. Known inconsistencies are tolerated: `Backend::NotFoundError` on delete is ignored ("backend was out of sync", project.rb:1425-1428, package.rb:832-835); `sync=lost` on notifications just skips; `ConsistencyCheckJob` reconciles weekly.
* **Timeouts**: Rails waits up to 1000 s on GET and 100000 s on POST (e.g. `cmd=copy&withhistory`, `cmd=diff` of huge tarballs, `cmd=release`). The backend serialises slow operations by forking per connection; blocking endpoints (`block=1`, `_log` streaming, `waitservice`) are handed to the single-process AJAX/event server (BSHandoff, see D).
* **Validation**: outside production every XML response is validated against the RNG/XSD schemas in `docs/api/api` (`Suse::Validator`, application_controller.rb:31,189) — the schemas there are effectively the contract documentation for both the public API and the backend answers.


---

# B. The public OBS API and WebUI: feature inventory

Sources: `src/api/config/routes/api.rb` (477 lines, the XML API used by osc), `src/api/config/routes/webui.rb` (530 lines, the HTML UI), the OpenAPI description `src/api/public/apidocs/OBS-v2.10.50.yaml` + 403 path files under `src/api/public/apidocs/paths/*.yaml` (served at `/apidocs`), and the RelaxNG/XSD schemas in `docs/api/api/*.{rng,xsd,xml}` (about, activity, aggregate, architecture, assignment, attrib, attrib_type, attribute_namespace_meta, backlog, binary_released, buildhistory, buildresult, buildstatus, channel, check, comment_history, configuration, constraints, directory, distribution(s), excluded_requests, group, issue, issue_tracker(s), jobstatus, link, multibuild, new_announcement, package, packageresult, patchinfo, pattern, productlist, project(s), report, request, required_checks, revisionlist, service, service-definition, staged_requests, staging_project(s), status, status_message(s), status_report, token(list), user, workerstatus, ...). Routing is split by `RoutesHelper::APIMatcher` / `WebuiMatcher` (config/routes.rb) on host/format.

Legend for the "Where" column: **DB** = implemented entirely against MySQL; **BE** = requires backend calls; **DB+BE** = DB authoritative with backend write-through or backend reads for enrichment; **proxy** = verbatim passthrough after a Rails permission check.

## B.1 Authentication & identity

| Feature | API routes | Where | Notes |
|---|---|---|---|
| HTTP Basic auth | every API request (`Authenticator#extract_user`, controllers/concerns/authenticator.rb:16-31, 94-118) | DB | `X-HTTP_AUTHORIZATION`/`Authorization` headers; passwords in `users` (bcrypt/legacy); no kerberos code exists in the tree (`grep -ri kerberos src/api` → nothing); LDAP exists only as a message "LDAP passwords can not be changed in OBS" (person_controller.rb:253-256) — the LDAP backend itself was removed, `proxy_auth_mode` is the replacement |
| Identity-proxy / SSO | `CONFIG['proxy_auth_mode']` = `:off | :on | :mellon | :ichain` (options.yml.example:36-68); headers `HTTP_X_USERNAME`, `HTTP_X_EMAIL`, `HTTP_X_FIRSTNAME/LASTNAME`; `proxy_auth_login_page/logout_page/register_page/account_page` | DB | `find_or_create_proxy_user` auto-creates users (authenticator.rb:79-92) |
| Anonymous / `_nobody_` | `CONFIG['allow_anonymous']`, `Configuration.anonymous`; `/public/*` always anonymous | DB | disabling anonymous also disables interconnect |
| Session login (WebUI) | `GET/POST /session`, `/session/reset`, `/signup` | DB | |
| Registration / password change | `POST /person?cmd=register`, `/person/register`, `/person/changepasswd`, `POST /person/:login?cmd=change_password|lock|delete`, `Configuration.registration` (allow/confirmation/deny) | DB | |
| Person API | `GET /person`, `GET/PUT/POST /person/:login`, `GET /person/:login/group`, `GET/PUT /person/:login/watchlist` | DB | user XML: login, email, realname, state, globalroles, watchlist, owner, ignore_auth_services |
| Groups | `GET /group`, `GET/PUT/DELETE /group/:title`, `POST /group/:title?cmd=add_user|remove_user|set_email`, `CONFIG['disallow_group_creation_with_api']` | DB | group maintainers |
| Roles / global permissions | `globalroles` in user XML (Admin, Staff, Moderator...), static permissions (`roles_static_permissions`), per-object relationships via `_meta` `<person userid role>` / `<group groupid role>` | DB+BE (roles land in `_meta` copy) | the backend never evaluates roles |
| Tokens (`/person/:login/token`) | `GET/POST /person/:login/token`, `PUT/DELETE /person/:login/token/:id`; types `Token::Service`, `Token::Rebuild`, `Token::Release`, `Token::Workflow`; WebUI `/my/tokens`, token users/groups sharing, token enable/disable (`Event::TokenStateChange`) | DB | secret string, optional project/package scope, HMAC `X-OBS-SIGNATURE` verification for webhooks (`TriggerControllerService::TokenExtractor`) |
| Blocked users, censor, moderation (reports/decisions/appeals) | `/reports/*` (`resources :reports`, per request/package/project/comment/user), `/decisions`, `/appeals`, `/users/:login/block`, `/users/:login/censor`, comment locks | DB | notification + email only |

## B.2 Projects, packages, sources

| Feature | API routes | Where | Notes |
|---|---|---|---|
| List projects | `GET /source` (`/build`, `/published` aliases) | DB (`Project.order(:name).pluck(:name)`), `?deleted=1` → proxy | |
| Project meta | `GET/PUT /source/:p/_meta`, `DELETE /source/:p` | DB+BE | PUT parses XML → DB → re-render → `PUT _meta` backend; remote projects → proxy; `remove_linking_repositories`, `force`, `comment` params; `lock` flag; creation adds caller as maintainer unless `prevent_adding_maintainer_in_project_creation_with_api` |
| Project config (`_config`) | `GET/PUT /source/:p/_config` (`rev`, `user`, `comment`) | BE | X-forwardable |
| Project commands | `POST /source/:p?cmd=` `showlinked` (DB), `lock`/`unlock` (DB flags → `_meta`), `freezelink` (BE), `addchannels`/`modifychannels` (DB+BE), `extendkey`/`createkey` (BE), `createmaintenanceincident` (DB+BE), `undelete` (BE + DB import), `release` (DB+BE: `MaintenanceHelper#release_package` → binary copies, source copies, `_link` rewrite), `move` (BE + DB rename), `copy` (DB + async `ProjectDoProjectCopyJob` → BE `cmd=copy`), `createpatchinfo` (BE `_patchinfo` write), `set_flag`/`remove_flag` (DB → `_meta`) | mixed | source_project_command_controller.rb:4-300 |
| Global commands | `POST /source?cmd=createmaintenanceincident|branch|orderkiwirepos`, `POST /public/source?cmd=triggerscmsync` | DB+BE / proxy | `cmd=branch` = `osc mbranch` (`BranchPackage` by attribute search, DB) |
| Package list / source info | `GET /source/:p` (`view=info` proxy; `view=productlist|verboseproductlist|issues|versions` DB; `expand` DB `expand_all_packages`) | DB / proxy | |
| Package meta | `GET/PUT /source/:p/:pkg/_meta`, `DELETE /source/:p/:pkg` (`force`, `comment`) | DB+BE | deleting cleans requests/reviews/devel refs (package.rb before_destroy chain :80-87) |
| Files | `GET/PUT/DELETE /source/:p/:pkg/:file` (`rev`, `expand`, `meta`, `deleted`, `limit`, `view=blame`, `keeplink`, `comment`), `GET /source/:p/_project/:file` | proxy (BE) | PUT validates special files (`PackageService::FileVerifier`: `_link`, `_service`, `_patchinfo`, `_channel`, `_constraints`, `_aggregate`, `_multibuild`, `_product`...) and refuses `_attribute`/`_staging_workflow` |
| Revisions / history | `GET /source/:p/:pkg/_history` (`meta`, `deleted`, `limit`, `startbefore`), `GET /source/:p/_project/_history` | BE (proxy) | |
| Package commands | `POST /source/:p/:pkg?cmd=` (31 commands, source_package_command_controller.rb): `updatepatchinfo`, `importchannel`, `unlock`, `addchannels`, `addcontainers`, `enablechannel` (DB+BE), `getprojectservices`, `showlinked` (DB), `collectbuildenv`, `instantiate` (BE copy/branch), `undelete`, `createSpecFileTemplate`, `rebuild`, `commit`, `commitfilelist`, `diff`, `linkdiff`, `servicediff`, `copy`, `release`, `waitservice`, `mergeservice`, `runservice`, `deleteuploadrev`, `linktobranch`, `branch`, `fork` (scmsync), `set_flag`, `remove_flag` | mostly proxy/BE | `branch`/`fork` are `BranchPackage` (DB-driven: devel package resolution, update-project attribute, maintenance incident lookup, auto-cleanup attribute) + BE `cmd=branch`/`copy`/`_link` |
| Links, aggregates, multibuild, constraints | files `_link`, `_aggregate`, `_multibuild`, `_constraints` | BE | DB tracks only `package_kinds` + `backend_packages.links_to` |
| Flags | `set_flag`/`remove_flag` cmds; `POST /flag/:p(/:pkg)` WebUI | DB → `_meta` | flags: build, publish, debuginfo, useforbuild, binarydownload, sourceaccess, access, lock (per repo/arch) |
| Attributes | `GET/POST/DELETE /source/:p(/:pkg(/:binary))/_attribute(/:attr)`, namespace/type admin `GET/PUT/POST/DELETE /attribute/:ns[/_meta]`, `/attribute/:ns/:name/_meta`; WebUI `/attribs/...` | DB + BE copy (GET reads BE copy) | attrib types: value counts, allowed values, defaults, issue lists, `modifiable_by` user/group/role |
| scmsync / SCM bridge | `scmsync` attr in project/package `_meta`; `fork`; `triggerscmsync`; `/build/_result?scmrepository=` | DB (meta) + BE (bs_srcserver `BSSrcServer::Scmsync`, obs_scm_bridge service) | writes to scmsync'ed packages refused in Rails (`ScmsyncReadOnly`) |
| Interconnect / remote projects | project `_meta` with `<remoteurl>`/`<remoteproject>` (admin only, `Project.validate_remote_permissions`); WebUI `/interconnects/new`; `GET /distributions/include_remotes` | DB + BE | all remote source/binary access goes backend→remote `/public` |
| Deleted objects | `?deleted=1` on project/package/file routes, `undelete` cmds; `ReadAccessOfDeleted` concern checks the backend `_meta` of the deleted object | BE | |
| Public (anonymous) read API | `GET /public/source/...`, `/public/build/...`, `/public/configuration[.xml|.json]`, `/public/distributions`, `/public/request/:number`, `/public/binary_packages/:p/:pkg` (1-click ymp), `/public/image_templates`, `/public/package_templates`, `/public/lastevents`, `/public/statistics/maintenance_statistics/:p` | proxy / DB | used by interconnect and download pages |

## B.3 Builds, results, binaries

| Feature | API routes | Where |
|---|---|---|
| Build results | `GET /build/:p/_result` (`view=status|summary|binarylist|versrel`, `package`, `repository`, `arch`, `code`, `lastbuild`, `multibuild`, `locallink`, `oldstate`), `?lastsuccess&package&pathproject` (Rails `PackageBuildStatus`) | proxy / BE |
| Repo/arch/package listing | `GET /build/:p`, `/build/:p/:r`, `/build/:p/:r/:a`, `/build/:p/:r/_buildconfig` | proxy |
| Package build status / history / reason / jobstatus / statistics / buildinfo / builddepinfo / jobhistory | `GET /build/:p/:r/:a/:pkg/_status|_history|_reason|_jobstatus|_statistics|_buildinfo` (POST `_buildinfo` with spec for `osc build`), `GET|POST /build/:p/:r/:a/_builddepinfo`, `_jobhistory` | proxy / BE |
| Build log | `GET /build/:p/:r/:a/:pkg/_log` (`nostream`, `start`, `end`, `last`, `lastsucceeded`, `view=entry`); WebUI live log (`/package/live_build_log/...`, `update_build_log` polling via `BuildLogSupport`) | proxy / BE |
| Binaries | `GET /build/:p/:r/:a/:pkg` (list), `GET .../:filename` (`view=fileinfo|fileinfo_ext|publishedpath`), `_buildenv`, rpmlint.log; `PUT`/`DELETE` only on `_repository` (admin); `POST /build/:p/:r/:a/:pkg` upload (admin), `cmd=copy` (internal) | proxy / BE |
| Project build commands | `POST /build/:p?cmd=wipe|restartbuild|killbuild|abortbuild|rebuild|unpublish|sendsysrq&package&repository&arch` | proxy |
| Published repositories | `GET /published/:p/:r/:a/:binary` (+ `view=ymp`), `/published/:p/:r?view=status|publishedpath` | proxy / BE |
| Download URLs / download-on-demand repos | `Repository#download_url` (BE publishedpath), DoD repos in `_meta` (`<download arch url repotype>`), WebUI `/download_repositories` | DB+BE |
| Worker status / monitor | `GET /worker/_status`, `/build/_workerstatus`, `/status/workerstatus`, `GET /worker/:worker` (capabilities), `POST /worker?cmd=checkconstraints`; WebUI `/monitor`, `/monitor/update_building`, `/monitor/events` (StatusHistory graphs) | BE (cached 30 s) + DB history |
| Dispatch priorities | `GET|PUT /build/_dispatchprios` (admin) | proxy |
| Job history / rebuild times | `GET /build/:p/_jobhistory`, `/build/:p/:r/:a/_jobhistory`; WebUI `/projects/:p/rebuild_time` | BE |
| Kiwi image editor | WebUI `/kiwi_images/:id`, `/kiwi_images/:id/build_result`, `autocomplete_binaries`, `/package/:id/kiwi_images/import_from_package` | DB (`kiwi_images`, `kiwi_package_groups`, `kiwi_repositories`...) + BE (`.kiwi` file read/write via `Package#save_file`, `_availablebinaries`) |
| Image templates / package templates | `GET /image_templates`, `/public/image_templates`, `/public/package_templates`; WebUI `/image_templates` | DB (projects carrying `OBS:ImageTemplates` / `OBS:PackageTemplates` attribute) |
| Cloud upload | **removed** (`grep -ri "cloud_upload\|ec2\|azure" app/{models,controllers,jobs}` → none) | — |
| Status reports / required checks (CI gating) | `/status_reports/{projects,repositories,built_repositories}/.../required_checks`, `POST|GET /status_reports/{published,built}/:p/:r[/:a]/reports/:uuid`, `/status_reports/requests/:number/reports` | DB (`status_reports`, `checks`); events `StatusCheckForBuild/Published/Request` | the backend passes `buildid` in `REPO_*` events so external CIs can attach checks |
| Badges | WebUI `/projects/:p/packages/:pkg/badge.svg` | BE (`Buildresult`) |

## B.4 Requests & reviews

| Feature | API routes | Where |
|---|---|---|
| Create / list / show | `POST /request?cmd=create`, `GET /request` (`view=collection`, filters: `user`, `group`, `project`, `package`, `states`, `types`, `roles`, `ids`, `limit`, `withhistory`, `withfullhistory`), `GET /request/:id` | DB (`bs_requests`, `bs_request_actions`, `reviews`, `history_elements`, `bs_request_action_accept_infos`) |
| Action types | submit, delete, add_role, set_bugowner, change_devel, maintenance_incident, maintenance_release, release (`bs_request_action_*.rb`) | DB; accept executes BE copies/branches (submit: `cmd=copy&withacceptinfo` or `cmd=branch`; maintenance_release: `release_package`; delete: backend delete via model destroy) |
| Commands | `POST /request/:id?cmd=addreview|approve|assignreview|cancelapproval|changereviewstate|changestate|setacceptat|setincident|setpriority|diff` | DB (+ BE diff via `cmd=diff` POSTs on sources, cached by `BsRequestActionWebuiInfosJob`) |
| Reviews | `by_user`, `by_group`, `by_project`, `by_package`; review state machine; `ReviewWanted`/`ReviewChanged` events | DB |
| Auto-accept (`accept_at`), auto-cleanup requests, superseding, priorities, `ignore_build_state`, `ignore_delegate` | `BsRequestAutoAcceptJob`, `ProjectCreateAutoCleanupRequestsJob` | DB |
| Request build-status check on accept / `lastsuccess` | `PackageBuildStatus` (BE `_result`, `_builddepinfo?view=pkgnames`) | BE |
| Request search | `GET /search/request[/id]` | DB (XPath → SQL) |
| WebUI request pages | `/request/show/:number`, `/requests/:number/{build_results,changes,mentioned_issues}`, `changes_diff/*filename`, inline comments, chart build results, watch, "seen by" | DB + BE diffs |
| Labels on requests/projects/packages, label templates, canned responses, assignments | `/labels/...`, `/label_templates[/projects/:p]`, `/canned_responses`, `/assignments/projects/:p[/packages/:pkg]` | DB |

## B.5 Staging workflows

| Feature | API routes | Where |
|---|---|---|
| Workflow lifecycle | `POST/PUT/DELETE /staging/:workflow_project/workflow` (managers group) | DB (`staging_workflows`) + BE (`PUT _project/_staging_workflow`) |
| Staging projects | `GET/POST /staging/:wp/staging_projects`, `GET .../:name`, `POST .../:name/copy/:new`, `POST .../:name/accept` (`StagingProjectAcceptJob`/`StagingProjectCopyJob`) | DB + BE (copy = `cmd=copy` project; accept = request accepts) |
| Staged / excluded requests, backlog | `GET/POST/DELETE .../staged_requests`, `/excluded_requests`, `GET /backlog` | DB + BE (`_link` creation in staging project, `Staging::StagedRequests#create_link`); build status via `failed_results` |
| WebUI | `/staging_workflows/:wp`, staging project pages with build/openQA/checks overview | DB+BE |

## B.6 Maintenance

| Feature | API/WebUI | Where |
|---|---|---|
| Maintenance projects (`kind=maintenance`, `<maintains>`), incidents (`kind=maintenance_incident`, counter `maintenance_incidents`), release projects (`kind=maintenance_release`) | `_meta`, `cmd=createmaintenanceincident`, `/projects/:p/maintenance_incidents`, `/projects/:p/maintenance_incident_requests`, `/statistics/maintenance_statistics/:p` | DB (+ `_meta` write-through) |
| Patchinfo | `_patchinfo` file; `cmd=createpatchinfo`, `cmd=updatepatchinfo`; WebUI `/patchinfo/*`, `/my/patchinfos` | BE file + DB-derived issues (`Patchinfo` model parses XML, `IssueTracker` resolution) |
| Channels | `_channel` files, `cmd=importchannel|addchannels|enablechannel|modifychannels`, `/search/channel[/binary[/id]]` | DB (`channels`, `channel_targets`, `channel_binary_lists`, `channel_binaries` parsed from `_channel`) + BE file |
| Release / updateinfo ids | `cmd=release` (project/package), `MaintenanceHelper#release_package` (`setupdateinfoid` via `UpdateinfoCounter`/`IncidentUpdateinfoCounterValue`), `Token::Release`, `maintenance_release_repositories_lifetime` (`PublishedRepositoriesCleanupJob` → `wipepublishedlocked`) | DB counters + BE copies |
| Released binaries tracking | `/search/released/binary[/id]` | DB (`binary_releases` from PACKTRACK) |
| Update projects / instantiate / `OBS:UpdateProject`, `OBS:MakeOriginOlder`, `OBS:BranchTarget`, `OBS:Maintained`, `OBS:MaintenanceProject`, `OBS:RejectRequests`, `OBS:AutoCleanup`, `OBS:RequestCloned`, `OBS:ApprovedRequestSource`... | attribute-driven behaviour in `BranchPackage`, `MaintenanceHelper`, request models | DB attributes + BE `cmd=copy&instantiate|makeoriginolder|vrevbump` |

## B.7 Search, statistics, status, misc

| Feature | API routes | Where |
|---|---|---|
| XPath search | `GET|POST /search/{project,package,person,request,issue,channel,repository,owner,missing_owner,attribute}[/id]` (`match=`, `limit`, `offset`, `CONFIG['limit_for_search_results']`) | DB (`lib/xpath_engine.rb` compiles XPath to SQL over attribs/packages/projects/users/repositories/issues/requests/channels) |
| Published/binary search | `/search/published/{binary,repoinfo,pattern}/id`, `/search`, `/published?view=collection` | BE (proxy) |
| Owner search (bugowner/maintainer by binary) | `/search/owner?binary=|user=|group=` (`OwnerSearch::*`, `OBS:OwnerRootProject` attribute) | DB + BE (`/published?view=collection`) |
| Statistics | `/statistics[/most_active_projects|most_active_packages|latest_added|latest_updated|global_counters|activity/:p|added_timestamp|updated_timestamp|active_request_creators]` | DB |
| Status messages / news | `GET/POST/PUT/DELETE /status/messages`, `/status_message`; WebUI `/news_items`, acknowledgements | DB |
| Announcements | `resources :announcements` | DB |
| Status of project (`/status/project/:p`) | `ProjectStatus::Calculator` | DB + BE jobhistory |
| Issue trackers | `GET/POST/PUT/DELETE /issue_trackers[/:name]`, `/issue_trackers/:name/issues/:id` | DB + BE (`PUT /issue_trackers` sync; Bugzilla/GitHub fetch jobs) |
| Architectures | `GET /architectures`, `GET/PUT /architectures/:name` (availability) | DB |
| Distributions | `GET/POST/PUT/DELETE /distributions[/:id]`, `include_remotes`, `refresh`, `bulk_replace` | DB (+ remote fetch) |
| Configuration | `GET/PUT /configuration`, `GET /public/configuration` | DB + BE push |
| About / version | `GET /about` (`X-Opensuse-APIVersion` header on every response) | DB |
| Comments | `GET/POST /comments/{request,package,project}/...`, `GET /comments/user`, `PUT/DELETE /comment/:id`, `/comment/:id/history` (versioned via PaperTrail); WebUI inline diff comments, moderation, locks | DB |
| Notifications / subscriptions / RSS | `GET /my/notifications`, `PUT /my/notifications/:id`, WebUI `/my/notifications`, `/my/subscriptions` (per event class & receiver role & channel instant_email/web/rss/scm), `/user/feed/:secret.rss`, `/main/news.rss`, `/main/latest_updates.rss`, `/project/latest_commits/:p.atom` | DB |
| Watchlist | `PUT /person/:login/watchlist`, WebUI `toggle_watched_item` for project/package/request | DB |
| Triggers / webhooks (`obs workflow`) | `POST /trigger` (+`/webhook`), `/trigger/rebuild`, `/trigger/release`, `/trigger/runservice`, `/trigger/workflow` (GitHub/GitLab/Gitea events → `Workflow` steps `branch_package`, `link_package`, `rebuild_package`, `trigger_services`, `set_flags`, `configure_repositories`, `submit_request`; `workflow_runs`, artifacts, status reporting back to SCM via `ReportToScmJob`) | DB + BE (branch/copy/link/rebuild/runservice) |
| Service list | `GET /service` | proxy |
| Reports / moderation | B.1 | DB |
| Feature toggles | Flipper (`/flipper`, `/my/beta_features`, `/global_feature_toggles`) | DB |
| Sitemaps, code of conduct, sponsors, themes | WebUI only | DB |
| Vendors / distros / releases / lifecycles | WebUI `/projects/:p/vendor`, `/distros`, `/releases`, `/lifecycles` | DB |
| Package versions (Anitya upstream vs local) | `GET /source/:p[/:pkg]?view=versions`, `SyncUpstreamPackageVersionJob`, `SyncLocalPackageVersionJob` | DB + BE (`view=info&parse=1`) |

## B.8 Summary of what the public API exposes that is pure backend passthrough

Everything under `/build/**` (except `/build` itself and `?lastsuccess`), `/published/**`, `/source/:p/:pkg/:file`, `/source/:p/_config`, `/source/:p/_pubkey`, `/source/:p/_keyinfo`, `/source/:p/:pkg/_history`, `cmd=diff|linkdiff|servicediff|commit|commitfilelist|copy|getprojectservices|collectbuildenv|waitservice|mergeservice|runservice|deleteuploadrev|orderkiwirepos|triggerscmsync`, `/search/published/**`, `/worker/**`, `/service`, `/lastevents`, and `/public/**` mirrors of the above. For all of these the XML wire format is defined by `BSXML.pm` (see D) and documented in `docs/api/api/*.rng` and `public/apidocs/paths/*.yaml`; a Go backend must keep those byte-level compatible because osc parses them directly.


---

# C. Source service server (bs_service) and the service pipeline

Files: `src/backend/bs_service` (354 lines), `src/backend/bs_servicedispatch` (212), `src/backend/BSSrcServer/Service.pm` (748, client side inside bs_srcserver), `src/backend/call-service-in-container` (251, bash/podman), `src/backend/run-service-containerized` (240, perl/docker, older), `dist/system-user-obsservicerun.conf`, `dist/setup_source_service_docker.sh`, frontend `src/api/app/models/service.rb`.

## C.1 Roles and data flow

```
 user commit (osc / webui / token)                     ┌──────────────────────────────┐
   │ PUT file / cmd=commit / cmd=runservice            │ bs_service  (:5152, obsservicerun) │
   v                                                   │  POST /sourceupdate/$p/$pkg    │
 bs_srcserver addrev ── genservicemark ──> runservice ─┤  cpio in  → run each service   │
   │   (BSSrcServer/Service.pm)                       │  cpio out → _service:* files   │
   │ if $BSConfig::servicedispatch:                    └──────────────────────────────┘
   │   write $bsdir/events/servicedispatch/<ev>  ── ping ──> bs_servicedispatch (BSStdRunner, maxchild 4)
   │                                                        │ runservice() → doservicerpc() → service server
   │                                                        │ addrev_service() → commit result tree "servicemark"
   │                                                        └ POST /source/$p/$pkg?cmd=notifypackagechange (→ repservers)
   └ else: fork a child in srcserver that does the same synchronously.
 Result: new revision whose expanded tree = sources + `_service:<name>:<file>` outputs (or `_service_error`);
         notifications SRCSRV_SERVICE_SUCCESS / SRCSRV_SERVICE_FAIL to the frontend (Service.pm:122-141).
```

## C.2 bs_service — the server (src/backend/bs_service)

* Runs as `$BSConfig::bsserviceuser/bsservicegroup` with `$BSConfig::bsdir` rebased to `$bsdir/service` (`log`, `run` beneath it) (lines 46-50). Port from `$BSConfig::serviceserver` URL (default 5152, `https` honoured) (57-60). `BSStdServer::server('src_service', ...)` with `maxchild = $BSConfig::service_maxchild`, `setkeepalive`, extra conf via `$BSConfig::serviceserver_extraconf` (344-354).
* Dispatch table (330-342; same as `routes/bs_service.txt`):

| Route | Handler | Semantics |
|---|---|---|
| `GET /` | `hello` | `<hello name="Source Service Server" />` |
| `GET /service` | `list_service` (301-310) | reads every `*.service` file in `$rootservicedir` (= `$BSConfig::serviceroot/$servicedir`, default `/usr/lib/obs/service`) as `$BSXML::servicetype` and returns `$BSXML::servicelist` (BSXML.pm:1644-1659: `<service name hidden><summary/><description/><parameter name><description/><required/><allowmultiple/><allowedvalue/>...`). Proxied to users by srcserver `/service` → frontend `GET /service`. |
| `GET /serverstatus` | `BSStdServer::serverstatus` | process slots |
| `POST /sourceupdate/$project/$package $timeout:num? noprefix:bool?` (marked `!-`: no auth needed) | `run_source_update` (131-294) | the only real endpoint; body is a cpio archive |
| `PUT /configuration` | `putconfiguration` (312-325) | receives the OBS `<configuration>` from srcserver (fan-out of frontend `PUT /configuration`); stored in `$bsdir/service/configuration.xml`; used for `api_url`/`obsname` env |

### `run_source_update` step by step
1. `timeout ||= $BSConfig::service_timeout` (137); work dir `$BSConfig::servicetempdir/$$` wiped and recreated with `src/` (139-144); `BSServer::read_cpio("$myworkdir/src")` unpacks the request (147). Must contain `_service` or `_serviceproject` (149).
2. Files from previous runs matching `/^_service[:_]/` are moved into `.old/` so services can reuse them (152-157).
3. Environment for the service processes (160-167): `OBS_SERVICE_PROJECT`, `OBS_SERVICE_PACKAGE`, `OBS_SERVICE_APIURL` (= `$BSConfig::api_url`), `OBS_NAME` (= `$BSConfig::obsname`), `OBS_SERVICE_DAEMON=1`, `http_proxy`/`https_proxy`/`no_proxy` from `$BSConfig::proxy`/`noproxy`.
4. Service selection (170-184): parse `_service` then `_serviceproject` with `$BSXML::services`; **skip** any `<service>` whose `mode` is `localonly`, `disabled`, `manual` or `buildtime`; everything else (no mode, `trylocal`, `serveronly` or any unknown value) runs server-side. `verify_filename` on the service name.
5. For each service (190-272): definition from `$rootservicedir/<name>.service` (`$BSXML::servicetype`), except the built-in `obs_scm_bridge` definition (70-87: params `url`, `projectmode`, `projectscmsync`). Command line = optional wrapper (`$BSConfig::service_wrapper->{$name}` or `->{'*'}`, e.g. `call-service-in-container`) + `$servicedir/$name` + `--<param> <value>` for every `<param name>` with content (the `outdir` param is ignored; **a param not declared in the `.service` file is a hard error** 209) + `--outdir $myworkdir/out`. Executed via `open(SERVICE,'-|')`+`exec` with stderr→stdout captured; `alarm($timeout)` kills with TERM on timeout → `500 timeout while execution of $name`.
6. Success: every plain file in `out/` (symlinks/dirs ignored, names starting with `:` or `.` skipped) is renamed to `_service:<name>:<basename>` (a leading `_service:` is stripped first) unless `noprefix` (246-255). Failure: `cleandir('.')`, exit status 3 → `500 remote execution error` (used by the container wrapper for "podman itself failed" = transient), otherwise `_service_error` file with "service <name> failed:\n<output>" and the loop stops (256-271).
7. `.old` removed, reply is a cpio of files matching `/^_service[_:]/` (or all files with `noprefix`), refusing symlinks/dirs (277-288); `BSServer::reply_cpio` streams it chunked with `.errors` collection.

## C.3 The `_service` file and the `mode` attribute

Schema `$BSXML::services` (BSXML.pm:1622-1633): `<services><service name="..." mode="..."><param name="...">value</param>...</service></services>`. The frontend validates names/params with `Service::NameValidator` (models/service.rb:19-27) when writing `_service`.

Where each `mode` value is interpreted (grep over Service.pm:92,98 and bs_service:177):

| mode | server side (bs_service / genservicemark) | client (osc) | build time |
|---|---|---|---|
| *(absent)* / `default` | runs on the server after every commit; result files `_service:<name>:*` are committed as the expanded tree; `genservicemark` counts it as "active" | osc runs it on checkout? no — osc only runs `trylocal`/`localonly` | no |
| `trylocal` | treated as active server-side (not in the skip list) | osc tries locally first and commits results | no |
| `localonly` | skipped (and not "active") | osc runs locally; results committed as regular files | no |
| `serveronly` | runs server-side (not in skip list) | osc never runs it | no |
| `buildtime` | skipped | — | run by `build`/worker inside the build root (service package must be installed there, `tar`, `recompress`, `set_version` in the scm flow, service.rb:122-129) |
| `manual` | skipped | only when `osc service manualrun` | no |
| `disabled` | skipped | osc skips | no |

If no active service exists (`genservicemark` returns undef, Service.pm:76-119) no servicemark is created and the commit is final immediately.

## C.4 Client side in bs_srcserver — BSSrcServer/Service.pm

* **servicemark** (76-119): per commit with an active service (`_service` in package or any project-level `_service` via `getprojectservices`), an md5 over "sourceservice/$proj/$pack/time + previous rev + file md5s" is generated and stored in the revision as `/SERVICE`; the expanded tree of the package is keyed by this mark until the service finishes (`BSSrcrep::getserviceerror` returns "service in progress" → readers of expanded sources get a 4xx "service in progress", handleservice 637-682). Disabled with `$BSConfig::old_style_services` (legacy lockfile `$eventdir/service/<p>::<pkg>` instead).
* `getprojectservices` (684-746): collects `_service` from the project's `_project` package (schema `$BSXML::services_link`, may contain `<link project>` to follow), then recursively from project links and the package's `_link` target; loop-protected. The merged list is sent as `_serviceproject` in the cpio (so project-wide services run on every package).
* `runservice` (470-607): called after `addrev`; skips `_project`, `rev=repository|upload`; serialises per servicemark (`BSSrcrep::addmeta_serialize_servicerun`); collects previous run outputs (`oldfiles`, up to the preceding commit) so the service can diff; expands `_link` (`$handlelinks`) so services see the link-expanded sources (`lxservicemd5`); if `$BSConfig::servicedispatch` → writes a `servicedispatch` event (`$BSXML::event` with `job=servicemark srcmd5 rev linksrcmd5 projectservicesmd5 oldsrcmd5`, 326-352) and pings `$eventdir/servicedispatch/.ping`; else forks and calls `doservicerpc` inline (requires `$BSConfig::serviceserver`).
* `doservicerpc` (383-468): `POST $serviceserver/sourceupdate/$p/$pkg?timeout=$service_timeout[&noprefix=1]`, chunked `application/x-cpio` body built by `BSHTTP::cpio_sender` from `revcpiofile` entries, response unpacked with `BSHTTP::cpio_receiver` into `$uploaddir/runservice$$` with md5s. Error classes: transient (`5xx`, `rpc timeout`, ENOSPC/EMFILE...) → `die "Transient error"` (retried by the dispatcher in 10 min, bs_servicedispatch:188-193); RPC error; "service daemon error". Result validation: with prefix, every returned file must match `^_service[_:]`; with `noprefix` (obs_scm_bridge) `_link`/`_meta` are forbidden and `_service_*` files may only update known rev fields (`_service_info`); `BSVerify::verify_filename`. New files are added with `BSSrcrep::addfile`; for `_project` obsscm runs a cpio is kept (`writeresultascpio`).
* `addrev_service` (202-245, read earlier via grep only) commits the result tree under the servicemark and calls `notify_serviceresult` → `SRCSRV_SERVICE_SUCCESS`/`FAIL` (`project package rev user comment requestid [error]`) which the frontend turns into `Event::ServiceSuccess/ServiceFail` (email to maintainers/bugowners, `package.service_fail` bus key) and `UpdateBackendInfosJob`.
* scmsync flow: `runservice_obsscm`/`generate_obs_scm_bridge_service` (269-324) synthesise a `_service` running `obs_scm_bridge` with `url`, `projectmode`, `projectscmsync`; results are committed via `cmd=commitobsscm` (bs_servicedispatch:51-77) with `servicemark`+`run` ids and optional cpio body.
* `servicemark_noservice` (609-635): `noservice=1` uploads fake a service run so that `_service:*` files in the upload are taken as the result.

## C.5 bs_servicedispatch (src/backend/bs_servicedispatch)

A `BSStdRunner` event-loop daemon (`BSStdRunner::run('servicedispatch', ...)`, 198-212): event dir `$bsdir/events/servicedispatch`, `maxchild = $BSConfig::servicedispatch_maxchild` (default 4), `inprogress` marker files `servicedispatch:<p>::<pkg>::<srcmd5>::<mark>::inprogress`, optional per-project concurrency limit `$BSConfig::servicedispatch_limitinprogress` (157-169). Event pattern `servicedispatch $project $package $job $srcmd5 $rev? $linksrcmd5? $projectservicesmd5? $oldsrcmd5?` → `runservice` (105-155): re-reads the revision (`getrev`, also deleted-srcmd5 trees), checks the servicemark still says "service in progress" (idempotent), builds the cpio list (expanded link tree if `linksrcmd5`, project services file, old outputs), calls `BSSrcServer::Service::doservicerpc` and `addrev_service`, then `notify_repservers` → `POST $srcserver/source/$p/$pkg?cmd=notifypackagechange` (42-49) so the scheduler re-evaluates. Statistics line `servicedispatch statistics: <job> <p>/<pkg>/- <code> <create>-<start>-<end>` (171-178). Hooks: `$BSSrcServer::Service::notify = \&BSNotify::notify` (24-26) — i.e. the dispatcher sends the SERVICE_* notifications itself via `/notify/`.

## C.6 Container isolation wrappers

`call-service-in-container` (bash, podman; configured as `$BSConfig::service_wrapper->{'*'}`):
* reads config with `obs_admin --query-config container_image|container_custom_opt|containers_root|servicetempdir|gems_mirror|obs_service_user|obs_service_pass|obs_service_network|geminabox_container|container_network_opt` (43-50, 76, 182).
* network policy by service name (71-97): `download_url download_src_package update_source download_files generator_pom snapcraft kiwi_import appimage node_modules cargo_vendor go_modules bundle_gems` → network on; `tar_scm obs_scm download_assets` → network + SCM cache; `obs_scm_bridge` → network + `/etc/obs/services/scm-bridge` config + CA certs; everything else `--network none`.
* argument handling (99-125): `--outdir` is intercepted, all other `--k v` pairs are single-quoted (quotes replaced by `_`); `--scm`/`--url` captured to derive a per-URL `scm-cache/<sha256(url)>` volume (190-198).
* layout: `MOUNTDIR=dirname(outdir)` (= `$servicetempdir/<pid>`), inner base `/var/cache/obs/XXXXXXXX` with `out/`, `src/`, `scripts/inner.sh(.command)`, `home/`; volumes mounted rw except scripts; container name `src-service-<id>`; `podman run --userns keep-id ... --rm $CONTAINER_IMAGE inner.sh` (222) with optional `--root $CONTAINERS_ROOT --storage-opt overlay.mount_program=fuse-overlayfs` (214-216); env exported into `inner.sh` (158-168): `OBS_SERVICE_APIURL`, `OBS_SERVICE_BUNDLE_GEMS_MIRROR_URL`, `OBS_SERVICE_DAEMON`, `OBS_SERVICE_USER/PASS`, `OBS_SERVICE_PROJECT/PACKAGE`, `OBS_NAME`, `HOME`, `CACHEDIRECTORY`.
* exit codes (226-243): 0 ok (stale `_service:*` in cwd that were not in `src/` are removed); podman failure 125/126/127 → exit 3 (bs_service treats 3 as transient "remote execution error" → retried); any other failure → 2 with output. SIGTERM (bs_service timeout) → `cleanup_container timeout` → `podman rm --force`.

`run-service-containerized` is the older Perl/docker equivalent (`docker run -u 2:2 --rm ... --net=bridge|none`, `$BSConfig::docker_image`, `docker_custom_opt`, `gems_mirror`, `geminabox_container`; 36-62, 182-186) with the same exit-code contract (221). `dist/system-user-obsservicerun.conf` creates user `obsservicerun` (home `/srv/obs/service`, member of `obsrun`); `dist/setup_source_service_docker.sh` provisions the container image/LVM for a dedicated service host.

## C.7 obs-service-* plugin contract (what a Go service runner must honour)

1. Discovery: `/usr/lib/obs/service/<name>` executable + `/usr/lib/obs/service/<name>.service` XML (`$BSXML::servicetype`) declaring every parameter; undeclared params are rejected before execution. `GET /service` lists them (hidden="true" hides from UIs).
2. Invocation: `<name> --<param> <value> ... --outdir <dir>` in a cwd containing the package sources (and `.old/` with previous `_service:*` outputs), environment variables listed in C.2 step 3, no stdin, stdout+stderr captured, hard timeout `$BSConfig::service_timeout`.
3. Output: only regular files in `--outdir`; they are renamed `_service:<name>:<file>`; `.`/`:`-prefixed files ignored; non-zero exit → `_service_error`; exit 3 → transient.
4. Chaining: services run in `_service` order (then `_serviceproject`), each seeing the previous services' outputs already prefixed in cwd (`_service:tar_scm:foo.tar` → `recompress` → `_service:recompress:foo.tar.xz` ...).
5. Result semantics in the source repo: the expanded tree of a revision with a servicemark = original files + `_service:*` files (or `_service_error`); `osc` shows them as generated; a later commit without changes to inputs can reuse the old servicemark (`can_reuse_oldservicemark`).

## C.8 Frontend handling of `_service`

* `Service` model (src/api/app/models/service.rb): ActiveModel wrapper over the backend file: `document` = `GET /source/:p/:pkg/_service` (Nokogiri, strict); `add_download_url` (download_url / download_src_package; `git@`/`.git` → `obs_scm` + buildtime `tar`/`recompress`/`set_version`, 30-58, 122-129); `add_kiwi_import` (writes, then `cmd=waitservice` + `cmd=mergeservice`, 60-70); `add_service(name, params, mode)`; `save` (81-100): refuses scmsync packages, deletes `_service` when empty else `PUT _service?comment=Modified via webui&user=` then `cmd=runservice` and `Package#sources_changed`.
* `Package#save_file` auto-adds `kiwi_import` for `*.kiwi.txz` uploads (package.rb:1248-1253); uploads of `_service` are verified (`Service.verify_xml!`) and wait synchronously for backend info refresh (`sources_changed(wait_for_update: true)`).
* Triggers: `Token::Service` (`POST /trigger/runservice`, SCM webhooks), `Webui::Packages::TriggerController#services`, `Workflow::Step::TriggerServices`, `Workflow::Step::BranchPackageStep`/`SubmitRequest` (`waitservice` before creating requests) — all map to `cmd=runservice` / `cmd=waitservice`.
* Service errors surface via `Package#service_error` (reads `_service_error` through `PackageServiceErrorFile`) and `Event::ServiceFail` notifications; `cmd=servicediff` (proxy) shows the diff produced by a run; `cmd=getprojectservices` (proxy) exposes the merged project service list; `cmd=mergeservice` turns generated files into real sources (used by the "kiwi import" and by users to "disable services").


---

# D. Infrastructure modules shared by every backend daemon

This first part covers the server/client/HTTP/util/auth/notify modules read directly. The three following parts (written as separate documents, concatenated below) cover BSXML/XML::Structured, BSVerify + BSDispatch + BSConfiguration/BSConfig, and the admin tools/dist packaging.

## D.1 BSServer.pm — the forking HTTP server (src/backend/BSServer.pm, 1000 lines)

Model: one master process per daemon, `fork()` per accepted connection, the child handles exactly one request (or a keep-alive sequence) and `exit(0)`s. No threads, no event loop in the main server; blocking work is handed off to a separate single-process "AJAX" event server (D.2/D.6).

| Function | Lines | Behaviour |
|---|---|---|
| `deamonize(@args)` | 70-80 | `-f` → fork and exit parent; `setsid`, ignore SIGPIPE, autoflush |
| `serveropen($port,$user,$group,$family,$bindaddr)` | 84-144 | creates 1 or 2 listening TCP sockets (`"port,port2"`; a port may be `&N` = inherited fd for `--restart`, or a scalar ref for ephemeral); IPv6 autodetect via `getaddrinfo`; `SO_REUSEADDR`, `listen 512`; then **drops privileges** (`BSUtil::drop_privs_to`) and records `starttime` |
| `serveropen_unix($path)` | 146-170 | unix socket + `$path.lock` flock (used by AJAX servers) |
| `setstatus / serverstatus / serverstatus_str` | 216-263 | per-slot 256-byte records in the `serverstatus` file (`$rundir/<name>.status`): `N starttime, N pid, C group, C extra, n state(1=F forked,2=R running,3=K keepalive), Z244 request`; `bs_serverstatus` and `/serverstatus` read it |
| `maxchildreached` | 272-296 | logs when `maxchild`/`maxchild2` is hit with a dump of the status table (slow-request log) |
| `server($conf)` | 328-579 | main loop: `select` on listen sockets (only while below `maxchild` / `maxchild2`; second socket = "group 1", e.g. worker port with its own limit), `periodic` callback every `periodic_interval` (default 3 s; BSStdServer sets 1 s) used for exit/restart files, config reload, memoize; reaps children (blocking `waitpid` when at the limit); child: closes listeners, writes status slot, optional `accept` hook, TLS `BSSSL::tossl` accept if `proto=https` (needs `ssl_keyfile`/`ssl_certfile`, optional client verify), `readrequest` with a 60 s alarm, calls `$conf->{dispatch}` and `stdreply`/`reply`, error → `reply_error`; keep-alive: if the handler set `allow_keepalive` and the client did not send `Connection: close`, waits `keepalive_maxidle` (10 s) / `keepalive_maxage` for the next request on the same socket (`goto redo_keepalive`, 555-575) — used only by `/notify/redis` |
| `reply($str,@hdrs)` | 589-646 | writes `HTTP/1.1 <status>` (from a `Status:` pseudo-header, default 200), adds `Cache-Control: no-cache`, `Connection: close` unless keep-alive, `Content-Length`; discards unread request body; **drains the client read side before writing** (Linux RST workaround) |
| `parse_error_string / reply_error` | 648-723 | errors are perl `die` strings: `"404 not found\n"` → status 404, tag "not found"; no numeric prefix → 400 (500 for EMFILE/ENOSPC/EAGAIN); `errorreply` hook renders `<status code summary details>`; 401 adds `WWW-Authenticate` from `conf->{wwwauthenticate}` |
| `readrequest` | 771-839 | parses request line (GET HEAD POST PUT DELETE PATCH), HTTP/0.9 GET, headers via `BSHTTP::gethead`, `%XX`-unescapes path; `Expect: 100-continue` support; **POST with `application/x-www-form-urlencoded` body is appended to the query string** (this is why BSRPC `formurlencode` works and why form POSTs can't be chunked); otherwise body left for `read_data`/`read_file`/`read_cpio` |
| `have_content / get_content_type / header` | 846-862 | |
| `read_file($fn)`, `read_cpio($dir)`, `read_data($max,$exact)` | 884-908 | stream request body to file / unpack cpio to dir / read bytes (chunked-aware) — see BSHTTP |
| `reply_stream($sender,$param,@hdrs)`, `reply_cpio($files)`, `reply_file($file,@hdrs)` | 910-962 | streaming replies; `reply_cpio` = chunked `application/x-cpio` with `.errors` collection; `reply_file` auto Content-Length for plain files else chunked |
| `reply_receiver($req,$param)` | 964-989 | BSRPC `receiver` that **forwards a backend answer verbatim to our client** (status/content-type/length/chunked preserved) — how srcserver proxies repserver answers |
| `forward_sender` | 991-999 | BSRPC `data` sender that streams our request body to another server (proxy uploads) |
| `getpeerdata`, `getpeerfingerprint`, `getsubjectdn` | 733-769 | peer IP (IPv4-mapped handled), TLS client cert fingerprint/DN |

## D.2 BSStdServer.pm — standard daemon wrapper (src/backend/BSStdServer.pm, 614 lines)

`BSStdServer::server($name, \@ARGV, $conf, $aconf)` (406-612) is what every daemon calls. It:
* handles CLI: `--logfile F` (reopen stdout/stderr, `BSUtil::openlog`), `--test` (compile dispatch tables and exit — used by `periodic` before restart), `--stop|--exit` and `--restart` (touch `$rundir/<runname>.exit|.restart` and wait; `isrunning` probes by trying to bind the port), `--req 'METHOD:/path?query' [--req-content file]` (run one request inline without a socket, `req_mode`, as `bsuser`; `AJAX:` prefix runs it through the AJAX dispatch) (417-460, 519-556).
* fills defaults (469-516): `verifiers = $BSVerify::verifiers`, `dispatch = \&dispatch`, `stdreply`, `errreply`, `authorize`, `periodic` (1 s), `serverstatus = $rundir/<runname>.status`, `setkeepalive`, `slowrequestlog = $logdir/<name>.slow.log` if `slowrequestthr`, `critlogfile = $logdir/<name>.crit.log`, TLS files from `$BSConfig::ssl_keyfile/ssl_certfile/ssl_verify`, `bindaddress = $BSConfig::global_bindaddress`; `BSDispatch::compile` builds the route matcher. AJAX conf: `socketpath`, `dispatches_call = BSWatcher::dispatches_call`, `getrequest_recvfd = BSHandoff::receivefd`, timeouts `getrequest 10 s`, `replrequest 60 s`, `replstream 120 s`, `run = BSEvents::schedule`.
* starts: `deamonize`, `serveropen` (with `--restart` fd inheritance), forks one AJAX child per partition (`aconf->{partitions}` → sockets `<socketpath><idx>`, `ajaxpartitionmap` maps project prefixes to partitions), sets the critical logger, pre-warms XML::Structured, logs "started on port", optional `memoize` list file, then `conf->{run}` (= `BSServer::server`).

Other pieces:
* `stdreply(@rep)` (50-63): handler return conventions — `($hashref, $dtd)` → `XMLout`; `($hashref, \&code)`; `($string, 'Content-Type: ...')`; a lone string gets `text/xml`. `$return_ok = <status code="ok" />`.
* `errreply` (65-75): `<status code="N"><summary>tag</summary><details>err</details></status>` via `$BSXML::opstatus`.
* `authorize($conf,$req,$auth)` (77-106): only active with `$BSConfig::subjectdnaccess` (TLS client DN regex → comma list of rights; or `trusted_subjectdn_header`) or `$BSConfig::ipaccess` (peer IP regex → rights, e.g. `'127\..*' => 'rw'`, `'.*' => 'worker'`); route tags like `!rw` require one of the listed rights; `_reject` denies. Otherwise every request is allowed.
* `dispatch` (108-155): logs `METHOD (peer) path?query`, honours `trusted_peerip_header` (X-Forwarded-For style) and `client_time_header`, captures `x-request-id` into the log line and `$BSRPC::autoheaders` (propagated downstream), writes the status slot, and in the AJAX process clones the connection (`BSServerEvents::cloneconnect`) before calling `BSDispatch::dispatch`.
* `periodic` (159-201): exit/restart handling (restart = `exec $0 --restart <fds>` after a successful `--test`), file memoization refresh, `BSConfiguration::check_configuration()` every ~10 s (re-reads `configuration.xml` from the frontend).
* `periodic_ajax` (203-222): the AJAX child exits when the parent's lock file disappears.
* memoization (224-325): `memoize_files` keeps hot files (`memoize` list) in RAM keyed by `mtime/size/inode`; `readstr_memoized`, `readxml_memoized`, `retrieve_memoized` used by repserver/srcserver for project XML / `:full` metadata.
* `serverstatus` route handler (327-345): `$BSXML::serverstatus` (`<serverstatus starttime><job id starttime pid request group/>`).
* `setup_authenticator` (365-373): installs `BSSigAuth` signature-based authenticators (`$BSConfig::signature_auth_keyfile{'user@host'}`; `BSSSHSign` for ssh keys) into `$BSRPC::authenticator`.
* `run_ajax_server` (375-392): unix socket server + `BSServerEvents::addserver`.

## D.3 BSDispatch.pm — route table syntax
Documented in the "BSDispatch route syntax" section of the verify/config part below (`$x` positional path parts, `name:type?`/`*` cgi params with BSVerify-typed validation, `cmd=foo` discriminators, `!rw`/`!-` auth tags, `GET|HEAD:` method prefixes, `$...` rest-of-path, `$_type` untyped, multi-valued params become arrays).

## D.4 BSRPC.pm — the synchronous HTTP client (src/backend/BSRPC.pm, 696 lines)

`BSRPC::rpc($param_or_uri, $xmlargs, @args)` (373-683). Parameters (comment block 345-371): `uri`, `request` (GET default), `headers` [list of "K: v"], `data` (string, or coderef sender called repeatedly, or with `datafmt` a hash→`XMLout` / coderef), `chunked`, `timeout` (implemented with `alarm` + die "rpc timeout", 379-399), `formurlencode` (POST @args as body), `verbatim_uri`, `socket` (reuse/continuation), `https` (custom tossl), `continuation`/`async` (fire-and-forget: returns a continuation hash with the non-blocking socket; `rpc_isfinished`, later `rpc($cont)` reads the answer — used by the scheduler to talk to many repservers), `verbose`, `sender` (custom request writer, e.g. `BSHandoff::handoffsender`), `replyheaders` (ref to receive `%headers`), `receiver` (`BSHTTP::cpio_receiver`, `file_receiver`, `null_receiver`, `BSServer::reply_receiver`, or custom) + `receiverarg`, `ignorestatus` (1 = return body on non-2xx; 2 = don't follow redirects), `maxredirects` (302/307 followed, auth stripped on host change `_stripauthhost`), `proxy` (http proxy, `CONNECT` tunnel for https, `Proxy-Authorization`), `sslpeerfingerprint` (`sha256:hex` pinning), `ssl_verify`/`ssl_keyfile`/`ssl_certfile`/`ssl_ctx`, `keepalive` (hashref holding socket+cookie reused across calls, 320-343, 460-480), `toomanyrequests` (429 + `Retry-After` sleep/retry), `authenticator` (401 → `call_authenticator` with the `WWW-Authenticate` header, retried once; `$BSRPC::authenticator` global may be a hash keyed by `user@host[:port]` realm, array of callbacks, or code — BSBearer/BSSigAuth plug in here).

Flow: URL-encode (`createuri`, 79-94; host part untouched, `@args` appended as `k=v` query), `createreq` (124-171: `Host`, `User-Agent: BSRPC 0.9.1`, `Connection: close` unless keepalive, Basic auth from `user:pass@` in the URI, cookies from a per-domain `%cookiestore` (iChain legacy, max 10), proxy handling), DNS cache (`lookuphost`, 1 h TTL, IPv4 preferred then IPv6), `opensocket` with `SO_KEEPALIVE`, write request + body (chunk-encoded 64 KiB pieces if `chunked`; sender coderef loop), `readanswerheaderblock`, status handling (2xx ok; bodyless 1xx/204/304/HEAD), then body via `receiver` or `BSHTTP::read_data(...,undef,1)`; finally `XMLin($xmlargs, $ans)` if a schema was given (or `$xmlargs->($ans)` if code). `rpc` dies with `"<code> remote error: <msg> (<uri>)\n"` on failure so callers can pattern-match the status. `BSRPC::args($hash, @keys)` turns a cgi hash into `k=v` list (arrays expanded). `':https'` import loads BSSSL. Globals: `$useragent`, `$noproxy`, `$logtimeout`, `$autoheaders` (e.g. `X-Request-ID`), `$dnscachettl`, `$logrequests`, `$ssl_keyfile/certfile/verify`, `set_clientcert`.

## D.5 BSHTTP.pm — wire helpers (src/backend/BSHTTP.pm, 566 lines)

* `urlencode/urldecode/queryencode/querydecodekv` (56-94): OBS-specific escaping (`%3D` → `=` restored in queryencode; cgi mode uses `+`).
* `gethead(\%h, $text)` (96-117): header folding, lowercase names, duplicates joined with `,`.
* `forwardheaders($req, @except)` (119-133), `makemultipart` (135-147), `parseauthenticate` (155-176: parses `WWW-Authenticate` into scheme → {param → [values]}; used by BSBearer).
* `read_data($req, $maxl, $exact)` (208-305): the universal body reader; `$req->{headers}`, `__socket`, `__data` (already-read bytes), `__cl` (remaining content length; -1 = EOF); handles chunked transfer encoding including trailers (`gethead` into headers), returns '' at EOF; `unexpected_eof` marks `__eof`.
* `str2req`, `fd2req` (307-324): wrap a string/file as a pseudo request for the receivers.
* Receivers (`($req,$param)` → result): `null_receiver`, `file_receiver` (`filename`, `withmd5` → `{size,md5}`), `cpio_receiver` (404-502: parses SVR4 "newc" (`070701`) archives streamed from the socket; options `directory` (write files; else data in memory), `withmd5`, `acceptsubdirs`/`createsubdirs`, `accept` regex|code, `map` prefix|code (rename / skip by returning undef), `no_unlink`, `cpiopostfile` hook (e.g. reset alarm per file); only type-8 regular files (and dirs with createsubdirs); refuses `/`, `.`, `..`, names > 8192; returns `[{name,size,mtime,mode,md5?,data?}...]`).
* Senders (`($param,$sock)`): `swrite($sock,$data,$chunked)` (504-516: hex-length chunk framing; EINTR/EWOULDBLOCK retry), `create_writer`, `cpio_sender` (523-528: `BSCpio::writecpio` of `cpiofiles` entries `{name, file|data, mtime...}` with `collecterrors` → a trailing `.errors` file), `file_sender` (530-553: `filename` path or fd, optional `bytes` limit), `reply_sender` (555-564: copy from another request = proxying).

## D.6 Event-driven side: BSEvents, BSServerEvents, BSWatcher, BSHandoff

* **BSEvents** (189 lines): minimal select()-loop scheduler — `new(type, handler)` (`read`,`write`,`timeout`,`never`), `add($ev,$timeout)`, `rem`, `set_timeout`, `schedule` (never returns), `allevents`.
* **BSServerEvents** (699 lines): a single-process HTTP server on top of BSEvents: `addserver` (unix/tcp listener), `getrequest` (reads a request incl. an fd passed over SCM_RIGHTS via `getrequest_recvfd` = handoff), `reply/reply_error/reply_stream/reply_file/reply_cpio` (non-blocking writers with `replrequest_timeout`/`replstream_timeout`), `cloneconnect` (answer "OK" to the forking parent while keeping the real client socket), `background` (detach a job), `stream_read_handler/stream_write_handler` (pump data between an upstream rpc socket and the client), `periodic_handler`, `concheck_handler` (detect client disconnects), `getrequestevents`. `$BSServerEvents::gev` = the current job event (`$jev`).
* **BSWatcher** (1304 lines) — the AJAX toolkit used inside those handlers:
  - `reply/reply_file/reply_cpio` wrappers that pick BSServerEvents in AJAX mode (50-81);
  - `redo_request($jev)` (83-101): re-runs the handler from scratch (handlers are written re-entrantly: they are called again whenever a watched resource changes and must return `undef` while "not ready");
  - **file watchers** `addfilewatcher($file, $periodic)` (153-181): polls `stat` (mtime/size/inode) every `$filewatchers_interval` (1 s) and re-dispatches every job waiting on the file — this is how `/lastnotifications?block=1`, `/lastevents`, `_log` streaming (`:logfiles.*`), `_jobstatus`, `waitservice` block without holding a process;
  - **serialisation** `serialize($file)`/`serialize_end` (209-270): a per-file mutex among AJAX jobs (first job proceeds, others are parked and re-dispatched when it ends; `serlialize_advance` priority);
  - **async rpc** `BSWatcher::rpc($uri|\%param, $xmlargs, @args)` (1096-1214): same parameters as BSRPC but returns `undef` immediately and registers a write event; on completion the job is re-dispatched and the second call returns the cached `rpcresult` (`rpcdone`); identical URIs are **joined** (`joinable`) so N clients waiting for the same upstream resource cost one upstream request; supports chunked/unchunked streaming forward of the upstream body to all joined clients (`rpc_recv_forward_*`, 525-737: "joining stream"), file receiver (`rpc_recv_file`), string, null, redirects, 401 authenticate, TLS (`rpc_tossl`), proxy tunnel, connect timeout 60 s, `background` jobs; `rpc_deljob` cancels upstream when the last client leaves;
  - `getstatus` (1252-1290): `/ajaxstatus` payload (watchers, rpcs, serializations, joblist);
  - `dispatches_call` (1292-1298): records handler+args on `$jev` so `redo_request` can replay.
* **BSHandoff** (116 lines): `handoff($path|\%opts, @args)` (59-99) in a forked request child connects to the AJAX unix socket (optionally the partition from `ajaxpartitionmap`), sends the HTTP request text **together with the client socket fd** (`SCM_RIGHTS` via `Socket::MsgHdr`, `handoffsender` 32-45) and `exit(0)`s — the AJAX process now owns the client connection. For https the child instead stays alive and relays (`reply_receiver`, `nullhandoff`). `handoff_part($partition, ...)` picks the partition; `BSHandoff::rpc` = ask the AJAX server something and get the answer back (`noexit`). `receivefd` (47-57) is the AJAX side.

## D.7 BSUtil.pm — file/process primitives (src/backend/BSUtil.pm, 963 lines)

| Area | Functions | Semantics |
|---|---|---|
| Atomic writes | `writexml($tmp,$final,$data,$dtd)` 71-82, `writestr($tmp,$final,$str)` 84-96, `store($tmp,$final,$data)` 562-575 (Storable `nstore`) | write to `$tmp` (convention: `.name.$$` or `.name` in the same dir), optional `fdatasync` when `$fdatasync_before_rename` (set by `set_fdatasync_before_rename` unless `$BSConfig::disable_data_sync`), then `rename($tmp,$final)`; if `$final` undef the file is left in place. `appendstr` 98-106. |
| Reads | `readstr($fn,$nonfatal)` 108-119, `readxml($fn,$dtd,$nonfatal)` 121-132 (non-XML content → die/undef), `fromxml`, `toxml` 134-143, `retrieve($fn|$fd,$nonfatal)` 583-596 (Storable; nonfatal=2 warns), `tostorable`/`fromstorable` 598-613 (`'pst0'.nfreeze` — the on-the-wire serialisation used for `/notify/redis`, `PACKTRACK` payload, `:schedulerstate`, etc.) |
| Dirs | `touch`, `ls` (no `.`/`..`, unsorted), `mkdir_p` (race-safe), `chown_multiple`, `mkdir_p_chown`, `cleandir` (recursive delete of contents), `linktree` (hardlink copy), `treeinfo`, `cp` (145-343) |
| Privileges | `drop_privs_to($user,$group)` 242-263 (setgid/setuid, dies on failure) |
| Processes | `xfork` (retries on EAGAIN every 5 s) 317-326; `xsystem($stdin,@cmd)` 699-787 (run a command with pipes, select-multiplexed stdin/stdout/stderr capture; dies with stderr on failure) |
| Locking | `lockopen($fh,$mode,$fn,$nonfatal)` 444-460: open + `flock LOCK_EX` + re-check the inode still has links (stale-lock-after-rename protection, loops); `lockcheck` 462-477 (non-blocking probe: -1 missing, 0 locked, 1 free); `lockopenxml` 479-492 (lock + parse); `lockcreatexml($fh,$tmp,$final,$data,$dtd)` 494-521 (create-if-absent via `link()`, returns undef if it already existed) |
| Wake-ups | `ping($pingfile)` 615-622: non-blocking write of one byte into a **FIFO**; `openping` 624-631 (`mkfifo 0666`, opened RDWR so it never gets EOF); `drainping` 633-639; `waitping($fh,$timeout)` 641-657 (blocking read, or 1 s polling with timeout). Every event-queue daemon (scheduler, dispatcher, publisher, signer, warden, servicedispatch, notifyforward, dodup, deltastore) sleeps in `waitping` on `$eventdir/<name>/.ping` and producers call `BSUtil::ping` after dropping an event file. |
| Daemon control | `openrunlock($lock,"$rundir/<name>",$name)` 659-665 (`<name>.lock` flock, dies "already running"); `restartexit($arg,$name,$runfile,$pingfile)` 667-697 (`--stop/--exit/--restart` via `.exit`/`.restart` marker files + ping; removes stale `.exit` on start); `waituntilgone` 434-442 |
| Flags | `enabled($repoid,$disen,$default,$arch)` 523-560: the **build/publish/debuginfo/useforbuild flag resolution algorithm** shared with the frontend's `FlagHelper`: `<enable>`/`<disable>` entries with optional `repository`/`arch`; score = 1 (disable) +2 (arch match) +4 (repo match); most specific wins, ties → disable (`return 0 if $score == 7`) |
| Encoding | `checkutf8`, `str2utf8`, `data2utf8`, `str2utf8xml`, `data2utf8xml` 344-432 (latin1 fallback, XML-safe control-char stripping) |
| Misc | `unify`, `clone` (dclone), `identical` (deep compare with exception sets), `identicalfile`, `isotime`, `rfc3339time` 789-884 |
| Logging | `openlog($logfile,$logdir,$user,$group)` 911-928 (reopen STDOUT/STDERR append), `printlog($msg,$level,$id)` 930-936 (`YYYY-MM-DD hh:mm:ss: [pid] msg`, debug levels via `setdebuglevel`), `setcritlogger`/`logcritical`/`diecritical` 938-962 (critical events also go to `<name>.crit.log` through BSStdServer::critlogger) |

## D.8 File queues and file DBs: BSFileDB.pm, BSFileQueue.pm, BSRedis.pm

* **BSFileDB** (360 lines): append-only, `|`-separated line records with a fixed layout (`[qw{number type time data []}]` where `[]` means "rest of fields is an array"); `fdb_add_i` (auto-incrementing first field under lock), `fdb_add`, `fdb_add_multiple`, `fdb_getlast`, `fdb_getmatch`, `fdb_getall($fn,$lay,$filter)`, `fdb_getall_reverse` (reads the file backwards in 64 KiB blocks — `/lastnotifications`, `/lastevents`, `.failhistory`, `_history`-style revision files `*.rev`, `*.mrev` all use this). `lockopen` wraps BSUtil::lockopen. Records are decoded by `decode_line` (escapes `|` and newlines).
* **BSFileQueue** (97 lines): `add($queue,@fields)` appends one record under lock and returns whether a ping is needed; `renamequeue` (atomic grab `queue` → `queue.send`), `openqueue`/`getnext` (iterate records with their byte length), `openmark`/`markdone($markfd,$off)` (mark a record consumed in place so a crash resumes) — used by `bs_notifyforward` and `BSRedisnotify`.
* **BSRedis** (176 lines): tiny RESP client (`new`, `connect`, `send`, `recv`, `run(@cmd)`, quit) used by `bs_redis`/srcserver `add_redis_notifications` to mirror build results into `$BSConfig::redisserver` (`redis://[pass@]host:port`).

## D.9 Auth to external services: BSBearer, BSSigAuth/BSSSHSign, BSSSL, BSPGP

* **BSBearer** (`generate_authenticator($creds,%opts)` 84-91 → closure `authenticator_function` 48-82): BSRPC authenticator for registries: on `401 WWW-Authenticate: Bearer realm=... service=... scope=...` it calls the realm with Basic creds (or JSON `post_creads`) and caches `Bearer <token>`; for `Basic` challenges answers with the creds. Users: `bs_regpush`, `BSPublisher::Registry`, `BSPublisher::Container`, `bs_notar`, `BSRekor`.
* **BSSigAuth / BSSSHSign** (installed by `BSStdServer::setup_authenticator`): signature-based auth for `$BSConfig::signature_auth_keyfile{'user@host'}` — signs the server challenge with a local gpg/ssh key (used for OBS→OBS and OBS→registry push without passwords).
* **BSSSL** (297 lines): `Net::SSLeay` tie-handle: `newctx(keyfile, certfile, verify_file|verify_dir | default paths)` 53-77 (TLS context with `OP_ALL`, ECDH for old OpenSSL), `tossl($sock, mode=>accept|connect, ...)` wraps a socket so `sysread/syswrite/print/readline` go through TLS (TIEHANDLE 150-260), `ssl_connect_with_timeout`, `data_available` (for async rpc), `peerfingerprint($type)` and `subjectdn` (used by BSStdServer authorize / sslpeerfingerprint pinning). Server side needs `$BSConfig::ssl_keyfile/ssl_certfile[/ssl_verify]`.
* **BSPGP** (310 lines): pure-Perl OpenPGP packet parsing — `pkdecodepacket/pkencodepacket`, `pk2times`, `pk2userid`, `pk2expire`, `pk2keydata`, `pk2algo`, `pk2keysize`, `pk2fingerprint(_keyid)`, `pk2sigdata`, `pk2signtime`, `unarmor`, `onepass_signed_message`, `compress_packets`. Used by `bs_signer`, `BSSrcServer::Signkey` (`_keyinfo`: expiry/algo/fingerprint of project keys), `bs_publish`/`BSConSign` (container/cosign signatures), `bs_dodup` (verify upstream repo signatures), `bs_notar`. Signing itself is external (`$BSConfig::sign` = `/usr/bin/sign` talking to `signd`).
* **BSX509 / BSASN1 / BSTUF / BSRekor / BSConSign**: certificate and TUF/notary/sigstore encoders for the container publishing path (not read in detail here).

## D.10 BSAirBrake.pm and BSNotify.pm

* **BSAirBrake** (190 lines): Airbrake/Errbit v3 JSON client (`new(server, api_key, project_id)`, `add_error($msg,$backtrace,$skip_frames)`, `send($opts)` with `environment/session/params`, `notify`), transport `BSRPC ':https'`; wired via `$BSConfig::airbrake` in `bs_srcserver`/`BSSched` critical paths (uses `BSUtil::getdebuglevel`).
* **BSNotify** (73 lines): the single outbound notification API `BSNotify::notify($type, \%params, $payload)` described in A.6: POST `$BSConfig::srcserver/notify/$type` form-encoded (`formurlencode`, 60 s), payload as `application/octet-stream` with a size-scaled timeout (max 3600 s); errors are warnings except payload transfers which are fatal. Everything non-scalar in `%params` is dropped (`$p` is filtered to defined scalars, 46).

## D.11 Operational protocol summary for a re-implementation

1. **Process model**: fork-per-request servers with hard `maxchild` limits and a side event-loop process for anything that blocks (long-poll, log streaming, upstream fan-in). A Go server replaces both with goroutines, but must preserve: blocking semantics of `block=1`, `nostream=0` log tailing, `waitservice`, request joining for identical upstream fetches, and the `serverstatus`/`ajaxstatus` introspection used by `bs_serverstatus`.
2. **Error contract**: `die "NNN message\n"` → HTTP NNN + `<status code="NNN"><summary>message</summary></status>`; clients (frontend `Backend::Error#summary`, osc, BSRPC `"$1 remote error: $2"`) pattern-match on this.
3. **Request parsing quirks**: form-urlencoded POST bodies are merged into the query string; multi-valued params; path components `%XX`-decoded before dispatch; `Expect: 100-continue`; chunked bodies; cpio streaming with `.errors` trailer file; `X-Request-ID` propagation; `Connection: close` default.
4. **Persistence primitives**: write-temp-then-rename everywhere, flock + inode-recheck locks, FIFO pings, append-only `|`-separated file DBs read backwards, Storable (`pst0`) blobs on disk and on the wire, XML via XML::Structured schemas (next part).
5. **Trust**: no auth except optional IP/DN maps; `user=` is a label.


---

# D. XML::Structured DSL and the BSXML schema catalogue

Sources read (all paths under `/home/jeffm/src/open-build-service/src/backend/`):

| File | Lines | Role |
|---|---|---|
| `worker/Structured.pm` | 1-~520 (code to line ~380, POD after) | The reference pure-Perl implementation of `XML::Structured` v1.3 shipped with the worker. No `Structured.xs`/`.so` exists in the tree or on this host; the XS fast path (`XSLoader::load`) is optional and falls back to the pure-Perl handlers (`_handle_*_slow`). |
| `build/PBuild/Structured.pm` | 21-... | A `PBuild::Structured` re-implementation on top of `Build::SimpleXML` with the same DTD semantics (used by the bundled `build` scripts, not by the OBS daemons). |
| `BSXML.pm` | 1-2221 | All OBS backend schemas (103 `our $…` schemas, 4 `our @…` fragment lists, 1 `my $rpm_entry`). |
| `BSKiwiXML.pm` | 45-460 | `$kiwidesc` (KIWI `<image>`). |
| `BSProductXML.pm` | 47-385 | product definition schemas. |
| `BSUtil.pm` | 71-84, 121-143 | `writexml`, `readxml`, `fromxml`, `toxml`. |
| `BSStdServer.pm` | 50-75, 306-314 | `stdreply`/`errreply` (XMLout of replies), `readxml_memoized`. |
| `BSRPC.pm` | 405-413, 674-681 | request body `datafmt` → `XMLout`; reply `$xmlargs` → `XMLin`. |

---

## 1. The XML::Structured DSL

### 1.1 Shape of a schema ("dtd")

A schema is a Perl array reference. Element 0 is the **XML element name**; every
following element is a *spec* describing one attribute or child. `=>` is just a
fat comma, so `[ 'package' => 'name', 'project' ]` is `[ 'package', 'name', 'project' ]`.
(`BSXML.pm:1704` uses a bare comma: `[ 'pkglist', [[ 'collection' => ... ]] ]` — identical meaning.)

There are exactly five spec forms. They are classified by `_toknown()`
(`worker/Structured.pm:257-272`) on parse and by `_workout()` (`:72-173`) on output:

| Spec syntax in BSXML.pm | `_toknown` code | Perl value in the hash | XML |
|---|---|---|---|
| `'foo'` (plain string) | `0` | scalar string (singleton) | attribute `foo="…"` **if it appears before the first array-ref spec**, otherwise child element `<foo>…</foo>` |
| `'_content'` (string) | `0` | scalar string | the text content of the element itself |
| `[ 'foo' ]` (1-element arrayref holding a string) | `1` | arrayref of scalar strings | repeated simple child `<foo>a</foo><foo>b</foo>` |
| `[ 'foo' => …specs… ]` (arrayref with ≥2 entries, first is a string) | `[0, known]` | hashref (singleton) | one sub-element `<foo …>…</foo>` with its own attrs/children |
| `[[ 'foo' => …specs… ]]` or `[ $otherschema ]` (1-element arrayref whose only entry is itself an arrayref) | `[1, known]` | arrayref of hashrefs | zero-or-more sub-elements `<foo …/>` |
| `[]` (empty arrayref) | ignored (`!@$_ ? ()`) | — | marker: "the attribute list ends here"; every string after it becomes a child element on output |

Notes on the 5th form: `[ $repo ]` where `$repo = [ 'repository' => ... ]` is literally
"a 1-element arrayref whose single element is an arrayref", i.e. the same thing as `[[ 'repository' => ... ]]`.
The sub-element's name is taken from `$repo->[0]`. A bare `$repo` (not wrapped) inside a parent spec
is the ≥2-entry form, i.e. a **singleton** sub-element (`$proj` uses `$maintenance` that way at `BSXML.pm:157`; `$dir` uses `$linkinfo`/`$serviceinfo` at `:451-452`).

There are **no** `@`/`$`/`*` prefix conventions in the DSL. The only magic strings are
`'_content'` and the empty-string element name `''` (see 1.6). `@flags`, `@roles`,
`@disableenable`, `@constraint` are ordinary Perl arrays (not refs) of specs that get
*spliced* into a parent list; `@$download[1 .. $#$download]` (`BSXML.pm:65`) splices the
body of another schema minus its root name; `@$revision` (`:817`) splices a whole schema
including its root name so that `$revision_acceptinfo` has root `revision`; `$services->[1]`
(`:1640`) reuses a single sub-spec; `@$jobhistlay`/`@$buildstatslay` are plain name lists
(no root) used as spec lists.

### 1.2 Parsing (`XMLin`, `XMLinfile`)

`XMLin($dtd, $string_or_handle)` (`worker/Structured.pm:374-379`) picks a parser once
(`XML::Parser` expat if available, else `XML::SAX` with `XML::LibXML::SAX`; external
entities/DTD loading disabled), seeds the work stack with `{ $dtd->[0] => [0, _toknown(@$dtd)] }`
and returns `$ret->{$dtd->[0]}` – i.e. the hash for the root element (not wrapped in
another level). `XMLinfile($dtd, $fn_or_fh)` opens the file and calls `XMLin` (`:381-391`).

Rules enforced by `_handle_start_slow` / `_handle_char_slow` / `_handle_end_slow` (`:175-255`):

* The root element name must equal `$dtd->[0]` or parsing dies `unknown element 'x'`.
* Any element or attribute name not in the schema → `die "unknown element"` / `"contains unknown attribute"`. **Strict**: unlike XML::Simple, extra data is an error.
* On input, **attributes and simple child elements are interchangeable**: `<pkg name="x"/>` and `<pkg><name>x</name></pkg>` produce the same `{name => 'x'}` (both go through the `!ref($ke)` scalar branch; attributes are applied via the `while (@a > 1)` loop at `:214-224`). Only the *schema shape* decides scalar-vs-array-vs-hash.
* A spec that is a hash (`[ 'foo' => … ]`) cannot be given as an attribute (`die "attribute 'foo' must be element"`).
* Code `0` (scalar) appearing twice → `die "must be singleton"`. Code `1` (`[ 'foo' ]`) is always an arrayref, even with exactly one occurrence, and the key is absent (not an empty array) when there are zero occurrences.
* Hash sub-elements with code `[0,…]` appearing twice → `die "element 'x' must be singleton"`.
* Character data inside an element whose schema has no `'_content'` and which is not a scalar leaf → `die "element 'x' contains content"` (whitespace-only text is ignored).
* `_content`: pre-initialised to `''` at element start, accumulated, then at element end leading/trailing `[ \t\r\n]` is stripped and the key is **deleted if the result is empty** (`:240-246`). So `<size unit="M"/>` parses to `{unit=>'M'}` with no `_content` key.
* Scalar leaf elements are *not* trimmed in the pure-Perl path (the end handler only trims when there is an enclosing `$out`, which is `undef` for leaves) – `<title> x </title>` gives `' x '`. An empty element `<foo/>` gives the defined empty string `''`. This is how **boolean flags** are represented in OBS: `<nosources/>`, `<disabled/>`, `<dirty/>` etc. are tested with `exists $h->{nosources}` or `defined`, never by value.
* Strings are flagged as UTF-8 characters (`Encode::_utf8_on`) unless `:bytes` mode is on (see 1.5).

### 1.3 Output (`XMLout`, `XMLoutfile`)

`XMLout($dtd, $hashref)` (`:393-404`) returns an XML string (no `<?xml …?>` prolog). `_workout`
(`:72-173`) walks the **schema in order**, not the hash:

1. While no array-ref spec has been seen (`!$inelem`), each plain-string spec whose key `exists` in the hash is written as an attribute `name="value"` (undef value → `name=""`). Attribute order = schema order.
2. The first ref spec (including `[]`) flips `$inelem`; from then on plain strings are written as child elements: `<foo>value</foo>`, or `<foo/>` when the value is `undef`. Note: the empty string `''` gives `<foo></foo>`, not `<foo/>`.
3. `'_content'`: if nothing else has been emitted yet the text is placed directly after `>` (`<size unit="M">500</size>`); otherwise it goes on its own indented line.
4. `[ 'foo' ]` → `die "'foo' must be array"` unless the value is an arrayref; each item → `<foo>item</foo>`.
5. `[[ … ]]`/`[ $sub ]` → `die "must be array"` / `"must be array of hashes"` unless an arrayref of hashrefs; each item recurses.
6. `[ 'foo' => … ]` → `die "'foo' must be hash"` unless a hashref; recurses.
7. Keys in the hash that are **not** in the schema → `die "excess hash entries: a, b"`. Hash keys are consumed as they are written (a `%d2` copy is deleted from), so a key that is in the schema is always accepted exactly once.
8. An element with nothing inside becomes `<foo/>`. Indentation is two spaces per level, one element per line, trailing `\n`. Values are escaped (`& < > "`), control characters 0x00-0x08/0x0B/0x0C/0x0E-0x1F are **silently deleted**, and output is always UTF-8 bytes (`_escape`, `:42-70`).

Round-trip is therefore stable but *not* byte-identical to arbitrary input (attribute-vs-element choice, whitespace, and `_content` placement are normalised).

### 1.4 `XMLoutfile`, `BSUtil` wrappers, server and RPC integration

* `XMLoutfile($dtd, $d, $fn_or_fh)` (`:421-434`) streams to a file (flushes per top-level child).
* `BSUtil::writexml($fn, $fnf, $data, $dtd)` (`BSUtil.pm:71-83`): `XMLout`, `syswrite` to `$fn`, optional `fdatasync`, then `rename($fn, $fnf)` if `$fnf` is defined – the standard "write temp + atomic rename" pattern used everywhere in the backend (`writexml("$dir/.meta.$$", "$dir/meta", $proj, $BSXML::proj)`).
* `BSUtil::readxml($fn, $dtd, $nonfatal)` (`:121-132`): slurps the file, requires it to match `/<.*?>/s` (else `"not xml"`), then `XMLin`. With `$nonfatal` returns `undef` on any error (missing file, not XML, schema violation) instead of dying.
* `BSUtil::fromxml($str, $dtd, $nonfatal)` / `BSUtil::toxml($data, $dtd)` (`:134-143`): string variants.
* `BSStdServer::readxml_memoized($fn, $dtd, $nonfatal)` (`BSStdServer.pm:306-314`): memoised string cache in front of `fromxml`.
* HTTP replies: a handler returns `($hashref, $BSXML::schema)`; `BSStdServer::stdreply` (`:50-63`) runs `XMLout($schema, $hash)` and sets `Content-Type: text/xml`. Errors go through `errreply` (`:65-75`) as `XMLout($BSXML::opstatus, {code, summary, details})`.
* RPC: `BSRPC::rpc({uri, datafmt => $BSXML::x, data => $hash}, $BSXML::y, @args)` serialises the request body with `XMLout($datafmt, $data)` (`BSRPC.pm:405-413`) and parses the answer with `XMLin($y, $ans)` after a `"answer is not xml"` sanity check (`:674-681`). Either side can be a `CODE` ref instead of a schema.

### 1.5 `':bytes'` import

`use XML::Structured ':bytes'` sets the **package-global** `$XML::Structured::bytes = 1`
(`:30-40`). Effect: parsed strings get `Encode::_utf8_off` (raw octets, no UTF-8 character
semantics) instead of `_utf8_on`; output is unaffected (always UTF-8 octets, with a
re-encode fallback for mis-flagged strings in `_escape`). Because it is global, one
`:bytes` import switches the entire process. All daemons do it
(`bs_srcserver`, `bs_repserver`, `bs_sched`, `bs_dispatch`, `bs_publish`, `bs_signer`,
`bs_worker`, `bs_warden`, `bs_service`, `bs_servicedispatch`, `bs_dodup`, `bs_deltastore`,
`bs_redis`, `bs_admin`, `bs_check_consistency`, `bs_archivereq`, `bs_getbinariesproxy`,
`bs_notifyforward`, `bs_productconvert`, `bs_sourcepublish`), while the library modules
(`BSUtil.pm`, `BSRPC.pm`, `BSStdServer.pm`, `BSWatcher.pm`) import plainly. Practical
consequence for Go: **the backend treats all XML string values as opaque byte strings**;
md5s/paths/names are never re-encoded, and invalid UTF-8 in input is tolerated.

### 1.6 The `''` alternative-element group

`$linkpatch` (`BSXML.pm:941-958`) has root name `''`. Used as `[ 'patches' => [ $linkpatch ] ]`
inside `$link` (`:969-971`), this creates `known{''} = [1, {add=>…, apply=>…, delete=>…, branch=>0, topadd=>0}]`.
On parse, an element under `<patches>` whose name is not directly known falls back to the `''`
entry (`:184-196`): a fresh hash is pushed onto `$out->{''}` and the element is then looked up in
the inner table. Result: `$link->{patches}->{''}` is an **ordered array of one-key hashes**, each
key being `add`/`apply`/`delete` (hash) or `branch`/`topadd` (scalar `''`). On output, `_workout`
with `$am eq ''` emits no wrapper element (`:81-85`), and top-level `XMLout` on a `''` schema
requires exactly one key and only handles ref specs (`:395-402`). This is the one place where
child **order across different element names** is preserved; everywhere else, order is only
preserved within a single `[[ … ]]` list.

### 1.7 Worked examples

**Example A – `$pack` (`BSXML.pm:163-181`), package `_meta`:**

```perl
our $pack = [ 'package' => 'name', 'project', [], 'title', 'description', 'releasename', 'scmsync',
              [ 'devel' => 'project', 'package' ], @roles, @disableenable, @flags, 'url', 'bcntsynctag' ];
```
XML:
```xml
<package name="hello" project="home:me">
  <title>Hello</title>
  <description/>
  <devel project="devel:tools" package="hello"/>
  <person userid="me" role="maintainer"/>
  <person userid="you" role="bugowner"/>
  <build>
    <disable arch="i586"/>
    <enable/>
  </build>
  <url>https://example.org</url>
</package>
```
Perl:
```perl
{ name => 'hello', project => 'home:me', title => 'Hello', description => '',
  devel => { project => 'devel:tools', package => 'hello' },
  person => [ { userid => 'me', role => 'maintainer' }, { userid => 'you', role => 'bugowner' } ],
  build => { disable => [ { arch => 'i586' } ], enable => [ {} ] },
  url => 'https://example.org' }
```
Observe: `description` is `''` (empty element → empty string, key present); `person` is always an
array; `build/enable` with no attributes is an array holding an empty hash `{}`; `group`, `lock`,
`publish`, … keys are simply absent.

**Example B – `$dir` (`:437-464`) with `$linkinfo` (`:409-424`), the `GET /source/prj/pkg` listing:**

```xml
<directory name="hello" rev="4" vrev="7" srcmd5="d41d8…">
  <linkinfo project="base" package="hello" srcmd5="aaa…" xsrcmd5="bbb…" lsrcmd5="ccc…"/>
  <entry name="hello.spec" md5="e3b0…" size="1234" mtime="1700000000"/>
  <entry name="_link" md5="…" size="80" mtime="1700000001"/>
</directory>
```
```perl
{ name => 'hello', rev => '4', vrev => '7', srcmd5 => 'd41d8…',
  linkinfo => { project => 'base', package => 'hello', srcmd5 => 'aaa…', xsrcmd5 => 'bbb…', lsrcmd5 => 'ccc…' },
  entry => [ { name => 'hello.spec', md5 => 'e3b0…', size => '1234', mtime => '1700000000' },
             { name => '_link',      md5 => '…',     size => '80',   mtime => '1700000001' } ] }
```
`linkinfo` is a bare `$linkinfo` reference → singleton hash; `entry` is `[[ … ]]` → array. All numbers are strings.

**Example C – `$repo` inside `$proj` (`:68-95`, `:137-161`), with `[ 'arch' ]` and `@flags`:**

```xml
<project name="home:me">
  <title>me</title>
  <description/>
  <repository name="openSUSE_Tumbleweed" rebuild="local">
    <path project="openSUSE:Factory" repository="snapshot"/>
    <arch>x86_64</arch>
    <arch>i586</arch>
  </repository>
</project>
```
```perl
{ name => 'home:me', title => 'me', description => '',
  repository => [ { name => 'openSUSE_Tumbleweed', rebuild => 'local',
                    path => [ { project => 'openSUSE:Factory', repository => 'snapshot' } ],
                    arch => [ 'x86_64', 'i586' ] } ] }
```
`[ 'arch' ]` → array of strings; `[[ 'path' … ]]` → array of hashes; `[ $download ]` would give
`download => [ {…} ]`; `[ 'base' => … ]` is a singleton hash (`base => {…}`).

**Example D – `$link` with `$linkpatch` (`:941-972`), a `_link` file:**

```xml
<link project="base" package="hello" rev="3" missingok="true">
  <patches>
    <branch/>
    <apply name="fix.patch"/>
    <add name="new.patch" type="patch" popt="1"/>
    <delete name="old.patch"/>
    <topadd>%define foo 1</topadd>
  </patches>
</link>
```
```perl
{ project => 'base', package => 'hello', rev => '3', missingok => 'true',
  patches => { '' => [ { branch => '' },
                       { apply  => { name => 'fix.patch' } },
                       { add    => { name => 'new.patch', type => 'patch', popt => '1' } },
                       { delete => { name => 'old.patch' } },
                       { topadd => '%define foo 1' } ] } }
```

**Example E – `$constraints` (`:1895-1941`) using `_content` and `[ 'disk' => $size ]`:**

```xml
<constraints>
  <hostlabel exclude="true">slow</hostlabel>
  <sandbox>kvm</sandbox>
  <hardware>
    <processors>4</processors>
    <disk><size unit="G">20</size></disk>
    <memory><size unit="M">4096</size></memory>
  </hardware>
</constraints>
```
```perl
{ hostlabel => [ { exclude => 'true', _content => 'slow' } ],
  sandbox   => [ { _content => 'kvm' } ],
  hardware  => { processors => '4',
                 disk   => { size => { unit => 'G', _content => '20' } },
                 memory => { size => { unit => 'M', _content => '4096' } } } }
```

### 1.8 Gotchas checklist for a Go port

1. **Ordering**: output order is schema order; input order is irrelevant except within one repeated element and within a `''` group. A Go struct with field order = schema order and `encoding/xml` struct tags reproduces this.
2. **Attribute vs element** is an output-only distinction: everything before the first array-ref spec (or `[]`) is an attribute; parsing accepts either. A Go decoder must accept `<x><name>v</name></x>` for a field tagged `name,attr` to be fully compatible (OBS frontend and `osc` always use the canonical form, so in practice attribute-only is usually enough, but `XMLin` accepts both).
3. **Singleton vs array**: `[ 'x' ]`/`[[ 'x' … ]]` → always slice (nil when absent, never empty slice on parse); `'x'`/`[ 'x' => … ]` → single value, duplicate occurrences are a hard parse error.
4. **Presence semantics**: absent key ≠ present empty. `<dirty/>` → `dirty => ''`; code tests `exists`/`defined`. Model as `*string` or a bool with `omitempty`-style custom handling; a plain `string` loses the distinction (`XMLout` writes `<foo></foo>` for `''` and `<foo/>` for undef, and omits the element when the key is absent).
5. **All values are strings**; no numeric/boolean typing in the DSL. Booleans in OBS XML are usually `"true"`/`"false"` strings in attributes (`missingok`, `exclude`, `hidden`) or presence-only elements (`disabled`, `nosources`, `retracted`).
6. **`_content`** is trimmed and dropped if empty; scalar leaves are not trimmed.
7. **Strict validation**: unknown elements/attributes are errors on parse; unknown hash keys are errors on output. A Go port that wants wire-compatibility with existing Perl peers must not emit extra elements.
8. **Namespaced names** (`xmlns:rpm`, `rpm:entry`, `xml:lang`) are plain strings containing a colon; the DSL has no namespace support.
9. **`[]` after a list** (e.g. `$patchinfo` `issue`: `'tracker','id','documented', [], '_content'`) is purely an output hint and is harmless if omitted.
10. **No `<?xml?>` declaration** is emitted, no DOCTYPE; XMLin ignores any that are present (expat handles them); external entities are disabled.
11. **Control characters are dropped on output** silently; invalid UTF-8 is re-encoded as char refs rather than rejected.

---

## 2. Every schema in BSXML.pm

Type legend used in the detail lists: **A** = attribute (string before `[]`/first ref);
**E** = simple child element (string after first ref; also accepted as attribute on input);
**E[]** = `[ 'x' ]` list of strings; **H** = `[ 'x' => … ]` singleton sub-element (hash);
**H[]** = `[[ 'x' => … ]]` / `[ $ref ]` repeated sub-element (array of hashes);
**C** = `_content`. "→ `$name`" means the sub-element is the referenced schema.

### 2.1 Catalogue (in file order)

| # | Name | Root element | Lines | What it represents |
|---|---|---|---|---|
| 1 | `$download` | `download` | 46-59 | Download-on-demand repo definition inside `<repository>` (arch, repotype, url, archfilter, master, pubkey). |
| 2 | `$doddata` | `doddata` | 61-66 | Same body as `$download` plus `project`/`repository`; the DoD worklist item passed to `bs_dodup` (`BSSched/DoD.pm`). |
| 3 | `$repo` | `repository` | 68-95 | Repository definition in project meta (paths, releasetargets, hostsystem, arches, DoD). |
| 4 | `@disableenable` | (fragment) | 97-106 | `[[disable arch repository]]`, `[[enable …]]` pairs used in every flag block. |
| 5 | `@flags` | (fragment) | 108-117 | `lock`, `build`, `publish`, `debuginfo`, `useforbuild`, `binarydownload`, `sourceaccess`, `access` flag blocks, each containing `@disableenable`. |
| 6 | `@roles` | (fragment) | 119-128 | `[[person userid role]]`, `[[group groupid role]]`. |
| 7 | `$maintenance` | `maintenance` | 130-135 | `<maintenance><maintains project=…/></maintenance>` block in a project meta. |
| 8 | `$proj` | `project` | 137-161 | Project `_meta`. |
| 9 | `$pack` | `package` | 163-181 | Package `_meta`. |
| 10 | `$packinfo` | `info` | 183-219 | Per-recipe build info of a package inside `$projpack` (deps, prereqs, constraints, paths, extrasource) computed by the source server for the scheduler. |
| 11 | `$linked` | `linked` | 221-225 | `<linked project package/>` – origin of a link chain; used in `$projpack`, `$linkinfo`, `$sourceinfo`. |
| 12 | `$aggregatelist` | `aggregatelist` | 227-247 | `_aggregate` file: list of binaries to copy from other project/repos. |
| 13 | `$patchinfo` | `patchinfo` | 249-283 | `_patchinfo` file for maintenance incidents (issues, category, rating, packager, …). |
| 14 | `$channel` | `channel` | 285-316 | `_channel` file: product/target/binaries for maintenance channels. |
| 15 | `$modulemd` | `modulemd` | 318-332 | Modularity (RHEL modules) metadata attached to a package in `$projpack`. |
| 16 | `$projectlink` | `link` | 334-338 | `<link project vrevmode/>` project-link entry (used in `$projpack`). |
| 17 | `$projpack` | `projpack` | 340-407 | The big source-server → scheduler dump: projects with repos, flags, packages with `$packinfo`, remotemaps, channeldata. |
| 18 | `$linkinfo` | `linkinfo` | 409-424 | Link resolution info inside a `$dir` listing. |
| 19 | `$serviceinfo` | `serviceinfo` | 426-435 | Source service run state (`code`, md5s, error) inside a `$dir` listing. |
| 20 | `$dir` | `directory` | 437-464 | Source file listing (`GET /source/prj/pkg`, also used for project/package lists). |
| 21 | `$providedby` | `providedby` | 466-475 | Provider record used by `$fileinfo`'s `*_ext` sections. |
| 22 | `$fileinfo` | `fileinfo` | 477-527 | Binary file header info (`/build/.../_fileinfo` / `?view=fileinfo`), rpm header fields + extended dep resolution. |
| 23 | `$sourceinfo` | `sourceinfo` | 529-556 | Parsed-source summary of one package (`?view=info`): md5s, name/version/release, subpacks, deps, exclarch. |
| 24 | `$sourceinfolist` | `sourceinfolist` | 558-561 | List of `$sourceinfo` (project-wide `?view=info`). |
| 25 | `$buildinfo` | `buildinfo` | 563-694 | The **build job description** sent to a worker (`/build/.../_buildinfo`; job file in `jobs/<arch>/`). |
| 26 | `$jobstatus` | `jobstatus` | 696-714 | Status file next to a dispatched job (`jobs/<arch>/<job>:status`). |
| 27 | `$buildreason` | `reason` | 716-726 | Why a package was rebuilt (`_reason` file / `?view=reason`). |
| 28 | `$buildstatus` | `status` | 728-752 | Per-package build status (`_status` / inside `$result`). |
| 29 | `$builddepinfo` | `builddepinfo` | 754-769 | `_builddepinfo` view: package build deps, cycles, SCCs. |
| 30 | `$event` | `event` | 771-794 | Event file written into daemon event dirs (`type` + target + servicedispatch/sourcepublish extras). |
| 31 | `$events` | `events` | 796-801 | Event feed (`/lastevents`, remote watcher) with `next`/`sync` cursor. |
| 32 | `$revision` | `revision` | 803-814 | One commit in `_history`. |
| 33 | `$revision_acceptinfo` | `revision` | 816-827 | `$revision` plus `<acceptinfo>` (request accept md5s). |
| 34 | `$revisionlist` | `revisionlist` | 829-832 | `GET /source/prj/pkg/_history`. |
| 35 | `$buildhist` | `buildhistory` | 834-844 | `_history` of builds for a package/repo/arch. |
| 36 | `$binaryversionlist` | `binaryversionlist` | 846-859 | Binary names + hdrmd5/metamd5/evr for one package (`?view=binaryversions`). |
| 37 | `$packagebinaryversionlist` | `packagebinaryversionlist` | 861-879 | Same, batched per package with cookie (scheduler ↔ repserver). |
| 38 | `$packagebinarychecksums` | `packagebinarychecksums` | 881-887 | Per-package `_content` blob of binary checksums (`?view=binarychecksums`). |
| 39 | `$worker` | `worker` | 889-923 | Worker registration record (`workers/<state>/<id>` files; `bs_worker` → repserver). |
| 40 | `$packstatuslist` | `packstatuslist` | 925-939 | Per-prj/repo/arch package status counts (scheduler `_status` summary). |
| 41 | `$linkpatch` | `''` (alternative) | 941-958 | One `<patches>` child of a `_link`: `add`/`apply`/`delete`/`branch`/`topadd`. Not referenced outside `$link`. |
| 42 | `$link` | `link` | 960-972 | `_link` file. |
| 43 | `$workerstatus` | `workerstatus` | 974-1031 | `/build/_workerstatus`: idle/building/down/dead/away workers, queue stats, daemon partitions. |
| 44 | `$workerstate` | `workerstate` | 1033-1041 | Worker's own state file (bs_worker: state, nextstate, jobid, pid, log limits). |
| 45 | `$jobhistlay` | (name list, no root) | 1043-1058 | Field list for job history entries, shared by `$jobhist` and `$buildstatslay`. |
| 46 | `$jobhist` | `jobhist` | 1060-1065 | One `_jobhistory` entry (repository, arch + `@$jobhistlay`). |
| 47 | `$jobhistlist` | `jobhistlist` | 1067-1070 | `GET /build/prj/repo/arch/_jobhistory`. |
| 48 | `$ajaxjob` | `job` | 1072-1081 | One in-flight request in the AJAX (event-loop) server status. |
| 49 | `$ajaxstatus` | `ajaxstatus` | 1083-1109 | `BSWatcher`/ajax server internals: watchers, rpcs, serialize queues, joblist. |
| 50 | `$ajaxstatuslist` | `ajaxstatuslist` | 1111-1114 | List of `$ajaxstatus`. |
| 51 | `$serverstatus` | `serverstatus` | 1116-1129 | `/serverstatus`: forked request handlers (id, pid, peer, request, group). |
| 52 | `$binarylist` | `binarylist` | 1131-1140 | Binary file listing of a package build result (`GET /build/prj/repo/arch/pkg`). |
| 53 | `$summary` | `summary` | 1142-1148 | `<statuscount code count/>` list inside `$result` (`?view=summary`). |
| 54 | `$schedulerstats` | `stats` | 1150-1156 | lastchecked/checktime/lastfinished/lastpublished inside `$result`. |
| 55 | `$buildresultinfo` | `info` | 1158-1174 | Build provenance (worker, md5s, times, jobid) in `_result?view=…` and `$buildstatistics`. |
| 56 | `$result` | `result` | 1176-1193 | One prj/repo/arch result block. |
| 57 | `$resultlist` | `resultlist` | 1195-1200 | `GET /build/prj/_result`. |
| 58 | `$opstatus` | `status` | 1202-1220 | Generic operation/error reply (`code`, `summary`, `details`, `data`, `exception`); used by `errreply`. |
| 59 | `$rpm_entry` (`my`) | `rpm:entry` | 1222-1230 | rpm-md dependency entry used in `$pattern`. |
| 60 | `$pattern` | `pattern` | 1232-1268 | One rpm-md `<pattern>` (patterns.xml). |
| 61 | `$patterns` | `patterns` | 1270-1277 | rpm-md `patterns.xml` root. |
| 62 | `$ymp` | `metapackage` | 1279-1317 | One-click-install `.ymp` file. |
| 63 | `$binary_id` | `binary` | 1319-1334 | Search result row for `/search/published/binary`. |
| 64 | `$pattern_id` | `pattern` | 1336-1347 | Search result row for `/search/published/pattern`. |
| 65 | `$repoinfo_id` | `repoinfo` | 1349-1354 | Search result row for published repositories. |
| 66 | `$issues` | `issues` | 1356-1365 | Issue references (tracker/name/label/url/state) in diffs and requests. |
| 67 | `$sourcediff` | `sourcediff` | 1367-1406 | `?view=xml` source diff (old/new, per-file diffs, issues). |
| 68 | `$request` | `request` | 1408-1507 | OBS request (actions, state, reviews, history). |
| 69 | `$repositorystate` | `repositorystate` | 1509-1512 | `<repositorystate><blocked>…</blocked></repositorystate>` list of blocked things per repo (scheduler/remote). |
| 70 | `$collection` | `collection` | 1514-1525 | Search result container (`matches`, `limited`, union of request/project/package/binary/pattern/repoinfo/value). |
| 71 | `$quota` | `quota` | 1527-1534 | Per-project package quota. |
| 72 | `$schedulerinfo` | `schedulerinfo` | 1536-1562 | Scheduler self-report (`/build/_schedulerinfo?arch=`). |
| 73 | `$person` | `person` | 1564-1587 | User record (login, email, realname, owner, state, globalrole, watchlist) – frontend format. |
| 74 | `$comps` | `comps` | 1589-1609 | yum `comps.xml` groups. |
| 75 | `$dispatchprios` | `dispatchprios` | 1611-1620 | `/build/_dispatchprios`: per prj/repo/arch priority adjustments for the dispatcher. |
| 76 | `$services` | `services` | 1622-1633 | `_service` file: list of services with params. |
| 77 | `$services_link` | `services` | 1635-1642 | Project-level `_service` that may `<link project/>` to other projects. |
| 78 | `$servicetype` | `service` | 1644-1659 | Service definition (`/service` listing: name, hidden, summary, parameters). |
| 79 | `$servicelist` | `servicelist` | 1661-1664 | List of `$servicetype`. |
| 80 | `$updateinfoitem` | `update` | 1666-1725 | One `<update>` of rpm-md `updateinfo.xml` (plus OBS-internal extensions). |
| 81 | `$updateinfo` | `updates` | 1727-1731 | `updateinfo.xml` root. |
| 82 | `$deltapackage` | `newpackage` | 1733-1753 | drpm delta entry (deltainfo/prestodelta). |
| 83 | `$deltainfo` | `deltainfo` | 1755-1758 | `deltainfo.xml` root. |
| 84 | `$prestodelta` | `prestodelta` | 1760-1763 | `prestodelta.xml` root. |
| 85 | `$configuration` | `configuration` | 1765-1803 | Instance configuration pushed from the frontend (`/configuration`). |
| 86 | `$issue_trackers` | `issue-trackers` | 1805-1822 | Issue tracker definitions pushed from the frontend. |
| 87 | `$appdataitem` | `application` | 1824-1855 | AppStream appdata entry. |
| 88 | `$appdata` | `applications` | 1857-1861 | AppStream appdata root. |
| 89 | `$attribute` | `attribute` | 1863-1873 | One `<attribute namespace name>` with values/issues. |
| 90 | `$attributes` | `attributes` | 1875-1878 | `_attribute` file. |
| 91 | `$size` | `size` | 1880-1885 | `<size unit>N</size>` helper (constraints, buildstatistics). |
| 92 | `$time` | `time` | 1887-1893 | `<time unit>N</time>` helper (buildstatistics). |
| 93 | `@constraint` | (fragment) | 1895-1929 | hostlabel/sandbox/linux/hardware constraint specs. |
| 94 | `$constraints` | `constraints` | 1931-1941 | `_constraints` file (base constraints + `overwrite` blocks with conditions). |
| 95 | `$buildstatistics` | `buildstatistics` | 1943-2001 | `_statistics` of a build (disk, memory, times, download, `$buildresultinfo`). |
| 96 | `$buildstatslay` | (name list, no root) | 2003-2050 | Flattened field names for build stats history entries (append-only!). |
| 97 | `$buildstatslist` | `buildstats` | 2052-2058 | `_buildstats` history list. |
| 98 | `$notifications` | `notifications` | 2060-2073 | `/notifications` feed (next/sync/limit_reached + notification type/time/data). |
| 99 | `$frozenlinks` | `frozenlinks` | 2075-2085 | `_frozenlinks` file (frozen link targets per project). |
| 100 | `$report` | `report` | 2087-2121 | Product/container `.report` file: list of binaries with provenance. |
| 101 | `$publishedpath` | `publishedpath` | 2123-2131 | `/published/.../?view=publishedpath` reply. |
| 102 | `$multibuild` | `multibuild` | 2133-2138 | `_multibuild` file (flavors). |
| 103 | `$pubkeyinfo` | `pubkey` | 2140-2150 | GPG key info (`/source/prj/_keyinfo` sub-element). |
| 104 | `$certinfo` | `sslcert` | 2152-2165 | SSL cert info (`_keyinfo` sub-element). |
| 105 | `$keyinfo` | `keyinfo` | 2167-2172 | `GET /source/prj/_keyinfo`. |
| 106 | `$binannotation` | `annotation` | 2174-2196 | Container/binary annotation (repos used, registry digests, buildtime…) embedded in `.containerinfo`/bdeps. |
| 107 | `$availablebinaries` | `availablebinaries` | 2198-2212 | `?view=availablebinaries`: package/product/pattern names per arch. |
| 108 | `$regrepoowner` | `regrepoowner` | 2214-2219 | Registry repo ownership record (`BSPublisher/Registry.pm`). |

Names mentioned in the task that **do not exist** in BSXML.pm: `$signkey`, `$sslcert` (it is `$certinfo`, root `sslcert`), `$regrepo` (it is `$regrepoowner`), `$deltastore`, `$scheduler` (closest: `$schedulerinfo`, `$schedulerstats`), `$publishstate`, `$lastevents` (it is `$events`), `$notification` (only inside `$notifications`), `$frontendnotification`, `$path`/`$releasetarget`/`$status`/`$entry`/`$action` (inline sub-specs, not top-level), `$kiwidesc` (in BSKiwiXML.pm), `$productlist` (BSProductXML `$products`, root `productlist`).

### 2.2 Detailed field lists for the important schemas

#### `$proj` – `<project>` (`BSXML.pm:137-161`)
| Field | Type | Notes |
|---|---|---|
| name | A | |
| kind | A | `standard`/`maintenance`/`maintenance_incident`/`maintenance_release` |
| title | E | |
| description | E | |
| url | E | |
| scmsync | E | |
| config | E | only with `withconfig` |
| link | H[] `{project, vrevmode}` | project links |
| remoteurl | E | remote project |
| remoteproject | E | |
| mountproject | E | |
| devel | H `{project}` | |
| person | H[] `{userid, role}` | from `@roles` |
| group | H[] `{groupid, role}` | |
| maintenance | H → `$maintenance` `{maintains: H[] {project}}` | |
| lock, build, publish, debuginfo, useforbuild, binarydownload, sourceaccess, access | H each, `{disable: H[] {arch, repository}, enable: H[] {arch, repository}}` | from `@flags` |
| repository | H[] → `$repo` | |

#### `$repo` – `<repository>` (`:68-95`)
| Field | Type | Notes |
|---|---|---|
| name | A | |
| rebuild | A | `transitive`/`direct`/`local` |
| block | A | `all`/`local`/`never` |
| linkedbuild | A | `off`/`localdep`/`all`/`alldirect` |
| download | H[] → `$download` `{arch, repotype, url (A); archfilter, master{url, sslfingerprint}, pubkey (E)}` | |
| releasetarget | H[] `{project, repository, trigger}` | |
| path | H[] `{project, repository}` | |
| hostsystem | H[] `{project, repository}` | |
| base | H `{project, repository}` | expanded repo only |
| arch | E[] | |
| status | E | |
| crosshostarch | E | internal |

#### `$pack` – `<package>` (`:163-181`)
name A, project A, title E, description E, releasename E, scmsync E, devel H `{project, package}`, person H[], group H[], disable H[] `{arch, repository}`, enable H[] (top-level `@disableenable`, legacy), lock/build/publish/debuginfo/useforbuild/binarydownload/sourceaccess/access H (flags), url E, bcntsynctag E.

#### `$packinfo` – `<info>` inside projpack package (`:183-219`)
repository A, name A, file A, error A; dep E[], prereq E[], buildtimeservice E[], onlynative E[], alsonative E[], constraint E[], imagetype E[], imagearch E[]; nodbgpkgs E, nosrcpkgs E, nativebuild E, nouseforbuild E, hasbuildenv E, bcntsynctag E; path H[] `{project, repository, priority}`, containerpath H[] `{project, repository, priority}`, extrasource H[] `{project, package, srcmd5, file}`.

#### `$projpack` – `<projpack>` (`:340-407`)
| Field | Type |
|---|---|
| repoid | A |
| project | H[]: name A, kind A; error, title, description, config, patternmd5 E; link H[] → `$projectlink`; remoteurl, remoteproject, scmsync, scminfo E; `@flags` H; person/group H[]; repository H[] → `$repo`; **package** H[] (below); missingpackages E |
| project.package | name, releasename, rev, srcmd5, versrel, verifymd5, scmsync, scminfo, originproject, revtime, constraintsmd5 (A); linked H[] → `$linked`; error (E); info H[] → `$packinfo`; aggregatelist H → `$aggregatelist`; patchinfo H → `$patchinfo`; channelmd5 E; modulemd H → `$modulemd`; `@flags` H; bcntsynctag, hasbuildenv, hasbigdep E |
| remotemap | H[]: project, kind, root, remoteurl, remoteproject, remoteroot, partition, proto (A); link H[] → `$projectlink`; config E; `@flags`; `@roles`; repository H[] → `$repo`; error E |
| channeldata | H[]: md5 A; channel H → `$channel` |

#### `$dir` – `<directory>` (`:437-464`)
name, count, rev, vrev, srcmd5, tproject, tpackage, trev, tsrcmd5, lsrcmd5, error, xsrcmd5 (all A); linkinfo H → `$linkinfo`; serviceinfo H → `$serviceinfo`; entry H[] `{name, md5, hash, size, mtime, error, id, originproject, originpackage}` (all A).

`$linkinfo` (`:409-424`): project, package, rev, srcmd5, baserev, missingok, xsrcmd5, lsrcmd5, error, lastworking (A); linked H[] → `$linked` `{project, package}`.
`$serviceinfo` (`:426-435`): code, xsrcmd5, lsrcmd5, lxsrcmd5 (A); error (E).

#### `$revision` / `$revisionlist` (`:803-832`)
`revision`: rev A, vrev A; srcmd5, version, time, user, comment, requestid E. `revisionlist`: revision H[]. `$revision_acceptinfo` adds acceptinfo H `{rev, srcmd5, osrcmd5, xsrcmd5, oxsrcmd5, oproject, opackage}`.

#### `$fileinfo` – `<fileinfo>` (`:477-527`)
filename A; name, epoch, version, release, arch, source, summary, description, disturl, size, mtime E; provides, requires, prerequires, conflicts, obsoletes, recommends, supplements, suggests, enhances, filelist E[]; provides_ext H[] `{dep A; requiredby H[] {name, epoch, version, release, arch, project, repository}}`; requires_ext / recommends_ext / supplements_ext H[] `{dep A; providedby H[] → $providedby {name, epoch, version, release, arch, project, repository}}`.

#### `$binarylist` (`:1131-1140`)
package A; binary H[] `{filename, md5, size, mtime}` (all A).

#### `$buildinfo` – `<buildinfo>` (`:563-694`)
| Field | Type | Notes |
|---|---|---|
| project, repository, package, releasename, srcserver, reposerver, downloadurl | A | |
| job, arch, hostarch, crossarch, error, srcmd5, verifymd5, rev, vcs, disturl, reason, needed, revtime, readytime, buildtime, specfile, file, versrel, bcnt, release, config, debuginfo, ccache, constraintsmd5 | E | |
| prjconfconstraint, constraint, subpack, imagetype | E[] | |
| nodbgpkgs, nosrcpkgs, genmetaalgo, forcebinaryidmeta, logidlelimit, logsizelimit, genbuildreqs, nouseforbuild | E | |
| obsgendiff | H `{project, repository}` | |
| dep | E[] | expanded build deps (names) |
| bdep | H[] | name, preinstall, vminstall, cbpreinstall, cbinstall, runscripts, notmeta, noinstall, installonly, sysroot, epoch, version, release, arch, hdrmd5, project, repository, repoarch, binary, package, srcmd5, annotation (all A) |
| pdep | E[] | obsolete |
| path, syspath, containerpath | H[] `{project, repository, server, url}` | |
| containerannotation, expanddebug, followupfile, followupsteps, masterdispatched, nounchanged | E | |
| module | E[] | |
| modularity_package, modularity_srcmd5, modularity_macros, modularity_label, modularity_platform, modularity_meta, modularity_yaml | E | |
| slsaprovenance, slsadownloadurl, slsabuilder, signflavor, reprorepoid, reprojobid | E | |
| preinstallimage | H `{project, repository, repoarch, package, filename, hdrmd5 (A); binary E[]; url E}` | |

#### `$jobstatus` (`:696-714`)
code, result, details (A); starttime, endtime, lastduration, workerid, hostarch, uri, arch, job, jobid, attempt (E).

#### `$buildstatus` – `<status>` (`:728-752`)
package, code, status, error, dirty, versrel, reproduciblecheck (A); details, workerid, hostarch, readytime, starttime, endtime, buildid, job, uri, arch (E).

#### `$result` / `$resultlist` (`:1176-1200`)
`result`: project, repository, arch, code, state, details, dirty (A); scmsync, scminfo (E); status H[] → `$buildstatus`; binarylist H[] → `$binarylist`; info H[] → `$buildresultinfo`; summary H → `$summary` `{statuscount H[] {code, count}}`; stats H → `$schedulerstats` `{lastchecked, checktime, lastfinished, lastpublished}`.
`resultlist`: state A, retryafter A; result H[].

#### `$buildresultinfo` – `<info>` (`:1158-1174`)
package A; workerid, hostarch, srcmd5, verifymd5, rev, vcs, file, buildtype, readytime, starttime, endtime, jobid E.

#### `$buildreason` – `<reason>` (`:716-726`)
explain, time, oldsource (E); packagechange H[] `{change, key}` (A).

#### `$workerstatus` (`:974-1031`)
clients A; idle H[] `{uri, workerid, hostarch}`; building H[] `{uri, workerid, hostarch, project, repository, package, arch, starttime}`; down/dead/away H[] `{workerid, hostarch}`; waiting/blocked H[] `{arch, jobs}`; buildavg H[] `{arch, buildavg}`; partition H[] `{name A; daemon H[] {type, arch, state, starttime (A); queue H {high, med, low, next}}}`.

#### `$worker` (`:889-923`)
hostarch, ip, port, uri, registerserver, workerid (A); buildarch E[], hostlabel E[]; sandbox E; linux H `{version, flavor (E)}`; hardware H `{cpu H {flag E[]}; processors, jobs, nativeonly, memory, swap, disk (E)}`; owner, tellnojob, job, arch, jobid, reposerver (E).

#### `$jobhistlist` / `$jobhist` (`:1043-1070`)
`jobhist`: repository, arch + `@$jobhistlay` = package, rev, srcmd5, versrel, bcnt, readytime, starttime, endtime, code, uri, workerid, hostarch, reason, verifymd5 — all A. `jobhistlist`: jobhist H[].

#### `$patchinfo` (`:249-283`)
incident, version (A); package E[], binary E[]; releasetarget H[] `{project, repository}`; issue H[] `{tracker, id, documented (A); _content C}`; category, rating, name, summary, description, message, swampid, packager, retracted, stopped, seperate_build_arch, blocked_in_product, embargo_date, zypp_restart_needed, reboot_needed, relogin_needed (E).

#### `$attributes` / `$attribute` (`:1863-1878`)
`attribute`: namespace, name, binary (A); value E[]; issue H[] `{name, tracker}`. `attributes`: attribute H[].

#### `$link` / `$linkpatch` (`:941-972`)
`link`: project, package, rev, vrev, cicount, baserev, missingok (A); patches H `{'' : H[] alternative of add{name,type,after,popt,dir} | apply{name} | delete{name} | branch '' | topadd str}`.

#### `$aggregatelist` (`:227-247`)
resign A; aggregate H[] `{project, arch, sourcearch (A); nosources, noupdateinfo (E); package, binary, binaryarch E[]; repository H[] {target, source}}`.

#### `$services` / `$services_link` / `$servicetype` / `$servicelist` (`:1622-1664`)
`services`: service H[] `{name, mode (A); param H[] {name A; _content C}}`.
`services_link`: link H[] `{project}` + the same service H[].
`servicetype` (`service`): name, hidden (A); summary, description (E); parameter H[] `{name A; description, required, allowmultiple E; allowedvalue E[]}`.
`servicelist`: service H[] → `$servicetype`.

#### `$constraints` (`:1895-1941`)
`@constraint`: hostlabel H[] `{exclude A; _content}`; sandbox H[] `{exclude A; _content}`; linux H `{version H {max, min E}; flavor E}`; hardware H `{cpu H {flag H[] {exclude A; _content}}; processors, jobs E; disk/memory/memoryperjob/physicalmemory H {size H → $size {unit A; _content}}}`.
`constraints`: `@constraint` + overwrite H[] `{conditions H {arch E[], package E[]}; @constraint}`.

#### `$multibuild` (`:2133-2138`)
buildemptyflavor A; package E[] (obsolete); flavor E[].

#### `$sourceinfo` (`:529-556`)
package, rev, vrev, srcmd5, lsrcmd5, verifymd5, metamd5 (A); filename, error, originproject, originpackage (E); linked H[] → `$linked`; revtime, changesmd5, name, version, release (E); subpacks, deps, prereqs, exclarch, badarch, buildtimeservice (E[]).

#### `$collection` (`:1514-1525`)
matches, limited (A); request H[] → `$request`; project H[] → `$proj`; package H[] → `$pack`; binary H[] → `$binary_id`; pattern H[] → `$pattern_id`; repoinfo H[] → `$repoinfo_id`; value E[].

#### `$notifications` (`:2060-2073`)
next, sync, limit_reached (A); notification H[] `{type, time (A); data H[] {key A; _content}}`.

#### `$configuration` (`:1765-1803`)
All E (list starts with `[]`): title, tos_url, code_of_conduct, description, name, anonymous, registration, default_access_disabled, default_tracker, allow_user_to_create_home_project, multiaction_notify_support, disallow_group_creation, change_password, cleanup_after_days, hide_private_options, gravatar, enforce_project_keys, download_on_demand, download_url, obs_url, api_url, ymp_url, errbit_url, bugzilla_url, http_proxy, no_proxy, admin_email, theme, cleanup_empty_projects, disable_publish_for_branches; schedulers H `{arch E[]}`; unlisted_projects_filter, unlisted_projects_filter_description.

#### `$dispatchprios` (`:1611-1620`)
prio H[] `{project, repository, arch, adjust}` (all A).

#### `$keyinfo` / `$pubkeyinfo` / `$certinfo` (`:2140-2172`)
`keyinfo`: project A; pubkey H → `$pubkeyinfo` `{keyid, userid, algo, curve, keysize, expires, fingerprint (A); _content}`; sslcert H → `$certinfo` `{serial, issuer, subject, algo, curve, keysize, begins, expires, fingerprint, keyid (A); _content}`.

#### `$request` (`:1408-1507`)
| Field | Type |
|---|---|
| id, actions, creator, type, key, retryafter | A |
| action | H[]: type A; source H `{project, package, rev, repository}`; target H `{project, package, releaseproject, repository}`; person H `{name, role}`; group H `{name, role}`; options H `{sourceupdate, updatelink, makeoriginolder (E)}`; acceptinfo H `{rev, srcmd5, osrcmd5, xsrcmd5, oxsrcmd5}`; sourcediff H[] → `$sourcediff` |
| issues | H → `$issues` `{issue H[] {state, tracker, name, label, url}}` |
| submit | H (legacy) `{source H {project, package, rev}; target H {project, package}}` |
| priority | E |
| state | H `{name, who, when, created, approver, superseded_by (A); comment E}` |
| review | H[] `{state, by_user, by_group, by_project, by_package, who, when (A); comment E; history H[] {who, when (A); comment, description E}}` |
| history | H[] `{name, who, when, superseded_by (A); comment, description E}` |
| accept_at, title, description | E |

#### `$event` / `$events` (`:771-801`)
`event`: type A; project, repository, arch, package, job, worker, time, due, srcmd5, rev, linksrcmd5, projectservicesmd5, oldsrcmd5, details (E); included H[] `{project, package, srcmd5}`. `events`: next, sync (A); event H[].

#### `$opstatus` – `<status>` (`:1202-1220`)
code, origin (A); summary, details (E); data H[] `{name A; _content}`; exception H `{type, message (A); backtrace H {line E[]}}`.

#### `$channel` (`:285-316`)
disabled E; product H `{project, name}`; target H[] `{project, repository, id_template, requires_issue (A); disabled E}`; binaries H[] `{project, repository, arch (A); binary H[] {name, binaryarch, project, repository, package, arch, supportstatus, superseded_by}}`.

#### `$buildstatistics` (`:1943-2001`)
disk H `{usage H {size H {unit A; _content}; io_requests, io_sectors E}}`; memory H `{usage H {size → $size}}`; times H `{total, preinstall, install, main, postchecks, rpmlint, buildcmp, deltarpms, download, cpusteal: each H {time → $time {unit A; _content}}}`; download H `{size → $size; binaries, cachehits, preinstallimage E}`; info H → `$buildresultinfo`.

#### `$report` (`:2087-2121`)
epoch, version, release, binaryarch, buildtime, buildhost, disturl, binaryid (A); binary H[] `{name, epoch, version, release, binaryarch, buildtime, buildhost, disturl, license, binaryid, supportstatus, superseded_by, cpeid, summary, isbase, project, repository, package, arch (A); _content}`.

#### `$binannotation` – `<annotation>` (`:2174-2196`)
repo H[] `{url, project, repository, priority}`; buildtime, buildhost, disturl, binaryid E; installed E[]; registry_refname, registry_digest, registry_fatdigest, package, epoch, version, release, binaryarch, hdrmd5 E.

---

## 3. BSKiwiXML.pm and BSProductXML.pm

### BSKiwiXML.pm (one schema)
| Name | Root | Lines | One-liner |
|---|---|---|---|
| `$kiwidesc` | `image` | 45-460 | KIWI image description (`config.xml`/`*.kiwi`): attrs name/schemeversion/schemaversion/displayname/id/kiwirevision; children description H, preferences H (type H[] with ~60 attrs, version, packagemanager, …), size H, instsource H (product/media metadata), users H[], split H, profiles H, drivers H[], strip H[], repository H[] (source + alias/priority), deploy H[], packages H[] (type, profiles; package/opensusePattern/product/archive/ignore lists), vmwareconfig H, xenconfig H. Parsed by the scheduler/worker to derive image build deps and types. |

### BSProductXML.pm
| Name | Root | Lines | One-liner |
|---|---|---|---|
| `$group` | `group` | 47-81 | Product package group (`*.group` files): name/version/release attrs; conditional H[], packagelist with package entries, include. |
| `$zypp` | `zypp` | 83-88 | Private: `<zypp name alias><disable/></zypp>` repo registration hint inside a product. |
| `$product` | `product` | 91-240 | Single product definition (id, schemeversion; vendor, name, version, release, productline, register, upgrades, updaterepos, summary, description, linguas, urls, buildoptions, …) as installed on a system. |
| `$productdesc` | `productdefinition` | 243-354 | Complete product definition file: `products` (H[] → `$product`), `conditionals`, `repositories`, `archsets`, `mediasets`, `group` list – describes how media are built. |
| `$products` | `productlist` | 357-360 | List of `$productdesc`. |
| `$productrepositories` | `product` | 362-380 | Per-product `distrotarget` → repository (zypp/update/…) mapping for release. |
| `$productlistrepositories` | `productrepositories` | 382-385 | List of `$productrepositories`. |

`BSProductXML.pm` additionally contains `mergexmlfiles()` (`:387+`), which resolves
`xi:include` by textual merging before `XMLin` – the DSL itself has no include support.


---

# Part D: BSVerify, BSDispatch route syntax, BSConfiguration, BSConfig variables

All paths are relative to `/home/jeffm/src/open-build-service/src/backend/` unless
stated otherwise. Line numbers were read from the working tree at commit
`db93ae665b`.

---

## BSVerify

File: `BSVerify.pm` (511 lines). Pure functions, no state. Every function
either returns normally or `die`s with a `"...\n"`-terminated message (the
trailing newline suppresses Perl's "at FILE line N" suffix, so the message is
the literal HTTP error body the dispatcher sends back). None of these functions
return a value that callers use.

### Verifier registry (`$BSVerify::verifiers`, lines 483-508)

This hash is what the dispatcher consults when a route parameter has a type
suffix (see "BSDispatch route syntax"). The *key* is the type name used in a
dispatch table; the *value* is the sub.

| type name in dispatch table | sub | note |
|---|---|---|
| `project` | `verify_projid` | |
| `package` | `verify_packid` | |
| `repository` | `verify_repoid` | |
| `arch` | `verify_arch` | |
| `job` | `verify_jobid` | |
| `package_repository` | `verify_packid_repository` | packid or literal `_repository` |
| `package_repositorybuild` | `verify_packid_repositorybuild` | also `_repository:<flavor>` |
| `filename` | `verify_filename` | |
| `md5` | `verify_md5` | |
| `srcmd5` | `verify_srcmd5` | md5 or 40-hex git id |
| `rev` | `verify_rev` | |
| `linkrev` | `verify_linkrev` | rev or `base` |
| `bool` | `verify_bool` | |
| `num` | `verify_num` | |
| `intnum` | `verify_intnum` | |
| `port` | `verify_port` | |
| `prp` | `verify_prp` | |
| `prpa` | `verify_prpa` | |
| `resultview` | `verify_resultview` | |
| `jobid` | **`verify_md5`** (line 503) | note: `jobid` type is md5, *not* `verify_jobid` |
| `workerid` | `verify_workerid` | no-op |
| `regrepo` | `verify_regrepo` | container registry repo path |
| `regtag` | `verify_regtag` | container registry tag |
| `module` | `verify_module` | no-op |

Because a bare parameter name with no `:type` suffix uses *its own name* as the
type (BSDispatch.pm:176, 212), the registry keys above double as the set of
"self-typed" parameter names: a route param spelled `rev?`, `linkrev?`,
`srcmd5?`, `workerid?`, `jobid?`, `module*`, `project*`, `package*`, `arch*`,
`job`, `port`, etc. is verified by the sub of the same name. Any other bare name
(e.g. `comment?`) would fail at compile time with `no verifier for comment`.

### Scalar verifiers

Regex semantics to keep in mind when porting (Perl):
* `$` without `/m` matches at end-of-string **or before a trailing `\n`**;
  `\z` is a hard end. Only the multibuild/flavor regexes use `\A...\z`; the
  rest (`md5`, `num`, `rev`, `prp`, ...) would accept a value with one trailing
  newline. (Not exploitable via the dispatcher: the query string decoder does not
  produce trailing newlines unless `%0A` is sent, which it would.)
* `/s` only changes `.`; it is irrelevant for the character-class regexes.
* `die(...) unless $x` with `$x` being the string `"0"` **dies** (Perl falsiness).
  So `verify_projid("0")`, `verify_packid("0")`, `verify_repoid("0")`,
  `verify_arch("0")` all reject the single character `0`.
* `\000-\037` is octal: bytes 0x00-0x1F. `\200-\377` is 0x80-0xFF (so raw UTF-8
  bytes pass `verify_simple`, `verify_url`, `verify_regtag`).

| sub | lines | rules (in order evaluated) | dies with | calls |
|---|---|---|---|---|
| `verify_projid($projid)` | 28-35 | 1. defined and `ne ''`. 2. `!~ /[\/\000-\037]/` (no slash, no control chars). 3. `":$projid:" !~ /:[_\.:]/` i.e. no colon-separated component may start with `_` or `.`, and no component may be empty (so no leading/trailing `:` and no `::`). 4. truthy (rejects `"0"`). 5. `length <= 200`. No other character restrictions (spaces, UTF-8, `%` etc. allowed). Comment at line 27: keep in sync with `src/api/app/models/project.rb`. | `projid is empty`, `projid '$projid' is illegal`, `projid '$projid' is too long` | none |
| `verify_projkind($kind)` | 37-40 | must be exactly one of `standard`, `maintenance`, `maintenance_incident`, `maintenance_release`. No defined check (undef warns, then dies). | `projkind '$projkind' is illegal` | none |
| `verify_packid($packid)` | 43-61 | 1. defined and `ne ''`. 2. `length <= 200` (checked on the whole string, before any split). 3. **Multibuild branch** (line 47): if `/(?<!^_product)(?<!^_patchinfo):./` matches, i.e. there is a `:` followed by at least one char, and that `:` is *not* the one immediately after a leading `_product` or `_patchinfo`: then it must match `\A([^:]+):([^:]+)\z` (exactly one colon, both sides non-empty) else illegal; `$p1` must not be `_project` or `_pattern`; `verify_packid($p1)` recursively; `$p2` (the flavor) must match `\A[^_\.\/:\000-\037][^\/:\000-\037]*\z` (first char not `_` `.` `/` `:` control; rest not `/` `:` control). Return. 4. Exact special names accepted: `_product`, `_pattern`, `_project`, `_patchinfo` (line 56). 5. `_product:<flavor>` and `_patchinfo:<flavor>` accepted with the same flavor syntax (line 57). 6. Otherwise: `!~ /[\/:\000-\037]/` (no slash, colon, control), `!~ /^[_\.]/` (no leading underscore or dot), truthy. Used for both source package names and build-container names (comment line 42). | `packid is empty`, `packid '$packid' is too long`, `packid '$packid' is illegal` | `verify_packid` (recursive on `$p1`) |
| `verify_repoid($repoid)` | 63-70 | defined/non-empty; `!~ /[\/:\000-\037]/`; `!~ /^[_\.]/`; truthy; `length <= 200`. | `repoid is empty`, `repoid '$repoid' is illegal`, `repoid '$repoid' is too long` | none |
| `verify_jobid($jobid)` | 72-77 | defined/non-empty; `!~ /[\/\000-\037]/`; `!~ /^[\.]/`. No length limit, colons allowed (job names are `proj::repo::pack-<md5>`). | `jobid is empty`, `jobid '$jobid' is illegal` | none |
| `verify_arch($arch)` | 79-86 | defined/non-empty; `!~ /[\/:\.\000-\037]/` (no slash, colon, dot, control); truthy; `length <= 200`; then `verify_simple`. | `arch is empty`, `arch '$arch' is illegal`, `arch '$arch' is too long`, `illegal characters` | `verify_simple` |
| `verify_packid_repository($p)` | 88-90 | if `$p` is truthy and `eq '_repository'` accept; else `verify_packid($p)` (so undef/empty dies `packid is empty`). | (via packid) | `verify_packid` |
| `verify_packid_repositorybuild($p)` | 92-100 | if `/\A_repository:([^:]+)\z/s`: flavor must match `\A[^_\.\/:\000-\037][^\/:\000-\037]*\z`; else `verify_packid_repository`. | `packid '$packid' is illegal` | `verify_packid_repository` |
| `verify_simple($name)` | 122-125 | `!~ /[^\-+=\.,0-9:%{}\@#%A-Z_a-z~^\200-\377]/s`. Allowed set: `- + = . , 0-9 : % { } @ # A-Z _ a-z ~ ^` and bytes 0x80-0xFF. No defined/empty check (empty string passes; undef warns and passes). | `illegal characters` | none |
| `verify_filename($f)` | 127-132 | defined/non-empty; `!~ /[\/\000-\037]/`; `!~ /^\./` (no leading dot, which also excludes `.` and `..`). No length limit. | `filename is empty`, `filename '$filename' is illegal` | none |
| `verify_url($url)` | 134-139 | defined/non-empty; `!~ /[^\041-\176\200-\377]/s` (every byte printable ASCII 0x21-0x7E or 0x80-0xFF: no space, no control, no DEL); `=~ /^[a-zA-Z]+:/s`. | `url is empty`, `illegal characters in url`, `url does not start with a scheme` | none |
| `verify_md5($m)` | 141-144 | truthy and `/^[0-9a-f]{32}$/s` (lowercase hex only). | `not a md5 sum` | none |
| `verify_srcmd5($m)` | 147-150 | truthy and (`/^[0-9a-f]{32}$/s` or `/^[0-9a-f]{40}$/s`). Comment: "can be a md5sum or a git id". | `not a srcmd5 sum` | none |
| `verify_rev($rev)` | 152-159 | defined/non-empty; accept 32-hex, 40-hex (git id), literal `upload`, `build`, `latest`, `repository`; else must be `/^\d+$/s`. | `revision is empty`, `bad revision '$rev'` | none |
| `verify_linkrev($rev)` | 161-165 | accept literal `base`; else `verify_rev`. | (via rev) | `verify_rev` |
| `verify_port($p)` | 167-172 | defined/non-empty; `/^\d+$/s`; numeric `>= 1024`. | `port is empty`, `bad port '$port'`, `illegal port '$port'` | none |
| `verify_num($n)` | 174-178 | defined/non-empty; `/^\d+$/`. | `number is empty`, `not a number: '$num'` | none |
| `verify_intnum($n)` | 180-184 | defined/non-empty; `/^-?\d+$/`. | same messages as num | none |
| `verify_bool($b)` | 186-189 | defined and exactly `'0'` or `'1'`. (`?flag` with no value becomes `1` in parse_cgi, so `?lastbuild` works but `?lastbuild=true` is rejected.) | `not boolean` | none |
| `verify_prp($prp)` | 191-197 | `/^([^\/]*)\/(.*)$/s` splits at the **first** slash; `verify_projid($1)`, `verify_repoid($2)`. | `not a prp: '$prp'` | `verify_projid`, `verify_repoid` |
| `verify_prpa($prpa)` | 199-205 | `/^(.*)\/([^\/]*)$/s` splits at the **last** slash; `verify_prp($1)`, `verify_arch($2)`. | `not a prpa: '$prpa'` | `verify_prp`, `verify_arch` |
| `verify_resultview($v)` | 207-210 | one of `summary`, `status`, `binarylist`, `stats`, `versrel`, `info`, `reproduciblecheck`. | `unknown view parameter: '$view'` | none |
| `verify_workerid` | 212-213 | **no-op** | | |
| `verify_regrepo($repo)` | 215-224 | defined/non-empty; `!~ /^[:\.]/s`; `"/$repo/" !~ /\/\//` (no empty path component: no leading, trailing or doubled slash); for each `/`-separated component: `!~ /[\/\000-\037]/s` and `!~ /^[:\.]/s`. | `bad repo name '$repo'`, `component '$p' is illegal` | none |
| `verify_regtag($tag)` | 226-230 | `!~ /[^\-+=\.,0-9%{}\@#%A-Z_a-z~\200-\377]/s` (same as `verify_simple` **minus** `:` and `^`); `!~ /^\./`. No empty check. | `illegal characters in tag '$tag'`, `illegal tag '$tag'` | none |
| `verify_module` | 480-481 | **no-op** | | |

### Structural (XML-document) verifiers

These take a hash as produced by `BSXML` parsing (`readxml`/`XMLin`) and
validate it field by field. They are called by the srcserver on PUT bodies and
by several library modules.

| sub | lines | rules | dies with | calls |
|---|---|---|---|---|
| `verify_service($p)` | 102-108 | `name` -> filename if defined; every `param[].name` -> filename (dies on undef name). | via filename | `verify_filename` |
| `verify_patchinfo($p)` | 110-120 | `name` -> filename if defined; `category`, if defined, in `security recommended optional feature ptf no_updateinfo`; each `releasetarget[]`: `project` -> projid (required), `repository` -> repoid if defined. "verifies the absolute minimum". | `Invalid category defined in _patchinfo` | `verify_filename`, `verify_projid`, `verify_repoid` |
| `verify_disableenable($h)` | 232-238 | for each element of `disable[]` and `enable[]`: `repository` -> repoid if key exists, `arch` -> arch if key exists. | via callees | `verify_repoid`, `verify_arch` |
| `verify_repo($repo)` | 240-267 | `name` -> repoid; each `path[]`: project+repository; each `arch[]` -> arch; each `releasetarget[]`: project+repository; each `download[]` (DoD): `verify_dod`, `arch` must be one of the repo's `arch[]`, and at most one dod per arch; `base` element forbidden; each `hostsystem[]`: project+repository. | `dod arch X not in repo`, `dod arch X listed more than once`, `repo contains a 'base' element` | `verify_repoid`, `verify_projid`, `verify_arch`, `verify_dod` |
| `verify_proj($proj, $projid?)` | 269-304 | if `$projid` given it must `eq $proj->{name}`; name -> projid; `kind` -> projkind if present; `package[].name` -> packid, unique; `repository[]` -> verify_repo, names unique; `link[].project` -> projid, `vrevmode` if present must be `extend` or `unextend`; flags `build publish debuginfo useforbuild lock binarydownload sourceaccess access` -> verify_disableenable; `mountproject` forbidden; `maintenance.maintains[].project` -> projid; `config` element forbidden. | `name does not match data`, `package X listed more than once`, `repository X listed more than once`, `bad vrevmode attribute: ...`, `project must not have a mountproject`, `project must not have a 'config' element` | `verify_projid`, `verify_projkind`, `verify_packid`, `verify_repo`, `verify_disableenable` |
| `verify_pack($pack, $packid?)` | 306-322 | if `$packid` given: must not be `_project`, must `eq $pack->{name}`; `project` -> projid if present; `name` -> packid; `verify_disableenable($pack)` on the pack itself (marked obsolete); the same 8 flags as proj; `devel.project` -> projid, `devel.package` -> packid. | `illegal package name '_project'`, `name does not match data` | `verify_projid`, `verify_packid`, `verify_disableenable` |
| `verify_link($l)` | 324-355 | `project` -> projid, `package` -> packid, `rev`/`baserev` -> rev, `vrev` -> simple (all optional); at least one of `project`/`package`/`rev` must exist; `cicount` in `add copy local`; `missingok` must be `1` or `true`; patches (`$l->{patches}->{''}` list): each patch hash has exactly one key = type; `branch`: requires `baserev`, must be the only patch, must have no content; `add`/`apply`/`delete`: `name` -> filename; `topadd`: ok; anything else: unknown. | `link must contain some target description`, `unknown cicount`, `missingok in link must be '1' or 'true'`, `more than one type in patch`, `branch link must have baserev`, `branch link must not have other patches`, `branch element contains data`, `unknown patch type` | `verify_projid`, `verify_packid`, `verify_rev`, `verify_simple`, `verify_filename` |
| `verify_aggregatelist($al)` | 357-385 | `resign` in `true false 0 1`; each `aggregate[]`: project -> projid, arch/sourcearch -> arch, `nosources` and `noupdateinfo` must be empty elements, `package[]` -> packid except literal `_repository`, `binary[]` -> filename, `repository[].source/target` -> repoid. | `'resign' attribute must be boolean`, `'nosources' element must be empty`, `'noupdateinfo' element must be empty` | `verify_projid`, `verify_arch`, `verify_packid`, `verify_filename`, `verify_repoid` |
| `verify_channel($ch)` | 387-405 | each `binaries[]`: project -> projid, arch -> arch; each `binary[]`: `name` -> filename, `project` -> projid, `package` -> packid, **`arch` -> `verify_packid`** (line 397, as written), and line 394 checks `$binaries->{binaryarch}` with `verify_arch` but gated on `$binary->{binaryarch}` (as written; looks like a latent bug: verifies the wrong hash). each `target[]`: must have project or repository; project -> projid; repository -> repoid. | `bad target specification` | `verify_projid`, `verify_arch`, `verify_filename`, `verify_packid`, `verify_repoid` |
| `verify_nevraquery($q)` | 407-417 | `arch` -> arch; `name` and `version` must be defined; `"$name-$version[-$release]"` -> filename and simple; `epoch` -> simple if defined. | `binary has no name`, `binary has no version` | `verify_arch`, `verify_filename`, `verify_simple` |
| `verify_attribute($a)` | 419-426 | `namespace` and `name` must be defined and pass simple; `binary` -> simple if present. | `no namespace defined`, `no name defined` | `verify_simple` |
| `verify_attributes($as)` | 428-433 | each `attribute[]` -> verify_attribute | | `verify_attribute` |
| `verify_frozenlinks($fl)` | 435-449 | each `frozenlink[]`: `project` -> projid if present (absent = `/all`); a project (or `/all`) may appear only once, and nothing may follow `/all`; each `package[]`: `name` -> packid, `srcmd5` -> srcmd5, `vrev` -> simple if defined. | `project listed multiple times in frozenlinks` | `verify_projid`, `verify_packid`, `verify_srcmd5`, `verify_simple` |
| `verify_dod($dod)` | 451-461 | `arch` -> arch; `repotype` -> simple; `url` -> url; `master.url` -> url and `master.sslfingerprint` -> simple if present. | via callees | `verify_arch`, `verify_simple`, `verify_url` |
| `verify_multibuild($mb)` | 463-478 | `buildemptyflavor` in `true false 0 1` if defined; cannot have both `package[]` and `flavor[]`; each entry of either list -> packid and must not contain `:`. | `'buildemptyflavor' attribute must be boolean`, `multibuild cannot have both package and flavor elements`, `packid X is illegal in multibuild`, `flavor X is illegal in multibuild` | `verify_packid` |

Note on `::` and `deleted`: there is **no** special `::` handling in any
BSVerify function; `verify_projid` forbids `::` outright. `::` is used in the
backend only as a separator in internal names: job names
`projid::repoid::packid-<md5>` (`BSSched/BuildJob.pm:1344-1345`, shortened to
`:<md5(prp)>::<packid>` if >200 chars), event names (`bs_srcserver:339`,
`BSSrcServer/Service.pm:347`), delta file names (`BSSched/PublishRepo.pm:725-740`),
DoD cache keys (`BSSched/DoD.pm:312`). "deleted" is just the `deleted:bool?` cgi
flag in the `/source` routes which makes the srcserver look in
`$projectsdir/_deleted/` (`bs_srcserver:2096-2101, 2317-2343`).

### Direct callers of BSVerify outside the dispatcher

| caller | what |
|---|---|
| `bs_srcserver:753,1735` | `verify_patchinfo` on `_patchinfo` file content |
| `bs_srcserver:1045-1046` | `verify_projid`/`verify_packid` |
| `bs_srcserver:1692` | `verify_aggregatelist` (`_aggregate`) |
| `bs_srcserver:1760` | `verify_channel` (`_channel`) |
| `bs_srcserver:2250` | `verify_proj($proj, $projid)` on `PUT /source/$project/_meta` |
| `bs_srcserver:2420` | `verify_pack($pack, $packid)` on `PUT /source/$project/$package/_meta` |
| `bs_srcserver:3262,3266` | `verify_attributes` (`_attribute`), `verify_frozenlinks` (`_frozenlinks`) |
| `bs_srcserver:3359` | `verify_linkrev($cgi->{olinkrev})` unless it is the literal `linkrev` |
| `bs_srcserver:3637,3657-3658` | `verify_md5(servicemark)`, per-entry `verify_filename`/`verify_md5` in commitfilelist |
| `bs_srcserver:7441-7506` | sourcepublish event verification (projid, packid, md5) |
| `bs_repserver:1040`, `BSRepServer.pm:144`, `BSRepServer/{Registry,Containerinfo,DoD}.pm`, `BSSched/{Bininfo,BuildResult,BuildJob/Patchinfo,BuildJob/Aggregate}.pm`, `bs_signer:1217` | `verify_nevraquery` on binary metadata (often inside `eval {}` to just skip bad entries) |
| `BSRepServer/Registry.pm:158-159`, `BSRepServer/DoD.pm:100-138`, `BSRepServer/Containerinfo.pm:178-179` | `verify_filename` + `verify_simple` on package/file names |
| `BSPublisher/Registry.pm:74,719` | `verify_regrepo`, `verify_regtag` (in eval) |
| `BSSched/BuildResult.pm:142` | `verify_arch` on each arch entry |
| `BSSrcServer/Multibuild.pm:87` | `verify_multibuild` (in eval) on `_multibuild` |
| `BSSrcServer/Link.pm:460,556` | `verify_filename`, `verify_link` on `_link` |
| `BSSrcServer/Scmsync.pm:266-286` | `verify_packid`, `verify_pack`, `verify_link` for scmsync-imported data |
| `BSSrcServer/Product.pm:133` | `verify_pack` for product-generated packages |
| `BSSrcServer/Service.pm:445`, `bs_service:176`, `bs_publish:2352` | `verify_filename` |
| `bs_worker:3622-3624` | projid/repoid/packid of binaries ("not exactly, but hopefully good enough") |

---

## BSDispatch route syntax

File: `BSDispatch.pm` (285 lines). It has three subs: `parse_cgi` (82-129),
`parse_cgi_singles` (132-142), `compile` (144-244), `dispatch` (246-283).

### How it is wired in

`BSStdServer::server($name, \@ARGV, $conf, $aconf)` (`BSStdServer.pm:406-...`)
is the entry point used by `bs_srcserver` (8250), `bs_repserver` (4849),
`bs_service` (356), `bs_signer`, `bs_publish`, `bs_getbinariesproxy`, etc.
For the main (forking) server config `$conf` it sets, if not already set
(`BSStdServer.pm:468-489`):

```
$conf->{'verifiers'}    ||= $BSVerify::verifiers;     # line 471
$conf->{'dispatch'}     ||= \&dispatch;               # BSStdServer::dispatch, line 472
$conf->{'stdreply'}     ||= \&stdreply;
$conf->{'errorreply'}   ||= \&errreply;
$conf->{'authorize'}    ||= \&authorize;              # BSStdServer::authorize, line 475
$conf->{'ssl_keyfile'}/{'ssl_certfile'}/{'ssl_verify'}/{'bindaddress'} from $BSConfig::...   # 485-488
BSDispatch::compile($conf);                           # line 489
```

and for the AJAX (event-loop) config `$aconf` (`BSStdServer.pm:491-511`) the same
plus `dispatches_call => \&BSWatcher::dispatches_call` (503), which records the
handler and its args in the current event (`BSWatcher.pm:1292-1298`) so a
request that must wait on a remote fetch can be re-run.

`--test` (`BSStdServer.pm:421-430`) only compiles both tables and exits; this is
how "no verifier for X" typos are caught.

`BSStdServer::dispatch` (`BSStdServer.pm:108-155`) is the per-request wrapper:
it applies `trusted_peerip_header`/`client_time_header`, logs
`"METHOD (peer) path?query"`, sets server status, handles `X-Request-ID`, and
then calls `BSDispatch::dispatch($conf, $req)`.

`bs_worker` uses `BSDispatch::compile`/`dispatch` directly (`bs_worker:4925-4928`)
without setting `verifiers`, so its table may only use untyped params.

### Dispatch-table grammar

A table is a flat list of `pattern => \&handler` pairs (`'dispatches'`). Each
pattern string is:

```
[ '!' AUTH ] [ METHODS ':' ] PATH [ ' ' PARAM ]*
```

Parsing happens in `compile` (`BSDispatch.pm:144-244`):

1. **Auth prefix** (lines 154-159): `/^!([^\/\s]*)\s*(.*?)$/`. `!` followed by a
   non-space, non-slash token sets `$needsauth`. `!` alone means `auth`.
   `!-` means "no auth required" (stored as `undef`, line 240). Comma-separated
   lists (`!worker,rw`) are allowed; any one listed class suffices (see
   authorize below). Entries with handler `undef` (e.g. `'!rw :' => undef`) are
   pure auth-setting rows.
2. **Split on whitespace** (line 162): first token is the path, the rest are
   cgi parameter specs. `%XX` escapes in all tokens are decoded (line 163), so a
   literal space or `%` in a cgi name is expressible.
3. **Path** is split on `/` with `split('/', $p, -1)` (keeps trailing empty
   field) (line 171). For each component:
   * `$name` or `$name:type` (lines 173-182): a capture. `$...` captures
     greedily across slashes (`(.*)`), anything else captures `([^\/]*)`. The
     captured value is stored in `$cgi->{name}` and, if `type` is non-empty,
     verified with `$verifiers->{type}`. If no `:type` is given, **the type is
     the name itself** (line 174-176: `$vartype = $var` unless `name:type`).
     `$name:` with an empty type means "no verification". Every `$` path
     variable is also pushed onto the positional-argument list `@args`.
     Compile dies `no verifier for $vartype` (177) if the type is unknown.
   * Otherwise the literal component is `quotemeta`'d (line 184).
4. **Method fixup** (lines 187-195). The first path component is where the
   `METHOD:` prefix ends up (because `'POST:/source'` splits to `('POST:', 'source')`):
   * If the whole pattern is just `METHOD:` with no path (`@p == 1`, e.g. `':'`,
     `'GET:'`), `.*` is appended so it matches any path.
   * If the first component contains `\:` : the method part `$m` is extracted;
     `\|` is un-escaped so `GET|HEAD:` becomes `(?:GET|HEAD):`; an *empty*
     method (`':'`) becomes `[^:]*:` = any method.
   * If there is no method prefix at all (pattern starts with `/`), the default
     is `(?:GET|HEAD|POST):` (line 194). **So an un-prefixed route also answers
     POST.** `PUT`/`DELETE` must always be explicit.
   * The regex is anchored: `'^' . join('/', @p) . '$'` (line 196).
5. **CGI parameter specs** (lines 198-223). Each token:
   ```
   [ '$' ] NAME [ ':' TYPE ] [ QUANT ]        or        [ '$' ] NAME '=' VALUE
   ```
   * Leading `$` (line 203): also pass the value positionally to the handler
     (after the path captures, in table order).
   * Trailing quantifier `?`, `*`, `+` (line 204). None means exactly once
     (required single); `?` optional single; `*` zero or more (array); `+` one
     or more (array).
   * `NAME=VALUE` (lines 205-210): a **fixed-value selector**. Recorded in
     `$cgisingles->{NAME} = VALUE` (stored as `$cpld->[2]`) and the param
     becomes `NAME:` with quantifier `?` (optional, untyped). A quantifier on a
     fixed param is a compile error. This is how `cmd=copy`, `view=info`,
     `aidx=all` route the same path to different handlers: at dispatch time the
     entry only matches if the request's single-valued cgi `NAME` equals `VALUE`
     (line 269-271).
   * `NAME:TYPE` / bare `NAME` (lines 211-213): exactly like path vars, bare
     name => type = name; `NAME:` => untyped. `*:*` is the wildcard
     "accept any other parameters, as arrays"; `*:?` is "accept any other
     parameters, single-valued".
   * Required (no `?`/`*`): generated code dies `parameter 'NAME' is missing`
     (line 214).
   * `+`/`*` go into the `multis` set and are verified element-wise (216-217);
     everything else goes into `singles` and is verified if present (219-220).
6. **Code generation** (lines 224-238). Each entry compiles (via string `eval`)
   to a closure:
   ```perl
   sub { my ($conf, $req) = @_;
     my $cgi = parse_cgi($req, {multis...}, {singles...});
     $cgi->{'pathvar'} = $1; ...                       # path captures
     $verifiers->{'type'}->($cgi->{'pathvar'});        # path verifications
     die("parameter 'x' is missing\n") unless exists $cgi->{'x'};
     $verifiers->{'t'}->($_) for @{$cgi->{'multi'} || []};
     $verifiers->{'t'}->($cgi->{'single'}) if exists $cgi->{'single'};
     my @args; push @args, $cgi->{'a'}; ...            # $-marked params
     $f->($cgi, @args);          # or $callfunction->($f, $cgi, @args) for AJAX
   }
   ```
   Note the order: `parse_cgi` (unknown-parameter check) runs first, then path
   verifiers, then per-param "missing" and verifiers in table order.
   The compiled entry is `[ qr/$pathre/, $closure, $cgisingles?, $needsauth? ]`.

### `parse_cgi` (lines 82-129)

* Query is decoded with `BSHTTP::querydecodekv` (`BSHTTP.pm:83-94`): split on
  `&`, each part split on the first `=`, then url-decoded (`+` -> space,
  `%XX`). A part without `=` has `undef` value, which `parse_cgi` turns into
  `1` (line 104: "assume boolean"). Empty names are skipped (103).
* `$multis` hash: key present => multiple values allowed; value `undef` =>
  collected into an array ref; value string => joined with that separator
  (the compiler always uses `undef`).
* `$singles` hash: key present => a second occurrence dies
  `parameter 'NAME' set multiple times`.
* `'*'` key in either hash is the wildcard for unknown names.
* Any name in neither hash is collected and finally dies
  `unknown parameter 'a', 'b'` (line 127) listing all unknown names sorted.
* The compiler always passes a (possibly empty) hashref for `$singles`, so the
  `||= {'*' => undef}` default on line 97 never applies for compiled routes.
* Values are plain strings; no further normalisation.

`parse_cgi_singles` (132-142) is a lax variant used for the `NAME=VALUE`
selector check and by `external_notification_redis` (`bs_srcserver:7409`): it
returns only names that occur exactly once (duplicates are dropped), `undef`
values become `1`.

### Runtime `dispatch` (lines 246-283)

1. `$path = "$req->{action}:$req->{path}"` (254), e.g. `POST:/source/foo`.
   The path is already url-decoded by BSServer; both path and query must be
   valid UTF-8 (`400 path is not utf-8`, `400 query string is not utf-8`,
   257-260; the query is `%XX`-decoded only for the check).
2. Trailing slashes are stripped (264) except for `METHOD:/` itself.
3. Entries are tried **in table order** (267). For each whose regex matches:
   * if it has `cgisingles` (a `NAME=VALUE` selector), the request's
     single-valued cgi params are parsed once (lazily) and every selector must
     match exactly (`($cgisingles->{$_} || '') ne $value` => skip) (269-272);
   * if the entry has an auth field (4th element), `$auth` is set from it
     (273) **even for handler-less entries**, so `'!rw :' => undef` followed by
     `'!- GET:' => undef` and `'!- HEAD:' => undef` (as in `bs_srcserver` and
     `bs_repserver`) establishes "everything needs `rw`, except GET/HEAD need
     nothing" as a running default for all later entries that lack their own
     `!` prefix;
   * entries with no handler are skipped (274) after setting `$auth`;
   * if `$auth` is set, `$conf->{authorize}->($conf, $req, $auth)` is called;
     if it returns a non-empty list that is returned as the reply (276-278),
     otherwise the handler closure runs (280).
4. No match: `400 unknown request: $path` (282).

### `BSStdServer::authorize` (`BSStdServer.pm:77-106`)

Called with the comma-separated auth-class string (`rw`, `worker`, `worker,rw`,
`auth`). Returns `()` (allow) or dies `500 access denied`.
* If neither `$BSConfig::subjectdnaccess` nor `$BSConfig::ipaccess` is set:
  always allow (80).
* `$BSConfig::subjectdnaccess` (`{ dn-regex => 'class,class' }`): the client
  certificate subject DN (for `https`, via `BSServer::getsubjectdn`) or the
  header named by `$conf->{trusted_subjectdn_header}` is matched against each
  key regex (anchored `^...$`); matching keys add their classes.
* `$BSConfig::ipaccess` (`{ ip-regex => 'class,class' }`): the peer IP (possibly
  overridden by `trusted_peerip_header`, `BSStdServer.pm:113-115`) likewise.
* The special class `_reject` denies outright (102).
* Allowed iff any class required by the route is in the granted set (103).

The template default (`BSConfig.pm.template:35-40`) grants `rw` to `::1`,
`127.*` and the host's own IP, and `worker` to everything else; so with the
`!rw :` / `!- GET:` rows all non-GET/HEAD requests are localhost-only unless a
route says `!worker ...` or `!- ...`.

### Worked examples

| table entry | method regex | path captures (and verifier) | cgi singles | cgi multis | positional args |
|---|---|---|---|---|---|
| `'/source/$project/$package/$filename rev? expand:bool? meta:bool? deleted:bool? view:? noajax:bool?'` (bs_srcserver) | `(?:GET\|HEAD\|POST)` | project(project), package(package), filename(filename) | rev(rev), expand(bool), meta(bool), deleted(bool), view(none), noajax(bool) | | project, package, filename |
| `'POST:/source/$project/$package cmd=copy rev? ... oproject:project? opackage:package? ... file:* ...'` | `POST` | project, package | cmd must equal `copy`; rev(rev), oproject(project), opackage(package), ... | file (untyped array) | project, package |
| `'POST:/source/$project cmd: *:*'` => `unknowncmd` | `POST` | project | cmd (required, untyped) | `*` wildcard: any other param, arrays | project |
| `'/build/$project/$repository/$arch/$package_repository view:? binary:filename* ... module*'` | GET/HEAD/POST | project, repository, arch, **package_repository** (variable name is literally `package_repository`, verifier of the same name: packid or `_repository`) | view | binary(filename), module(module=no-op) | project, repository, arch, package_repository |
| `'!worker,rw /build/$project/$repository/$arch/$package:package_repository view:? ...'` (bs_repserver) | GET/HEAD/POST, auth `worker` or `rw` | package (verified as package_repository) | | | |
| `'/published/$project/$repository/$arch:filename/$filename/$subfilename:filename view:?'` | | arch is verified as a *filename* (published dirs contain non-arch names), subfilename as filename | | | project, repository, arch, filename, subfilename |
| `'GET\|HEAD:/registry/$gunprefix:/$...:regrepo/_trust/tuf/$filename'` | `(?:GET\|HEAD)` | gunprefix (unverified, one component), `...` (greedy `(.*)`, verified as regrepo), filename | | | gunprefix, ..., filename |
| `'POST:/notify/$_type: *:?'` | POST | `_type` unverified | any other params, single | | `_type` |
| `'/request/$id:num'` | | id verified by num | | | id |
| `'!worker /worker $arch $port $state: workerid? working:bool? memory:num? disk:num? buildarch:arch* tellnojob:bool? proto:?'` (bs_repserver) | auth worker | (none) | arch(arch, required), port(port, required), state (required untyped), workerid(no-op), working, memory, disk, tellnojob, proto | buildarch(arch) | arch, port, state |
| `'/_result $prpa+ oldstate:md5? package* code:* ...'` (bs_repserver) | | | oldstate(md5) | prpa(prpa, 1+), package(package), code(untyped) | prpa (an array ref) |
| `'!- /lastevents $filter:* start:num? obsname:? client:? block:bool?'` | no auth | | start(num), obsname, client, block(bool) | filter (untyped) | filter (array ref) |
| `'/ajaxstatus aidx=all'` then `'/ajaxstatus aidx:num?'` | | | first entry only when `aidx=all` literally; otherwise second with numeric check | | |

Handler signature is therefore always `sub { my ($cgi, @positional) = @_ }`
where `@positional` = path `$vars` in path order followed by `$`-marked cgi
params in table order; array-quantified `$` params arrive as array refs.

---

## BSConfiguration logic

File: `BSConfiguration.pm` (98 lines). Loaded by every daemon
(`use BSConfiguration;`) and it in turn does `use BSConfig;` (27), so **loading
BSConfiguration is what loads BSConfig.pm** (from `@INC`, i.e. the file next to
the daemons; the unit tests substitute `t/lib/Test/Mock/BSConfig.pm`, which sets
only `bsdir`, `srcserver`, `reposerver`, `repodownload`, `debuglevel`).

BSConfig.pm itself (template `BSConfig.pm.template`) is plain Perl executed at
`use` time. Noteworthy behaviour of the template beyond variable assignments:
* `$hostname = Net::Domain::hostfqdn() || 'localhost'` and its resolved IP
  (lines 28-30) feed the default `$srcserver`/`$reposerver`/`$serviceserver`
  URLs and the `$ipaccess` table.
* Optional `$frontend` FQDN adds the API host's IP to `$ipaccess` with `rw`
  (32, 43-46).
* **Host-specific override**: after the main body, `bsconfig.<hostname>` in the
  same directory is `require`d if readable (255-260), so a shared BSConfig.pm
  can be specialised per host.
* `$ipaccess`, `$relsync_pool`, `$container_registries`, `$publish_containers`
  are hashes/arrays with non-trivial defaults.

### Precedence: BSConfig.pm beats `configuration.xml`

Lines 31-40: at load time it records which of eight variables were *already
defined by BSConfig.pm*: `obsname`, `api_url`, `proxy`, `noproxy`,
`repodownload`, `enable_download_on_demand`, `forceprojectkeys`,
`schedulerarchs`. Those are never overwritten by the XML.

### `update_from_configuration` (lines 47-66)

Reads `$BSConfig::bsdir/configuration.xml` (path fixed at line 44, **computed
before the `bsdir` default on line 85 is applied**, so if BSConfig.pm does not
set `bsdir` the path would be `/configuration.xml`; in practice the template
always sets it) with `$BSXML::configuration` (non-fatal: missing file => `{}`).
This file is written by the API: `PUT /configuration` on bs_srcserver
(`putconfiguration`, dispatch row `'PUT:/configuration'`), bs_repserver and
bs_service all accept it. Mapping (only for values not pinned by BSConfig.pm):

| XML element | `$BSConfig::` variable | transform |
|---|---|---|
| `name` | `obsname` | as is; final fallback `"build.some.where"` (65) |
| `api_url` | `api_url` | as is |
| `http_proxy` | `proxy` | as is |
| `no_proxy` | `noproxy` | as is |
| `download_url` | `repodownload` | as is |
| `download_on_demand` | `enable_download_on_demand` | `eq 'on'` => 1 else 0 |
| `enforce_project_keys` | `forceprojectkeys` | `eq 'on'` => 1 else 0 |
| `schedulers/arch` (list) | `schedulerarchs` | only set if present |

A change id `"$mtime/$size/$inode"` is remembered (49).

* `check_configuration()` (68-72): re-stat, re-read if the id changed. Called
  periodically by long-running daemons.
* `check_configuration_once()` (75-79): guarded by a flag; "useful for BSServer
  where we fork for every request" - the child checks once.
* `update_from_configuration()` is executed once at module load (81), and
  `BSUtil::setdebuglevel($BSConfig::debuglevel)` (82).

### Defaults applied after loading (lines 84-96)

```
bsdir                    ||= '/srv/obs'
logdir                   ||= "$bsdir/log"
rundir                   ||= "$bsdir/run"
servicetempdir           ||= "$bsdir/service"
scm_cache_high_watermark ||= 80
scm_cache_low_watermark  ||= 70
service_timeout          ||= 3600
redisserver                = undef unless truthy
notifyforward              = undef unless defined
local_registry_signatures_extension = undef unless truthy
sourcepublish_sync         = undef unless truthy
```

Nothing else is defaulted centrally. **Other derived values live in the
consumers**, notably:

* Listen port/proto of each server are parsed out of its own URL:
  `bs_srcserver:126-129` (`5352`, from `$srcserver`), `bs_repserver:95-98`
  (`5252`, from `$reposerver`), `bs_service:57-60` (`5152`, from
  `$serviceserver`), `bs_getbinariesproxy:55-58` (`5254`, from
  `$getbinariesproxyserver`): `$port = $1 if URL =~ /:(\d+)$/`,
  `$proto = 'https' if URL =~ /^(https):/`.
* Worker-facing second port: `$workersrcserver`/`$workerreposerver`
  (`bs_srcserver:8233-8240`, `bs_repserver:4837-4844`) yield `$conf->{port2}`;
  proto must equal the main proto. `BSSched/BuildJob.pm:121-122` falls back to
  `$srcserver`/`$reposerver` when unset.
* `BSStdServer::server` (418, 423, 470, 494): `rundir ||= $BSConfig::rundir || "$bsdir/run"`,
  log file via `BSUtil::openlog($logfile, $logdir, $bsuser, $bsgroup)`, creates
  `$bsdir` chowned to `bsuser:bsgroup`.
* Partitioning (`BSSrcServer/Partition.pm:26-90`): `projid2reposerver` walks
  `@$partitioning` as `(regex, name)` pairs matched with `/^$regex/` in order;
  falls back to `$partition`; dies if still undefined or if the name is not a
  key of `$partitionservers`; returns `$workerpartitionservers->{name}` for
  worker requests when that hash exists, else `$partitionservers->{name}`;
  if `$partitionservers` is unset everything is `$reposerver`.
* Published-repo path mapping (`BSUrlmapper.pm:67-85`): `map_to_extrep($prp)`
  defaults to `"$bsdir/repos/" . ($prp with ':' => ':/')`; `$publishredirect`
  may override by exact prp key, or by regex (longest-key-first) when
  `$publishedredirect_use_regex`; the value can be a string path, a code ref
  (called with `($prp, $prp_ext)`), or an array
  `[path, subpath, downloadurl, stageserver(s), triggerfile]` (template 109-117,
  `get_extrep_stageservers` 93-99, `get_downloadurl` 102-120).

---

## BSConfig variable table

Census: `grep -ho '\$BSConfig::[A-Za-z_0-9]*'` over `bs_*`, `BS*.pm`,
`BSSrcServer/`, `BSSched/` (incl. `BuildJob/`, `EventSource/`), `BSPublisher/`,
`BSRepServer/`, `BSDispatcher/`, `BSSetup/`, plus `build/`, `worker/`, `plugins/`,
`t/`. 163 distinct names. "Template" column: the value in
`BSConfig.pm.template`; `#` means it only appears commented out (with the shown
example); `-` means absent from the template (documented only by usage).
"Files" lists the referencing files (test files omitted).

### Paths and directories

| variable | template | files | meaning |
|---|---|---|---|
| `bsdir` | `'/srv/obs'` (80); default `/srv/obs` (BSConfiguration:85) | 45 files: every daemon and most libs (`bs_srcserver`, `bs_repserver`, `bs_sched`, `bs_publish`, `bs_dispatch`, `bs_signer`, `bs_service`, `bs_warden`, `bs_dodup`, `bs_admin`, `bs_dbtool`, `BSSrcrep.pm`, `BSUrlmapper.pm`, `BSRevision.pm`, `BSRedisnotify.pm`, `BSSrcServer/*`, `BSRepServer/*`, `BSPublisher/*`, ...) | Root of all backend state: `projects/`, `sources/`, `trees/`, `build/`, `repos/`, `jobs/`, `events/`, `workers/`, `db/`, `run/`, `log/`, `service/`, `configuration.xml` ... |
| `logdir` | `"$bsdir/log"` (121) | `BSStdServer.pm`, `BSStdRunner.pm`, `bs_dispatch`, `bs_dodup`, `bs_service`, `bs_redis`, `bs_notifyforward`, `bs_warden` | Log files (`<name>.log`, `<name>.slow.log`, `<name>.crit.log`, `BSStdServer.pm:481-484`). |
| `rundir` | `"$bsdir/run"` (120) | `BSStdServer.pm`, `BSStdRunner.pm`, all daemons | `<runname>.pid/.status/.exit/.restart/.lock` control files (`BSStdServer.pm:418,434-445`). |
| `servicedir` | `"/usr/lib/obs/service/"` (60) | `bs_service:62` | Directory containing source-service executables; default `/usr/lib/obs/service`. |
| `servicetempdir` | `# /srv/obs/service` (61); default `"$bsdir/service"` | `bs_service:55` | Scratch dir where bs_service unpacks sources and runs services. |
| `serviceroot` | `# /opt/obs/MyServiceSystem` (62) | `bs_service:63` | chroot-like prefix: services are run from `"$serviceroot/$servicedir"`. |
| `containers_root` | `# "$bsdir/service/containers"` (356) | `bs_admin:82` | Where `call-service-in-container` stores container images (default `$bsdir/service/containers`). |
| `bsquotafile` | `# '/srv/obs/quota.xml'` (89) | `bs_srcserver:1064,1422` | XML (`$BSXML::quota`) with per-project size quotas enforced on source commits. |
| `keyfile` | `# '/srv/obs/openSUSE-Build-Service.asc'` (210) | `bs_signer:806-807`, `bs_publish:365-367`, `bs_srcserver` | Global public GPG key used when a project has no own key (returned by `/getsignkey`, put into published repos). |
| `certfile` | `# '/srv/obs/openSUSE-Build-Service.cert'` (212) | `bs_srcserver:6382-6383,6421-6422` | Global SSL cert returned by `/getsslcert` / `_keyinfo withsslcert` when the project has none. |
| `gpg_standard_key` | `# "/etc/obs-default-gpg.asc"` (211) | `bs_publish:902-903` | Path written as `gpgkey=` in generated `.repo` files when no project key exists (dies otherwise). |
| `dodup_pubkeydir` | - | `bs_dodup:61` | Directory of trusted public keys for verifying download-on-demand repo metadata signatures. |
| `ssl_keyfile`, `ssl_certfile` | `# "/path/to/tls.key"`, `# "/path/to/tls.pem"` (48-49) | `BSStdServer.pm:485-486`, `bs_worker:4931-4932` | TLS key/cert for servers whose URL is `https://`. |

### Server URLs, ports, bind address

| variable | template | files | meaning |
|---|---|---|---|
| `srcserver` | `"http://$hostname:5352"` (52) | 22 files: `bs_srcserver` (own port), `bs_repserver`, `bs_sched`, `bs_dispatch`, `bs_publish`, `bs_signer`, `bs_worker`, `bs_servicedispatch`, `bs_notifyforward`, `BSNotify.pm`, `BSRepServer.pm`, `BSRepServer/{BuildInfo,ProjPacks,YMP}.pm`, `BSSched/{BuildJob,ProjPacks,Remote,DoD}.pm`, `BSSched/BuildJob/{Aggregate,Patchinfo}.pm`, `BSSched/EventSource/RemoteWatcher.pm`, `BSPublisher/Registry.pm` | Source server URL. The srcserver derives its own listen port/proto from it (`bs_srcserver:126-129`); everyone else uses it as RPC base (`/getprojpack`, `/source/...`, `/notify/<type>` (`BSNotify.pm:49`), `/lastevents`, `/getsignkey`, ...). |
| `reposerver` | `"http://$hostname:5252"` (53) | `bs_repserver` (own port, 95-98), `bs_dispatch`, `bs_srcserver`, `bs_worker`, `BSSrcServer/{Partition,Registry}.pm`, `BSSched/{DoD,BuildJob}.pm` | Repository server URL for this partition; also the default `remoteurl` handed to workers. |
| `serviceserver` | `"http://$hostname:5152"` (54) | `bs_service` (own port, 57-60), `BSSrcServer/Service.pm`, `bs_srcserver` | Source service server URL; srcserver POSTs `/sourceupdate/...` to it. |
| `workersrcserver`, `workerreposerver` | `# "http://$hostname:5353"`, `# "http://$hostname:5253"` (57-58) | `bs_srcserver:8233-8240`, `bs_repserver:4837-4844`, `BSSched/BuildJob.pm:121-122` | Alternative URLs/ports used only by workers; the server listens on `port2` in addition; proto must match the main one. Embedded into job `buildinfo` as `srcserver`/`reposerver`. |
| `getbinariesproxyserver` | - | `bs_getbinariesproxy:57-58` | URL of the optional `bs_getbinariesproxy` daemon (default port 5254); parsed for port/proto. |
| `masterdispatcher` | - | `bs_dispatch:517-863`, `bs_repserver:2049` | URL of a master dispatcher in a multi-partition setup; local dispatcher forwards `PUT /jobs/$arch/$job`, `DELETE`, `/jobs/_mdload` to it and events are relayed unless it equals `$reposerver`. |
| `global_bindaddress` | - | `BSStdServer.pm:488` | Address to bind all servers to (default any). |
| `partitionservers` | `# { 'home' => 'http://home-backend-server:5252', 'main' => ... }` (250-252) | `BSSrcServer/Partition.pm`, `bs_srcserver` | Partition name => repserver URL. |
| `workerpartitionservers` | - | `BSSrcServer/Partition.pm:41`, `BSSched/BuildJob.pm:1368-1369` | Partition name => worker-facing repserver URL (overrides `partitionservers` for worker requests). |
| `api_url` | - (from `configuration.xml`) | `bs_service:162` (exported as `OBS_SERVICE_APIURL`), `BSSched/BuildJob.pm:1304-1307` (SLSA `builder` id) | Public API URL. |
| `repodownload` | - (from `configuration.xml` `download_url`) | `BSUrlmapper.pm:49-120`, `BSRepServer/BuildInfo.pm:224` (`downloadurl` in buildinfo), `bs_srcserver:2675` (`%_download_url` macro) | Public base URL of published repos, e.g. `http://download.opensuse.org/repositories`. |
| `sourcepublish_downloadurl` | - | `BSSched/BuildJob.pm:1305-1306` | Public URL of the source-publish rsync target; `"$url/_slsa"` becomes `slsadownloadurl` in buildinfo. |
| `markfileorigins` | - | `bs_publish:2017-2024, 3444-3466` | Base URL; publisher issues `HEAD $markfileorigins/$prp_ext/$file` for every added/deleted published file (CDN origin marking). |
| `stageserver`, `stageserver_sync` | `# 'rsync://127.0.0.1/put-repos-main'`, `# 'rsync://127.0.0.1/trigger-repos-sync'` (184-185) | `bs_publish:1830-1842, 1904-1918` | rsync targets: published repo content is pushed to `stageserver` (string or list), then a trigger rsync to `stageserver_sync`. Per-prp override via `publishredirect` element 3. |
| `sourcepublish_sync` | `# 'rsync://127.0.0.1/sources'` (192); undef unless truthy | `bs_sourcepublish:45,176`, `bs_publish:2328`, `bs_srcserver:5218` | rsync(-ssl) URL for publishing sources; enables the `sourcepublish` event queue. |

### Users and privileges, access control

| variable | template | files | meaning |
|---|---|---|---|
| `bsuser`, `bsgroup` | `'obsrun'`, `'obsrun'` (81-82) | `BSStdServer.pm`, `BSStdRunner.pm`, all daemons | Unprivileged identity daemons drop to (`BSUtil::drop_privs_to`), owner of `$bsdir`. |
| `bsserviceuser`, `bsservicegroup` | `'obsservicerun'`, `$bsgroup` (87-88) | `bs_service:46-47` (overrides `bsuser/bsgroup`), `bs_admin:84`, `bs_setup:106` | Identity of bs_service. |
| `ipaccess` | hash (35-40): `'^::1$'=>'rw'`, `'^127\..*'=>'rw'`, `"^$ip\$"=>'rw'`, `'.*'=>'worker'` | `BSStdServer.pm:80-99` | Peer-IP regex => comma-separated auth classes (`rw`, `worker`, `auth`, `_reject`), see authorize above. |
| `subjectdnaccess` | - | `BSStdServer.pm:80-91` | Client-cert subject DN regex => auth classes (https or `trusted_subjectdn_header`). |
| `signature_auth_keyfile` | - | `BSStdServer.pm:365-372` (`setup_authenticator`), `BSSched/Remote.pm:799-801` | `{ 'user@realm' => keyfile-or-sshsign-config }`: installs `BSSigAuth` signature authenticators into `$BSRPC::authenticator` for outgoing RPCs to remote OBS instances that require signature auth. |
| `putjob_verify_peerip` | - | `bs_repserver:2343-2350` | If set, `/putjob` is only accepted from the IP the job was dispatched to. |
| `checkclientconnectivity` | - | `bs_repserver:1814-1822` | If set, `/worker` registration first probes `http://peer:port/status` to make sure the server can reach the worker. |
| `norootexceptions` | `# { "my_project/my_package" => 1, "openSUSE:Factory.*/installation-images" => 1 }` (229) | `bs_worker:4253` | Anchored regexes over `proj/pack`; matching packages may build as root (otherwise `--norootforbuild`). |
| `secure_sandboxes` | - | `bs_dispatch:108-109`, `BSDispatcher/Constraints.pm:35-41` | List of sandbox types considered secure (default `xen kvm zvm`); jobs needing a secure sandbox only go to such workers. |
| `forceprojectkeys` | - (from `configuration.xml` `enforce_project_keys`) | `bs_srcserver:2202,2258` | Auto-create a signing key for every new project. |
| `enable_download_on_demand` | - (from `configuration.xml` `download_on_demand`) | `bs_sched:768`, `BSSched/ProjPacks.pm:430,458`, `BSSched/BuildRepo.pm` | Enables DoD repositories (`<download>` elements) in the scheduler. |

### Worker / dispatcher / scheduler

| variable | template | files | meaning |
|---|---|---|---|
| `dispatch_constraint` | `# sub { my ($job, $worker) = @_; ... }` (265-270) | `bs_dispatch:736-755,1359-1360` | Code ref `($info, $worker, $constraints)` => bool; extra site policy for job/worker matching. |
| `dispatch_adjust` | - | `bs_dispatch:1101-1106` | List of `(prpa-regex, adjust)` pairs added to the scheduling priority of matching `prp/arch`. |
| `powerpkgs`, `powerhosts` | - | `bs_dispatch:100-104, 1204-1276` | Package names that are "power" jobs and hostname prefixes of workers reserved for them. |
| `secure_sandboxes` | - | see above | |
| `workerhostcheck` | `# 'my_check_script'` (225) | `bs_worker:138` | Script run by the worker to decide if the host may take a job. |
| `buildlog_maxsize`, `buildlog_maxidle` | `# 500*1000000`, `# 8*3600` (219-220) | `bs_worker:126-127` | Kill a build when the log exceeds the size or no output for that many seconds. |
| `nokiwitree` | - | `bs_worker:5108` | Disable sending the kiwi tree back with the result. |
| `deltagen_mopt` | - | `bs_worker:4130` | Memory option passed to `makedeltarpm`/deltagen on the worker. |
| `localarch` | `# "x86_64"` (215) | `bs_worker:1705`, `BSSched/{BuildJob,Checker,Remote}.pm`, `BSSched/BuildJob/{KiwiProduct,ProductCompose,Patchinfo}.pm`, `bs_srcserver` | Real host arch to use for the pseudo arch `local` (product builds). |
| `schedulerarchs` | - (from `configuration.xml` `schedulers/arch`) | `bs_repserver:2968-2969` | Arch list used by `/workerstatus` to know which scheduler daemons should exist. |
| `sched_asyncmode` | `# 1` (93) | `bs_sched:77` | Asynchronous remote fetches in the scheduler. |
| `sched_startupmode` | `# 0` (98) | `bs_sched:78` | 0 fetch all, 1 skip remote, 2 delay all package data fetches at cold start. |
| `sched_maxserverload` | - | `bs_sched:81,591` | Max parallel load the scheduler puts on the srcserver (default 1). |
| `genmetaalgo` | - | `bs_sched:572-578` | Force the `.meta` generation algorithm version (else auto from BSSolv). |
| `delayscale_med`, `delayscaler_med`, `delayscale_high` | - | `BSSched/Lookat.pm:263-264` | Scale factors for the scheduler's "look at later" backoff (note: `delayscale_med` only gates, the multiplier actually used is `delayscaler_med`, default 10; high default 1). |
| `maxmetahardlink` | - | `BSSched/BuildRepo.pm:138,187` | When a `.meta` file reaches this many hard links, duplicate it (`.linkedmetadup`) to avoid link-count limits. |
| `deep_check_dependent_projects_on_macro_change` | - | `BSSched/ProjPacks.pm:225` | Trigger deep checks of dependent prps when a project config/path changes critically. |
| `related_projects` | - | `BSSched/ProjPacks.pm:898` | `{ "projA/projB" => 1 }`: treat two projects as related (like parent/child) for rebuild/prefer decisions. |
| `usesolvstate` | - | `BSSched/Remote.pm:391` | Allow the `solvstate` view for remote repositories outside of partitions. |
| `interconnect_serialize_slots` | - | `BSSrcServer/Remote.pm:662-664` | Hash remote-call serialization keys into N slots to bound concurrency to remote instances. |
| `relsync_pool` | hash (154-181) arch => pool name | `bs_sched:272`, `bs_srcserver:6891` | Release-number counters are synced only between archs of the same pool. |
| `limit_projects`, `limit_projects_use_regex` | `# { "ppc" => [ "openSUSE:Factory", "FATE" ] }` (146-149) | `bs_srcserver:1175-1184` | Per-arch allow-list (names or anchored regexes) of projects the scheduler for that arch gets to see. |
| `kiwiprojects` | - | `bs_srcserver:1930-1935` | If set, kiwi image builds are only allowed in the listed projects. |
| `ignore_project_config_errors` | - | `BSSched/Checker.pm:402,503`, `BSRepServer/Checker.pm:121,130`, `bs_srcserver:1859,1899` | Do not mark packages broken on prjconf parse errors. |
| `extraconfig` | - | `bs_srcserver:2758-2761`, `BSSched/ProjPacks.pm:1678-1681`, `BSRepServer/ProjPacks.pm:131-134` | `{ projid-regex => "prjconf text" }` appended to every matching project's build config. |
| `extramacros` | `# { ".*" => "%packager ...\n%vendor ...\n" }` (199-203) | `bs_srcserver:2753-2755` | `{ projid-regex => macros }` appended to the project macros. |
| `slsaprovenance` | `# [ "openSUSE:Leap:.*:Update", ... ]` (142) | `BSSched/BuildJob.pm:1302` | prp prefix regexes for which SLSA provenance is generated (keeps all used binaries). |
| `partition` | `# 'home'` (242) | `bs_dispatch:468-475`, `BSSrcServer/Partition.pm`, `BSSched/{ProjPacks,DoD}.pm`, `BSSched/EventSource/RemoteWatcher.pm`, `bs_repserver`, `BSRepServer/ProjPacks.pm` | Name of the partition this binary backend serves; sent as `partition=` to `/getprojpack` and `/jobs`. |
| `partitioning` | `# [ 'home:' => 'home', '.*' => 'main' ]` (246-248) | `BSSrcServer/Partition.pm`, `BSSrcServer/Registry.pm:78`, `bs_srcserver` | Ordered `(projid-regex, partition)` pairs. |
| `disable_data_sync` | `# 1` (105) | `bs_srcserver:?`, `bs_repserver:102`, `bs_sched:88`, `bs_dispatch:153`, `bs_service:52`, `BSStdRunner.pm:213` | Skip `fdatasync` before rename (faster, unsafe on XFS). |
| `nosharedtrees` | `2` (128) | `BSSrcrep.pm:36,85,282-283`, `bs_admin:65` | 0 trees shared under `sources/<pack>`, 1 per-project `trees/<proj>/<pack>`, 2 per-project with fallback to shared. |
| `source_db_sqlite`, `published_db_sqlite` | `1`, `1` (134-135) | `bs_srcserver`, `bs_dbtool`, `BSSrcServer/{LinkinfoDB,ScmsyncDB}.pm` | Use SQLite instead of BSDB for the source/linkinfo and published-binary search databases. |
| `scm_cache_high_watermark`, `scm_cache_low_watermark` | defaults 80 / 70 (BSConfiguration:89-90) | only BSConfiguration.pm in backend (used by the `cleanup_scm_cache` script) | Disk-usage percentages for pruning the scm cache. |
| `new_full_handling` | - | `bs_admin:67-68` | bs_admin `:full` repository handling mode (default 1). |

### Concurrency limits / server tuning

| variable | template | files | meaning |
|---|---|---|---|
| `srcserver_maxchild` | `# 20` (65) | `bs_srcserver:8230` | Max forked request handlers. |
| `service_maxchild` | `# 20` (67) | `bs_service:68` | Same for bs_service. |
| `binproxy_maxchild` | `# 40` (69) | (not referenced; the real name is `getbinariesproxyserver_maxchild`) | stale template entry |
| `getbinariesproxyserver_maxchild`, `getbinariesproxyserver_cachesize` (MB), `getbinariesproxyserver_extraconf` | - | `bs_getbinariesproxy:63,637-638` | Proxy limits, cache size (default 1 GiB), extra server conf. |
| `servicedispatch_maxchild`, `servicedispatch_limitinprogress` | `# 4` (280), - | `bs_servicedispatch:38,210` | Parallel service runs; cap on in-progress per the BSRunner `limitinprogress` option. |
| `publish_maxchild`, `publish_maxchild_flavor`, `publish_getflavor` | - | `bs_publish:72-73,4058` | Publisher parallelism; `getflavor` is a code ref `($req) => flavor` and `maxchild_flavor` `{ flavor => n }` lets BSRunner (`BSRunner.pm:54-83`) cap concurrency per flavor (e.g. per project class). |
| `signer_maxchild`, `signer_maxchild_flavor`, `signer_getflavor` | - | `bs_signer:63-64,1304` | Same for bs_signer. |
| `sourcepublish_maxchild` | - | `bs_sourcepublish:48` | Same for bs_sourcepublish. |
| `publish_penalty_multiplier` | - | `bs_publish:3883-3890` | If the last publish of a prp took >300 s, delay the next by duration x multiplier (clamped 1 h); 0 disables. |
| `srcserver_extraconf`, `reposerver_extraconf`, `serviceserver_extraconf` | - | `bs_srcserver:8241`, `bs_repserver:4845`, `bs_service:352` | Hash merged into the BSServer `$conf` (e.g. `maxchild`, timeouts, `trusted_peerip_header`). |
| `srcserver_ajaxpartitions`, `reposerver_ajaxpartitions` | - | `bs_srcserver:8231`, `bs_repserver:4835` | Number/list of extra AJAX (event-loop) server partitions (`BSStdServer.pm:396-404`). |
| `srcserver_notification_keepalive_maxcount` | - | `bs_srcserver:7431` | Allow HTTP keep-alive on `/notify/redis` up to N requests per connection. |
| `service_timeout` | default 3600 (BSConfiguration:91) | `bs_service:137,233`, `bs_servicedispatch:108`, `BSSrcServer/Service.pm:392` | Max seconds a source service may run. |
| `debuglevel` | `# 1` (283) | `BSConfiguration.pm:82` | `BSUtil::setdebuglevel`. |
| `ssl_verify` | - | `BSStdServer.pm:487`, `BSServer.pm:354-356` | Enable client-cert verification; hash `{ mode, verify_file, verify_dir }`. |

### Publishing

| variable | template | files | meaning |
|---|---|---|---|
| `publishredirect`, `publishedredirect_use_regex` | `# { 'My:Project/repo' => [ dir, subpath, url, rsync-stage, trigger ], 'My:Other:Project/images' => '/srv/local/other/directory/' }` (109-117) | `BSUrlmapper.pm:75-85`, `bs_repserver` | Per-prp override of the published directory (string, array, or code ref); regex keys when `_use_regex`. |
| `publishedhook`, `publishedhook_use_regex` | - | `bs_publish:3654-3670` | `{ prp => [prog, args...] }` executed after a prp is published (regex keys, longest-first, when `_use_regex`). |
| `unpublishedhook`, `unpublishedhook_use_regex` | - | `bs_publish:3637-3650` | Same for unpublish. |
| `publishprogram` | - | `bs_publish:3490-3499,3674,3748` | `{ prp => program }`: run `program prp extrep` instead of the built-in repo generation (forked, `.pplock`). |
| `publisher_compile_content_hook` | - | `bs_publish:100-102,2460-2461` | `{ prp => code }` (or a filename `require`d to produce it) called while compiling repo content. |
| `createrepo_rpmmd_hook` | - | `bs_publish:96-98,864-865` | Code ref `($projid, $repoid, $extrep, \%options, $data, \@legacyargs)` to tweak createrepo invocation. |
| `createrepo`, `modifyrepo` | `# '/usr/bin/createrepo_c'`, `# '/usr/bin/modifyrepo_c'` (273-275) | `bs_publish:657-658,1588,1647` | Binaries to use. |
| `rsync_extra_options` | `# "--whole-file"` (189) | `bs_publish:1866`, `bs_sourcepublish:161` | Extra rsync flags. |
| `packtrack` | `[]` (131) | `bs_publish:2044,2322` | prp prefix regexes for which binary release tracking is reported to the API (always on for `maintenance_release`). |
| `publish_nodb` | - | `bs_publish:200` | `{ prp => 1 }`: skip the published-binary database for these prps. |
| `publish_diffnotify` | - | `bs_publish:2338-2346` | `(prp-regex, value)` pairs enabling publish diff notifications (`BSPublisher/Diffnotify.pm`). |
| `extradeltarepos` | - | `BSSched/PublishRepo.pm:576` | `{ prp => dir }`: alternative old-repo dir to compute deltarpms against. |
| `prp_ext_map` | - | `BSUrlmapper.pm:40-41,108-109,131-132` | `{ prp => public URL or undef }`: explicit download-URL per prp (undef = not published); also used in reverse by `urlmapper`. |
| `sourcepublish_filter` | `# [ "openSUSE:.*", "SUSE:.*" ]` (194) | `bs_publish:2329-2330` | prp prefix regexes whose sources are published. |
| `sourcepublish_blocklist` | `# [ "flash-player", ... ]` (196) | `bs_srcserver:7450-7507` | Package names never source-published. |
| `markfileorigins` | - | see URLs | |
| `obsname` | - (from `configuration.xml` `name`, fallback `build.some.where`) | `bs_publish`, `bs_sched:465`, `bs_productconvert`, `bs_service:163` (`OBS_NAME`), `BSSched/{BuildJob,EventHandler}.pm`, `BSSrcServer/Signkey.pm`, `bs_srcserver` | Instance name; used as `obs://<obsname>` disturl prefix, in signkey user ids, and in `/lastevents obsname=`. |

### Signing / keys / containers / registry

| variable | template | files | meaning |
|---|---|---|---|
| `sign` | `# '/usr/bin/sign'` (206) | `bs_signer`, `bs_publish`, `bs_notar`, `bs_regpush`, `bs_srcserver`, `bs_repserver`, `BSSrcServer/Signkey.pm`, `BSSched/BuildJob/Aggregate.pm`, `BSRepServer/Registry.pm`, `BSPublisher/{Container,Registry}.pm` | Path of the `sign` client (obs-sign). Unset disables all signing. |
| `sign_project` | `# 1` (208) | `bs_signer:79-93,792,957`, `bs_publish`, `BSSrcServer/Signkey.pm`, `bs_srcserver`, `BSPublisher/*`, `BSRepServer/Registry.pm` | Pass `--project NAME` to sign (per-project keys on the signd side). |
| `sign_package` | - | `bs_signer:795,958` | Additionally pass `--package NAME`. |
| `sign_type` | - | `bs_signer:235,465-533,834-840`, `bs_publish:411,446`, `BSSrcServer/Signkey.pm`, `BSPublisher/Container.pm` | Enable `--signtype <cms|appx|helm|...>` arguments for non-RPM signing (`setsigntype`, `bs_signer:832`). |
| `sign_flavor` | - | `bs_signer:940-941`, `bs_publish:2303-2305`, `BSSrcServer/Signkey.pm:78-79`, `BSSched/BuildJob.pm:1311-1313`, `bs_repserver` | Allowed list of sign flavors; a project's `buildflags:signflavor` must be in it. |
| `disable_sslcert_cache` | - | `bs_srcserver:2137,2192,6347` | `{ projid => 1 }`: never cache the project's SSL cert. |
| `container_registries` | hash (285-302): `'local' => { server => $hostname, pushserver => 'local:' }` plus commented remote examples with `user`, `password`, `repository_base`, optional `mapper` code ref (308-320) | `BSPublisher/Container.pm:54-67`, `bs_repserver:4459-4480`, `bs_srcserver:7817`, `bs_regpush:1127` | Registry definitions by name. |
| `publish_containers` | `[ '.*' => ['local'] ]` (323-330) | `BSPublisher/Container.pm:54-56`, `bs_repserver:4451`, `bs_srcserver` | Ordered `(projid-or-prp regex, [registry names])` pairs. |
| `local_registry_signatures_extension` | `1` (334); undef unless truthy | `bs_srcserver:7874` | Announce `X-Registry-Supports-Signatures: 1`. |
| `dodcreds` | - | `bs_dodup:487,563`, `BSRepServer/Registry.pm:129` | `{ url => "user:pass" }` credentials for DoD registries/repos. |
| `dodupblacklist` | - | `bs_dodup:797-798` | Package-name prefix regexes to ignore in DoD updates. |
| `dodup_checkinterval_ok`, `dodup_checkinterval_error` | - | `bs_dodup:59-60` | Re-check intervals for DoD repos after success/failure. |
| `notification_namespace` | - | `plugins/notify_rabbitmq.pm:49` | Prefix for event type names (default `OBS`). |

### Notifications / messaging

| variable | template | files | meaning |
|---|---|---|---|
| `notification_plugin` | `# "notify_hermes notify_rabbitmq"` (76) | `BSSrcServer/Notify.pm:39-41` | Space-separated plugin module names in `plugins/` invoked via `/notify_plugins/$type`. |
| `hermesserver`, `hermesnamespace` | `# "http://$hostname/hermes"`, `# "OBS"` (72-73) | `plugins/notify_hermes.pm:39-64` | Hermes notification endpoint. |
| `rabbitmqconfig` | - | `plugins/notify_rabbitmq.pm:57` | `{ server, user, password, vhost, ... }` for Net::AMQP::RabbitMQ. |
| `redisserver` | - ; undef unless truthy | `bs_redis`, `bs_dbtool`, `bs_dispatch`, `bs_signer`, `bs_warden:96`, `bs_repserver`, `bs_srcserver`, `BSSched/{BuildJob,Checker,EventHandler,BuildResult}.pm`, `BSSrcServer/ScmsyncDB.pm` | `redis[s]://[user:pass@]host[:port]` (regex at `bs_redis:268`); enables the redis result/notification cache maintained by `bs_redis`. |
| `notifyforward` | - ; undef unless defined | `BSSched/Checker.pm:123`, `bs_repserver:2076` | Explicit on/off for the `bs_notifyforward` daemon (default: on iff `redisserver`). |
| `publish_diffnotify` | - | see Publishing | |

### Remote / proxy / interconnect

| variable | template | files | meaning |
|---|---|---|---|
| `proxy` | - (from `configuration.xml` `http_proxy`) | `BSRPC.pm`, `bs_dodup:64`, `bs_service:66`, `bs_sched:475`, `BSSrcServer/Remote.pm`, `bs_repserver`, `BSRepServer/{BuildInfo,DoD,Remote}.pm`, `bs_srcserver`, `BSSched/EventHandler.pm` | HTTP proxy URL for outgoing RPC/DoD/service traffic. |
| `noproxy` | - (from `configuration.xml` `no_proxy`) | `BSRPC.pm:102`, `bs_service:67` | Comma list of hosts bypassing the proxy. |
| `interconnect_serialize_slots`, `usesolvstate`, `signature_auth_keyfile`, `masterdispatcher`, `partition*` | - | see above | |

### Source services

| variable | template | files | meaning |
|---|---|---|---|
| `servicedispatch` | `1` (278) | `BSSrcServer/Service.pm:309,566`, `bs_srcserver:5216` | Run services asynchronously through `bs_servicedispatch` (event queue) instead of inline. |
| `old_style_services` | `# 1` (232) | `BSSrcServer/Service.pm:79,296,474`, `BSSrcServer/Scmsync.pm:48`, `bs_srcserver:456` | Legacy service mode (no servicemark; incompatible with scm bridge). |
| `service_wrapper` | `# { '*' => '/usr/lib/obs/server/call-service-in-container', '<name>' => ... }` (342-345) | `bs_service:200-203` | Wrapper executable per service name (`*` default). |
| `container_image`, `container_custom_opt`, `container_network_opt`, `docker_custom_opt`, `gems_mirror`, `geminabox_container`, `obs_service_user`, `obs_service_pass`, `obs_service_network` | 350-375 | only `container_image` (`bs_admin:83`) is referenced by Perl code; the rest are consumed by the shell wrapper `call-service-in-container` | Container-based service execution options. |
| `extended_service_configuration` | - | not referenced in the backend tree | (name from the task brief; does not exist here) |

### Variables that appear in the template but are never read by backend Perl

`binproxy_maxchild` (69), `xenstore_maxsize`, `gettimeout` (221-222; the
worker has a hard-coded `$gettimeout = 3600`), `container_custom_opt`,
`container_network_opt`, `docker_custom_opt`, `gems_mirror`,
`geminabox_container`, `obs_service_user`, `obs_service_pass`,
`obs_service_network` (shell-consumed).


---

# Part D — Operational / Admin surface of the OBS backend

Source tree: `/home/jeffm/src/open-build-service` (branch master, db93ae665b).
All paths below are relative to that root unless absolute. Line references are
`file:line` or `file:start-end` as of this checkout.

Conventions used in this document:

- `$bsdir` = `$BSConfig::bsdir`, default `/srv/obs` (`src/backend/BSConfig.pm.template:80`,
  fallback in `src/backend/BSConfiguration.pm:85`).
- `$rundir` = `$bsdir/run`, `$logdir` = `$bsdir/log` (`BSConfig.pm.template:120-121`,
  `BSConfiguration.pm:86-87`).
- "event" = an XML file (`$BSXML::event`) dropped into `$bsdir/events/<queue>/` plus a
  `BSUtil::ping` on `$bsdir/events/<queue>/.ping`. This is the universal
  admin → daemon control channel; **bs_admin never speaks BSRPC** (verified:
  `grep -n "BSRPC\|BSServer" src/backend/bs_admin` returns nothing).

---

## 1. `src/backend/bs_admin` — the admin tool

File: `src/backend/bs_admin` (1431 lines). Installed as
`/usr/lib/obs/server/bs_admin`; `/usr/sbin/obs_admin` is a 2-line
`exec /usr/lib/obs/server/bs_admin "$@"` wrapper (`dist/obs_admin:1-2`).

### 1.1 Startup, globals, privilege model

- `BEGIN` block (`bs_admin:24-35`): pushes `$wd/build` and `$wd` on `@INC`; special-cases
  `--query-config <var>` *before* loading anything else so the shell wrappers
  (`dist/obsscheduler:28`) can call it cheaply. Prints `$BSConfig::<var>`.
- Modules (`bs_admin:37-58`): `Build`, `BSConfiguration`, `BSFileDB`, `BSWatcher`,
  `BSUtil`, `BSXML`, `BSKiwiXML`, `BSProductXML`, `BSDB`, `BSDBIndex`, `BSSolv`,
  `BSSrcrep`, `BSRevision`, `BSSrcServer::LinkinfoDB`. It installs
  `$BSRevision::storelinkinfo = \&BSSrcServer::LinkinfoDB::storelinkinfo` (`bs_admin:60`).
- Directory globals (`bs_admin:62-78`):
  `$reporoot=$bsdir/build`, `$eventroot=$bsdir/events`, `$projectsdir=$bsdir/projects`,
  `$srcrepdir=$bsdir/sources`, `$configfile=$bsdir/configuration.xml`,
  `$treesdir` = `$bsdir/trees` if `$BSConfig::nosharedtrees` else `$srcrepdir`,
  `$sourcedb=$bsdir/db/source`, `$rundir=$bsdir/run`, `$dodsdir=$bsdir/dods`.
- Service-container globals (`bs_admin:80-85`): `$sc_root =
  $BSConfig::containers_root || "$bsdir/service/containers"`,
  `$sc_container = $BSConfig::container_image || "localhost/obs-source-service-podman:latest"`,
  `$sc_serviceuser = $BSConfig::bsserviceuser || "obsservicerun"`.
- **Privileges**: the tool is meant to be run as root. Only three subcommands call
  `BSUtil::drop_privs_to($BSConfig::bsuser,$BSConfig::bsgroup)`:
  `--update-request-db` (`bs_admin:740`), `--update-source-db` (`:748`),
  `--update-scmsyncdb` (`:1419`). `--update-project-signing-key` deliberately stays
  root and `chmod 0600`/`chown`s the result (`bs_admin:1380-1392`).
- Argument parsing is a hand-rolled `while (@ARGV) { my $arg = shift; if/elsif ... }`
  loop (`bs_admin:694-1431`), **not** `Getopt::Long` (Getopt::Long is `use`d at line 40
  but never called). Multiple subcommands may be chained on one command line;
  unknown args print help and `exit(1)` (`bs_admin:1427-1430`).

### 1.2 Helper primitives (what the subcommands are built from)

| Helper | Lines | Effect |
|---|---|---|
| `write_event($project,$repo,$arch,$event,$package,$job)` | 541-555 | Builds name `type::project::package::repo` (md5-hashed if >200 chars), writes `$eventroot/$arch/<name>` atomically (`.name$$` then rename via `writexml`), pings `$eventroot/$arch/.ping`. Target: **the scheduler for `$arch`** (`bs_sched` reads `$bsdir/events/$arch`, `bs_sched:93,448`). |
| `write_publish_event($project,$repo)` | 557-565 | Writes `$eventroot/publish/<project>::<repo>` type=`publish`; pings. Target: **bs_publish** (`bs_publish:84`). |
| `write_source_publish_event($project,$package,$md5,$included)` | 567-577 | Writes `$eventroot/sourcepublish/<project>::<package>::<md5>` type=`sourcepublish`; pings. Target: **bs_sourcepublish** (`bs_sourcepublish:50`). |
| `prefer_publish_event($name)` | 579-583 | Renames `$eventroot/publish/$name` → `_$name` and touches `$rundir/bs_publish.rescan`. |
| `scan_repo`, `dump_repo`, `wipe_notyet`, `dump_state`, `dump_memstats`, `dump_projectstats`, `dump_pmat`, `shutdown_scheduler`, `rebuild_full_tree`, `check_project`, `check_package` | 585-662 | Thin wrappers around `write_event` with event types `scanrepo`, `dumprepo`, `wipenotyet`, `dumpstate`, `memstats`, `projectstats`, `dumppmat`, `exitcomplete`, `useforbuild`, `recheck`/`admincheck`/`package`. All are dispatched in `BSSched/EventHandler.pm:46-75`. |
| `check_scheduler_status($arch)` | 619-625 | Returns 0 (running) if `BSUtil::lockcheck('>>', "$rundir/bs_sched.$arch.lock")` sees the lock held. |
| `shutdown_all_schedulers($waittime)` | 627-650 | Discovers archs from `$rundir/bs_sched.*.lock`, sends `exitcomplete` to each running one, optionally polls up to `$waittime` s; exit 2 if any still running. Used by `dist/obsscheduler stop` (`dist/obsscheduler:61-62`). |
| `dump_nStore($file,@sel)` | 374-380 | `BSUtil::retrieve` (Storable) + `Data::Dumper`. |
| `dump_solv($fn)` | 390-399 | Loads a `.solv` via `BSSolv::pool->repofromfile`, dumps `pkg2data` for each package. |
| `clone_repository(...)` | 401-452 | See `--clone-repository`. |
| `update_request_db`, `insert_request_db` | 454-485 | Rebuild `$bsdir/db/request` (BSDB index, `noindex id`) from `$bsdir/requests/*`. |
| `check_xml_file`, `check_product_xml_file`, `check_kiwi_xml`, `check_constraints_xml`, `check_meta_xml` | 487-540 | Parse-only validators. |
| `update_scmsyncdb($projid)` | 664-673 | Re-stores `scmsync` URLs for `_project` and every package via `BSSrcServer::ScmsyncDB::storescmsync`. |
| `findfile` / `find_latest_file` | 288-321 | Copy of srcserver's "find the kiwi/_constraints file in latest rev" logic. |

### 1.3 Complete subcommand catalogue (dispatch order in `bs_admin:694-1431`)

Grouped as in the usage text (`bs_admin:106-282`). "Target" names the daemon that
consumes the side effect.

#### General
| Option | Args | Lines | What it does / touches |
|---|---|---|---|
| `--help` | – | 701-704 | Prints `echo_help` (`:106-282`), exit 0. |
| `--query-config` | `<variable>` | 28-34 (BEGIN), 1333-1337 | Prints `$BSConfig::<variable>`. Pure read of `BSConfig.pm`. |

#### Job controlling (scheduler events; target = `bs_sched <arch>`)
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--shutdown-scheduler` | `<arch>` | 865-868 | `write_event('', undef, $arch, 'exitcomplete')` → scheduler dumps state to `$rundir/bs_sched.$arch.state` and exits (`EventHandler.pm:66,479`). Used as `ExecStop` of `obsscheduler@.service`. |
| `--shutdown-all-schedulers` | `[<waitsecs>]` | 869-873 | See helper; exit 0/2. |
| `--check-project` | `<project> <arch>` or `<project> <repo> <arch>` | 874-880 | event `recheck` for project[/repo]. |
| `--check-all-projects` | `<arch>` | 881-884 | event `recheck` with no project (whole arch). |
| `--deep-check-project` | `<project> [<repo>] <arch>` | 906-912 | event `package` for project (re-fetches sources: "in case of lost events"). |
| `--check-package` | `<project> <package> <arch>` | 885-890 | event `package` with package set. |
| `--force-check-project` | `<project> <repo> <arch>` | 983-990 | event `wipenotyet` + `admincheck` (high prio, bypasses the "blocked due to calculating time" throttle; `EventHandler.pm:230-236`). |
| `--rescan-repository` | `<project> <repo> <arch>` | 976-982 | event `wipenotyet` + `scanrepo` → scheduler rescans `$bsdir/build/<prj>/<repo>/<arch>/:full` and rewrites the `:full.solv`/cache. |
| `--rebuild-full-tree` | `<project> <repo> <arch>` | 900-905 | event `useforbuild` → scheduler rebuilds the `:full/` tree content (`EventHandler.pm:69`). |
| `--dump-repository-state` | `<project> <repo> <arch>` | 859-864 | event `dumprepo` (scheduler prints its in-memory repo state into its log). |
| `--dump-state` | `<arch>` | 815-818 | event `dumpstate` → scheduler writes `$rundir/bs_sched.$arch.state` **without exiting** (`EventHandler.pm:68,472-479`). |
| `--dump-memstats` | `<arch> [what]` | 819-823 | event `memstats` with `job`=what. |
| `--dump-pmat` | `<arch>` | 824-827 | event `dumppmat` (Devel::MAT heap dump). |
| `--dump-projectstats` | `<arch>` | 828-831 | event `projectstats`. |
| `--dump-project-from-state` | `<project> <arch>` | 832-845 | **Local read** of `$rundir/bs_sched.$arch.state` (Storable); dumps `remoteprojs{project}` and `projpacks{project}`. Requires a prior `--dump-state`. |
| `--recheck-dod` | `<URL-or-project prefix>` | 1352-1357 | Writes `$bsdir/dods/<md5(prefix)>.recheck` containing the prefix and touches `$bsdir/dods/.changed`. Target: **bs_dodup** (`bs_dodup:46`, dods dir). |

#### Publishing (target = `bs_publish` / `bs_sourcepublish`)
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--publish-repository` | `<project> <repo>` | 936-957 | Just `write_publish_event`. Publisher may skip if `:repoinfo` state unchanged. |
| `--republish-repository` | `<project> <repo>` | same | Loads `$reporoot/<prj>/<repo>/:repoinfo`, deletes its `state` key, stores it back atomically (`.:repoinfo` → `:repoinfo`), then publish event → forces a publish. |
| `--unpublish-repository` | `<project> <repo>` | same | For each arch dir having `:repodone`, `rm -rf <arch>/:repo` (and — bug — `"$repo/$a/:repodone"` uses `$repo` not `$repodir`, line 950), then publish event so the publisher removes the published tree and updates the published DB. Works on locked projects. |
| `--prefer-publish-event` | `<eventfilename>` | 958-961 | Renames event to `_<name>` in `$eventroot/publish/` and touches `$rundir/bs_publish.rescan` (the publisher sorts `_`-prefixed names first). |
| `--publish-source` | `<project> <package> <md5>` | 930-935 | `sourcepublish` event (no `included` list). |
| `--publish-source-via-report` | `<reportfile>` | 913-929 | Parses a `$BSXML::report`; extracts project/package/srcmd5 from the root `disturl` (`obs://host/prj/repo/md5-pkg`) and each `<binary disturl>` into `included`; writes `sourcepublish` event. |

#### Repository cloning
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--clone-repository` | `[--volatile] <srcprj> <srcrepo> [<dstprj>] <dstrepo>` | 962-975, impl 401-452 | Requires `$reporoot/<dst>` to exist (scheduler must have created it). `cp -al` (hardlink copy) of `$reporoot/<src>/<srcrepo>` into `$bsdir/tmp/bs_admin.$$`; unlinks `:repoinfo`; per arch unlinks `:jobhistory`, `rm -rf :repo :repodone`; with `--volatile` and `$BSConfig::new_full_handling` (default 1, `bs_admin:65-66`) replaces `_volatile` with a hardlink copy of `:full`. Then `rename` dst→`tmp.old`, `rename` tmp→dst, writes a `scanrepo` event for every arch dir, `rm -rf tmp.old`. |

#### Maintenance / DB rebuilds
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--update-source-db` | `[<project>]` | 747-757 | Drops privs. For each project (or the one given) and each package: `BSRevision::getrev_local` + `BSRevision::updatelinkinfodb(projid,packid,rev,lsrev(rev))` → rewrites the linkinfo index in `$bsdir/db/source` (BSDB or sqlite, per `$BSConfig::source_db_sqlite`). |
| `--update-request-db` | `[<requestfile>]` | 739-746 | Drops privs. Rebuilds `$bsdir/db/request` (BSDB) from `$bsdir/requests/*` (all, or single id). Note `bs_check_consistency:380` calls this db "obsolete". |
| `--update-scmsyncdb` | `[<project>...]` | 1418-1426 | Drops privs. Re-stores scmsync URLs via `BSSrcServer::ScmsyncDB` for all/given projects (sqlite `scmsync` table + redis job). |
| `--remove-old-sources` | `<days> <keep-revs> [--debug]` | 1082-1332 | **Experimental GC of `$bsdir/sources`.** Scans every `*.rev`, `*.mrev`, `*.rev.del`, `*.mrev.del` in `$projectsdir/<prj>.pkg/`; keeps the last `<keep-revs>` lines per file and any rev newer than `<days>`; collects `<md5>-MD5SUMS` tree files from `$treesdir` and `$srcrepdir`; resolves `_link` files (`$BSXML::link`) so linked-to revisions are kept; unlinks unreferenced `$srcrepdir/<pkg>/<md5>-<file>` (skipping `deltastore`), then rewrites the `.rev` files dropping deleted hashes. Reports bytes freed. |
| `--create-patchinfo-from-updateinfo` | `<updateinfo.xml> <pooldir>` | 1047-1081 | Reads `$BSXML::updateinfoitem`, writes `./<id>-<version>/_patchinfo` + copies the referenced rpms (and their SOURCERPMs, via `rpm -qp` + `find`) from `<pooldir>` and an `updateinfo.xml`. Operates in CWD; no `$bsdir` writes. |

#### Debug / inspection
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--dump-cache` | `<project> <repo> <arch>` or `<file.solv|file.cache>` | 787-798, impl 382-399 | Dumps `$reporoot/<prp>/<arch>/:full.solv` (preferred) or `:full.cache` (Storable). |
| `--dump-relsync` / `--dump` | `<file> [key...]` | 799-803 | Generic Storable dumper (`BSUtil::retrieve`), optionally descending into keys. Intended for `:relsync`, `:relsync.max`, `:relsync.sent`, `:repoinfo`, `:depends`, `dispatchprios`, etc. |
| `--set-relsync` | `<file> <key> <value>` | 804-814 | Loads Storable, sets `$s->{key}=value`, `BSUtil::store`s it back (no atomic temp name). |
| `--check-meta-xml` | `<project> [<package>]` | 705-713, impl 522-540 | Parses `$projectsdir/<prj>.xml` (`$BSXML::proj`) or `$projectsdir/<prj>.pkg/<pkg>.xml` (`$BSXML::pack`). |
| `--check-product-xml` | `<file>` | 718-721 | `BSProductXML::readproductxml($file,0,1)` (expands xi:include). |
| `--check-product-group-xml` | `<file>` | 714-717 | `$BSProductXML::group`. |
| `--check-pattern-xml` | `<file>` | 722-725 | `$BSXML::pattern`. |
| `--check-request-xml` | `<file>` | 726-729 | `$BSXML::request`. |
| `--check-kiwi-xml` | `<file>` or `<project> <package>` | 758-767 | `$BSKiwiXML::kiwidesc`; prj/pkg form resolves latest rev's `*.kiwi` in `$srcrepdir/<pkg>/<md5>-<file>`. |
| `--check-constraints` | `<file>` or `<project> <package>` | 768-777 | `$BSXML::constraints`. |
| `--parse-build-desc` | `<file> [<arch> [<buildconfig>]]` | 782-786 | `Build::read_config` + `Build::parse`, Dumper output. |
| `--parse-hdrmd5` | `<rpm|deb>` | 787-791 | `Build::queryhdrmd5`. (undocumented in help) |
| `--show-scheduler-architectures` | – | 778-783 | Reads `$bsdir/configuration.xml` (`$BSXML::configuration`) and prints `schedulers/arch`. Used by `dist/obsscheduler:28`. |
| `--show-delta-file` | `<file.obscpio>` | 991-1002 | `BSSolv::isobscpio` check; `BSSolv::obscpioinstr(file[, <dir>/deltastore])` prints delta instructions. |
| `--cat-delta-file` | `<file.obscpio>` | 1003-1012 | `BSSolv::obscpioopen(file, <dir>/deltastore, FH, "$srcrepdir/:upload")` and streams the reconstructed content to stdout. (undocumented) |
| `--show-delta-store` | `<file-or-dir>` | 1013-1020 | `BSSolv::obscpiostorestats(<dir>/deltastore)`. |

#### Backend configuration
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--update-project-signing-key` | `<project> <gpg_pub.asc> <signkey-hex> [<ssl_cert.pem>]` | 1358-1393 | `BSRevision::addrev_meta_replace({'comment'=>'manually update key'}, $projid, undef, [pub → $projectsdir/<prj>.pkg/_pubkey], [signkey → _signkey], [cert|undef → _sslcert])` — i.e. commits a new `_project` meta revision via the source-rep layer (files land in `$bsdir/sources/_project/<md5>-_signkey` etc.). Then `chmod 0600 _signkey` and `chown $bsuser` on `_signkey`/`_pubkey`. Help text (`:260-281`) documents the pem2openpgp/gpg/`sign --encrypt`/hex-dump recipe for producing `_signkey`. |

#### Dispatcher maintenance (target = `bs_dispatch`)
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--list-badhosts` | – | 1341-1351 | Dumps Storable `$rundir/dispatch.badhosts`. |
| `--drop-badhosts` | – | 1338-1340 | Touches `$rundir/bs_dispatch.dropbadhosts` (dispatcher polls `$rundir/$runname.dropbadhosts`). |

Note: the user-requested "changes dispatch priorities" is *not* a bs_admin feature; the
dispatcher's priorities live in `$bsdir/jobs/dispatchprios` (Storable, written by the
API through the repserver, checked by `bs_check_consistency:313`) and can only be
*viewed* via `--dump <file>`.

#### Service containers
| Option | Args | Lines | Effect |
|---|---|---|---|
| `--sc` / `--service-container` | `<podman-cmd> [-d|--debug] [args]`; special `launch [cmd]` and `help` | 1394-1417; help text 87-105 | Finds `podman` via `$SHELL -c "type -P podman"`; runs `su -s /bin/bash -c "<podman> --root $sc_root <cmd> args" - $sc_serviceuser`. `launch` = `podman --root $sc_root run -it --rm $sc_container [cmd|/bin/bash]`. Exit status propagated. |

---

## 2. Other admin utilities

### 2.1 `src/backend/bs_check_consistency` (633 lines) — filesystem validator

Purpose (header `:21-24`): "Combining instable storage and fragile filesystem
implementations is a bad idea. Try to find out what junk ended up in the tree today."
Pure local FS walker; no RPC. Uses `BSConfig` directly (not `BSConfiguration`).

Options (`Getopt::Long`, `:84-98`; usage `:54-82`):

| Option | Effect |
|---|---|
| `--check-jobs` | `$bsdir/jobs`: verifies Storable `jobs/dispatchprios` and `jobs/load` (warns if missing, `:308-317`); per `jobs/<arch>/`: `*:status` must have a job file and parse as `$BSXML::jobstatus` (warns if code `dispatching`); `*:dir` must be a dir whose `meta` lines match `^[0-9a-f]{32}  .+`; other files must parse as `$BSXML::buildinfo` (`:318-374`). |
| `--check-db` | `$bsdir/db/{published,source}` (skips `sqlite`; `request` is "obsolete"): expects 2-hex-char shard dirs containing 30-hex-char Storable entries (`:376-426`). |
| `--check-event` | `$bsdir/events/*/*` must parse as `$BSXML::event`; top-level files other than `lastevents`/`lastnotifications` are "strange" (`:428-452`). |
| `--check-worker` | `$bsdir/workers/<state>/<worker>` must parse as `$BSXML::worker` (dir `disable` exempt) (`:454-480`). |
| `--check-build` | Walks `$bsdir/build/<prj>/<repo>/<arch>/`: validates Storables `:depends`, `:full.cache`, `:relsync`, `:relsync.max`, `:relsync.sent`, `:repoinfo`; `:packstatus` (Storable or `$BSXML::packstatuslist`); `:jobhistory` via `BSFileDB::fdb_getall($BSXML::jobhistlay)`; `:repostate` (`$BSXML::repositorystate`); known-but-unchecked list (`:full.solv`, `:bininfo*`, `:lastfailures`, `:lastcheck`, `:full.metacache*`, `:full.xcache`, `:full.useforbuild`, `.bininfo`, `.checksums`, `.preinstallimages`, `:preinstallimages`, `:packstatus.finished`, `.meta.success`, `_deltas`), dirs `:logfiles.fail`, `:logfiles.success`, `:reproduciblecheck.fail`, `:meta` (with `--do-check-meta`), `:full` and package dirs via `check_package_dir` (`:482-631`). The project-level file `_repoid` and repo-level `:repoinfo`/`.finishedlock` are accepted. |
| `--check-all` | All of the above. |
| `--do-check-project-build PROJECT` | Restrict `--check-build` to one project. |
| `--do-check-package-source PRJ PKG` | Parses `$projectsdir/PRJ.pkg/PKG.xml` and the `.rev/.mrev/.rev.del/.mrev.del` files via `BSFileDB::fdb_getall` with layout `rev vrev srcmd5 version time user comment requestid` (`:289-304`). |
| `--do-check-meta` | Also validate `.meta` / `:meta/*` line format. |
| `--do-check-signatures` | `rpm --checksig --nosignature` on every `.rpm` (`:154-163`). |
| `--fix` | `warnORremove`: unlink files that can be regenerated (scheduler cold start needed). |
| `--forced-fix` | `warnORbreak`: also unlink data that needs manual repair (`:122-141`). |

`check_package_dir` (`:179-282`) is the canonical whitelist of what may appear inside a
package result dir / `:full` / `:repo`: `:bininfo` (checks every referenced rpm exists),
`*.meta`, `history`, `logfile`, `*.desktopfiles`, `reason` (`$BSXML::buildreason`),
`status` (`$BSXML::buildstatus`), `*.rpm` (magic `\xed\xab\xee\xdb`), `updateinfo.xml`,
`.updateinfo`/`.updateinfodata` (Storable), `.waiting_for_*`, `*-Media*` dirs, debian
files, `:lastcache`, `:full.useforbuild`, `:full.metacache`, `.bininfo`, `.checksums`,
`.channelinfo`, `_statistics`, `.stats`, `_buildenv`, `_channel`, `pesign-*`,
`*appdata.xml`, `*.applications`, `.meta.success*`, `.nosourceaccess`, `rpmlint.log`,
`preinstallimage.info`, `.preinstallimage*`, `*-Build*`.

### 2.2 `src/backend/bs_cleanup` (759 lines) — source GC (Ericsson, 2015)

Standalone (uses `XML::Simple`, `File::Find`; **hard-codes** `/srv/obs/sources`,
`/srv/obs/trees`, `/srv/obs/projects` at `:33-35` — ignores BSConfig). Marked
"EXPERIMENTAL". Modes (`GetOptions` `:709-721`):

| Mode | Effect |
|---|---|
| `--remove-old-sources <days> <revs>` | `remove_old_srcs` (`:294-651`): same algorithm as bs_admin's `--remove-old-sources` (only `.rev`/`.rev.del`, not `.mrev`), but with project/package bookkeeping, `--include-packages`/`--exclude-packages` filters, and link resolution through `$OBS_SRC_DIR/<pkg>/<md5>-_link`. Rewrites `.rev` files after deletion. |
| `--service-cleanup` | `search_srcs` (`:258-284`): for each `<md5>-_service*` file in `sources/<pkg>/` checks whether any `trees/<prj>/<pkg>/<rev>-MD5SUMS` references it via a `/LSERVICE` or `/LINK` marker and whether that rev is still in `projects/<prj>.pkg/<pkg>.rev`; unreferenced files are reported/removed/moved. |
| `--full-cleanup <days> <revs>` | Both. |
| `--remove-deleted-project <project>` | Must be in `projects/_deleted/`; deletes all its revs' files, the `_deleted/<prj>.pkg` dir and `trees/<prj>`. |
| Options | `--search|-s` (default, report only), `--remove|-rm`, `--move|-mv <dst>`, `--include-packages <pkg...|file>`, `--exclude-packages <pkg...|file>`, `--debug|-d`, `--help`. |

### 2.3 `src/backend/bs_dbtool` (341 lines) — published/source DB inspector & converter

Drops privs to `$bsuser` (`:59`). Usage: `bs_dbtool [--old|--new] <table> <cmd> [args]`
(`:61`). `--old` forces the legacy BSDB (Storable-shard) backends, `--new` forces
sqlite (`:46-53`). Tables → storage (`:66-82`):

- `linkinfo`, `scmsync` → `$bsdir/db/source` (sqlite via `BSSrcServer::SQLite::opendb`
  when `$BSConfig::source_db_sqlite`, else BSDB with `allkeyspath=project`; `scmsync`
  requires sqlite).
- `binary`, `pattern`, `repoinfo` → `$bsdir/db/published` (sqlite or BSDB).
- `result`, `oldresult`, `jobs`, `scmsync.redis` → **Redis** via an inline
  `BSSrcServer::Redis` shim (`:17-42`) parsing `$BSConfig::redisserver`
  (`redis[s]://[user:pass@]host[:port]`), `KEYS table.*` / `HGETALL`.

Commands: `keys [path value]`, `values <path>`, `fetch <key>`, `repoinfo <prp>`,
`repoorigins <prp>` (binaryorigins), `projectkeys [project]`, `locallinks`, `linkers`,
`linkpackages` (sqlite only), `convert` (BSDB → sqlite for `linkinfo`, `pattern`,
`binary`(from `repoinfo`); `scmsync` → redis jobs via `BSSrcServer::ScmsyncDB::addredisjob`),
`deleteobsolete` (for `result`: drop redis result keys for prp/arch no longer in the
project meta; for `scmsync.redis`: re-queue keys absent from sqlite) (`:84-339`).

### 2.4 `src/backend/bs_serverstatus` (191 lines) — live server/ajax status viewer

Wrapper: `/usr/sbin/obs_serverstatus` (`dist/obs_serverstatus:1-2`). Usage:
`bs_serverstatus [--top] [--nolastevents] [-w] <statusfile|ajax-socket>` (`:63`).

- If the argument is a **unix socket** (`-S`), it does a `BSHandoff::rpc({uri=>'/ajaxstatus',
  handoffpath=>$socket}, $BSXML::ajaxstatus)` and prints a slot table of jobs
  (`age ev fd request [requestid]`), watchers, outstanding RPCs and serialize locks;
  `--nolastevents` collapses `GET /lastevents?` / `GET /lasteventsproxy?` pollers into
  counts (`:65-158`). The sockets are `$rundir/bs_srcserver.ajax` and
  `$rundir/bs_repserver.ajax`.
- Otherwise it treats the argument as the **BSServer status file** (fixed 256-byte
  records `N N C C n Z244` = time, pid, group, extra, state, data) and prints
  `state(F/R/K)+group age pid request` per slot (`:160-191`). `--top` redraws every
  second with ANSI clears. State files are `$rundir/<daemon>` (`BSStdServer.pm`; see §5).

### 2.5 `src/backend/bs_archivereq`
Listed in spec `%files` (`dist/obs-server.spec:854`); drops privs (`:22`); works on
`$bsdir/requests`, `$bsdir/requests.old`, `$bsdir/db/request` (from the path grep in §5).
Not further analysed here.

---

## 3. `bs_setup` and `BSSetup::*`

Files: `src/backend/bs_setup` (267 lines), `src/backend/BSSetup/Base.pm` (178 lines),
`src/backend/BSSetup/SUSE.pm` (25 lines). Wrapper `/usr/sbin/obs-setup`
(`dist/obs-setup:1-2`). Only grep hits for `BSSetup` are those three files.

- `bs_setup` detects the distro from `/etc/os-release` `$ID` (`:71-73`); only
  `suse|sles` is implemented (`BSSetup::SUSE`), everything else `die`s (`:22-31`).
- Global option `--debug` (sets `$::LOGLEVEL=1`). Commands (`:35-40`):
  - `service-containers` / `sc` → `setup_service_containers` (`:91-113`):
    1. `install_pkg('podman')` (`zypper -n install`, `SUSE.pm:6-22`);
    2. `prepare_mounts_conf` (`Base.pm:38-73`): comments out `/etc/SUSEConnect` and
       `/etc/zypp/credentials.d/SCCcredentials` in `/etc/containers/mounts.conf` if those
       files don't exist;
    3. `prepare_storage_conf` (`Base.pm:75-102`): adds `"$bsdir/service/containers"` to
       `additionalimagestores` in `/etc/containers/storage.conf`;
    4. `configure_bsserviceuser($BSConfig::bsserviceuser)` (`Base.pm:104-124`):
       `loginctl enable-linger <uid>`, `usermod -v 200000-265535 -w 200000-265535 <user>`
       (subuid/subgid ranges for rootless podman);
    5. `install_pkg('obs-source-service-podman-image')`;
    6. `generate_bsconfig` (`Base.pm:137-160`): **appends** to
       `/usr/lib/obs/server/bsconfig.<hostname>` (the per-host override file BSConfig.pm
       `require`s, `BSConfig.pm.template:254-260`):
       `$api_url='http://api-opensuse.suse.de:80'` (hard-coded SUSE default),
       `$containers_root="$bsdir/service/containers"`,
       `$container_image='localhost/obs-source-service-podman:latest'`,
       `$service_wrapper = { '*' => '<libdir>/call-service-in-container' }`;
    7. `restart_bs_service` → `<libdir>/bs_service --restart` (`Base.pm:162-176`).
  - `gitea` → `setup_gitea` (`:115-256`): creates MySQL DB `gitea` (root/opensuse),
    writes `/etc/gitea/conf/app.ini` (ROOT_URL `https://<fqdn>/gitea/`, HTTP 3000, mysql
    127.0.0.1:3306, LFS on, secrets in `/etc/gitea/{INTERNAL_TOKEN,JWT_SECRET,LFS_JWT_SECRET,SECRET_KEY}`
    generated with `gitea generate secret`), runs `gitea migrate`, creates admin user
    `obsadmin`/`opensuse`, `systemctl enable --now gitea.service`.
  - `obs-gitea-bridge` → `systemctl enable --now obs-gitea-bridge` (`:258-261`).

---

## 4. `dist/` — units, sysconfig, init scripts, helpers

### 4.1 Inventory and install targets (`dist/Makefile`)

`dist/Makefile:4-9` enumerates what is installed:

- `OBS_BIN_SCRIPTS` (→ `/usr/bin`): `obs_productconvert`.
- `OBS_SBIN_SCRIPTS` (→ `/usr/sbin`): `obs_admin obs_serverstatus obsscheduler obsworker obsstoragesetup obs-setup`.
- `SYSTEMD_TARGET_FILES`: `obs-api-support`.
- `SYSTEMD_SERVICE_FILES` (→ `/usr/lib/systemd/system/`): `obs-clockwork`, 9×
  `obs-delayedjob-queue-*`, `obs-sphinx`, `obsdeltastore`, `obsdispatcher`, `obsdodup`,
  `obswarden`, `obssrcserver`, `obsrepserver`, `obspublisher`, `obssigner`, `obsservice`,
  `obsservicedispatch`, `obssourcepublish`, `obsgetbinariesproxy`, `obsscheduler`,
  `obsworker`, `obsstoragesetup`, `obsapisetup`, `obsnotifyforward`, `obsredis`;
  plus template `obs-delayedjob-queue-quick@`. (`obsscheduler@.service` and
  `obsworker@.service` exist in `dist/systemd/` but are **not** in the install list and
  not in the spec `%files`.)
- `install_apache` (`:21-29`): installs `$(APACHE_VHOST_CONF)`=`obs-apache24.conf` as
  `/etc/apache2/vhosts.d/obs.conf`, substituting `@@APACHE_LOGDIR@@`,
  `@@OBS_BACKEND_DATA_DIR@@`, `@@OBS_API_PREFIX@@`, `@@OBS_DOCUMENT_ROOT@@`.
- `install_systemd_services` (`:31-40`): substitutes `@@OBS_BUNDLE_BIN@@`,
  `@@OBS_RUBY_ABI_VERSION@@`, `@@APACHE_USER@@`, `@@APACHE_GROUP@@`, `@@OBS_API_PREFIX@@`
  in all units.
- `install_logrotate` → `/etc/logrotate.d/{obs-api,obs-server,obs-source_service}`;
  `install_slp` → `/etc/slp.reg.d/obs.{source,repo}_server.reg`;
  `install_crontabs` → `/etc/cron.d/cleanup_scm_cache`;
  `install_obsapisetup` → `/usr/lib/obs/server/{setup-appliance.sh,functions.setup-appliance.sh}`;
  `install_overview` → `/srv/www/obs/overview/overview.html.TEMPLATE`;
  `install_tests_appliance` → `/usr/lib/obs/tests/appliance/`.
- Substitution values come from `Makefile.include:1-18`: `OBS_BACKEND_PREFIX=/usr/lib/obs/server`,
  `OBS_BACKEND_DATA_DIR=/srv/obs`, `OBS_DOCUMENT_ROOT=/srv/www/obs`,
  `OBS_API_PREFIX=/srv/www/obs/api`, `APACHE_USER=wwwrun`, `APACHE_GROUP=www`,
  `APACHE_CONFDIR=/etc/apache2`, `APACHE_LOGDIR=/var/log/apache2`,
  `OBS_RUBY_BIN=/usr/bin/ruby.ruby4.0`, `OBS_BUNDLE_BIN=/usr/bin/bundle.ruby4.0`,
  `OBS_RUBY_ABI_VERSION=4.0.0`.

Other files in `dist/` not installed by the Makefile: `aws_credentials.example`,
`install_rh.sh`, `obs_import_srcdebtree`, `obs_import_srcrpmtree`, `obs_mirror_project`,
`obs_rebuild_db`, `obs-apache2.conf` (apache 2.2 variant), `obs-lighttpd.conf`,
`nginx-obs-api.conf`, `internal_redirect.include`, `munin/`, `openstack/`,
`openQA_mail_notification.rb`, `resolve_swagger_yaml.rb`, `run_shellcheck.sh`,
`setup_source_service_docker.sh`, `gem_build_cleanup.sh`, `obs-rpmlintrc`,
`obs-bundled-gems.spec`, `obs-api-testsuite-rspec.spec`, `README.SETUP`, `README.UPDATERS`,
`t/` (appliance test suite). `obs_project_update` is installed by the spec into
`obs-utils` (`obs-server.spec:1077`).

### 4.2 Backend daemon units (`dist/systemd/`)

All backend daemon units share the same shape: `EnvironmentFile=/etc/sysconfig/obs-server`,
`ExecStart=/usr/lib/obs/server/bs_<x> --logfile <x>.log`, `ExecStop=... --stop`,
`ExecReload=... --restart`, `WantedBy=multi-user.target`, no `User=` (the daemon starts
as root and drops to `$BSConfig::bsuser`/`bsgroup` itself — `BSStdServer.pm:515`,
`BSStdRunner.pm:212`, and explicit `drop_privs_to` in `bs_sched:86`, `bs_dispatch:152`,
`bs_warden:107`, `bs_dodup:1075`, `bs_redis:249`, `bs_notifyforward:192`).
`--logfile <name>` is resolved by `BSUtil::openlog($logfile, $BSConfig::logdir, bsuser, bsgroup)`
(`BSStdServer.pm:412-415`, `BSStdRunner.pm:194-197`), i.e. into `$bsdir/log/<name>`;
each daemon additionally gets `$logdir/<name>.crit.log` (`BSStdServer.pm:483`,
`BSStdRunner.pm:247`). `--stop`/`--exit`/`--restart` are handled in
`BSStdServer.pm:432-442,550` by touching `$rundir/<runname>.exit` / `.restart` (see §5).

**Important**: no Perl daemon reads `OBS_BASE_DIR`, `OBS_RUN_DIR`, `OBS_LOG_DIR`,
`OBS_SRC_SERVER` etc. from the environment (`grep -rn "ENV{'OBS_" src/backend` only
finds `bs_service` *exporting* `OBS_SERVICE_*` to source services, `bs_service:160-164`).
The `EnvironmentFile=` lines are therefore only effective for the units that interpolate
`$VAR` in `ExecStart` (`obsworker@.service`) and for the shell wrappers. Daemon locations
come from `BSConfig.pm` (`$bsdir`, `$rundir`, `$logdir`, `$srcserver`, `$reposerver`).

| Unit | ExecStart | Ordering | Notes |
|---|---|---|---|
| `obssrcserver.service` (`systemd/obssrcserver.service`) | `bs_srcserver --logfile src_server.log` | After `network-online.target obsstoragesetup.service obsservice.service` | `OOMPolicy=continue`. Source server, port 5352 (`BSConfig.pm.template:52`). |
| `obsrepserver.service` | `bs_repserver --logfile rep_server.log` | After `network-online.target obsstoragesetup.service` | Port 5252. Also serves `/getworkercode` to workers (`bs_repserver:3820,4742`). |
| `obsscheduler.service` | `/usr/sbin/obsscheduler start` / `stop` | `Requires=obsrepserver.service`; After `network-online obssrcserver obsrepserver obsapisetup` | `Type=oneshot`, `RemainAfterExit`, `KillMode=none`, `TimeoutStopSec=infinity` — the shell script forks one `bs_sched <arch>` per architecture (see §4.5). |
| `obsscheduler@.service` (not installed) | `bs_sched %i`; ExecStop `bs_admin --shutdown-scheduler %i` | After `network-online obsstoragesetup` | Per-arch alternative to the wrapper. |
| `obsdispatcher.service` | `bs_dispatch --logfile dispatcher.log` | `Requires=obsrepserver`; After `network-online obssrcserver obsscheduler obsrepserver` | `OOMPolicy=continue`. |
| `obspublisher.service` | `bs_publish --logfile publisher.log` | After `network-online.target` | `OOMPolicy=continue`. |
| `obssigner.service` | `bs_signer --logfile signer.log` | After `network-online obsapisetup obsstoragesetup signd.service` | needs the external `signd` (obs-signd package). |
| `obswarden.service` | `bs_warden --logfile warden.log` | After `network-online.target` | monitors workers (`$bsdir/workers/building`). |
| `obsdodup.service` | `bs_dodup --logfile dodup.log` | After `network-online`, `obsapisetup` | download-on-demand metadata updater; `$bsdir/dods`. |
| `obsdeltastore.service` | `bs_deltastore --logfile deltastore.log` | After `network-online obsstoragesetup` | events `$bsdir/events/deltastore`; writes `$bsdir/sources/<pkg>/deltastore`, uses `$bsdir/upload`. |
| `obsservicedispatch.service` | `bs_servicedispatch --logfile servicedispatch.log` | After `network-online obsstoragesetup obsservice` | events `$bsdir/events/servicedispatch`. |
| `obsservice.service` | `bs_service --logfile src_service.log` | After `network-online obsstoragesetup` | runs as `obsservicerun` (`BSConfig::bsserviceuser`); port 5152 (`BSConfig.pm.template:54`); log owned `obsservicerun:obsrun` (`obs-server.spec:625-628`). |
| `obssourcepublish.service` | `bs_sourcepublish --logfile sourcepublish.log` | After `network-online obsstoragesetup` | events `$bsdir/events/sourcepublish`. |
| `obsgetbinariesproxy.service` | `bs_getbinariesproxy --logfile getbinariesproxy.log` | After `network.target obsstoragesetup` | `LimitNOFILE=16384`; cache in `$bsdir/getbinariesproxycache`. |
| `obsnotifyforward.service` | `bs_notifyforward --logfile notifyforward.log` | After `network-online.target` | events `$bsdir/events/notifyforward`. |
| `obsredis.service` | `bs_redis --logfile redis.log` | After `network-online.target` | "redis forwarder"; events `$bsdir/events/redis`. |
| `obsworker.service` | `/usr/sbin/obsworker start` / `stop` | After `network-online obsstoragesetup obssrcserver obsrepserver xendomains.service` | oneshot + RemainAfterExit; script runs workers inside `screen` (§4.6). |
| `obsworker@.service` (not installed) | `ExecStartPre` mkdir/chmod `/var/cache/build/{root,state}_%i`, `systemctl set-environment HOSTNAME=$(hostname)`; `ExecStart=bs_worker --hardstatus --root /var/cache/build/root_%i --statedir /var/cache/build/state_%i --id ${HOSTNAME}:%i --reposerver http://${OBS_REPO_SERVERS} --jobs $OBS_WORKER_JOBS` | After `network-online.target` | The only unit where `EnvironmentFile` variables are actually consumed (`OBS_REPO_SERVERS`, `OBS_WORKER_JOBS`). `ExecReload=kill -HUP`. |
| `obsstoragesetup.service` | `/usr/sbin/obsstoragesetup start` / `stop` | After `network-online xendomains haveged`; **Before** `mysql.service sshd.service obsapisetup.service` | oneshot + RemainAfterExit (§4.4). |
| `obsapisetup.service` | `/usr/lib/obs/server/setup-appliance.sh --non-interactive --setup-only --with-gitea` | Before `apache2.service`; After `obsstoragesetup network-online mariadb.service` | oneshot + RemainAfterExit (§4.8). |

### 4.3 API-side units

- `obs-api-support.target` (`systemd/obs-api-support.target`): `Wants=` clockwork, the
  delayed-job queues (`consistency_check`, `default`, `issuetracking`, `mailers`,
  `project_log_rotate`, `quick@0..2`, `releasetracking`, `staging`, `scm`) and
  `obs-sphinx`; `After=network.target`; `AllowIsolate=yes`.
- `obs-clockwork.service`: `User/Group=@@APACHE_USER@@/@@APACHE_GROUP@@`
  (`wwwrun:www`), `WorkingDirectory=@@OBS_API_PREFIX@@`, `RAILS_ENV=production`,
  `bin/clockworkd --log-dir=log -l -c config/clock.rb start`, `Type=forking`,
  `PIDFile=tmp/clockworkd.clock.pid`, `BindsTo=obs-api-support.target`, After
  `mariadb.service obsapisetup.service`.
- `obs-delayedjob-queue-<q>.service` (×9 + `quick@`): same user/dir/env;
  `@@OBS_BUNDLE_BIN@@ exec script/delayed_job.api.rb --queue=<q> start -i <id>` with
  fixed ids `releasetracking=1000`, `issuetracking=1010`, `mailers=1020`,
  `default=1030`, `project_log_rotate=1040`, `consistency_check=1050`, `staging=1060`,
  `scm=1070`, `quick@%i`; `PIDFile=tmp/pids/delayed_job.<id>.pid`; `Restart=on-failure`.
- `obs-sphinx.service`: `bin/rails sphinx:start` / `ts:stop`, `Conflicts=searchd.service`,
  `PIDFile=log/production.sphinx.pid`.

### 4.4 `/etc/sysconfig/obs-server` — every variable (`dist/sysconfig.obs-server`)

Installed as a fillup template on SUSE (`%{_fillupdir}/sysconfig.obs-server`,
`obs-server.spec:536-537,745,1065`) or directly as `/etc/sysconfig/obs-server` on
Fedora/RHEL (`:533-534,1063`). Header note (`:1-3`): all options can also be declared
per worker in `/etc/buildhost.config` (which `obsstoragesetup` regenerates — §4.5).

| Variable (default) | Lines | Comment / meaning | Consumed by |
|---|---|---|---|
| `OBS_BACKENDCODE_DIR=""` | 5-13 | Backend code dir; empty → `/usr/lib/obs/server/`. | `obsscheduler:20-24`. |
| `OBS_RUN_DIR=""` | 15-23 | Communication dir; empty → `/srv/obs/run`. | `obsscheduler:12-14` (default `/srv/obs/run`); `obsworker:46-48` (default **`/var/run/obs`**). |
| `OBS_LOG_DIR=""` | 25-33 | Log dir; empty → `/srv/obs/log`. | `obsscheduler:16-18`, `obsworker:49-51`, spec `%pre` (`:618-628`). |
| `OBS_BASE_DIR=""` | 35-43 | Base dir; empty → `/srv/obs`. | `obsstoragesetup:25-29`, `setup-appliance.sh:33-35`. |
| `OBS_API_AUTOSETUP="no"` | 45-53 | Auto-setup api/webui; "will replace config files"; appliance only. | `setup-appliance.sh:67-70` (exits unless `yes`; `--force` sets it). |
| `OBS_SRC_SERVER=""` | 58-66 | Source server host; empty → `localhost:5352`. | **Nothing in the tree reads it** (worker uses `--srcserver` only from reposerver code). |
| `OBS_REPO_SERVERS=""` | 68-76 | Repo server host(s); empty → `localhost:5252`. | `obsworker:52-54,84-92` (→ `--reposerver http://host` per entry); `obsworker@.service:13`. |
| `OBS_SCHEDULER_WRAPPER=""` | 78-87 | Wrapper for preloading libs (e.g. `jemalloc.sh`). | `obsscheduler:57`. |
| `OBS_WORKER_INSTANCES="0"` | 89-97 | Number of build instances; 0 → number of CPUs. | `obsworker:352-357`, `obsstoragesetup:175-193`. |
| `OBS_WORKER_INSTANCE_NAMES=""` | 99-111 | z/VM guest names (need minidisks 0150 root / 0250 swap, secondary console). | `obsworker:361-364,432-433,470`. |
| `OBS_WORKER_DIRECTORY=""` | 113-120 | Base dir for per-worker subdirs. Default `/var/cache/obs/worker` (`obsworker:41-43`, `obsstoragesetup:31-33`). | `obsworker`, `obsstoragesetup`. |
| `OBS_WORKER_PORTBASE="0"` | 122-130 | Base port for worker instances; 0 → OS assigns. | `obsworker:421-425` (`--port $((PORTBASE+I))`). |
| `OBS_WORKER_JOBS="1"` | 132-140 | Parallel compile jobs per worker (`make -j`). | `obsworker:59-61` (`--jobs`); `obsworker@.service`. |
| `OBS_WORKER_TEST_MODE=""` | 142-148 | "yes": results ignored, no job blocking. | `obsworker:56-58` (`--test`). |
| `OBS_WORKER_HOSTLABELS=""` | 150-159 | Labels to pin packages to hosts. | `obsworker:414-416` (`--hostlabel` each). |
| `OBS_WORKER_SECURITY_LEVEL=""` | 161-170 | Extends hostlabels with `OBS_WORKER_SECURITY_LEVEL_<x>`. | `obsworker:80-82`. |
| `OBS_USE_SLP="yes"` | 172-179 | Register/discover via SLP. | `obsworker:386-394` (`slptool findsrvs service:obs.repo_server`). |
| `OBS_CACHE_DIR=""` | 181-190 | Common package cache dir; "content will be removed there!". | `obsworker:100-104` (`--cachedir`). |
| `OBS_CACHE_SIZE=""` | 192-200 | Cache size MB; "50% of partition". | `obsworker:100-104` (`--cachesize`); both must be set. |
| `OBS_WORKER_NICE_LEVEL=18` | 202-214 | nice level (18 so testsuites can still go to 19). | `obsworker:65-69,484`. |
| `OBS_VM_TYPE="auto"` | 216-223 | `auto|xen|kvm|lxc|zvm|emulator:$arch[:$script]|qemu:$arch|none|openstack`. | `obsworker:140-159`. |
| `OBS_VM_KERNEL="none"` | 225-233 | Kernel for kvm (z/VM: `/boot/image`). | `obsworker:106-108` (`--vm-kernel`, not for openstack). |
| `OBS_VM_INITRD="none"` | 235-251 | initrd for kvm (mkinitrd recipe with virtio+binfmt_misc); z/VM: created on demand. | `obsworker:110-112` (`--vm-initrd`), `create_initrd` `:306-339`. |
| `OBS_VM_DISK_AUTOSETUP_ROOT_FILESIZE="4096"` | 253-260 | Root disk MB for XEN/KVM/TMPFS autosetup. | `obsworker:183-184` (`--vmdisk-rootsize`). |
| `OBS_VM_DISK_AUTOSETUP_SWAP_FILESIZE="1024"` | 262-269 | Swap MB. | `obsworker:185-187` (`--vmdisk-swapsize`). |
| `OBS_VM_DISK_AUTOSETUP_FILESYSTEM="ext3"` | 271-279 | `ext2|ext3|ext4|reiserfs|btrfs|xfs`; buildflag `vmfstype` overrides per job. | `obsworker:188-190,163-165` (`--vmdisk-filesystem`). |
| `OBS_VM_DISK_AUTOSETUP_MOUNT_OPTIONS=""` | 281-288 | mount options. | `obsworker:191-193,166-168` (`--vmdisk-mount-options`). |
| `OBS_WORKER_USE_MKFS_COPYIN=""` | 290-298 | mkfs copy-in support. | `obsworker:76-78` (`--vm-use-mkfs-copyin`). |
| `OBS_VM_USE_TMPFS=""` | 300-308 | Build in memory ("requires much memory"). | `obsworker:450-452` (`--tmpfs`). |
| `OBS_VM_CUSTOM_OPTION=""` | 310-318 | Extra VM handler options. | `obsworker:114-116` (`--vm-custom-option`). |
| `OBS_VM_ENABLE_NET=""` | 320-328 | Allow VM internet (kvm only; unsafe/non-reproducible). | `obsworker:175-179` (`--vm-network`); `obsstoragesetup:417` propagates. |
| `OBS_INSTANCE_MEMORY=""` | 330-337 | Memory per VM, default 512. | `obsworker:134,455,476-478` (`--vm-memory`); autodetected by `obsstoragesetup:391-396`. |
| `OBS_STORAGE_AUTOSETUP=""` | 339-348 | "may destroy data"; mass-deployed workers. | `obsstoragesetup:16-19` (exits unless `yes`). |
| `OBS_SETUP_WORKER_PARTITIONS="use_obs_vg"` | 350-359 | `take_all` (erase all LVM!) / `use_obs_vg` (re-setup VG "OBS") / `none`. | `obsstoragesetup:72-129,174`. |
| `OBS_WORKER_CACHE_SIZE=""` | 361-368 | MB for LVM cache LV (default 25 GB, `obsstoragesetup:210-213`). | `obsstoragesetup`. |
| `OBS_WORKER_ROOT_SIZE=""` | 370-377 | MB per worker root LV (else computed). | `obsstoragesetup:221-245`. |
| `OBS_WORKER_SWAP_SIZE=""` | 379-386 | MB per worker swap LV (default 512). | `obsstoragesetup:216-219`. |
| `OBS_WORKER_BINARIES_PROXY=""` | 388-395 | URL of `bs_getbinariesproxy`. | `obsworker:122-124` (`--getbinariesproxy`). |
| `OBS_ROOT_SSHD_KEY_URL=""` | 397-405 | URL of root ssh pubkey (PXE workers). | `obsstoragesetup:55-64` (curl → `/root/.ssh/authorized_keys`, enable sshd). |
| `OBS_WORKER_SCRIPT_URL=""` | 407-415 | URL of a hook script run at boot. | `obsstoragesetup:443-451`. |
| `OBS_WORKER_CLEANUP_CHROOT=""` | 417-424 | Empty chroot/lxc after build. | `obsworker:70-72` (`--cleanup-chroot`). |
| `OBS_WORKER_WIPE_AFTER_BUILD=""` | 426-433 | Wipe build env after build. | `obsworker:73-75` (`--wipeafterbuild`). |
| `OBS_WORKER_CONTROL_INSTANCE=""` | 435-442 | OpenStack control instance name/id. | `obsworker:200-204,461` (`--vm-server`). |
| `OBS_WORKER_OS_FLAVOR=""` | 444-451 | OpenStack flavor. | `obsworker:206-209,462` (`--openstack-flavor`). |
| `OS_AUTH_URL`, `OS_PROJECT_ID`, `OS_PROJECT_NAME`, `OS_USER_DOMAIN_NAME`, `OS_USERNAME`, `OS_PASSWORD`, `OS_REGION_NAME` (all `""`) | 453-467 | OpenStack client env (only `OBS_VM_TYPE=openstack`). | `obsworker:215-279` (exported; `OS_INTERFACE=public`, `OS_IDENTITY_API_VERSION=3` defaults). |
| `OBS_WORKER_PREFIX=""` | 469 | OpenStack worker name prefix (default `worker`). | `obsworker:419,463-466`. |
| `OBS_OPENSTACK_DISK_SIZE`, `OBS_OPENSTACK_SWAP_SIZE`, `OBS_OPENSTACK_MEMORY_SIZE` (`""`) | 471-473 | OpenStack disk/swap/memory. | `obsworker:281-291`. |

Variables honoured by the scripts but **absent from the template**: `OBS_WORKER_THREADS`
(`--threads`, `obsworker:62-64`), `OBS_WORKER_LOCALKIWI_DIRECTORY` (`--localkiwi … --arch local`,
`:118-120`), `OBS_WORKER_BINARIES_PROXY_KIWIPRODUCT` (`:126-128`), `OBS_VM_ENABLE_CONSOLE`
(`:130-132`), `OBS_VM_DISK_CLEAN` (`--vmdisk-clean`, `:169-171,194-196`),
`OBS_WORKER_OWNER` (`--owner`, `:367-369`), `OBS_WORKER_PROTO` (`--proto`, `:427-429`),
`OBS_VM_USE_HUGETLBFS` (`--hugetlbfs`, `:456-458`, written by `obsstoragesetup:409`),
`OBS_SCHEDULER_ARCHITECTURES` (legacy, `obsscheduler:31-40`; superseded by
`configuration.xml` via `bs_admin --show-scheduler-architectures`), `ZVM_VOLUME_SWAP`
(`:311`). The worker script also sources `/etc/sysconfig/proxy` (`http_proxy`,
`HTTPS_PROXY`, `NO_PROXY`; `obsworker:15-22`) and the legacy `/etc/sysconfig/obs-worker`
(`:28-30`).

### 4.5 `dist/obsstoragesetup` (462 lines) — worker/backend storage autosetup

Runs once at boot via `obsstoragesetup.service` (`Before=mysql sshd obsapisetup`).
Sources `/etc/sysconfig/obs-server`, `functions.setup-appliance.sh`, and
`/etc/buildhost.config` (`:8-14`). **No-op unless `OBS_STORAGE_AUTOSETUP=yes`** (`:16-19`).
`stop` does nothing (`:454-457`). `start`:

1. sshd key bootstrap from `OBS_ROOT_SSHD_KEY_URL` (`:55-64`).
2. `mdadm --assemble --scan` + `boot.lvm` if present (`:67-70`).
3. LVM policy (`:72-129`):
   - `take_all`: refuses if `/dev/OBS/server` exists; otherwise `vgremove -ff OBS`,
     `pvremove` every PV, creates one `8e` partition on every unpartitioned disk
     found by `hwinfo --disk`, `pvcreate --metadatasize 499k` all `8e` partitions,
     `vgcreate OBS`.
   - `use_obs_vg`: `vgchange -ay OBS; lvremove -f /dev/OBS/worker_* /dev/OBS/cache`.
4. Backend volume (`:131-166`): if `/dev/OBS/server` exists, mounts it on `$backenddir`
   (`/srv/obs`), and if `$backenddir/run` is owned by a non-root user that is not
   `$BSConfig::bsuser`, `find $backenddir -uid <old> -exec chown bsuser:bsgroup` (uid
   migration). Records `obs_server_size` from `lvdisplay`.
5. `prepare_obssigner` (shared function, §4.8) and `touch $backenddir/MySQL/.run-mysql_upgrade` (`:169-172`).
6. Worker LVs (`:174-320`) when `take_all|use_obs_vg`: `NUM` = `OBS_WORKER_INSTANCES` or
   #CPUs capped so each gets ≥512 MB RAM (minus 1 for Dom0). Reads PVs of VG OBS and
   `PE_SIZE`; defaults cache=25 GiB, swap=512 MiB; root size = (VG − server − cache −
   NUM×swap)/NUM, must be ≥4 GiB; all rounded to PE. If `NUM ≥ #PVs` creates
   `worker_root_N`/`worker_swap_N` round-robin over PVs with root and swap on different
   PVs (the "MAGIC AT WORK" offset trick, `:252-294`); else stripes each LV over all PVs
   (`-i`). Finally `lvcreate -n cache -i … -L ${CACHE}M OBS` + `mkfs -text4 /dev/OBS/cache`.
7. Mounts `/dev/OBS/cache` on `$OBS_WORKER_DIRECTORY` and creates `…/cache` (`:326-331`).
8. **Rewrites `/etc/buildhost.config`** (`:333-428`), preserving the previous file as
   `/etc/buildhost.config.presets` (appended at the end): `OBS_WORKER_DIRECTORY`,
   `OBS_CACHE_DIR=$OBS_WORKER_DIRECTORY/cache`, `OBS_CACHE_SIZE` = half of the cache
   FS size (`df -m`), detects XEN (`/sys/hypervisor/type`) or KVM (`svm|vmx` + `modprobe kvm`);
   for each `/dev/OBS/worker_root*` with matching swap: in VM mode symlinks
   `$OBS_WORKER_DIRECTORY/root_N/{root,swap}` → LVs, in chroot mode `mkfs ext4` + mount +
   `mkswap`/`swapon`; writes `OBS_WORKER_INSTANCES`, `OBS_WORKER_JOBS` =
   `#CPU/(instances−1)` (min 1), `OBS_INSTANCE_MEMORY` = `MemTotal/(2×instances)` MB if
   unset, and tries to reserve 2 MiB hugepages (`nr_hugepages`) for KVM, mounting
   `/dev/hugetlbfs` and writing `OBS_VM_USE_HUGETLBFS="/dev/hugetlbfs"`; propagates
   `OBS_VM_ENABLE_NET`.
9. If the backend code is installed: `mkdir -p $backenddir/{run,log}`, `chown bsuser:bsgroup
   $backenddir $backenddir/log $backenddir/run`, `chmod g+w` (`:430-440`).
10. Optional hook script from `OBS_WORKER_SCRIPT_URL` with `OBS_WORKER_DIRECTORY`/`OBS_WORKER_DISK`
    exported (`:443-451`); ensures `$OBS_WORKER_DIRECTORY/cache` exists (`:452`).

### 4.6 `dist/obsworker` (517 lines) — worker launcher

Entry: `obsworker.service` → `/usr/sbin/obsworker start|stop|run`. Inputs: `/etc/sysconfig/proxy`,
`/etc/sysconfig/obs-server`, legacy `/etc/sysconfig/obs-worker`, `/etc/buildhost.config`
(`:15-39`). Defaults: `OBS_WORKER_DIRECTORY=/var/cache/obs/worker`, `OBS_RUN_DIR=/var/run/obs`,
`OBS_LOG_DIR=/srv/obs/log`, `OBS_REPO_SERVERS=localhost:5252` (`:41-54`).

Flag assembly (`:56-292`):
- `--test` (`OBS_WORKER_TEST_MODE`), `--jobs N`, `--threads N`, nice level, `--cleanup-chroot`,
  `--wipeafterbuild`, `--vm-use-mkfs-copyin`, hostlabels (+security level label).
- `--reposerver <url>` for every entry in `OBS_REPO_SERVERS` (auto-prefixed `http://`);
  the last one becomes `$WORKER_CODE`, the server to fetch the worker code from (`:84-92`).
  With `OBS_USE_SLP=yes`, additional `--reposerver` entries are discovered via
  `slptool findsrvs service:obs.repo_server` (skipping localhost/127.*) (`:386-394`).
- `--cachedir $OBS_CACHE_DIR --cachesize $OBS_CACHE_SIZE` (only if both set; creates the dir).
- `--vm-kernel`, `--vm-initrd`, `--vm-custom-option "<opt>"`, `--localkiwi <dir> --arch local`,
  `--getbinariesproxy <url>`, `--getbinariesproxy-kiwiproduct <url>`, `--vm-enable-console`.
- VM type (`:136-159`): explicit `OBS_VM_TYPE` → `--emulator --arch <a> [--emulator-script s]`,
  `--vm-type qemu --arch <a>`, or `--<type>` (`--xen`, `--kvm`, `--lxc`, `--zvm`, `--pvm`…);
  `auto` → `--kvm` if `/dev/kvm` and `/usr/bin/qemu-kvm`, `--xen` if hypervisor type is xen
  and `xl`/`xm` exist, else chroot (no flag).
- z/VM: `--vmdisk-filesystem`, `--vmdisk-mount-options`, `--vmdisk-clean`; `check_vmcp`
  (`modprobe vmcp`, `vmcp set smsg iucv`…) and `create_initrd` (mkinitrd + dasd udev rule
  for swap device `ZVM_VOLUME_SWAP`, default 0200) at start (`:294-339,358-365`).
- kvm: `--vm-network` if `OBS_VM_ENABLE_NET=yes`.
- xen/kvm/qemu/emulator: `--vmdisk-rootsize`, `--vmdisk-swapsize`, `--vmdisk-filesystem`,
  `--vmdisk-mount-options`, `--vmdisk-clean` from the `OBS_VM_DISK_AUTOSETUP_*` vars.
- openstack: validates `OBS_WORKER_CONTROL_INSTANCE`, `OBS_WORKER_OS_FLAVOR`, `OS_*`;
  forces ≥1 instance; `--vmdisk-rootsize/--vmdisk-swapsize/--vm-memory` from `OBS_OPENSTACK_*`.

`start|run` (`:342-498`):
1. Recreates `$OBS_RUN_DIR/worker/boot` (`chown obsrun:obsrun $OBS_RUN_DIR`) and a
   `screenrc` (`zombie on`, `defscrollback 10000`, caption).
2. `NUM` = `OBS_WORKER_INSTANCES` or #CPUs (`/sys/devices/system/cpu/cpu[0-9]*`); for z/VM
   the count of `OBS_WORKER_INSTANCE_NAMES`.
3. Fetches the worker code: `curl -s $WORKER_CODE/getworkercode | cpio --extract` into the
   boot dir, retrying 10× with 10 s sleeps; `ln -s . XML`; `chmod 755 bs_worker` (`:396-412`).
   (Served by `bs_repserver` `'!worker /getworkercode'`, `bs_repserver:4742`; the worker
   later self-updates via the same URL, `bs_worker:4670`.)
4. For `I` in 1..NUM: `--port $((PORTBASE+I-1))` if PORTBASE>0; `--proto`; `WORKERID=$HOSTNAME:$I`
   (z/VM: `$HOSTNAME:<guestname>`); `R=$OBS_WORKER_DIRECTORY/root_$I`; for VM types
   `--device $R/root --swap $R/swap --vm-memory $OBS_INSTANCE_MEMORY [--tmpfs] [--hugetlbfs …]`
   (errors out if no autosetup size and `$R/root` missing); openstack adds
   `--vm-worker <prefix>I --vm-kernel <prefix>I-grub-image --device <prefix>I-root --swap <prefix>I-swap --vm-server … --openstack-flavor …`;
   z/VM adds `--vm-worker <guest> --vm-worker-nr I`; lxc only `--vm-memory`.
   Appends to screenrc:
   `screen -t $WORKERID nice -n $OBS_NICE ./bs_worker --hardstatus $vmopt $port $proto --root $R --statedir $OBS_RUN_DIR/worker/$I --id $WORKERID $REPO_PARAM $HUGETLBFS $HOSTLABELS $HOSTOWNER $OBS_JOBS $OBS_THREADS $OBS_TEST $OBS_WORKER_OPT $TMPFS $DEVICE $SWAP $MEMORY $OBS_CLEANUP_CHROOT $OBS_WIPE_AFTER_BUILD $OBS_WORKER_USE_MKFS_COPYIN $ARCH $EMULATOR`
   (`:484-488`) and `mkdir -p $workerdir/$I`.
5. `screen -S obsworker -m [-d] -c $screenrc` from the boot dir (`-d` only for `start`).

`stop` (`:499-512`): for each `$workerdir/*/state`, runs `./bs_worker --statedir <dir> --exit`
in parallel, waits, `killall bs_worker`, `killall -9 bs_worker`, `screen -S obsworker -X quit`.

Note: there is no `--srcserver` flag passed by this script; the worker only gets
`--reposerver`. The `--arch` flag is only set for emulator/qemu/localkiwi modes.

### 4.7 `dist/obsscheduler` (68 lines) — scheduler launcher

Sources `/etc/sysconfig/obs-server`; defaults `OBS_RUN_DIR=/srv/obs/run`,
`OBS_LOG_DIR=/srv/obs/log`, code dir `OBS_BACKENDCODE_DIR` or `/usr/lib/obs/server`
(`:6-26`). Architecture list = `bs_admin --show-scheduler-architectures` (from
`$bsdir/configuration.xml`), overriding legacy `OBS_SCHEDULER_ARCHITECTURES`; falls back
to `i586` with a warning (`:28-40`). Reads `bsuser`/`bsgroup` via `perl -MBSConfig`.
`start`: `mkdir -p rundir logdir; chown`, then for each arch `touch+chown
$logdir/scheduler_<arch>.log` and `$OBS_SCHEDULER_WRAPPER ./bs_sched <arch> >> log 2>&1 &`
(no pidfile — "FIXME: not nice", `:50-58`). `stop`: `bs_admin --shutdown-all-schedulers 600`,
and `pkill bs_sched` if exit code 2 (`:60-63`).

### 4.8 `dist/setup-appliance.sh` + `functions.setup-appliance.sh` and `obsapisetup.service`

`obsapisetup.service` runs `setup-appliance.sh --non-interactive --setup-only --with-gitea`
once per boot (After `obsstoragesetup`, `mariadb`; Before `apache2`). Other flags:
`--enable-optional-services`, `--force` (sets `OBS_API_AUTOSETUP=yes`),
`--disable-forceprojectkeys` (`setup-appliance.sh:55-65`). It is a no-op unless
`OBS_API_AUTOSETUP=yes` (`:67-70`); single-instance via `/run/setup-appliance.pid`.
Defaults `apidir=/srv/www/obs/api`, `backenddir=/srv/obs` (or `OBS_BASE_DIR`),
`MYSQL_USER=root`, `MYSQL_PASS=opensuse` (`:16-35`). Flow (`:72-178`):

1. `prepare_os_settings` (`functions:625-661`): per distro picks `MYSQL_SERVICE`
   (`mysql`/`mariadb`), `HTTPD_SERVICE` (`apache2`/`httpd`), `HTTPD_USER/GROUP`
   (`wwwrun:www` / `apache:apache`), passenger conf path, trust-anchor dir, package
   installer, `APACHE_ADDITIONAL_PACKAGES` (`apache2 apache2-mod_xforward rubygem-passenger-apache2 memcached`),
   `OBS_SIGND` (`obssignd`/`signd`).
2. `prepare_obssigner` (`functions:552-621`): if `/usr/lib/obs/server/BSConfig.pm` exists,
   uncomments/sets `$gpg_standard_key` and `$keyfile` to `/srv/obs/obs-default-gpg.asc`,
   `$sign = "/usr/bin/sign"`, `chmod 4755 /usr/bin/sign`; generates a 4096-bit RSA
   "private OBS <defaultkey@localobs>" key without passphrase in `$backenddir/gnupg`
   (`gpg2 --homedir … --gen-key`), exports it to `obs-default-gpg.asc`, touches
   `gnupg/phrases/defaultkey@localobs`; appends `user: defaultkey@localobs`,
   `server: 127.0.0.1`, `allowuser: obsrun`, `allow: 127.0.0.1`, `phrases: …/gnupg/phrases`
   to `/etc/sign.conf`; restarts signd. If the key was created, `systemctl reload
   obssrcserver obsrepserver` (`setup-appliance.sh:81-84`).
3. `check_required_backend_services` (`obsrepserver obssrcserver obsscheduler obsdispatcher obspublisher`
   → enable --now), `check_recommended_backend_services` (`obsdodup obsdeltastore obssigner $OBS_SIGND obsservicedispatch`,
   ask), `check_optional_backend_services` (`obswarden obsapisetup obsstoragesetup obsworker obsservice obssourcepublish`)
   (`functions:453-513`; all skipped in `--setup-only`), then `check_unit $MYSQL_SERVICE 1`.
4. `get_hostname` (`functions:154-208`): waits up to 30 s for an FQDN, falls back to
   `hostnamectl`, `/etc/hostname`, default-route IP. Tracks hostname changes in
   `$backenddir/.oldfqhostname`; on change runs `adapt_worker_jobs`.
5. `prepare_database_setup` (`functions:244-313`): if `rails db:migrate:status` fails →
   first-time init: rewrites `datadir` in `/etc/my.cnf*` to `/srv/obs/MySQL`,
   `mysql_install_db`, `chown mysql:mysql -R /srv/obs/MySQL`, restarts mysql, sets root
   password `opensuse`; then `rails db:setup writeconfiguration` (first time) or
   `rails db:migrate:with_data` (logged to `$apidir/log/db_migrate.log`), `chown obsrun:obsrun $backenddir`.
6. `enable_forceprojectkeys` (`mysql … update configurations SET enforce_project_keys=…`,
   `functions:677`), `check_server_key`, `generate_proposed_dnsnames`, `check_server_cert`
   (self-signed cert in `$backenddir/certs/server.{key,crt}`), `import_ca_cert`,
   `relink_server_cert`, `fix_permissions`.
7. `prepare_apache2` (`functions:515-542`): installs missing packages, `a2enmod passenger
   rewrite proxy proxy_http headers socache_shmcb xforward`, `a2enflag SSL`;
   `prepare_passenger` fixes `PassengerRuby` to `/usr/bin/ruby.ruby4.0`;
   `check_unit apache2`, `check_unit memcached`, `check_unit obs-api-support.target`.
8. `create_issue_file`, `create_overview_html` (fills `overview.html.TEMPLATE`
   `___API_URL___`/`___REPO_URL___`), `add_login_info_to_issue`, `set_gpg_expiry_date`,
   `create_sign_cert`, `prepare_gitea` (→ `obs-setup gitea` when `--with-gitea`).

### 4.9 `dist/obs-server.spec` — operationally relevant parts

- Users (`:352-408`): package `system-user-obsrun` provides user+group `obsrun`
  (uid 103 on sysusers systems, home `/usr/lib/obs`, shell nologin;
  `dist/system-user-obsrun.conf`); `system-user-obsservicerun` provides `obsservicerun`
  in group `obsrun`, home `/srv/obs/service` (`system-user-obsservicerun.conf`).
  `obs-server` `Requires: user(obsrun) user(obsservicerun)` (`:213-214`).
  `%triggerin -- docker` adds `obsservicerun` to group `docker` (`:930-931`).
- Sub-packages (`:238-350`): `obs-worker` (obsworker unit + script), `obs-common`
  (sysconfig template, `obsstoragesetup`, `setup-appliance.sh`), `obs-api` (Rails app at
  `/srv/www/obs/api`, api units, apache vhost), `obs-utils` (`obs_project_update`),
  `obs-tests-appliance`, `perl-OBS-XML` (`BSXML.pm` renamed to `OBS::XML`), `obs-backend-testsuite`.
- `%install` (`:480-550`): `make install` (both Makefiles), creates `rc<unit>` symlinks
  to `/usr/sbin/service` on SUSE, creates empty owned dirs
  `MySQL certs db diffcache remotecache repos sources trees upload workers` under
  `/srv/obs` (`:508-510`), fails unless `/usr/lib/obs/server/build` is a symlink
  (`:521-524`), installs sysconfig template, sysusers files.
- `%pre` (`:600-630`): `%service_add_pre` for all 15 backend units; chowns existing logs
  `$OBS_LOG_DIR/{deltastore,dispatcher,dodup,obsgetbinariesproxy,publisher,rep_server,servicedispatch,signer,src_server,warden}.log`
  to `obsrun:obsrun` and `src_service.log` to `obsservicerun:obsrun`.
- `%post` (`:666-681`): `%service_add_post` for the same units. `%posttrans` (`:686-694`):
  creates `/srv/obs` owned `obsrun:obsrun` if missing; replaces a real
  `/usr/lib/obs/server/build` directory by the symlink `../../build` (→ `/usr/lib/build`,
  the `obs-build`/`build` package).
- `%postun` (`:696-713`): `service_del_postun_with_reload` (a locally defined macro that
  does `systemctl force-reload`/`set-property Markers=+needs-reload` on upgrade instead
  of restart, `:84-102`); `rmdir /srv/obs` if empty. `obs-common` and `obs-worker` are
  explicitly **not** restarted on upgrade (`:715-724`).
- `%post -n obs-api` (`:748-791`): migrates old `frontend/config/database.yml`,
  generates `config/secret.key` (`rails secret`, mode 0640 root:www), rewrites
  `adapter: mysql` → `mysql2`, touches `log/production.log`, enables
  `obs-api-support.target` when upgrading from SysV `obsapidelayed`.
- `%files` (`:798-931`): backend dir ownership table for `/srv/obs` (see §5 table);
  config(noreplace) `BSConfig.pm`, `/etc/slp.reg.d/*`, logrotate, cron;
  `%ghost /usr/lib/obs/server/build`.
- `%files -n obs-api` (`:941-1058`): `config/database.yml` 0640 root:www noreplace,
  `config/options.yml` 0644 root noreplace (+ `.example` files), `thinking_sphinx.yml`,
  `production.sphinx.conf`, `/etc/apache2/vhosts.d/obs.conf` noreplace, log/tmp/storage
  dirs owned by `wwwrun:www`, `%ghost` log files and `secret.key`.

### 4.10 Web-server, DB and API config templates

- `dist/obs-apache24.conf` (installed as `/etc/apache2/vhosts.d/obs.conf`): `Listen 82`;
  Passenger `smart`/pool 20; `LimitRequestLine/Fieldsize 20000`, `LimitRequestBody 0`;
  vhost `*:80` serves `@@OBS_DOCUMENT_ROOT@@/overview`; vhost `*:82` serves
  `@@OBS_BACKEND_DATA_DIR@@/repos` with `Indexes FollowSymLinks` (the public download
  tree); vhost `*:443` `ServerName api`, DocumentRoot `@@OBS_API_PREFIX@@/public`,
  SSL cert/key `@@OBS_BACKEND_DATA_DIR@@/certs/server.{crt,key}`, `XForward on`
  (mod_xforward; enabled by `use_xforward` in options.yml), gzip asset rewrite rules,
  `MAINTENANCE` define → 503 page, and **ProxyPass** `/v2`, `/extensions/v2` →
  `http://localhost:5352/registry`, `/sigstore` → `http://localhost:5352/sigstore`
  (container registry on the source server), `/gitea` → `http://localhost:3000`
  (`:138-147`). `obs-apache2.conf` is the Apache 2.2 variant (`Allow from all`, no
  placeholders, no gzip rules, no `LimitRequestBody`).
- `dist/nginx-obs-api.conf` + `internal_redirect.include`: Passenger-on-nginx vhost
  (`wwwrun:www`, `rails_env production`) and the `/internal_redirect/<proto>/<host>/<uri>`
  internal location used by `use_nginx_redirect` (X-Accel-Redirect) with `resolver 8.8.8.8`.
- `dist/obs-lighttpd.conf`: legacy lighttpd setup (443 rails, 80→https redirect, 82 repos).
- MySQL/MariaDB: there is **no** my.cnf template in `dist/`; the appliance script patches
  `datadir` in `/etc/my.cnf*` to `/srv/obs/MySQL` and the spec owns
  `/srv/obs/MySQL` as `mysql:mysql` (`obs-server.spec:887`). `obsstoragesetup` touches
  `/srv/obs/MySQL/.run-mysql_upgrade`. `src/api/config/database.yml.example`: adapter
  `mysql2`, db `api_production`, `root`/`opensuse`, utf8mb4, pool 30.
- API options: `src/api/config/options.yml.example` (installed as `options.yml`,
  `obs-server.spec:1031-1032`). Backend-relevant keys: `use_xforward`,
  `use_nginx_redirect: /internal_redirect`, `source_host: localhost`,
  `source_port: 5352`, `source_protocol`, `source_protocol_ssl_verification`,
  `allow_anonymous`, `proxy_auth_mode` (`:off|:on|:mellon|:ichain`) and
  `proxy_auth_{login,logout,register,account}_page`, `global_write_through`,
  `backup_*`, `notifications_lifetime`, `maintenance_release_repositories_lifetime`,
  `limit_for_search_results`; test/dev override `source_host: backend`,
  `memcached_host: cache` (`:1-70,125-180`). There is no `obs-api` sysconfig file.
- `dist/*.logrotate`: backend logs in `/srv/obs/log/*.log` rotated daily ×14, `su obsrun obsrun`,
  `copytruncate` (`src_service.log` as `obsservicerun`); `/srv/obs/service/log/*.log`
  same; api logs `/srv/www/obs/api/log/*log` by size 8 MB, `wwwrun www`.
- `dist/cleanup_scm_cache.cron`: hourly (`33 * * * *`) as `obsservicerun`:
  `/usr/lib/obs/server/cleanup_scm_cache --log=/srv/obs/service/log/cleanup_scm_cache.log`.
- SLP: `obs.source_server.reg` / `obs.repo_server.reg` announce
  `service:obs.source_server:http://$HOSTNAME:5352` and `…repo_server…:5252`
  (`watch-port-tcp`); BSConfig comments say to update them when changing ports
  (`BSConfig.pm.template:51`).
- `dist/munin/*`: plugins reading `/srv/obs/configuration.xml` for the arch list and
  `/srv/obs/info/schedulerinfo.<arch>` (queue high/med/low/next), plus dispatcher,
  srcserver file sizes/package counts, worker cpu/mem.
- `dist/openstack/`: README + `create-grub-image.sh`, `create-secgroups.sh`,
  `create-vm-volumes.sh` for `OBS_VM_TYPE=openstack` deployments.
- `dist/t/`: appliance QA (bash-tap `.ts`/`.ta` scripts checking users/groups, required
  services, certs, DB, webserver/API, signd; RSpec UI tests) — run by openQA.
- Misc helpers: `obs_project_update` (osc-based project copy between instances),
  `obs_mirror_project` (ruby, mirrors binaries of a base distro), `obs_rebuild_db`
  (re-imports projects/packages into a fresh API from backend XML), `obs_import_srcrpmtree`,
  `obs_import_srcdebtree`, `install_rh.sh` (RHEL bootstrap), `setup_source_service_docker.sh`
  (legacy docker-based service containment), `gem_build_cleanup.sh`,
  `openQA_mail_notification.rb`, `resolve_swagger_yaml.rb` (build-time API docs),
  `run_shellcheck.sh` (lint for `dist/`).

---

## 5. Install layout, users and the on-disk `$bsdir` tree

### 5.1 Code install (`src/backend/Makefile`, `Makefile.include`)

- `OBS_BACKEND_PREFIX=/usr/lib/obs/server` (`Makefile.include:2`). `make install`
  (`src/backend/Makefile:15-24`) copies **everything** in `src/backend/` (`cp -a ./*`)
  into it, then removes `build/`, `testdata/`, `examples/`, `Makefile`, and leaves a
  symlink `build → /usr/lib/build` (the `obs-build` package provides `Build.pm` et al.).
  `BSConfig.pm` is created from `BSConfig.pm.template` if absent (`:11-12`).
  Result: all `bs_*` programs, `*.pm` modules and module dirs (`BSSched/`, `BSSrcServer/`,
  `BSRepServer/`, `BSPublisher/`, `BSDispatcher/`, `BSSetup/`, `plugins/`, `templates/`,
  `worker/`) live flat in `/usr/lib/obs/server` (spec `%files`, `obs-server.spec:839-881`).
- Per-host override: `/usr/lib/obs/server/bsconfig.<fqdn>` is `require`d by `BSConfig.pm`
  if readable (`BSConfig.pm.template:254-260`); `bs_setup` appends to it.
- `install_data_dirs` (`src/backend/Makefile:4,26-30`) creates under
  `OBS_BACKEND_DATA_DIR=/srv/obs`: `build events info jobs log projects repos run sources
  trees workers service service/log` (0755) and `gnupg` (0700). The spec adds
  `MySQL certs db diffcache remotecache repos sources trees upload workers`
  (`obs-server.spec:508-510`).
- Users/ownership of `/srv/obs` per spec `%files` (`obs-server.spec:886-906`):

| Path | Mode | Owner |
|---|---|---|
| `/srv/obs` | 0775 | obsrun:obsrun |
| `MySQL` | 0755 | mysql:mysql |
| `build`, `db`, `diffcache`, `events`, `info`, `jobs`, `projects`, `remotecache`, `repos` | 0755 | obsrun:obsrun |
| `certs`, `gnupg` | 0700 | root:root |
| `log`, `run`, `sources`, `trees`, `upload`, `workers` | 0775 | obsrun:obsrun |
| `service`, `service/log` | 0755 | obsservicerun:obsrun |

- Runtime identities: every backend daemon starts as root from systemd and calls
  `BSUtil::drop_privs_to($BSConfig::bsuser,$BSConfig::bsgroup)` = `obsrun:obsrun`
  (`BSStdServer.pm:515`, `BSStdRunner.pm:212`, plus explicit calls listed in §4.2);
  `bs_service` uses `$BSConfig::bsserviceuser` = `obsservicerun` (`BSConfig.pm.template:87-88`;
  set to root if the lxc wrapper is used). The Rails API runs as `wwwrun:www`.
  Workers run as root on build hosts (no `User=`; the worker needs root for
  chroot/VM handling).

### 5.2 `$bsdir` directory map (from `grep -rnoE '"\$(BSConfig::)?bsdir/…'` over `src/backend`)

| Subpath | Used by (daemons/modules) | Purpose |
|---|---|---|
| `configuration.xml` / `.configuration.xml` | `BSConfiguration.pm:44`, `bs_srcserver`, `bs_repserver`, `bs_service`, `bs_admin` | Instance config pushed by the API (`PUT /configuration`); includes `schedulers/arch` list. |
| `issuetrackers.xml` / `.issuetrackers.xml` | `bs_srcserver` | Issue tracker config from the API. |
| `build/` | `bs_sched`, `bs_repserver`, `bs_publish`, `bs_dodup`, `BSRepServer/{BuildInfo,SLSA}.pm`, `bs_admin`, `bs_check_consistency` | `build/<prj>/<repo>/<arch>/` — scheduler/repserver result tree: per-package result dirs, `:full`, `:full.solv`, `:repo`, `:repoinfo`, `:packstatus`, `:jobhistory`, `:relsync*`, `:meta/`, `:logfiles.*`, `_volatile`; `build/_repoid`. |
| `events/` | all daemons (see list in §4.2) | Per-consumer queues: `events/<arch>/` (scheduler), `publish/`, `sourcepublish/`, `deltastore/`, `servicedispatch/`, `signer/`, `notifyforward/`, `redis/`, `undelivered/` (`bs_srcserver:7954`); files `lastevents`, `lastnotifications`; each queue has a `.ping` fifo/file. |
| `jobs/` | `bs_sched`, `bs_dispatch`, `bs_repserver`, `bs_signer`, `bs_warden` | `jobs/<arch>/<job>` buildinfo, `<job>:status`, `<job>:dir/`; `jobs/dispatchprios`, `jobs/load`, `jobs/mdload`, `jobs/finished`. |
| `workers/` | `bs_repserver`, `bs_dispatch`, `bs_warden` | `workers/{idle,building,down,disable}/<workerid>` worker XML state. |
| `run/` | `BSStdServer.pm`, `BSStdRunner.pm`, `bs_sched`, `bs_dispatch`, `bs_publish`, … | Control files: `<daemon>` (BSServer status file for `bs_serverstatus`), `<daemon>.lock`, `<daemon>.exit`, `<daemon>.restart`, `<daemon>.state`/`.state.new`, `<daemon>.dumpstate`, `bs_sched.<arch>.{lock,state,pmat,dead}`, `bs_srcserver.ajax` / `bs_repserver.ajax` (unix sockets), `dispatch.badhosts`, `dispatch.data`, `bs_dispatch.dropbadhosts`, `bs_publish.rescan`. |
| `log/` | `BSConfiguration.pm`, `bs_sched`, `bs_service`, shell wrappers | `<name>.log`, `<name>.crit.log`, `scheduler_<arch>.log`. |
| `sources/` | `bs_srcserver`, `BSSrcrep.pm`, `BSRevision.pm`, `BSSrcServer/{Link,Product,Remote,Scmsync,Service,Signkey}.pm`, `bs_deltastore`, `bs_servicedispatch`, `bs_sourcepublish`, `bs_admin` | Content-addressed source store `sources/<pkg>/<md5>-<file>`, `<pkg>/deltastore`, `<srcmd5>-MD5SUMS` trees (shared mode), `sources/:upload`, `sources/_project/<md5>-_signkey|_pubkey|_sslcert`. |
| `trees/` | `BSSrcrep.pm`, `bs_admin` | `trees/<prj>/<pkg>/<srcmd5>-MD5SUMS` when `$nosharedtrees` (default 2). |
| `projects/` | `bs_srcserver`, `BSRevision.pm`, `BSSrcrep.pm`, `BSSrcServer/*`, `bs_admin`, `bs_check_consistency` | `<prj>.xml`, `<prj>.conf`, `<prj>.pkg/<pkg>.xml`, `.rev`, `.mrev`, `.rev.del`, `_pubkey`, `_signkey`; `projects/_deleted/`. |
| `requests/`, `requests.old/` | `bs_srcserver`, `bs_archivereq`, `bs_admin` | Request XML by id, `.nextid`. |
| `db/` | — | `db/source` (linkinfo/scmsync; BSDB shards or sqlite), `db/published` (binary/pattern/repoinfo), `db/request` (legacy), `db/sqlite` (`bs_sourcepublish`, `BSSrcServer/SQLite.pm`). |
| `repos/` | `bs_publish`, `bs_repserver`, `bs_srcserver`, `BSUrlmapper.pm`, `BSRepServer/BuildInfo.pm` | Published repositories (`repos/<prj>/<repo>/…`), served on port 82. `repos_sync/` (`bs_publish`). |
| `upload/` | `bs_repserver`, `bs_publish`, `bs_signer`, `bs_service`, `bs_deltastore`, `BSPublisher/{Container,Registry}.pm`, `BSRepServer/{Containertar,Registry}.pm` | Temp dir for in-flight uploads. |
| `remotecache/` | `bs_sched`, `BSSrcServer/Remote.pm` | Cache for remote (interconnect) project data. |
| `info/` | `bs_sched`, `bs_repserver` | `info/schedulerinfo.<arch>` (queue stats, read by munin and the API). |
| `diffcache/` | `bs_srcserver` | Cached source diffs. |
| `dods/` | `bs_dodup`, `bs_sched`, `bs_admin` | Download-on-demand metadata; `.changed`, `<md5>.recheck`. |
| `registry/`, `blobs/`, `slsa/`, `zchunk/` | `BSPublisher/Registry.pm`, `BSRepServer/Registry.pm`, `BSSrcServer/Registry.pm`, `BSPublisher/Blobstore.pm`, `BSRepServer/SLSA.pm`, `bs_publish` | Local container registry (served via `/v2` proxy), blob store, SLSA provenance, zchunk data. |
| `notificationpayload/` | `bs_srcserver` | Stored notification payloads. |
| `getbinariesproxycache/` | `bs_getbinariesproxy` | Proxy cache. |
| `service/`, `service/containers`, `service/log` | `bs_service`, `BSConfiguration.pm:88` (`servicetempdir`), `BSSetup/Base.pm` | Source service temp dir, podman image store, service logs. |
| `tmp/` | `bs_admin` | `--clone-repository` scratch. |
| `MySQL/`, `certs/`, `gnupg/`, `obs-default-gpg.asc`, `.oldfqhostname` | appliance scripts / spec | MariaDB datadir, TLS cert, default signing key homedir, hostname tracking. |
| `events/redis` | `BSSrcServer/ScmsyncDB.pm` | redis job queue for scmsync. |

### 5.3 Ports and inter-service endpoints (for completeness)

`BSConfig.pm.template:52-58`: `$srcserver=http://<fqdn>:5352`, `$reposerver=http://<fqdn>:5252`,
`$serviceserver=http://<fqdn>:5152`; optional `$workersrcserver`/`$workerreposerver`
(5353/5253) for worker-only listeners. `$ipaccess` (`:34-46`) grants `rw` to
localhost/own IP/`$frontend`, `worker` to everyone else. Optional binary-backend
partitioning (`$partition`, `$partitioning`, `$partitionservers`, `:234-252`) lets
several repserver+scheduler+dispatcher+warden+publisher sets share one source server.
Signing is external (`$sign=/usr/bin/sign`, `$keyfile`, `$gpg_standard_key`, `$certfile`,
`:205-212`) via the `signd` daemon configured in `/etc/sign.conf` by `prepare_obssigner`.
