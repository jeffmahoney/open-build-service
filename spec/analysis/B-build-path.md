# OBS Build Execution Path: repserver, dispatcher, worker, warden, obs-build

Engineering analysis for a ground-up re-architecture as Go microservices.
All paths are relative to `/home/jeffm/src/open-build-service/src/backend/` unless
absolute. Line numbers are from the checked-out tree at the time of analysis
(master, HEAD db93ae665b). Every major claim cites `file:line`.

Scope covered here (the "build execution path"):

| Component | File | Lines |
|---|---|---|
| Repository server | `bs_repserver` | 4850 |
| Repserver helper modules | `BSRepServer.pm`, `BSRepServer/{Bininfo,BuildInfo,Checker,Containerinfo,Containertar,DoD,ProjPacks,Registry,Remote,SLSA,YMP}.pm` | ~2300 |
| Job dispatcher | `bs_dispatch` + `BSDispatcher/Constraints.pm` | 1510 + 293 |
| Worker | `bs_worker` | 5282 |
| Warden (dead-worker detector) | `bs_warden` | 310 |
| Binary proxy for workers | `bs_getbinariesproxy` | 641 |
| Shared infrastructure | `BSHandoff.pm`, `BSRunner.pm`, `BSStdRunner.pm`, `BSFileQueue.pm`, `BSCpio.pm`, `BSTar.pm`, `BSContar.pm`, `BSBuild.pm`, `BSCando.pm`, `BSXML.pm` | |
| obs-build submodule | `build/build`, `build/build-vm*`, `build/build-recipe*`, `build/init_buildsystem`, `build/Build.pm`, `build/getbinaryid`, `build/computeblocklists`, `build/extractbuild` | |
| Scheduler touch points (job creation / result ingest) | `BSSched/BuildJob.pm`, `BSSched/BuildJob/Package.pm`, `BSSched/BuildJob/PreInstallImage.pm`, `BSSched/EventHandler.pm` | cited only |

---

## 0. Executive summary / mental model

The build path is a **shared-filesystem state machine** with three daemons on the
"binary backend" host (repserver, dispatcher, warden; the per-arch schedulers and the
publisher share the same `$BSConfig::bsdir`) and N stateless-ish workers that reach the
backend only over HTTP.

* The **scheduler** (bs_sched, one per arch) decides *what* to build and writes a job
  file `jobs/<arch>/<job>` (an XML `buildinfo`) (`BSSched/BuildJob.pm:319-349`).
* The **dispatcher** (bs_dispatch, one per backend) decides *where* to build. It scans
  `jobs/<arch>/*` and `workers/idle/*`, picks a job/worker pair, `PUT`s the job XML to the
  worker's `/build` endpoint, and records a `jobs/<arch>/<job>:status` file
  (`bs_dispatch:237-365`, main loop `826-1476`).
* The **worker** (bs_worker) registers itself with every configured repserver
  (`POST /worker?state=idle`), runs a tiny HTTP server, accepts one job at a time,
  fetches sources from the srcserver and binaries from repserver(s), invokes the
  obs-build `build` script, and `POST`s results back as a cpio to `/putjob`
  (`bs_worker:4656-4727`, `3910-4404`, `5082-5262`).
* The **repserver** (bs_repserver) is the HTTP front for the on-disk repository
  tree (`build/<prj>/<repo>/<arch>/...`), the worker registry (`workers/{idle,building,
  away,down,dead}`), and the job directory (`jobs/<arch>/`). It writes the job result
  into `jobs/<arch>/<job>:dir` and drops an event file in `events/<arch>/` for the
  scheduler (`bs_repserver:2333-2619`).
* The **warden** (bs_warden) polls workers in `workers/building/*` and resets
  `:status` files of jobs whose worker vanished (`bs_warden:126-309`).
* The scheduler then ingests `jobs/<arch>/<job>:dir` into
  `build/<prj>/<repo>/<arch>/<pkg>/` and the `:full` tree (`BSSched/BuildJob.pm:498-712`).

Everything between scheduler, dispatcher, repserver, warden, signer and publisher is
**coordinated through files**: `:status` files with `flock`+`link` atomic creation
(`BSUtil.pm:494-509`), XML event files in `events/<consumer>/`, `.ping` FIFO-style
wakeups (`BSUtil.pm:615-660`), and Storable blobs. The worker is the only component
that is purely RPC-driven.

---

## 1. On-disk layout served and written by the build path

Root is `$BSConfig::bsdir` (default `/srv/obs`). Directories declared in
`bs_repserver:103-115`:

```
$bsdir/
  build/                 $reporoot  - repository tree (scheduler-owned, repserver-read)
  workers/               $workersdir- worker registry
  jobs/                  $jobsdir   - job queue, one subdir per arch
  events/                $eventdir  - event spool for schedulers/publisher/signer/dispatcher
  info/                  $infodir   - schedulerinfo.<arch> etc.
  upload/                $uploaddir - scratch for uploads
  run/                   $rundir    - lock files, daemon control files
  registry/              $registrydir - container registry served at /registry
  repos/                 $extrepodir  - published repositories (publisher-owned)
  db/published           $extrepodb
  slsa/                  SLSA artifact store (BSRepServer/SLSA.pm:31)
  blobs/sha256/<3>/<61>  DoD container blob store (BSRepServer/Registry.pm:38,87-102)
  getbinariesproxycache/ bs_getbinariesproxy cache (bs_getbinariesproxy:59)
  configuration.xml      global OBS configuration (bs_repserver:4259-4280)
```

### 1.1 `build/<prj>/<repo>/<arch>/` ("gdst")

Per package dir `build/<prj>/<repo>/<arch>/<pkg>/` (written by the scheduler in
`BSSched/BuildJob.pm:498-712` when it ingests a finished job):

| Entry | Writer | Purpose / reader |
|---|---|---|
| `<binaries>` (`*.rpm`, `*.deb`, `*.tar`, `*.containerinfo`, `*.helminfo`, `*.report`, `*.packages`, `*.sha256`, `_statistics`, `_buildenv`, `_modulemd.yaml`, `rpmlint.log`, `_slsa_provenance.json`, `_ccache.tar`, `.kiwitree` subtrees...) | scheduler (`rename` from `jobs/<arch>/<job>:dir`, `BSSched/BuildResult.pm:379-...`) | repserver `getbinarylist`/`getbinary` (`bs_repserver:1062-1245`, `1595-1625`) |
| `logfile` | scheduler | `getlogfile` (`bs_repserver:1329-1390`) |
| `status` (`$BSXML::buildstatus`) | scheduler: `{status=>'scheduled', readytime, job}` at job write (`BSSched/BuildJob.pm:331`), `succeeded`/`failed` on ingest | `getbuildstatus` (`bs_repserver:1268-1307`), `getjobstatus` |
| `reason` (`$BSXML::buildreason`) | scheduler (`BSSched/BuildJob.pm:334`) | `getbuildreason` (`:1260`) |
| `history` (BSFileDB, layout `versrel bcnt srcmd5 rev time duration`, `bs_repserver:102`) | scheduler `addsucceededhist` (`BSSched/BuildJob.pm:900-910`) | `getbuildhistory`, `nextbcnt` (build counter) |
| `.stats` (BSFileDB, `$BSXML::buildstatslay`) | scheduler `addbuildstats` (`:847-855`) | `getbuildstats` (`bs_repserver:1253`) |
| `.failhistory` | `addjobhist` (`:879-898`) | `notify_jobresult` successive_failcount (`bs_repserver:2325`) |
| `.bininfo` (Storable) | repserver `putjob` creates it in the job dir (`bs_repserver:2534`), scheduler merges | `read_bininfo` (`BSRepServer.pm:133-171`) |
| `.preinstallimage.<md5>` | scheduler `update_preinstallimage` (`BSSched/BuildJob/PreInstallImage.pm:172-176`) | `getbinary` special-case `_preinstallimage.<md5>` (`bs_repserver:1598-1605`) |
| `.preinstallimage`, `.nouseforbuild`, `.nosourceaccess`, `.channelinfo`, `.meta.success`, `.setrelease` | scheduler / copybuild | flags |
| `.checksums` | signer | `getbinarylist withmd5`/`cpioheaderchksums` (`bs_repserver:1132-1160`) |

Per-arch control files in `build/<prj>/<repo>/<arch>/`:

| Entry | Meaning |
|---|---|
| `:full/` | the "use for build" binary pool (one copy of each binary that other packages may build against), plus `:full.solv` (libsolv cache), `:full.xcache` (Storable cache served as `view=cache`, `bs_repserver:774-800`), `:full.metacache`, `:full.subdirs`, `:full/doddata`, `:full/doddata.needed` (DoD) |
| `:repo/` + `:repoinfo` | the published repository snapshot (publisher input) |
| `:meta/<pkg>` | the "meta" dependency fingerprint of the last build (`BSSched/BuildJob.pm:593,651`) used for rebuild decisions |
| `:logfiles.success/<pkg>`, `:logfiles.fail/<pkg>` | last success/fail logs (`:707-711`) |
| `:packstatus`, `:packstatus.finished` | scheduler status map + append-only overrides (`bs_repserver:133-166`, `BSSched/BuildJob.pm:834-845`) |
| `:schedulerstate`, `:schedulerstate.dirty` | per-repo scheduler state; repserver `dirty()` touches `.dirty` (`bs_repserver:1928-1942`) |
| `:jobhistory` (BSFileDB `$BSXML::jobhistlay`) | `addjobhist` (`BSSched/BuildJob.pm:879-898`); served by `getjobhistory` (`bs_repserver:3485-3534`) |
| `:bininfo`, `:bininfo.merge` | global bininfo (`BSRepServer.pm:110-131`) |
| `:preinstallimages` (Storable array) | index of available preinstall images (`BSSched/BuildJob/PreInstallImage.pm:131-189`) |
| `:relsync`, `:relsync.merge`, `:relsync.max` | release-number sync between partitions (`bs_repserver:4008-4058`) |
| `:depends` | builddepinfo source (`bs_repserver:3623-3709`) |
| `:lastfailures` | cache for `code=lastfailures` jobhistory queries (`:3406`) |
| `:repodone`, `:reproduciblecheck.fail/` | misc |

### 1.2 `jobs/<arch>/`

| Entry | Writer | Reader |
|---|---|---|
| `<job>` = `<prj>::<repo>::<pkg>-<srcmd5>` (XML `$BSXML::buildinfo`) | scheduler `writejob` (`BSSched/BuildJob.pm:345`); or repserver `addjob` when master-dispatched (`bs_repserver:4094-4107`); or `copybuild`/`uploadbuild` fake jobs | dispatcher, repserver, warden |
| `<job>:status` (XML `$BSXML::jobstatus`) | dispatcher `assignjob` creates with `code=dispatching` via `lockcreatexml` (`bs_dispatch:245`), rewrites `building` (`:349`); repserver `putjob` rewrites `finished`/`signing` (`:2572-2574`,`2600`); scheduler `killjob` creates `deleting` (`BSSched/BuildJob.pm:169-171`) | everyone. **Existence of this file is the "claimed" lock for the job.** |
| `<job>:dir/` | repserver `putjob` (rename of `.putjob.<pid>` tmp dir, `bs_repserver:2545`) | scheduler `jobfinished` |
| `<job>:<targetarch>:cross` | scheduler `add_crossmarker` (`BSSched/BuildJob.pm:387-402`) when a job for arch X must be built on hostarch Y: marker lives in `jobs/<hostarch>/` | dispatcher (`bs_dispatch:987-1004`) |
| `.logfile.badhost/<job>` | repserver `putjob` on `code=badhost` (`:2378-2395`) | humans |
| `finished` (`|`-separated lines, BSFileQueue style) | repserver `putjob` (`:2541-2544`), dispatcher `addbadhosttofinished` (`:600-609`) | dispatcher load accounting (`:828-860`) |
| `load` (Storable) | dispatcher (`:859`) | dispatcher |
| `dispatchprios` (Storable `$BSXML::dispatchprios`) | `PUT /build/_dispatchprios` (`bs_repserver:4060-4069`) | dispatcher (`:869-897`) |
| `mdload` (Storable) | slave dispatchers via `POST /jobs/_mdload` (`bs_repserver:4161-4189`) | master dispatcher (`:911-933`) |

Job naming: `jobname($prp,$packid)` replaces `/` by `::`; if the result is >200 chars it
becomes `:<md5(prp)>::<pkg or :md5(pkg)>` (`BSSched/BuildJob.pm:1341-1348`, duplicated in
`bs_repserver:125-131`). The full job name appends `-<srcmd5>` (`BSSched/BuildJob.pm:1090`).

### 1.3 `workers/`

```
workers/idle/<hostarch>:<workerid>      XML $BSXML::worker  (registration, capabilities, uri)
workers/building/<hostarch>:<workerid>  same + job, jobid, arch, [reposerver if masterdispatched]
workers/away/<...>                      worker said "building" for some other repserver
workers/down/<...>                      exited / unreachable / disabled / job finished
workers/dead/<...>                      (listed in workerstatus; set via nextstate=dead path)
workers/disable/<prefix>                admin drop-in: any workerid matching ^prefix is refused
```

`idlename = "$harch:$workerid"` with `/` replaced by `_` (`bs_repserver:1768-1769`).
State transitions are implemented by `updateworkerdata` which renames across the
state dirs (`bs_repserver:1734-1743`).

### 1.4 `events/`

Event files are XML `$BSXML::event` (`BSXML.pm:771-794`), named
`<type>:<project>::<package>::<repo>::<arch>...` (md5-hashed when >200 chars), written
with `writexml(tmp, final)` and followed by `BSUtil::ping("$eventdir/<consumer>/.ping")`
(`bs_repserver:2924-2991` `forwardevent`). Consumers: `events/<arch>/` (scheduler for
that arch), `events/publish/`, `events/signer/`, `events/dispatch/` (dispatcher),
`events/repository/` (dispatcher forwards these to the srcserver,
`bs_dispatch:613-631`). Event types emitted by the build path:

| Type | Emitter | Consumer / handler |
|---|---|---|
| `built` (`finished:<job>`) | repserver `putjob` (`:2600-2616`), `failjob` (`:4249-4257`) | scheduler `event_built` (`BSSched/EventHandler.pm:98-141`) -> `BSSched::BuildJob::jobfinished` |
| `built` to `events/signer/` (`finished:<arch>:<job>`) | `putjob` when `$BSConfig::sign` and signable files (`:2588-2598`) | bs_signer, which re-emits to scheduler |
| `badhost` to `events/dispatch/` | `sendbadhostevent` (`:2147-2162`) | dispatcher `forwardevents` (`:631-660`) |
| `dispatchdetails` | dispatcher via `setdispatchdetails` (`bs_dispatch:1478-1497`, repserver `:4208-4219`) | scheduler `event_dispatchdetails` appends `scheduled <pkg>/<job>/<details>` to `:packstatus.finished` (`BSSched/EventHandler.pm:687-698`) |
| `scanrepo`, `scanprjbinaries`, `rebuild`, `recheck`, `relsync`, `wipe`, `wipeallarch`, `force_publish`, `suspendproject`/`resumeproject`, `uploadbuild`, `copybuild`, `configuration`, `publish` | repserver handlers (`putbinary`, `delbinary`, `docommand`, `copybuild`, `uploadbuild`, `putconfiguration`, DoD fetches) | schedulers / publisher |

---

## 2. Repository server (`bs_repserver`)

### 2.1 Server configuration and authentication

* Listens on `$BSConfig::reposerver` port (default 5252) and optionally a *second*
  port for workers (`$BSConfig::workerreposerver`, e.g. 5253): `bs_repserver:4837-4845`
  sets `$conf->{'port2'}`; both must use the same proto. `maxchild` 20 (`:4824`).
* Route syntax (`BSDispatch.pm:144-243`): `'[!<auth>] [METHOD[|METHOD]:]/path/$var[:type] cgi[:type][?+*] fixed=value' => \&handler`.
  A leading `!worker` / `!rw` / `!-` sets the required authorization class; `!-` means
  "no auth". The repserver's route table starts with `'!rw :' => undef`,
  `'!- GET:' => undef`, `'!- HEAD:' => undef` (`bs_repserver:4696-4698`): the
  *default* for any mutating method is `rw`, GET/HEAD default to no auth, and
  individual worker routes are tagged `!worker`.
* Authorization is IP-based (`BSStdServer.pm:77-106`): `$BSConfig::ipaccess` maps
  peer-IP regexes to comma-separated classes (`BSConfig.pm.template:34-46`: localhost
  and the backend's own IP are `rw`; `.*` is `worker`). Optionally
  `$BSConfig::subjectdnaccess` (TLS client cert DN) and a `trusted_peerip_header`.
  **There is no per-worker credential**; a worker is anything that can reach the
  worker port from an allowed IP. Only `$BSConfig::putjob_verify_peerip` adds a check
  that the `putjob` comes from the IP recorded in the jobstatus `uri`
  (`bs_repserver:2341-2349`).
* Outgoing RPCs from repserver to the srcserver may use signature auth
  (`BSStdServer::setup_authenticator`, `BSStdServer.pm:365-373`, keyed by
  `$BSConfig::signature_auth_keyfile`); this is backend-to-backend, not worker-related.
* An AJAX (event-loop) twin process listens on the unix socket
  `run/bs_repserver.ajax` (`bs_repserver:4805-4815`, `4829-4835`); forking handlers
  hand long-running streaming requests over to it with `BSHandoff::handoff`
  (fd passing via `SCM_RIGHTS`, `BSHandoff.pm:58-103`). Routes available in the
  AJAX process: `getlogfile`, `getbinarylist`, `getbinary`, `getresult`,
  `getbinaries`, `getbinaryversions` (`:4805-4815`). `reposerver_ajaxpartitions`
  allows several AJAX processes keyed by partition name (`BSHandoff::handoff_part`).

### 2.2 Routes grouped by consumer

Route table: `bs_repserver:4695-4802` (main), `4805-4815` (ajax). Column "Auth" shows
the BSDispatch auth class after the defaults above.

#### 2.2.1 Consumed by the WORKER (`bs_worker`)

| Route | Auth | Handler | What it does |
|---|---|---|---|
| `POST /worker $arch $port $state workerid? working? memory? disk? buildarch* tellnojob? proto?` | worker | `workerstate` `:1752-1846` | Registration/heartbeat. `state=idle` with the worker XML as body -> write `workers/idle/<harch:workerid>` with `uri` computed from peer IP + port (`:1795-1805`). `state=building` -> `away`; `state=exit` -> `down`. If `workers/building/<idlename>` exists when a worker says idle, the repserver assumes a crash and **unlinks the job's `:status`** so the job is re-dispatched (`:1773-1794`), or forwards `cmd=idleworker` to the owning repserver if the job was masterdispatched. Optional `checkclientconnectivity` does a 1s probe of `http://peer:port/status` (`:1808-1823`). `workers/disable/<prefix>` refuses workers (`:1825-1834`). |
| `GET /getworkercode`, `GET /getbuildcode` | worker | `getcode` `:3795-3822` | cpio of the `worker/` resp. `build/` directory (including `Build/`, `emulator/` subdirs and legacy `qemu/` interpreters) so workers self-update. |
| `POST /putjob $arch $job $jobid $code? now? kiwitree? workerid?` | worker | `putjob` `:2333-2619` | Result upload (cpio). Detailed in 2.5. |
| `POST /workerdispatched $arch $job $jobid hostarch port proto? workerid?` | worker | `workerdispatched` `:1848-1881` | Used only for master-dispatched jobs: the worker tells the *owning* repserver that it started; creates `:status` with `code=building` via `lockcreatexml` and verifies `md5(jobfile) == jobid`. |
| `GET /getbinaries $project $repository $arch binaries nometa? metaonly? workerid? now? module* withannotation?` | worker (and ajax) | `getbinaries` `:441-507` | cpio of named binaries from the `:full` tree of the prpa (resolved via libsolv pool `setup_pool_with_repo`, `BSRepServer.pm:31-63`) plus their `.meta` files. Triggers DoD download (`BSRepServer::DoD::fetchdodbinary`) with handoff to ajax if a package is `d0d0d0d0...` placeholder. Containers are served as reconstructed tars (`Containertar::open_container`). |
| `GET /getbinaryversions ... binaries nometa? workerid? now? module* withevr?` | worker | `getbinaryversions` `:168-267` | Returns `$BSXML::binaryversionlist` (`name` with suffix, `hdrmd5`, `sizek`, `metamd5`, `evr`, `arch`). Used by the worker to decide cache hits before downloading. |
| `GET /getjobdata $arch $job $jobid workerid?` | worker | `getjobdata` `:2621-2639` | cpio of the existing `jobs/<arch>/<job>:dir` (used for `_delta`, `_ptf`, `_reproduciblecheck`, follow-up builds). Requires `:status` `code=building` and matching jobid. |
| `GET /getpackagebinaryversionlist $project $repository $arch $package* withcode? workerid?` | worker | `getpackagebinaryversionlist` `:269-308` | Per-package binary version list from `:bininfo` (used by `getbinaries_buildenv` / product builds). |
| `GET /badpackagebinaryversionlist ...` | worker | `badpackagebinaryversionlist` `:404-418` | Worker tells repserver that bininfo is stale -> deletes `.bininfo`/`:bininfo` and emits `scanprjbinaries`. |
| `GET /getpreinstallimageinfos $prpa+ match? workerid?` | worker | `getpreinstallimageinfos` `:1883-1915` | Returns Storable list of preinstall images whose 512-byte bitstring is a subset of the supplied `match` bitstring (POST body). See 2.7. |
| `GET /getobsgendiffdata $project $repository $arch jobid? workerid?` | GET=none | `getobsgendiffdata` `:1917-1926` | cpio of `*.obsgendiff` files from `:repo` (for `BuildFlags: obsgendiff`). |
| `GET /build/$project/$repository/$arch/$package[:_repository]` (`view=cpio|binaryversions|...`) | `!worker,rw` | `getbinarylist` `:1062-1245` | The worker uses `view=cpio` to fetch old packages (`getoldpackages`, `bs_worker:3183-3203`), product build inputs (`getbinaries_product`, `bs_worker:1690-2178`), containers (`view=cpio&binary=...&withannotation=1`, `bs_worker:2648-2661`), `view=binaryversions` for cache validation, `view=cpioheaderchksums` for `RPMHDRS_ONLY` product builds (`:1113-1131`). |
| `GET /build/$project/$repository/$arch/$package/$filename` | GET=none | `getbinary` `:1595-1625` | Single file download; `_preinstallimage.<md5>` is served from `.preinstallimage.<md5>` (`:1598-1605`); worker uses this for preinstall images (`bs_worker:2403-2411`), `_ccache.tar` of linked projects (`bs_worker:3163-3168`), and `getbinaries_buildenv` single-file fetches (`bs_worker:2655-2660`). |
| `GET /build/$project/$repository/$arch/_repository/$filename` | | `getbinary_repository` `:1523-1593` | Access-by-name into `:full` (with DoD fetch); `_buildconfig` returns the expanded project config from srcserver. |
| `GET /build/$project/$repository/$arch` `view=binarychecksums|rawbinarychecksums` | | `getbinarychecksums` `:348-369` | Worker fetches `.checksums` for product builds (`bs_worker:2130-2149`). |

Also used by the worker but on the **srcserver** (`bs_srcserver:8063-8085`):
`/getsources`, `/getconfig`, `/getsslcert`, `/getpubkey`, `/getmodulemd`,
`/getobsgendiffdata`, `/getbinaries`, `/getbinaryversions` (remote/interconnect),
`/source/<prj>/<pkg>/<file>?rev=` (follow-up `_buildenv`), `/build/<prpa>/...`
for remote repositories, and `/worker/<workerid>`.

#### 2.2.2 Consumed by the DISPATCHER (`bs_dispatch`) / master-dispatch peers

The dispatcher reads the filesystem directly for everything local. HTTP is used
only in the **master dispatcher** topology (`$BSConfig::masterdispatcher`):

| Route | Handler | Use |
|---|---|---|
| `GET /jobs`, `GET /jobs/$arch` | `listjobarchs`, `listjobs` `:4076-4092` | master: enumerate jobs |
| `PUT /jobs/$arch/$job` | `addjob` `:4094-4107` | slave dispatcher pushes its job XML (with `masterdispatched=<md5>` set) to the master (`bs_dispatch:506-521`); master writes `jobs/<arch>/<job>` and a `:cross` marker if `hostarch` differs. |
| `DELETE /jobs/$arch/$job` | `deljob` `:4122-4159` | slave removes jobs that vanished locally (`bs_dispatch:533-549`); if building, sends `/discard` to the worker. |
| `GET /jobs/$arch/$job view=status` | `getjob` `:4109-4120` | warden asks the slave for the status of a masterdispatched job (`bs_warden:155-163`). |
| `POST /jobs/$arch/$job cmd=idleworker workerid jobid?` | `idleworkerjob` `:4191-4206` | "the worker that had this job is idle again" -> unlink `:status` if it still says building on that worker. |
| `POST /jobs/$arch/$job cmd=setdispatchdetails details` | `setdispatchdetails` `:4208-4219` | emits `dispatchdetails` event to the scheduler. |
| `POST /jobs/$arch/$job cmd=fail message` | `failjob` `:4221-4257` | dispatcher fails a job without a worker (bad constraints): creates `:dir/logfile` with the message, `:status` `finished/failed/workerid=dispatcher`, notifies, emits `built`. |
| `POST /jobs/_mdload` | `postmdload` `:4161-4189` | slave uploads its load table every 120s (`bs_dispatch:565-595`). |
| `POST /event $type $project ...` | `forwardevent` `:2924-2991` | dispatcher forwards `badhost` events to the master (`bs_dispatch:368-379`, `641-642`). |
| `GET /worker/$workerid` | `getworkercap` `:4307-4317` | capabilities of a known worker (any state) minus ip/port/uri. |
| `POST /worker cmd=checkconstraints $project $repository $arch $package` (body: constraints XML) | `checkconstraints` `:4319-4380` | API helper: which registered workers satisfy prjconf+package constraints. |
| `PUT/GET /build/_dispatchprios` | `:4060-4074` | admin priority adjustments consumed by the dispatcher. |

#### 2.2.3 Consumed by the FRONTEND / API (`bs_srcserver` proxies most of these)

| Route | Handler | Notes |
|---|---|---|
| `/_result $prpa+ oldstate? package* code* lastbuild? withbinarylist? withstats? withinfo? withreproduciblecheck? summary? withversrel?` | `getresult` `:3052-3221` | Build results; `oldstate` long-poll is handed off to ajax with a file watcher on `:packstatus` (`:3054-3058`, `3211-3219`); `fixpackstatus` maps `scheduled` to the live jobstatus code (`dispatching`/`building on X`/`finished`/`signing`) by reading `jobs/<arch>/<job>:status` (`:2993-3050`). |
| `/build/$project/$repository/$arch/$package/_status`, `_jobstatus`, `_reason`, `_history`, `_buildstats`, `_log` | `:1268-1390` | `_log` streams from the **worker** (`$jobstatus->{uri}/logfile`) while building (`:1343-1360`), from `jobs/<arch>/<job>:dir/logfile` while finished/signing, else from the package dir. |
| `/build/$project/$repository/$arch/$package/_buildinfo add* internal? debug? deps?` and `POST ... _buildinfo` | `getbuildinfo`, `getbuildinfo_post` `:3537-3621` | see 2.6 |
| `/build/$project/$repository/$arch/_builddepinfo package* view?` | `getbuilddepinfo` `:3623-3709` | from `:depends` |
| `/build/$project/$repository/$arch/_jobhistory`, `/_jobhistory $prpa+` | `:3485-3534` | from `:jobhistory` |
| `/build/$project/$repository/$arch package* view=status\|binaryversions\|binaryversionscode\|gbininfo\|gbininfocode\|packstatus\|binarychecksums\|rawbinarychecksums\|binaryversionscookie withbininfocookie?` | `getpackagelist_build` `:386-401` | interconnect/scheduler views (Storable payloads) |
| `/build/$project/$repository/$arch/$package[:_repository] view=... binary* nometa? noajax? nosource? noimport? withmd5? module* withccache? withannotation? withevr? copybuild? aggregatemode?` | `getbinarylist` `:1062-1245` and `getbinarylist_repository` `:661-991` | `_repository` views: `solv`, `solvstate`, `cache` (+`:full.xcache`), `cpioheaders`, `cpio`, `binaryversions`, `availablebinaries`, `names`. `aggregatemode` implements aggregate expansion (`:1016-1060`). |
| `/build/$project/$repository/$arch/$package/$filename view=fileinfo\|fileinfo_ext withfilelist?` | `getbinary`/`getbinary_info` `:1403-1521` | `fileinfo_ext` builds a libsolv pool from the repo path to answer requires/provides-by. |
| `PUT /build/$project/$repository/$arch/_repository/$filename ignoreolder? wipe?`, `DELETE ...` | `putbinary`, `delbinary` `:1655-1732` | direct upload into `:full` (cpio or single file); emits `scanrepo`. |
| `POST /build/$project/$repository/$arch/$package cmd=copy ...` | `copybuild` `:2653-2873` | release/copy of build results: creates a fake `copy-<md5>` job with a `:dir` of hardlinks (`setrelease` renaming logic `:2690-2776`), emits `copybuild` event (to signer if `resign`). |
| `POST /build/$project/$repository/$arch/$package` | `uploadbuild` `:2875-2922` | upload a result dir as an `upload-<md5>` job; verifies `_blob.sha256:*`. |
| `POST /build/$project cmd=move oproject` | `moveproject` `:2641-2651` | rename on disk (unsafe while schedulers run). |
| `POST /build/$project/$repository/$arch/_repository match=` | `postrepo` `:3825-3858` | XPath over pool data |
| `POST /build/.../_repository cmd=missingdodresources resource* partition?` | `missingdodresources` `:1627-1634` | DoD needed-resource registration |
| `POST /_command $cmd $prpa+ package* code* lastbuild? sysrq? allarch?` | `docommand` `:3287-3404` | `rebuild` (unlink `:meta/<pkg>`, event), `killbuild`/`abortbuild` (-> worker `/kill`), `restartbuild` (-> worker `/discard` + unlink `:status`), `sendsysrq` (-> worker `/sysrq`), `wipe`, `wipeallarch`, `unpublish`, `wipepublishedlocked`, `force_publish`, `availablebinaries`. |
| `/workerstatus daemonsonly? arch* type*` | `workerstatus` `:1992-2145` | see 2.4 |
| `/published...` | `publisheddir`, `publishedfile`, `published_status` `:3880-4006` | published repo browsing + YMP generation |
| `GET\|HEAD /registry/...` | `registry_*` `:4382-4647` | OCI/Docker distribution API, see 2.8 |
| `/slsa/...`, `POST /slsa cmd=addrefs` | `:4649-4690` | see 2.9 |
| `PUT/GET /configuration` | `:4259-4280` | global config; emits `configuration` event to all schedulers + publisher |
| `/build/$project/$repository/$arch/_relsync` GET/POST | `:4008-4058` | partition relsync |
| `/serverstatus`, `/ajaxstatus` | | introspection |

#### 2.2.4 Consumed by the SCHEDULER (via filesystem, not HTTP)

The scheduler never calls the repserver over HTTP; it shares `$bsdir`. The repserver
*serves scheduler-produced state* and *pokes the scheduler with events*.
`forwardevent` (`:2924-2991`) is the only "notify scheduler" path; there is no
"lastevents" long-poll between repserver and scheduler (the `lastevents` route lives in
`bs_srcserver:8198-8199` for API/interconnect consumers). The srcserver, in turn, pushes
events to the repserver via `POST /event` (`:4713`, `newevent` in
`bs_srcserver:7990-7998`).

#### 2.2.5 Consumed by OTHER REPSERVERS / PARTITIONS (interconnect)

* `BSRepServer::Remote::addrepo_remote` (`BSRepServer/Remote.pm:36-79`) fetches a
  remote repo state with `GET /build/<prj>/<repo>/<arch>/_repository?view=cache`
  (or `view=solvstate&noajax=1` for partitions) and
  `read_gbininfo_remote` uses `view=gbininfo` / `view=binaryversions` (`:81-118`).
* Partitioning (`$BSConfig::partition`, `$BSConfig::partitioning`,
  `$BSConfig::partitionservers`, `BSConfig.pm.template:233-250`): each partition has
  its own repserver/schedulers/dispatcher/warden/publisher; the single srcserver
  routes. Job `path` entries carry a per-repo `server` so the worker fetches binaries
  from the right partition (`BSSched/BuildJob.pm:1356-1385` `path2buildinfopath`,
  `$BSConfig::workerpartitionservers`).
* `handoff_part('interconnect_in', ...)` is used for `view=cache|cpio|solvstate`
  so that interconnect traffic goes through a dedicated ajax partition
  (`bs_repserver:667-669`, `1067-1070`).
* `BSUrlmapper::urlmapper` / `get_downloadurl` map prp to public download URLs
  for `_buildinfo` and YMP (`BSRepServer/BuildInfo.pm:163-175`).

### 2.3 Worker registration and state (repserver view)

`workerstate` (`bs_repserver:1752-1846`):

1. `idlename = "$harch:$workerid"`; `workerid` defaults to `"$peerip:$peerport"` (`:1763`).
2. The body (if any) is the worker's capability XML (`$BSXML::worker`,
   `BSXML.pm:889-926`). `job`/`arch` are stripped; `hardware/nativeonly` is normalised
   to a present-but-empty element (`:1767`).
3. `state=building`: move to `away` (another repserver got the job).
   `state=exit`: move to `down`.
   `state=idle`: crash recovery as described, then the record
   `{hostarch, ip, port, uri, workerid, tellnojob, ...capabilities}` is written to
   `workers/idle/<idlename>` (`:1795-1836`).

The `uri` is **computed from the TCP peer address**, not from what the worker claims
(`:1800`). This matters for NAT/containers: the dispatcher will later connect to
`uri/build`.

Example of a worker registration file (`workers/idle/x86_64:lamb55:1`), assembled from
`bs_worker:788-902` and `bs_repserver:1795-1805`:

```xml
<worker hostarch="x86_64" ip="10.160.0.55" port="34567"
        uri="http://10.160.0.55:34567" registerserver="http://backend:5252"
        workerid="lamb55:1" tellnojob="">
  <hostlabel>OBS_WORKER_SECURITY_LEVEL_1</hostlabel>
  <sandbox>kvm</sandbox>
  <linux>
    <version>6.4.0</version>
    <flavor>default</flavor>
  </linux>
  <hardware>
    <cpu>
      <flag>fpu</flag> ... <flag>avx2</flag> <flag>x86-64-v2</flag> <flag>x86-64-v3</flag>
    </cpu>
    <processors>16</processors>
    <jobs>4</jobs>
    <memory>8192</memory>
    <swap>1024</swap>
    <disk>12288</disk>
  </hardware>
  <owner>build-admins</owner>
</worker>
```

(`registerserver` is the repserver URL the worker sent to, `bs_worker:975`.)

### 2.4 `/workerstatus`

`workerstatus` (`bs_repserver:1992-2145`) composes `$BSXML::workerstatus`
(`BSXML.pm:974-1031`):

* `idle`/`down`/`dead`/`away` lists by reading each `workers/<state>/*` file
  (`uri` only shown for idle).
* `building`: for every `jobs/<arch>/<job>:status` with `code=building`, an entry with
  `workerid, uri, hostarch, project, repository, package, arch, starttime`
  (`:2024-2053`). Project/repo/package are parsed from the job *name* when it has the
  `prj::repo::pkg-md5` form; otherwise the job XML is read.
* `waiting`: count of unlocked jobs per arch (only on the master dispatcher host).
* `blocked`, `buildavg`: from `info/schedulerinfo.<arch>` (`$BSXML::schedulerinfo`).
* `partition/daemon`: liveness of each daemon by trying `flock(LOCK_EX|LOCK_NB)` on
  `run/bs_sched.<arch>.lock`, `bs_dispatch.lock`, `bs_publish.lock`, `bs_signer.lock`,
  `bs_warden.lock`, `bs_dodup.lock` (`:2061-2130`).

Example:

```xml
<workerstatus clients="3">
  <idle uri="http://10.160.0.55:34567" workerid="lamb55:1" hostarch="x86_64"/>
  <building uri="http://10.160.0.56:34568" workerid="lamb56:1" hostarch="x86_64"
            project="openSUSE:Factory" repository="standard" package="glibc"
            arch="x86_64" starttime="1729500000"/>
  <down workerid="lamb57:1" hostarch="x86_64"/>
  <waiting arch="x86_64" jobs="1234"/>
  <blocked arch="x86_64" jobs="56"/>
  <buildavg arch="x86_64" buildavg="412"/>
  <partition>
    <daemon type="scheduler" arch="x86_64" state="running" starttime="1729400000">
      <queue high="0" med="3" low="120" next="12"/>
    </daemon>
    <daemon type="repserver" state="running" starttime="1729400000"/>
    <daemon type="dispatcher" state="running" starttime="1729400001"/>
    <daemon type="publisher" state="running" starttime="1729400002"/>
    <daemon type="warden" state="running" starttime="1729400003"/>
  </partition>
</workerstatus>
```

### 2.5 `putjob` -- result ingestion (the most important repserver handler)

`bs_repserver:2333-2619`. Step by step:

1. Preconditions: `jobs/<arch>/<job>` exists, `:status` exists, `lockopenxml` the
   status, `jobid` matches, `code eq 'building'`, `workerid` matches if supplied
   (`:2336-2341`). Optional peer-IP check (`:2342-2349`).
2. `md5(jobfile) eq jobid` (`:2351-2353`) -- the jobid *is* the MD5 of the job XML
   bytes, computed by the dispatcher at assignment (`bs_dispatch:259`) and by the
   worker at receipt (`bs_worker:4683`).
3. Move the worker record `workers/building/<idlename>` -> `workers/down/` (`:2361-2366`)
   (the worker will re-register as idle shortly).
4. `code=badhost`: emit `badhost` event to the dispatcher (`sendbadhostevent`), unlink
   `:status` (job becomes dispatchable again), stash the logfile in
   `.logfile.badhost/<job>`, return (`:2368-2395`).
5. Clock sanity: `now` must be within +-3600s of server time, otherwise treated as
   badhost and `die("time mismatch")` (`:2398-2404`).
6. Release the lock, read the cpio into `jobs/<arch>/.putjob.<pid>`
   (`BSServer::read_cpio`, `:2409-2420`).
7. Validate: `meta` lines must be `^[0-9a-f]{32}  .+` (`:2423-2441`); no
   `*.obsbinlnk`, `*.obsbininfo`, `_blob.*` files (`:2444-2458`).
8. Containers: for every `X.containerinfo` with `X.tar` present, verify the JSON and
   **normalize** the tar (`BSRepServer::Containertar::normalize_container`,
   `BSRepServer/Containertar.pm:33-94`): re-pack layers deterministically, split into
   `_blob.sha256:<digest>` files, delete the tar (`$deletetar=1`), and rewrite the
   containerinfo with `tar_manifest`, `tar_md5sum`, `tar_sha256sum`, `tar_mtime`,
   `tar_size`, `tar_blobids`, `imageid`, `goarch`, `goos`, `govariant`
   (`bs_repserver:2466-2492`). SBOM (`.spdx.json`, `.cdx.json`), `.intoto.json` and
   `_slsa_provenance.json` are parsed as JSON and must be <256MB (`:2494-2504`).
   Any failure turns the result into `failed` with only `meta`+`logfile` kept.
9. Re-acquire lock, re-check jobid/workerid/starttime, then
   `rename(.putjob.<pid>, <job>:dir)` (`:2515-2546`). `kiwitree=1`: before that,
   `receivekiwitree` reconstructs a directory tree from the `.kiwitree` manifest
   (`:2195-2300`): entries `d <dir>`, `l <link> <target>`, `f <file> [<leadsigmd5|md5>]`;
   files whose `leadsigmd5` is found in any input package's bininfo are **hardlinked**
   from the repo tree (`receivekiwitree_scan`, `:2164-2193`), the rest are fetched from
   the worker with `POST <uri>/kiwitree file=...` (`:2283-2298`).
10. Result code: `succeeded|failed|unchanged|genbuildreqs` (`:2547-2558`).
11. `notify_jobresult` -> `BSNotify` `BUILD_SUCCESS|BUILD_FAIL|BUILD_UNCHANGED`
    (`:2302-2331`).
12. For each `*.containerinfo` create `*.obsbinlnk` (`containerinfo2obsbinlnk`,
    `BSRepServer/Containerinfo.pm:90-144`): a Storable "package-like" record
    (`name=container:<name>`, version, release, arch, provides incl. tags, `hdrmd5` =
    tar md5, `path=../<pkg>/<file>`, annotation XML) so containers can be
    dependencies in `:full`.
13. `BSRepserver::Bininfo::create_bininfo_file($dir)` writes `.bininfo`
    (`BSRepServer/Bininfo.pm:30-35`).
14. Append `prj|repo|arch|pkg|start|end|result|workerid|hostarch` to `jobs/finished`
    for the dispatcher's load model (`:2541-2544`).
15. Decide `signing` vs `finished`: if `$BSConfig::sign` and the upload contains
    signable artifacts (`.rpm/.drpm/.sha256/.sha512/.iso/.pkg.tar.*/.AppImage/.deb/
    .appx/.apk/.helminfo/.raw/.efi/.roothash/.usrhash` or `_slsa_provenance.json` or
    kiwitree `.asc/.key/.p7s`) -> `code=signing` and event `finished:<arch>:<job>` into
    `events/signer/`; else `code=finished` and event `finished:<job>` into
    `events/<arch>/` (`:2588-2616`). `_reproduciblecheck` never signs.
16. `dirty(prj,repo,arch)` touches `:schedulerstate.dirty` unless signing (`:2605`).

### 2.6 `_buildinfo` generation (`BSRepServer/BuildInfo.pm`, `BSRepServer/Checker.pm`)

`getbuildinfo` (`bs_repserver:3537-3545`) and the POST variants (`:3547-3621`,
used by `osc build` which uploads the local spec, optional `buildenv`, `deps`
("ldepfile") and `_service` as a cpio) call `BSRepServer::BuildInfo::buildinfo`
(`BuildInfo.pm:44-129`). That function **re-uses the scheduler's job-creation code
in-process** instead of reimplementing it:

* Creates a fake global context `$gctx` with `reporoot`, `arch`, empty projpacks.
* Fetches project/package metadata from the srcserver with
  `/getprojpack?withdeps=1&buildinfo=1` (`BSRepServer/ProjPacks.pm:28-49`).
* Builds a `BSRepServer::Checker` (`Checker.pm:50-57`) which subclasses the scheduler
  checker API (`setup`, `preparepool`, `buildinfo`, `addrepo`, `read_gbininfo`,
  `writejob`) with `isreposerver=1`, so `BSSched::BuildJob::Package::check` short-cuts
  (`Package.pm:92`) and `BSSched::BuildJob::create` runs with `dobuildinfo` set
  (`BSSched/BuildJob.pm:1065-1338`). `writejob` just stores the result in
  `$ctx->{buildinfo}` (`Checker.pm:303-307`).
* Expansion uses `BSSolv::expander` over pools built with
  `BSRepServer::addrepo_scan` (local `:full`/`:full.solv`) or
  `BSRepServer::Remote::addrepo_remote` (`Checker.pm:259-272`).
* `fixupbuildinfo` (`BuildInfo.pm:189-211`) strips internal fields
  (`job needed constraintsmd5 prjconfconstraint nounchanged revtime reason nodbgpkgs
  nosrcpkgs`), folds `syspath`/`containerpath` into `path`, replaces `server` by
  public `url`s (`addurltopath`), restores `subpack` from the parsed recipe, sets
  `downloadurl`, selects a preinstall image (`addpreinstallimg`, `:131-161`) and
  attaches `modularity_yaml`.
* `add=` installs extra packages (`extradeps`), `debug=1` attaches the libsolv
  expansion trace (`expanddebug`), `internal=1` returns the raw job data.

The resulting document is the same `$BSXML::buildinfo` schema that is written to
`jobs/<arch>/<job>` -- the job file is literally a buildinfo (`BSXML.pm:563-693`).

### 2.7 Preinstall images

A **preinstall image** is a tarball of an already-unpacked build root for a set of
packages. It is produced by a normal build job for a package whose recipe is
`_preinstallimage` (`build/build-recipe-preinstallimage:33-74`; recipe detection
`build/build-recipe:131`). `init_buildsystem` records `<hdrmd5> <pkgid>` for each
installed package in `.preinstall_image/<pkg>` when `CREATE_PREINSTALL_DATA` is set
(`build/init_buildsystem:885-889`, `1110-1112`, `1136`, `1162`). The recipe tars the
root (excluding `.build*`, `.init_b_cache`, `.pkgs`, `proc`, `sys`...) to
`OTHER/preinstallimage.tar` + `OTHER/preinstallimage.info` (lines `<hdrmd5>  <pkg>`),
and `preinstallimage_compress` makes it `.tar.zst` (or `.gz`) (`:83-101`;
called from `build/build:2003` or `build-vm:1143`).

On ingest the scheduler (`BSSched/BuildJob/PreInstallImage.pm:131-189`):

* hardlinks the tar to `<pkg>/.preinstallimage.<id>` where
  `id = md5("$info/$mtime/$size/$inode")`,
* computes a 512-byte **bitstring**: bit `hex(first 3 hex chars of hdrmd5)` set for
  each contained package (`:160-166`),
* appends `{package, hdrmd5=>id, file, sizek, bitstring, hdrmd5s=>[...], bins=>[...]}`
  to `build/<prpa>/:preinstallimages`.

Selection happens on the **worker** (`bs_worker:2262-2467` `getpreinstallimage`):

1. Call `/getbinaryversions?withevr=1` for every `path` repo to learn the `hdrmd5`
   of every needed binary (`:2277-2316`); the `prpa`s that are on a repserver
   (not the srcserver) are candidates (`:2297`).
2. Build its own 512-byte bitstring from the needed hdrmd5s (`:2329-2334`) and
   `POST /getpreinstallimageinfos?match=body&prpa=...` to each server; the repserver
   returns images whose bitstring is a subset (`bs_repserver:1883-1915`; bitwise
   `($img->{bitstring} & $match) ne $img->{bitstring}` test at `:1900`).
3. Pick the image with the most packages such that **all** its hdrmd5s are in the
   needed set (`:2343-2360`); never its own package; for `_preinstallimage` builds at
   least one package must be new (`:2352-2356`).
4. Fetch the image via the worker cache, via the getbinaries proxy
   (`/getpreinstallimage`, `bs_getbinariesproxy:408-472`) or
   `GET <server>/build/<prpa>/<pkg>/_preinstallimage.<id>` (`:2365-2411`).
5. Fetch the `.meta` files of the packages in the image if this is the local
   project (`getpreinstallimage_metas`, `:2180-2260`) -- metas are needed to compute
   the build's dependency fingerprint even for packages that are never downloaded.
6. Return `(imagefile, imagebins, imagesource, metas, imageorigins)`; the rpmlist gets
   `preinstallimage: <path>` and `<bin> preinstallimage` lines
   (`bs_worker:4169-4191`), which `init_buildsystem` consumes
   (`build/init_buildsystem:738-749`, `768-772`, `906-909`: unpack the image with
   bsdtar and skip preinstall of contained packages).

Also disabled: for SLSA provenance jobs (`bs_worker:2266`) and for recipe types that
need all binaries (`needsbinariesforbuild`, `:4034-4038`).

### 2.8 Container registry endpoints (`/registry`)

Served by `registry_*` (`bs_repserver:4382-4647`) from `$bsdir/registry/<regrepo>/`
written by the publisher (`BSPublisher/Registry.pm`, out of scope):
`:tags/<tag>`, `:manifests/<sha256:...>`, `:blobs/<sha256:...>`, `:info`, `:sigs`,
`:cosign`, `:referrers/<subject>`, `:referrerinfo`, `:manifestinfos/`, `:tuf`,
`:tuf.old`, and `$registrydir/:repos`. The API covers `GET/HEAD /registry/<name>/
manifests/<tag|digest>` (with `Accept` negotiation and `select_manifest` for fat
manifests, `BSRepServer/Registry.pm:44-51`), `blobs/<digest>` (with `Range`),
`tags/list` (paginated), `_catalog`, cosign `signatures/`, OCI `referrers/`,
`manifestinfos/`, notary TUF `_trust/tuf/*.json|*.key` with timestamp re-signing
(`extend_timestamp`, `Registry.pm:53-80`). **Workers do not use these endpoints**;
workers get base containers through `/getbinaries` as tar files reconstructed from
`_blob.*` files (`BSRepServer::Containertar::open_container`, `Containertar.pm:149-166`)
or as DoD containers pulled from an upstream registry by the repserver
(`BSRepServer::DoD::fetchdodcontainer`, `DoD.pm:97-124`, blob download in
`Registry.pm:109-141` using `BSBearer` auth and the shared `blobs/sha256` store).

### 2.9 SLSA provenance support

`BSRepServer/SLSA.pm`: `POST /slsa cmd=addrefs $prpa` (`bs_repserver:4649-4657`)
is called by the scheduler/signer after a provenance-enabled build to record which
`prpa`s reference which input digests; `link_binaries` hardlinks the exact input
binaries (by sha256) into `$bsdir/slsa/<prpa>/<2>/<digest>` (+ `.prov`) and a
per-prpa sqlite `refs` table (`SLSA.pm:61-69`, `139-170`, `172-186`). This is what
makes the "all used binaries are kept" promise (`BSConfig.pm.template:137-142`).
`GET /slsa/<prpa>/<filename>/<digest>[?view=refs]` serves them (`:4659-4690`).
The worker generates the provenance statement itself (3.5.7).

### 2.10 YMP

`BSRepServer/YMP.pm:makeymp` renders a one-click-install XML for a published binary
by querying the binary (`Build::query`), the project path (via `/getprojpack
withconfig`) and `PublishFlags: ympdist:` (`YMP.pm:90-159`); served from
`/published/<prj>/<repo>/<arch>/<file>?view=ymp` (`bs_repserver:3900-3920`,
`3976-3984`).

---

## 3. Dispatcher (`bs_dispatch`)

A single-process, single-threaded loop (`bs_dispatch:826-1476`) with no HTTP server.
It reads/writes only the shared filesystem and makes outbound RPCs to workers
(`/build`), to the repserver (`/jobs/...`, `/event`), and to the srcserver
(`/source/<prj>/<pkg>/_constraints`, `/jobs/<arch>?partition=`).

### 3.1 Startup

* Lock `run/bs_dispatch.lock` (`:797-800`); restore `%infocache`, `%badhost`,
  `%newestsrcchange` from `run/bs_dispatch.state` if a previous instance dumped it
  (`:808-815`; dumped on `.exit`/`.restart`/`.dumpstate` control files in `filechecks`
  `:666-710`).
* `%cando` / `%harchcando` from `BSCando::cando` (`:159-172`): `harchcando{"$harch/$arch"}`
  is `''` for native or the helper name (`linux32`, `linux64`, `mips32`) for
  personality-switched builds (`BSCando.pm:32-74`).
* `%secure_sandboxes` = `$BSConfig::secure_sandboxes` or `{xen,kvm,zvm}` (`:107-114`).
* `%powerpkgs` from `$BSConfig::powerpkgs` (ordered list of package names that should
  go to `$BSConfig::powerhosts`) (`:97-103`).
* Option `--test-constraints <pkg> <arch> <workerinfo.xml> [<constraints.xml>
  [<prjconf-constraints.txt>]]` runs the oracle once and exits (`:122-153`).

### 3.2 Main loop, in order (`:826-1476`)

**(a) Load accounting** (`:828-860`). If `jobs/finished` is non-empty: unlink it,
parse `prj|repo|arch|pkg|start|end|result|workerid|hostarch` lines, and update
`jobs/load` (Storable `{prpa => [lastend, load]}`) with an exponential decay model:
`$decay = log(.5)/(4*3600)` (`:63`): a 4h build adds 0.5 to the prpa load, 4h of
idle halves it (`:845-855`). Entries older than 50 days are pruned.

**(b) Slave mode** (`$BSConfig::masterdispatcher` set and != own reposerver, `:862-868`):
run `dispatchslave` (`:454-597`): for every local `jobs/<arch>/<job>` without `:status`
(and `file ne '_aggregate'`), `PUT` the job to the master's `/jobs/<arch>/<job>` with
`masterdispatched=<md5 of original xml>` injected (`:498-521`); `DELETE` jobs that are
gone; every 600s re-sync with the srcserver's `/jobs/<arch>?partition=` list; every
120s upload load to `/jobs/_mdload`. Then `forwardevents`, sleep 1 if nothing synced,
`next`. (The master then dispatches and the worker reports back to the slave's
repserver through `reposerver` in the job -- see `assignjob` and `/workerdispatched`.)

**(c) Priorities** (`:869-897`): `jobs/dispatchprios` (`$BSXML::dispatchprios`,
`BSXML.pm:1611-1619`: list of `{project?, repository?, arch?, adjust}`) is reloaded
when its `mtime/size/inode` changes and indexed per project (`':all:'` for global).

**(d) Current load** (`:899-933`): decay every `jobs/load` entry to "now"; merge
`jobs/mdload` from slaves (`[lastend, load, buildingtime, nbuilding]`) and
`%masterdispatched` (jobs this master handed out that the slave hasn't reported yet).

**(e) Worker inventory** (`:944-967`): `workers/building/*` -> `%workerload{host}{idle}=1`,
`%numbuilding{harch}`; `workers/idle/*` -> `%workerload{host}{idle}=0`,
`%numidle{harch}`, and `%idlearch{arch} = [idle workers whose harch can build arch]`
via `%cando`. "host" is the idlename with the trailing `:<instance>` stripped
(`:949,958`), so `lamb55:1`..`lamb55:8` are one host.

**(f) Job inventory** (`:968-1066`): for each arch with idle workers (plus arches
reachable through `:cross` markers in `jobs/<hostarch>/`, `:987-1004`): list
`jobs/<arch>`, split into locked (`*:status`) and free jobs; **locked jobs add +1 to
their prpa's load** (`:1012-1030`), free jobs are collected in `%jobs{prpa}`. Job infos
are cached in `%infocache{arch}{job}` with `bdep`/`subpack` stripped to save memory
(`:1182-1186`); for the common `prj::repo::pkg-md5` name shape the info is
synthesised without reading the file (`:1038-1041`). `%maybesrcchange{prpa}` is set if
any job's `reason` is `new build`/`source change` within 24h of `revtime`
(`:1047-1055`).

**(g) Scale factors** (`:1095-1126`): per prpa `$scales{prpa} = exp(-adj*ln(10)/10)`
where `adj` is the sum of matching `$BSConfig::dispatch_adjust` regex adjustments and
the last matching `dispatchprios` entry (clamped to +-10000). An adjust of +10 makes
the prpa look 10x less loaded.

**(h) Project load smoothing** (`:1081-1093`): load per `project/arch` ("pra") is the
sum over its repos; each prpa with jobs gets `(load_prpa + load_pra)/2`, with a random
`rand(.01)` floor so ties are broken randomly.

**(i) Staleness bonus / source-change bias** (`:1133-1143`, `:381-402`): prpas whose
jobs are *not* source changes get `load *= 3` (`$nosrcchangescale`, `:95`) plus
`staleness = (lastbuild - newestsrcchange)/((now-lastbuild)*40 + 5e6)`; i.e.
rebuild-storms (no source change) are deprioritised against fresh commits.

**(j) Ordering**: `@jobprpas` sorted by `scale*load` ascending (`:1145`).

**(k) Assignment loop** over prpas (`:1150-1450`). For each prpa:

1. Skip if no idle worker can build this arch (direct or via cross) (`:1153-1156`).
2. Load job infos; jobs with `reason` new build/source change go first
   (`@srcchange`, `:1192-1198`); then shuffle, then stable-sort by `needed` desc,
   `readytime` asc (`:1201-1202`). `needed` is the number of packages whose expanded
   deps include this package's source (`BSSched/Checker.pm:1862-1871`, carried in
   the job as `<needed>`), i.e. "how many builds are blocked on me".
3. `powerpkgs` are moved to the front in configured order (`:1204-1218`);
   `_preinstallimage` jobs are moved to the front (`:1225-1232`).
4. For each job (`:1236-1440`):
   * if this is the first non-source-change job of the prpa, apply the `*3 +
     staleness` penalty now; if that pushes `scale*load` above the next prpa's,
     `rerun` (re-insert prpa into the sorted queue and continue with the next prpa)
     (`:1239-1247`, `:1441-1448`).
   * candidate workers: `shuffle(@{$idlearch{$info->{'hostarch'} || $arch}})`
     (`:1248`); remove workers marked `nativeonly` for this arch (`:1250-1254`); if
     more than one, **sort by host idleness** (fraction of the host's instances that
     are idle, descending) (`:1255-1269`) -- spreads load across physical hosts.
     **There is no cache-locality preference** (the worker's cache is not known
     to the dispatcher); the only locality-ish signal is per-host spreading.
   * powerjobs: restrict to `$BSConfig::powerhosts` prefixes (`:1271-1283`).
   * **constraints** (`:1289-1332`): three sources merged with
     `BSDispatcher::Constraints::mergeconstraints`:
     1. package `_constraints` file: `constraintsmd5` in the job; fetched from
        `srcserver/source/<prj>/<pkg>/_constraints?expand=1&rev=<srcmd5>&noajax=1`
        and cached by md5 (`getconstraints`, `:404-430`; a fetch error is retried
        after 600s, a parse error fails the job); `overwrite` sections with
        matching `conditions/arch|package` are applied (`overwriteconstraints`,
        `Constraints.pm:179-199`).
     2. recipe constraints (`<constraint>` lines from the spec's `#!BuildConstraint:`
        hints, carried as `info->{constraint}`), parsed with `list2struct`
        (`Constraints.pm:202-290`: `hardware:memory:size 4096` -> nested hash;
        `+`/`=`/`!` suffixes add/delete/modify).
     3. prjconf `Constraint:` lines (`prjconfconstraint`), same parser. Merge order:
        prjconf, then package, then recipe (`:1310-1331`). Bad constraint syntax
        -> `failjob` with a message (`:1297-1300`, `1315-1319`).
   * For each candidate worker (`:1335-1409`):
     - skip if `%badhost` has `prj/pkg/arch/idle` or `prj/:repo:repo/arch/idle`
       (`:1341-1344`);
     - read/cached `workers/idle/<idle>` (`:1346-1356`); vanished -> drop from
       `%idlearch`;
     - optional site hook `$BSConfig::dispatch_constraint->($info,$worker,$constraints)`
       (`:1358-1360`, `BSConfig.pm.template:265-271`);
     - if the job needs a personality helper (`harchcando{"$harch/$arch"}` non-empty,
       e.g. i586 on x86_64, armv7l on aarch64) and the worker has
       `hardware/nativeonly`, skip and remember in `%nativeonlycache`
       (`:1362-1367`); `nativeonly` is set by the worker for aarch64 CPUs without
       32-bit EL0 support (`bs_worker:832-872`);
     - **oracle** (`Constraints.pm:77-134`): returns 1 iff all of: every
       `hostlabel` present (or absent if `exclude=true`); `sandbox` equals
       (`secure` matches any of `%secure_sandboxes`); `linux/flavor` equal,
       `linux/version` within `min..max` (`Build::Rpm::verscmp`); `hardware/
       processors`, `jobs`, `disk` (MB), `memory` (<= memory+swap), `memoryperjob*jobs`,
       `physicalmemory` (<= memory), each `cpu/flag` present/absent. Sizes are
       normalised to MB by `getmbsize` (`:53-66`, units B/K/M/G/T/P). (The code has a
       "fractional oracle" path for `0<ora<1` with `__lastoracle` retry that waits
       for a better-fitting worker up to a minute of worker idleness / 5 minutes of
       job wait, `:1333-1340`, `:1368-1386`; the current oracle only returns 0/1.)
     - `assignjob($job,$idle,$arch)` (`:237-365`), max 5 tries per job once something
       was assigned in this pass (`:1392-1393`).
   * If nothing could be assigned and constraints/badhost were involved, every 5th
     attempt and at most every 300s run `checkconstraints` (`:712-788`) over
     **all** registered workers (idle, building, away, down): if fewer than half (or
     <=5) comply, set the scheduler-visible dispatch details
     ("waiting for N compliant workers (M of them down) (K of them bad)") via
     `setdispatchdetails`; if **none** comply, `failjob` with a hint listing the
     offending constraint parts (`:751-787`). Success disables the check for 4h.
   * Stop the pass after 50 assignments if it already took >60s (`:1449`).

**(l) Housekeeping**: `forwardevents()` (`:613-664`: forward `events/repository/*` to
the srcserver; process `events/dispatch/*`: `badhost` events populate `%badhost` with
three keys -- `prj/pkg/arch/worker`, `arch/job`, `arch/job/worker` -- and append a
30s "badhost" pseudo-build to `jobs/finished` (`:600-609`)); sleep 1s if nothing was
assigned (`:1454`); expire `%badhost` entries after 24h (`:1456-1464`); persist to
`run/dispatch.badhosts`; `filechecks()`. Debug snapshot of `load/scales/jobs/powerpkgs`
is written to `run/dispatch.data` each pass (`:1128-1138`).

### 3.3 `assignjob` -- the hand-off (`:237-365`)

1. Write `jobs/<arch>/<job>:status` = `<jobstatus><code>dispatching</code></jobstatus>`
   with `BSUtil::lockcreatexml` (`:241-247`): writes a temp file, flocks it, `link`s
   it to the final name -- `link` fails if the status already exists, so this is the
   atomic claim. If it fails -> `badjob` (another dispatcher/scheduler owns it).
2. Re-read the job file (gone -> unlink status, `badjob`); `jobid = md5_hex(infoxml)`
   (`:250-260`).
3. Every 20s recompute `workercode`/`buildcode` = md5 over the sorted list of
   `md5  filename` of all files in `worker/` and `build/` (+`Build/`, `emulator/`)
   (`getcodemd5`, `:175-210`, `:262-267`).
4. Read `workers/idle/<idlename>` (gone -> `undef`, meaning "try another worker").
5. Build args `port=<repserver port> workercode=<md5> buildcode=<md5>
   [registerserver=<worker's registerserver>] [nobadhost=<msg>]` -- `nobadhost` is set
   when `%badhost` shows the job already failed on `>5+20/nhosts` attempts
   (`checkbadhost`, `:224-235`, `:280-288`): the worker is then told to report
   `failed` instead of `badhost` so the job stops bouncing.
6. `PUT <worker uri>/build?<args>` with the raw job XML as body, timeout
   `10 + len/100000` seconds (`:289-300`).
7. Error handling (`:301-325`): worker replies `cannot build anything` -> drop
   worker from this pass; `cannot build this repository` / `cannot build this
   package` (from the worker's `--hostcheck` script, exit 2/other) -> record
   `%badhost` and return `badhost`; `bad job` -> `badjob`; any other error ->
   **move worker to `workers/down/`** and unlink status.
8. On success: status becomes `{code=>building, uri, workerid, starttime, hostarch,
   jobid, attempt}` (`:328-335`); worker record gains `job, jobid, arch[, reposerver]`
   and is moved `idle` -> `building` (`:338-349`); redis job-status notification.
   For masterdispatched jobs the master deletes its copy of the job and status
   (`:351-358`) -- the slave's repserver keeps the real one, which the worker will
   update through `/workerdispatched`.

Example `jobs/x86_64/openSUSE::Factory::standard::glibc-7a1b...:status` while building:

```xml
<jobstatus code="building">
  <uri>http://10.160.0.56:34568</uri>
  <starttime>1729500000</starttime>
  <workerid>lamb56:1</workerid>
  <hostarch>x86_64</hostarch>
  <jobid>d41d8cd98f00b204e9800998ecf8427e</jobid>
</jobstatus>
```

(`code` cycles: `dispatching` -> `building` -> `finished` | `signing` -> (scheduler
deletes). Scheduler-side values `deleting` (`BSSched/BuildJob.pm:169`) and fake
`finished` (`:721-727`) also exist. `$BSXML::jobstatus`, `BSXML.pm:696-714`.)

### 3.4 Example `_constraints` XML (`$BSXML::constraints`, `BSXML.pm:1895-1942`)

```xml
<constraints>
  <hostlabel>OBS_WORKER_SECURITY_LEVEL_1</hostlabel>
  <sandbox>secure</sandbox>
  <linux>
    <version><min>5.14</min></version>
    <flavor>default</flavor>
  </linux>
  <hardware>
    <cpu><flag>avx2</flag><flag exclude="true">hypervisor</flag></cpu>
    <processors>4</processors>
    <jobs>4</jobs>
    <disk><size unit="G">20</size></disk>
    <memory><size unit="M">8192</size></memory>
    <physicalmemory><size unit="G">4</size></physicalmemory>
  </hardware>
  <overwrite>
    <conditions><arch>aarch64</arch><package>kernel-default</package></conditions>
    <hardware><memory><size unit="G">16</size></memory></hardware>
  </overwrite>
</constraints>
```

Equivalent prjconf form (`Constraint:` lines parsed by `list2struct`):

```
Constraint: sandbox secure
Constraint: hardware:memory:size unit=G 8
Constraint: hardware:cpu:flag avx2
Constraint: hostlabel OBS_WORKER_SECURITY_LEVEL_1
```

### 3.5 Job states summary (dispatcher + repserver + warden + scheduler)

| Filesystem fact | Meaning |
|---|---|
| `jobs/<arch>/<job>` exists, no `:status` | ready, dispatchable |
| `:status code=dispatching` | dispatcher holds the flock during the worker RPC (transient) |
| `:status code=building` | assigned; `uri`/`workerid`/`jobid` recorded; worker file in `workers/building/` |
| `:status` unlinked by repserver (`workerstate` crash path, `putjob badhost`, `idleworkerjob`), warden, or `restartbuild` | re-dispatch |
| `:status code=finished` + `:dir` | waiting for scheduler `event_built` |
| `:status code=signing` + `:dir` | waiting for bs_signer, which then re-emits `built` |
| `:status code=deleting` | scheduler is purging |
| job file removed | scheduler `purgejob` (`BSSched/BuildJob.pm:145-160`) after ingest / when the source changed (`kill_otherjobs`, `:371-385`, which sends `/discard` to the worker) |

### 3.6 Warden (`bs_warden`)

Loop every 5s (`bs_warden:126-309`):

* For every `workers/building/<w>` (checked at a random interval of up to 1h per
  worker, `:138`): read the worker file; read the job status (locally or via
  `GET <reposerver>/jobs/<arch>/<job>?view=status` if masterdispatched); if
  `code=building` and `workerid` matches, `GET <uri>/worker?jobid=` (`:160-170`).
  On RPC failure: lock the status, re-verify jobid/workerid, **unlink `:status`**
  (`:188-202`), move the worker file to `workers/down/`. For masterdispatched jobs it
  POSTs `cmd=idleworker` to the owning repserver instead (`:172-186`).
* Every 60s orphan scan (`:207-305`): jobs with `:status` but no `workers/building`
  entry are probed every 30 min; a `:status` that is not `building` for >24h is
  restarted (unlinked) or, for fake jobs without `endtime`, removed with its `:dir`
  (`:236-253`); unparsable status files older than 1h are removed; orphaned `:dir`/
  `:status` without a job file are removed after 24h (`:281-301`).

There is no heartbeat from the worker to the backend while building other than the
5-minute idle re-registration (`bs_worker:4741-4744`, only in idle state) -- liveness
is detected by *polling the worker's HTTP endpoint*.

---

## 4. Worker (`bs_worker`)

### 4.1 Process model and command line

`bs_worker` is a single Perl process per *worker instance*; the production launcher
`dist/obsworker` starts one instance per CPU (or `OBS_WORKER_INSTANCES`) inside a
`screen` session (`dist/obsworker:303-430`). It first bootstraps the code with
`curl $WORKER_CODE/getworkercode | cpio` into `$OBS_RUN_DIR/worker/boot`
(`:353-366`), then for each instance I:

```
nice -n 18 ./bs_worker --hardstatus --kvm [--port P] [--proto https] \
   --root $OBS_WORKER_DIRECTORY/root_$I --statedir $OBS_RUN_DIR/worker/$I \
   --id $HOSTNAME:$I --reposerver http://backend:5252 [--reposerver ...] \
   [--hugetlbfs PATH] [--hostlabel L ...] [--owner O] [--jobs N] [--threads N] \
   [--test] [--cachedir DIR --cachesize MB] [--vm-kernel K] [--vm-initrd I] \
   [--vm-custom-option "..."] [--localkiwi DIR --arch local] \
   [--getbinariesproxy URL] [--getbinariesproxy-kiwiproduct URL] [--vm-enable-console] \
   [--vm-network] [--tmpfs] --device $ROOT/root_$I/root --swap $ROOT/root_$I/swap \
   --vm-memory 512 [--vmdisk-rootsize 4096 --vmdisk-swapsize 1024 --vmdisk-filesystem ext3 \
   --vmdisk-mount-options ... --vmdisk-clean] [--cleanup-chroot] [--wipeafterbuild] \
   [--vm-use-mkfs-copyin] [--arch ARCH] [--emulator-script S]
```

(`dist/obsworker:414-420`). The full option set is parsed in `bs_worker:410-751`
(usage text `:282-404`); VM type is `--kvm|--xen|--lxc|--emulator|--zvm|--docker|--pvm|
--openstack` or `--vm-type T` (`:448-452`, `:739-743`). `--statedir` holds `state`
(XML `$BSXML::workerstate`), `lock`, `job`, and the self-updated `worker/` and
`build/` code directories; `BUILD_DIR` is exported as `$statedir/build`
(`:775`). `--exit`/`--restart` talk to a running instance through the state file
(`:753-771`, `:4751-4766`).

### 4.2 Capability discovery (`:788-902`)

At startup the worker builds `$workerinfo` (`$BSXML::worker`): `hostlabel` list,
`sandbox` = vm type or `chroot`, `owner`, `linux/{version,flavor}` from
`/proc/version` (`:792-795`), `hardware/processors` and `hardware/cpu/flag` from
`/proc/cpuinfo` (x86 `flags`, s390 `features`, aarch64 `Features`, POWER synthesised
`power7/8/9` flags) (`:798-830`), aarch64 `nativeonly` + `EL0` flag heuristics by CPU
implementer/part (`:832-872`), synthetic `x86-64-v2/v3/v4` level flags (`:875-880`),
`jobs` from `--jobs`, `memory` from `--vm-memory`, `swap`/`disk` from `--vmdisk-*`
or `blockdev --getsize64` of the root/swap devices (`:883-900`). `hostarch` is
`uname -m` unless `--arch` (`:777-783`); it must be a key of `%BSCando::cando` or
`local` with `--localkiwi` (`:783`).

### 4.3 Registration and the state file

* `send_state($state, $port, $hostarch[, $exclude])` (`:956-980`) POSTs
  `/worker?state=&arch=&port=[&proto=][&workerid=]` to **every** `--reposerver`;
  for `idle` the body is the capability XML with `registerserver=<that server>`
  (`:971-975`), timeout 3s, errors only logged. For multiple repservers the loop
  aborts as soon as the local state is no longer idle (`:962-966`) -- first server to
  dispatch wins; the others are told `state=building` (`send_state('building', ...,
  exclude => the dispatching server)`, `:4966`) so they mark the worker `away`.
* State machine in `$statedir/state` (`BSXML.pm:1033-1041`:
  `state, nextstate, jobid, pid, logsizelimit, logidlelimit`), protected by
  `flock` on the same file (`lockstate`/`commitstate`, `:140-161`): `idle` ->
  `building` -> (`killed` | `discarded` | `badhost`) -> `idle` | `exit` | `rebooting`
  | `dead`.
* `periodic` runs every 10s inside the HTTP server (`:4729-4870`, `:4913-4918`):
  re-sends `idle` every 5 minutes (`:4741-4744`) (this is the only recovery after a
  repserver restart, see 5.4); honours `--oneshot`; executes `exit`/`dead`/`rebooting`
  (`rebooting` = `exec ./bs_worker @saveargv` from the updated `worker/` dir,
  `:4768-4772`); re-kills stuck jobs every 2 minutes in `killed/discarded/badhost`
  (`:4776-4790`); detects a dead build child (`pid`) (`:4793-4806`); and enforces
  **log limits**: kill when `.build.log` exceeds `logsizelimit` (default
  `$BSConfig::buildlog_maxsize` 500 MB) or has been idle for `logidlelimit` (default
  8h) (`:4808-4838`); and pattern-matches the log tail for known host bugs
  (`BUG: unable to handle kernel NULL pointer`, `rcu_sched self-detected stall`,
  soft lockup >1800s, `Data Storage Exception`) -> state `badhost` (`:4839-4866`).

### 4.4 The worker's HTTP server (`:4904-4931`)

| Route | Handler | Caller |
|---|---|---|
| `PUT /build jobid? buildcode? workercode? port? registerserver? nobadhost?` (body: job XML) | `startbuild` `:4656-4727` | dispatcher |
| `GET /logfile jobid? nostream? start? end? view=entry?` | `getlogfile` -> `stream_logfile` `:904-954`, chunked, follows the file until the build ends | repserver `_log` (live) |
| `GET /kill jobid?` | `killjob` `:4612` (state `killed`, `build --kill`) | repserver `abortbuild` |
| `GET /discard jobid?` | `discardjob` `:4624` (state `discarded`: result is thrown away, nothing uploaded) | repserver `deljob`/`restartbuild`, scheduler `killjob` |
| `GET /badhost jobid?` | `badhostjob` `:4636` | manual |
| `GET /sysrq jobid? sysrq=` | `sysrqjob` `:4648` -> `build --sysrq X` (kvm monitor `sendkey alt-print-X`, xen) | repserver `sendsysrq` |
| `GET /worker jobid?` | `workerinfo` `:4589` (hostarch, port, workerid) | warden liveness probe |
| `GET /info jobid?` | `jobinfo` `:4577` (the job XML) | debugging |
| `POST /kiwitree jobid? file*` | `getkiwitreefiles` `:4540-4575` (cpio of selected files under `KIWI/` or `PRODUCT/`) | repserver `receivekiwitree` |

All handlers verify `jobid` against the state file (`getstate`, `:4486-4493`).
`BSServer::server($conf)` **returns** when `/build` is accepted: `startbuild` sets
`$BSServer::request->{returnfromserver}` (`:4719`), the main program continues
synchronously with the build (`:4933-4934`); i.e. the HTTP server is *not* serving
while the worker is between `startbuild` and `BSServer::done` but the forked
children keep serving `/logfile`, `/kill`, etc. during the build.

`startbuild` (`:4656-4727`):

1. Xen: refuse if `/var/lib/xenstored/tdb` > 20 MB (`:4659-4661`).
2. Read the body to `job.new.<pid>`; state must be `idle` (`:4662-4664`).
3. **Code update**: if `workercode` differs from the local md5 and
   `--noworkerupdate` isn't set, fetch `<registerserver|http://peer:port>/getworkercode`
   into `worker.new`, run `./bs_worker --selftest`, swap directories, set state
   `rebooting` and `die("rebooting...")` -- the dispatcher sees an RPC error, moves
   the worker to `down`, and the worker re-execs and re-registers (`:4666-4677`,
   `getcode` `:1013-1075`). Same for `buildcode` -> `getbuildcode` (`:4697-4703`,
   without reboot).
4. Parse the job (`$BSXML::buildinfo`); `jobid = md5(xml)` (`:4679-4683`);
   `--localkiwi` workers accept only `kiwiproduct`/`productcompose` jobs (`:4690-4696`).
5. `--hostcheck SCRIPT --srcserver S <jobfile> precheck <buildroot>`: exit 3 -> `400
   cannot build anything`, 2 -> `cannot build this repository`, else `cannot build
   this package` (`:4705-4713`) -- mapped by the dispatcher to badhost bookkeeping.
6. Reply `200 <status code="ok">` (or `400 Testmode` in `--test`) and return the
   locked state + buildinfo + registerserver to the main program (`:4719-4725`).

### 4.5 Job lifecycle on the worker (`:4933-5262`)

```
BSServer::server() returns (state locked, state=idle)
  |-- masterdispatched? -> POST <job.reposerver>/workerdispatched (jobid := masterdispatched md5)   :4939-4947
  |-- truncate $buildroot/.build.log, state := building{jobid,pid,log limits}                      :4949-4963
  |-- send_state('building') to all other repservers                                              :4966
  |-- $ex = dobuild($buildinfo)                                                                    :4968-4983
  |     (die before download_complete -> $ex=3 badhost; "500 " -> 3; disk full -> 3; FATAL: -> 4)
  |-- follow-up builds: prepend old logfile, carry rpmlint.log/_statistics/_buildenv/_ccache.tar   :4985-5017
  |-- lockstate(); discarded? -> cleanup, back to idle, no upload                                  :5020-5039
  |-- badhost state -> $ex=3; other non-building state -> $ex=1                                    :5041-5045
  |-- empty log -> $ex=3                                                                           :5047-5056
  |-- --hostcheck post-build (failed|succeeded) -> $ex=3 on failure                                :5058-5064
  |-- _generated_buildreqs handling -> $ex=11 (genbuildreqs) if md5 differs from job's genbuildreqs :5067-5080
  |-- collect @send (4.5.6)                                                                        :5082-5174
  |-- $code := succeeded|unchanged|failed|badhost|genbuildreqs (4.5.7)                             :5176-5214
  |-- POST <reposerver>/putjob?job&arch&jobid&code&now[&workerid][&kiwitree=1] (chunked cpio)      :5234-5253
  |     on RPC failure: sleep 60 (then the result is lost; the warden/repserver re-dispatch)      :5250-5252
  |-- --wipeafterbuild -> build --wipe                                                             :5257
  |-- cleanup_job(); state := nextstate || idle; send_state('idle')                                :5260-5277
```

#### 4.5.1 `dobuild` -- download phase (`:3910-4133`)

Working directories (`:3934-3936`): `$buildroot/.build-srcdir` (sources),
`$buildroot/.pkgs` (binaries; may be a symlink), `$buildroot/.build.oldpackages`
(previous result for `--oldpackages`). `--tmpfs` mounts a tmpfs of
`vmdisk_rootsize` MB on `$buildroot` (`:3938-3943`).

`outbuildinfo` (the future `_buildenv` file) is seeded with
`project/package/repository/arch/srcmd5/verifymd5/versrel/bcnt/release/module/
crossarch/signflavor` and an empty `bdep` list (`:3952-3964`).

Then per job kind (`:3975-4064`):

| `file` / mode | Sources | Binaries | Notes |
|---|---|---|---|
| normal (`*.spec`, `*.dsc`, `Dockerfile`, `*.kiwi`, ...) | `getsources` (`:1111-1173`): `GET <srcserver>/getsources?project&package&srcmd5` as cpio with md5s; the md5 of the sorted `md5  name` list must equal `verifymd5` (`:1138-1145`); kiwi `repoarch=src` bdeps are fetched as extra sources into `images/<prj>/<pkg>/` (`:1152-1171`) | `getbinaries` (4.5.2) / `getbinaries_product` / `getbinaries_image` | recipe markers `# needsbinariesforbuild`, `# needspubkeyforbuild`, `# needssslcertforbuild`, `# needsappxsslcertforbuild`, `XS-Obs:`/`Obs:` tags, `@OBS_PACKAGE@` (`:4015-4030`); `mkosi/livebuild/appimage/simpleimage` imply needsbinariesforbuild and `forcebinaryidmeta` (`:4033-4039`); `readbuildenv` (`:3205-3249`) replaces the bdep list with a committed `_buildenv[.repo.arch]` file (reproducible rebuild from a frozen env); `getsslcert`/`getpubkey` write `_projectcert.crt` / `_pubkey` into the source dir (`:1247-1279`). |
| `_delta` (deltarpm generation) | `getjobdata` (old+new rpms placed by the scheduler in `:dir`) + `worker/worker-deltagen.spec` (`@mopt@` substituted from `Repotype: ... deltagen-mopt:N`) (`:3979-3989`, `:4136-4150`) | `getbinaries` | `file := deltagen.spec` |
| `_ptf`, `_reproduciblecheck` | `getjobdata` (`:3990-4001`); reproduciblecheck additionally reconstructs container tars from `_blob.*` (`reconstruct_containers`, `:3668-3697`) | `getbinaries` | `meta` is taken from the job data |
| follow-up (`followupfile` set, multi-stage) | `getfollowupsources` (`:1188-1211`): job data + `_buildenv` from the source package | product or normal | `rootforbuild` for kiwi, `_channel`/`*.report` copied into result (`:4002-4013`, `:4401-4403`) |

Build config: `getconfig?project&repository[&path=...]` from the srcserver
(or `config` embedded in the job), combined with `modularity_macros` (`:4072-4084`);
written to `$buildroot/.build.config` and parsed locally with `Build::read_config`
(`:4089`). For image builds the `path=` list makes the srcserver expand the config
over the image's repository path (`:4065-4069`). `getoldpackages` fetches the previous
build with `GET <reposerver>/build/<prpa>/<pkg>?view=cpio&noajax=1&noimport=1&withccache=N`
(`:3183-3203`) unless `nounchanged`; `getlinkedccache` pulls `_ccache.tar` from the
first path repo on another server that has one (`:3140-3181`). `getmodulemddata`
stores modulemd as `_modulemd.pst` (`:1227-1245`). Download statistics
(cache hits, bytes, preinstall image) go into `_statistics` (`:4113-4121`).

#### 4.5.2 `getbinaries` for package builds (`:2957-3138`)

Inputs: `bdep` entries with flags `preinstall, vminstall, runscripts, notmeta, noinstall,
installonly, sysroot` (`$BSXML::buildinfo`, `BSXML.pm:617-642`), `path` (ordered
repos, each with `server`), `syspath` (host repos for cross builds), `crossarch`,
`module`, `forcebinaryidmeta`.

1. `getpreinstallimage` (2.7) removes image-covered binaries from the todo list
   (`:2983-3002`).
2. For each `path` repo in order, call `getbinaries_cache($dir, server, prj, repo,
   arch, nometa, todo, modules, bvl, prefix)` (`:1481-1661`) with `nometa=1` unless the
   repo is the job's own project/repo (`:3011-3013`): metas are only needed for the
   local repo's packages because the dependency fingerprint ("meta") of a build is
   built from the `.meta` files of local deps and the hdrmd5 of foreign deps
   (`:3081-3135`).
3. Cross builds: target-arch `sysroot` deps first (prefixed `__sysroot__`), then the
   native (host-arch) deps from `syspath` with `arch = crossarch` (`:2963-2973`,
   `:3039-3057`).
4. Missing packages -> `die("404 getbinaries: missing packages: ...")`, which becomes
   `$ex=1`/failed (not badhost, since the message starts with 3 digits, `:4972`).
5. Compose `meta` (`:3060-3137`): one line per dependency `md5  name` (the hdrmd5 of
   the binary, or the first line of its `.meta`), plus the transitive lines of local
   deps' metas rewritten as `md5  dep/subdep`, then pruned by `BSBuild::gen_meta`
   (`BSBuild.pm:50-121`: sort by depth/path, drop duplicate `md5 lastpkg`, handle
   cycles via `subpack`; `genmetaalgo` 0/1 selects the pruning variant). The result is
   written to `$buildroot/.build.meta` and uploaded as `meta` -- the scheduler compares
   it with `:meta/<pkg>` to decide whether a rebuild is needed.

`getbinaries_cache` (`:1481-1661`) is the worker cache:

* Cache key: `sha256("$prpa/$hdrmd5")`; file `$cachedir/<2 hex>/<sha256>` (+`.meta`)
  (`get_cachefile`, `:1332-1338`). The hdrmd5 is the RPM signature-header MD5 (or
  Debian/Arch equivalents) obtained from `/getbinaryversions` (`:1510-1529`) or by
  `Build::queryhdrmd5` after download (`:1601-1607`).
* Hit path: `link_or_copy` cache file into the pkg dir, verify `queryhdrmd5` ==
  expected and (if needed) `.meta` md5 == `metamd5` (`:1537-1575`).
* Miss path: `GET /getbinaries?binaries=a,b,c` (or through the proxy,
  `trygetbinariesproxy`, `:1416-1439`, which passes
  `hdrmd5[metamd5]:sizek:suffix@name` tokens so the proxy can serve from its own cache
  without querying) as cpio into the pkg dir; new files are then **hardlinked into the
  cache** (`move_entry_into_cache`, `:1340-1361`).
* LRU: `$cachedir/content` is a Storable array `[[cacheid, size], ...]`, newest first,
  rewritten under flock by `manage_cache(prunesize, cacheold, cachenew)` (`:1370-1414`):
  used entries are moved to the front, new entries inserted, and entries beyond the
  byte budget `--cachesize` are unlinked. Space is reserved before big downloads
  (`:1583-1585`).
* Requires `--cachedir` *and* `--cachesize` (`:773`). The cache is keyed by content
  hash, so it is safe across jobs/projects; it also caches preinstall images
  (`<cachefile>.meta` = `"$hdrmd5  :preinstallimage\n"`, `:2371-2373`, `:2402-2408`) and
  product-build rpms (`:1790-1808`, cache key `hdrmd5[/rpmhdrs_only]` or
  `md5sum.extra`).

`bs_getbinariesproxy` (`:189-406`, `408-472`, `474-607`) is the same algorithm as a
shared HTTP cache (`$bsdir/getbinariesproxycache`, 1 GB default, `maxchild` 40): it
answers `/getbinaries` with files hardlinked from its cache and fetches misses from
the `server=` named in the request; `getbinaries_kiwiproduct` serves per-package
`view=cpio` fetches by `hdrmd5:sizek:rpm@name` or `md5sum:sizek:extra@name`;
`getpreinstallimage` caches images. It has no authentication at all.

#### 4.5.3 Image, container and product builds

* `getbinaries_image` (`:2852-2955`) for kiwi appliances, Dockerfile, fissile:
  two-path mode (`syspath` = packages to *install* in the build root, fetched
  `nometa`; `path` = packages that go into `$srcdir/repos/<prj>/<repo>/` for the
  image tool, with hdrmd5 lines in meta); `unorderedimagerepos`/bdep `project`
  pins which repo a binary comes from (`:2905-2912`); then `getbinaries_containers`
  (`:2761-2850`): each `container:` bdep is fetched with `/getbinaries?withannotation=1`
  into `$srcdir/containers/<prj>/<repo>/`, gzipped for kiwi, and the base-container
  annotation (`$BSXML::binannotation`, `BSXML.pm:2174-2194`) is written to
  `containers/annotation`; `containerurl` collects `dir://./containers/...` for
  `--set-container-derived-from`.
* `getbinaries_buildenv` (`:2483-2733`) for jobs with a frozen `_buildenv`: every
  bdep must carry `hdrmd5`; the worker resolves name+hdrmd5 against
  `/getpackagebinaryversionlist` of all path repos (so it can also find *old*
  versions still present in package dirs, not just `:full`), then downloads
  individual files from `/build/<prpa>/<pkg>/<file>` (`:2655-2660`).
* `getbinaries_product` (`:1690-2178`) for kiwi products and productcompose: the
  bdep list contains `package`/`repoarch`/`binary` entries produced by the scheduler
  (`BSSched/BuildJob/KiwiProduct.pm:616-640`, `ProductCompose.pm:710-734`); the
  worker builds `$srcdir/repos/<prj>/<repo>/<rarch>/<file>` trees per package,
  verifying `hdrmd5`/`leadsigmd5`/`md5sum` from `view=binaryversions` bininfo, with
  cache and proxy support, renames `_modulemd.yaml`/`updateinfo.xml` per package,
  spreads `_slsa_provenance.json` to per-rpm files (`:2100-2119`), merges
  `.checksums` into `repos/.createrepo_checksums` (`:2130-2165`), and for
  `RPMHDRS_ONLY` products fetches only rpm headers (`view=cpioheaderchksums`).
  `kiwiproduct_patch_recipe` (`:3251-3341`) rewrites the kiwi XML: `MEDIUM_NAME`
  gets `-Build<cicnt>.<bcnt>`, `obsrepositories:/` is expanded to `obs://prj/repo`
  entries, `<repopackage name="*"/>` is expanded to the full package list from meta.

#### 4.5.4 The `build` invocation (`:4152-4302`)

The rpmlist `$buildroot/.build.rpmlist` (`:4162-4191`), one line per installable bdep:

```
<name> <path>/.pkgs/<name>.rpm        # or .deb/.pkg.tar.*/.apk
<name> preinstallimage                # covered by the preinstall image
sysroot: <name> <path>/.pkgs/__sysroot__<name>.rpm
preinstallimage: <path>/.pkgs/preinstallimage.<file>
preinstallimagesource: <prj>/<repo>/<pkg> [<id>]
localkiwi <dir>/localkiwi.rpm
preinstall: a b c
vminstall: d e
runscripts: a
noinstall: x y            # not for image modes
installonly: z
```

(`init_buildsystem` parses these tags at `build/init_buildsystem:725-793`.)

Exact command line for a KVM worker (constructed `:4194-4302`; `qsystem` `:4304`):

```
[linux32|linux64]  $statedir/build/build \
  --root $buildroot/.mount --vm-type kvm --vm-disk $vm_root --vm-swap $vm_swap \
  [--emulator-script S] --statistics --vm-watchdog \
  --memory <--vm-memory or $buildroot/memory> [--vm-kernel K] [--vm-initrd I] \
  [--vm-custom-opt=...] [--vmdisk-rootsize N] [--vmdisk-swapsize N] \
  [--vmdisk-filesystem <buildflags:vmfstype or --vmdisk-filesystem>] \
  [--vmdisk-filesystem-options <buildflags:vmfsoptions or ...>] [--vm-use-mkfs-copyin] \
  [--vmdisk-mount-options=...] [--vmdisk-clean] [--hugetlbfs P] [--vm-worker N] \
  [--vm-worker-nr N] [--vm-network] [--vm-enable-console] \
  --clean --changelog [--oldpackages $buildroot/.build.oldpackages] \
  [--norootforbuild] --baselibs-internal \
  --dist $buildroot/.build.config --rpmlist $buildroot/.build.rpmlist \
  --logfile $buildroot/.build.log [--release R] [--debug] --arch <job arch> \
  [--jobs N] [--ccache --ccache-create-archive [--ccache-type=T]] [--threads N] \
  --reason "Building <pkg> for project '<prj>' repository '<repo>' arch '<arch>' srcmd5 '<md5>'" \
  [--disturl obs://...] [--linksources] [--signdummy] [--vcs URL] [--buildtime T] \
  [--obspackage=<pkg>] [--buildflavor=<flavor>] \
  $buildroot/.build-srcdir/<recipe> \
  [--kiwi-parameter --ignore-repos --kiwi-parameter --add-repo --kiwi-parameter dir://./repos/<prj>/<repo> \
   --kiwi-parameter --add-repotype --kiwi-parameter rpm-md --kiwi-parameter --add-repoprio --kiwi-parameter N ...] \
  [--kiwi-parameter --set-container-derived-from=dir://./containers/...]
```

Variants: `openstack` passes `--vm-server --vm-worker --vm-kernel --openstack-flavor`
(`:4225-4233`); `lxc`/`docker` pass `--root $buildroot --vm-type T [--memory]`
(`:4234-4237`); chroot passes just `--root $buildroot` (`:4238-4242`). The leading
helper (`linux32`/`linux64`) comes from `BSCando::cando` entries of the form
`arch:helper` for the job's `hostarch||arch` (`:3921-3922`). `--norootforbuild` is
forced unless `$BSConfig::norootexceptions` matches or `buildflags:allowrootforbuild`
or `rootforbuild` (follow-up kiwi) (`:4253`). `--jobs` is capped by
`buildflags:maxjobs` (`:4155-4156`). `--signdummy` only for products when
`build/signdummy` exists (`:4268`). Multibuild: a package `name:flavor` yields
`--buildflavor=flavor` and `--obspackage=name` (`:4273-4281`).

Exit code mapping (`:4304-4375`): obs-build returns 0 success, 1 failed, 2 unchanged
(or forced by `same_result_marker` unless `reason eq 'rebuild counter sync'`), 3 retry
on another host (badhost), 4 fatal host (badhost + worker `dead`), 9 generated
buildrequires changed (`build/build:457-461`, `1867`). On 0/2 the VM statistics
file `OTHER/_statistics` is parsed (`updatestatsfromfile`, `:4406-4439`: `TIME_*`,
`MAX_mb_used_on_disk`, `MAX_mb_used_memory`, `IO_*`) and the final
`$BSXML::buildstatistics` XML (`BSXML.pm:1943-2000`) is written to `OTHER/_statistics`
with `info{hostarch,starttime,endtime,workerid,readytime,srcmd5,verifymd5,rev,vcs,
file,buildtype,jobid}` (`:4340-4360`). For VM builds the result tree is renamed from
`$buildroot/.mount/.build.packages` to `$buildroot/.build.packages` (`:4328-4336`).
On success `OTHER/_buildenv` (the `outbuildinfo`) is written (`:4377-4379`) and
`.report`/`_channel` files are generated (4.5.5).

#### 4.5.5 Post-processing per image type (`:4381-4399`)

| Mode | Function | Produces |
|---|---|---|
| non-image with `.packages` present | `createreport(OTHER)` | `X.report` from `X.packages` (`:3540-3646`) |
| docker/podman/fissile | `createreport(DOCKER)` | `X.report` with origin project/repo/package per installed package (`kiwiorigins`), `isbase` from `.basepackages`, summaries from `.pkgsummaries`, cpeid for `-release` packages, version/release from the containerinfo |
| kiwi image | `createreport(KIWI)` | same for `KIWI/*.packages` |
| kiwi product | `createchannel(OTHER)` | annotates every `*.report` with origins and writes `_channel` XML (`:3392-3503`) |
| productcompose | `annotatereport(PRODUCT)` | origin annotation only (`:3505-3524`) |

`$BSXML::report` (`BSXML.pm:2087-2120`) is the content-tracking record the publisher
and release tooling consume.

#### 4.5.6 Result collection (`:5082-5174`)

Result dirs by recipe: `RPMS/<arch>/*` + `SRPMS` (default), `DEBS` (`.dsc`, `build.collax`,
`debian.control`), `DEBS/<arch>`+`SDEBS` (debbuild), `ARCHPKGS` (PKGBUILD), `APKS`
(APKBUILD), `KIWI` (`.kiwi`), `PRODUCT` (`.productcompose`), `DOCKER`
(`Dockerfile*`), `FISSILE`, `HELM` (`Chart.yaml`), always plus `OTHER` and the top
level (`:5086-5097`). Only regular files are sent (`:5105`); `same_result_marker`
and `.kiwitree` are excluded; `_generated_buildreqs` is sent only in the
genbuildreqs case. Subdirectories inside `KIWI`/`PRODUCT` are not sent as files:
instead a `.kiwitree` manifest is generated (`buildkiwitree`, `:4441-4484`:
`d <dir>` / `l <link> <target>` / `f <file> [leadsigmd5|md5]`), uploaded, and the
repserver pulls what it cannot hardlink via `POST /kiwitree` (2.5 step 9), unless
`$BSConfig::nokiwitree`.

Containers: if `BSContar`/`BSTar` loaded and `/usr/bin/obs-gzip-go` exists, every
`X.tar` with `X.containerinfo` is **normalized on the worker** (`normalize_container`,
`:3648-3666`: deterministic layer recompression with obs-gzip-go, mtime from the
image config, `.sha256` regenerated) and an `X.tar.recompressed` marker is uploaded
so the repserver skips recompression (`:5110-5122`; `Containertar.pm:46-48`).

SLSA: `_slsa_provenance.json` is generated for `slsaprovenance` jobs (in-toto
statement v0.2 or v1 per `buildflags:slsaversion`, `:3790-3884`): `subject` = sha256
of every uploaded artifact except `rpmlint.log/_statistics/_buildenv`; `materials`/
`resolvedDependencies` accumulated during download (`add_slsa_materials_*`,
`:3886-3908`: sources from `getsources`, binaries from every `getbinaries_*` path,
containers, repos, sysroot, and the build config with `intent=buildconfig`); URIs
point at `slsadownloadurl` (`/_slsa` publishing server or the API `/slsa/<prpa>/
<file>/<sha256>` route). `_slsa_provenance.config` (the exact prjconf) is uploaded too.
Follow-up/reproduciblecheck jobs reuse the stored provenance (`:5130-5135`).

Unchanged builds (`$ex==2`) upload only `_statistics` and possibly `_ccache.tar`
(`:5168-5174`); `meta` and `logfile` are always appended (`:5217-5218`); badhost
uploads only the log (`:5220-5230`).

#### 4.5.7 Result codes

| `$ex` | Source | `code=` sent to `/putjob` |
|---|---|---|
| 0 | build exit 0 | `succeeded` |
| 2 | build exit 2 / `same_result_marker` | `unchanged` |
| 1 | build exit 1, missing packages, kiwitree failure, non-building state, oversized `_generated_buildreqs` | `failed` |
| 3 | build exit 3; download-phase exception; `500 ...` errors; disk full; `Build.pm` not importable; empty log; post hostcheck failure; `badhost` state (log pattern) | `badhost`, **unless** the job carried `nobadhost` (-> `failed` + message appended, `:5185-5188`) or the log tail shows `Exec format error` / `error while loading shared libraries: lib` (a broken build config, not a broken host, `:5191-5205`) |
| 4 | build exit 4 / `FATAL:` | `badhost` and worker `nextstate=dead` (`:5206-5210`) |
| 11 | `_generated_buildreqs` md5 != job's `genbuildreqs` | `genbuildreqs` (scheduler re-expands with the new buildrequires, `BSSched/BuildJob.pm:414-467`, `:649-656`) |

### 4.6 Cross-arch and emulation

* **Personality switch** (same ISA family): `BSCando::cando` lists `i586:linux32` for
  x86_64, `armv7hl:linux32` for aarch64, `ppc:linux32` for ppc64, `s390:linux32` for
  s390x, etc. (`BSCando.pm:32-74`). The dispatcher treats these as native capable
  (`harchcando`), the worker prefixes the build command with the helper (`:3921`,
  `:4160`), and inside the VM obs-build re-execs itself with the recorded
  `PERSONALITY` (`build/build-vm:629-634`, `:1019-1026`).
* **Foreign ISA via qemu-user**: the job carries `hostarch` (prjconf `HostArch:`;
  `BSSched/BuildJob.pm:965`), the scheduler writes a `:cross` marker in
  `jobs/<hostarch>/`, the dispatcher assigns the job to a `hostarch` worker
  (`bs_dispatch:1248`), and obs-build registers binfmt handlers from
  `build/qemu-reg` via the static `initvm.<arch>` helper when the target arch is not
  in the host's `extend_build_arch` list (`build/common_functions:110-133`,
  `build/init_buildsystem:710-720`). Requires the `build-initvm-<arch>` static
  binaries and qemu-user binaries in the build root.
* **Real cross compile** (sysroot): `crossarch` + `syspath` in the job; worker splits
  bdeps into `sysroot` and native (`bs_worker:2963-2973`), obs-build installs the
  sysroot packages under `/build-sysroot` (`build/init_buildsystem:1160-1192`,
  `build/build-recipe-spec:206-209` adds `--nodeps`).
* **System emulation** (`--vm-type qemu` / `--emulator`): `build-vm-qemu` boots a
  full foreign-arch VM (`build/build-vm-qemu:38-...`), `build-vm-emulator` delegates
  to `emulator/emulator.sh` (`build/build-vm-emulator:32-62`).

### 4.7 Host requirements of a worker

From `bs_worker`, `dist/obsworker`, `build/build-vm*`:

* Runs as **root** (mounts, chroot, loop devices, `kill -9 -1` inside VMs; the
  non-root paths in obs-build are for pbuild/osc, not bs_worker).
* Perl with `XML::Structured` (pure-perl fallback is forced, `bs_worker:30`),
  `Digest::MD5/SHA`, `Storable`, and the shipped `BS*.pm` + `build/` code which the
  worker **fetches from the repserver** (`/getworkercode`, `/getbuildcode`) and
  replaces at runtime (`getcode`, `:1013-1075`). The worker's own installed perl
  modules must only be enough to run the bootstrap.
* KVM: `/dev/kvm` readable, `qemu-kvm`/`qemu-system-<arch>` (`build/build-vm-kvm:
  195-199`), a guest kernel+initrd (`/boot/vmlinuz`, `/boot/initrd-build|virtio`,
  `kernel.obs.guest`...) with virtio, `fuser` (kill), optional hugetlbfs and
  `qboot.rom`; `mkfs.*`, `tune2fs`, `fallocate`, optional `debugfs` and
  `mkfs.ext4 -d` (copy-in mode), `bsdtar` (mandatory for preinstall images in VMs,
  `build/init_buildsystem:212-220`). Root and swap images are files in
  `$buildroot.img`/`.swap` or block devices (LVM via `OBS_SETUP_WORKER_PARTITIONS`).
  Guest runs `init=/.build/build` (`build-vm-kvm:356`); swap device doubles as the
  result channel (`BUILDSTATUS` + block list, `build-vm:1217-1225`, `:1131-1160`).
* Xen: `xl`/`xm`; z/VM: `vmcp`, `iucvconn`, DASD minidisks; PowerVM: `pvmctl`;
  OpenStack: `nova`/`cinder` CLIs and cinder volumes; EC2: `ec2-*` tools.
* Containers (`lxc`, `docker`, `podman`, `nspawn`): the respective runtime; the
  build root is bind-mounted and `/.build/build` is exec'd as init inside
  (`build/build-vm-docker:30-44`, `build-vm-podman:30-42`, `build-vm-nspawn:30-44`,
  `build-vm-lxc:44-103`); these are "not safe" sandboxes for constraint purposes.
* chroot: no isolation; `killchroot` for kill.
* Network: outbound HTTP(S) to srcserver, all repservers / partition servers,
  optional getbinaries proxy; **inbound** HTTP from dispatcher, repserver(s) and
  warden on the worker's port (random unless `--port`/`OBS_WORKER_PORTBASE`).
* Disk: `--cachedir` sized by `--cachesize`; buildroot + `.pkgs` + VM images
  (default 4 GB root + 1 GB swap).
* Clock within +-1h of the repserver (`bs_repserver:2398-2404`).
* Optional `/usr/bin/obs-gzip-go` for container normalisation on the worker.

---

## 5. Worker <-> backend protocol timeline

### 5.1 Sequence diagram (normal job)

```
Worker W           Repserver R (port 5252/5253)   Dispatcher D        Scheduler S(arch)     Srcserver SRC        Warden
  |                      |                            |                    |                    |                 |
  |-- GET /getworkercode (bootstrap, dist/obsworker) ->|                  |                    |                 |
  |<- cpio worker/ ------|                            |                    |                    |                 |
  |-- POST /worker?state=idle&arch=H&port=P[&workerid=ID][&proto=https]  body=<worker/> ->|   |                 |
  |   (to EVERY --reposerver; bs_worker:956-980)       |                    |                    |                 |
  |                      | writes workers/idle/H:ID {uri=http://peerip:P,...} (bs_repserver:1795-1836)           |
  |                      |                            |                    |-- writejob: jobs/A/J (+status=scheduled in build/..., :cross marker)
  |                      |                            |<-- ls jobs/A, ls workers/idle (fs) --|                    |
  |                      |                            | lockcreatexml jobs/A/J:status {dispatching}              |
  |                      |                            |-- GET SRC/source/P/K/_constraints?expand=1&rev=md5 ->|  |
  |<- PUT /build?port=5252&workercode=M&buildcode=M[&registerserver=R][&nobadhost=msg]  body=<buildinfo/> --|  |
  |   (bs_dispatch:289-300; may answer 400 "cannot build ..." / die "rebooting...")                            |
  |-- 200 <status code="ok"/> -------------------------->|                                                        |
  |                      |                            | jobs/A/J:status := building{uri,workerid,starttime,hostarch,jobid=md5(xml)}
  |                      |                            | workers/idle/H:ID -> workers/building/H:ID (+job,jobid,arch)
  |-- [masterdispatched only] POST R'/workerdispatched?arch&job&jobid&hostarch&port ->|                         |
  |-- POST R2/worker?state=building (other repservers -> workers/away) --->|                                     |
  |-- GET SRC/getsources?project&package&srcmd5 ---------------------------------------------->|                 |
  |<- cpio sources (+md5 per file; verify verifymd5) ------------------------------------------|                 |
  |-- GET SRC/getconfig?project&repository[&path=..] ------------------------------------------>|                 |
  |-- GET SRC/getsslcert | /getpubkey | /getmodulemd (optional) ------------------------------->|                 |
  |-- GET R/getbinaryversions?project&repository&arch&binaries=a,b,c[&nometa]&withevr=1&now=T ->|                 |
  |<- <binaryversionlist> (hdrmd5, sizek, metamd5, evr) -------|                                                  |
  |-- POST R/getpreinstallimageinfos?match=body&prpa=... body=512B bitstring ->|  (optional)                      |
  |<- Storable [ {package,hdrmd5,file,sizek,hdrmd5s,path=pkg/_preinstallimage.<id>} ] --|                        |
  |-- GET R/build/P/Rp/A/K/_preinstallimage.<id> --------->|  (optional, or via getbinariesproxy)                 |
  |-- GET R/getbinaries?project&repository&arch&binaries=x,y[&nometa][&metaonly][&module=..]&now=T ->|          |
  |<- cpio x.rpm x.meta y.rpm y.meta ... ------------------|  (per path repo, cache misses only; proxy optional) |
  |-- GET R/build/P/Rp/A/K?view=cpio&noajax=1&noimport=1&withccache=N  (old packages) ->|                        |
  |-- GET R2/build/P2/Rp2/A/K/_ccache.tar  (optional, linked project) -------->|                                  |
  |   run: $statedir/build/build ... (4.5.4); .build.log grows                                                   |
  |<- GET /logfile?jobid=..&start=&end=&nostream=  (streamed to API users through R:getlogfile) --|              |
  |<- GET /worker?jobid=..  (liveness) ----------------------------------------------------------------------------|
  |<- GET /kill|/discard|/sysrq?jobid=..  (abort/restart/sysrq from R docommand, S killjob, R deljob) --|        |
  |-- POST R/putjob?job=J&arch=A&jobid=md5&code=succeeded|failed|unchanged|badhost|genbuildreqs&now=T[&workerid=ID][&kiwitree=1]
  |      Content-Type: application/x-cpio, chunked; entries: RPMS/.., OTHER/_statistics, _buildenv, meta, logfile, [_slsa_provenance.json], [.kiwitree]
  |                      |<- (kiwitree) POST W/kiwitree?file=..&file=.. -> cpio of missing subtree files
  |                      | jobs/A/J:dir written; :status := finished|signing; jobs/finished += line; events/A/finished:J (or events/signer/)
  |                      | workers/building/H:ID -> workers/down/H:ID
  |<- 200 ---------------|                            |                    |<-- event finished:J (events/A/, .ping)
  |                      |                            |                    | jobfinished: jobs/A/J:dir -> build/P/Rp/A/K/, :full update, status succeeded
  |                      |                            |                    | purgejob: rm jobs/A/J, J:status
  |-- build --wipe (optional)                          |                    |                    |                 |
  |-- POST /worker?state=idle (all repservers) -------->|  workers/down -> workers/idle  (loop)                    |
  |   ... every 5 min while idle: POST /worker?state=idle again (bs_worker:4741-4744)                            |
```

Timeouts: dispatcher `/build` 10s+size; worker RPCs to backend `$gettimeout` = 1h
(`bs_worker:118`); `send_state` 3s; `/workerdispatched` 600s; `putjob` default BSRPC
timeout (no explicit limit on the worker side, repserver reads the stream);
`/kiwitree` 600s per cpio file (`bs_repserver:2287-2292`); warden probes 60s.

### 5.2 Message formats

* Job (`PUT /build` body, `jobs/<arch>/<job>`): `$BSXML::buildinfo`. Example (trimmed)
  as written by `BSSched::BuildJob::create` / `writejob`:

```xml
<buildinfo project="openSUSE:Factory" repository="standard" package="zlib" job="openSUSE::Factory::standard::zlib-3f2a...e1"
           arch="x86_64" srcserver="http://backend:5352" reposerver="http://backend:5252">
  <srcmd5>3f2a9c...e1</srcmd5>
  <verifymd5>3f2a9c...e1</verifymd5>
  <rev>42</rev>
  <disturl>obs://build.opensuse.org/openSUSE:Factory/standard/3f2a9c...e1-zlib</disturl>
  <reason>source change</reason>
  <needed>131</needed>
  <revtime>1729490000</revtime>
  <readytime>1729499990</readytime>
  <file>zlib.spec</file>
  <versrel>1.3.1-2</versrel>
  <bcnt>7</bcnt>
  <release>2.7</release>
  <debuginfo>1</debuginfo>
  <constraintsmd5>0c4e...</constraintsmd5>
  <prjconfconstraint>sandbox secure</prjconfconstraint>
  <subpack>zlib-devel</subpack><subpack>libz1</subpack>
  <bdep name="rpm" preinstall="1" runscripts="1"/>
  <bdep name="glibc" preinstall="1" vminstall="1"/>
  <bdep name="gcc" notmeta="1"/>                 <!-- Support: package -->
  <bdep name="pkgconf-pkg-config"/>
  <bdep name="kiwi" noinstall="1"/>              <!-- in sysdeps-only images -->
  <bdep name="libz1" installonly="1"/>
  <path project="openSUSE:Factory" repository="standard" server="http://backend:5253"/>
  <path project="openSUSE:Tumbleweed" repository="standard" server="http://home-backend:5253"/>
  <syspath .../>  <containerpath .../>           <!-- image/cross builds only -->
  <logidlelimit>7200</logidlelimit>
  <genbuildreqs>8f1c...</genbuildreqs>
  <slsaprovenance>1</slsaprovenance><slsabuilder>https://api.example</slsabuilder>
  <module>nodejs:18</module>
</buildinfo>
```

* Worker state file (`$statedir/state`): `<workerstate state="building" jobid="d41d..." pid="4711" logidlelimit="7200"/>`.
* `/putjob` cpio entries are plain files named by basename only (`BSHTTP::cpio_receiver`
  rejects `/` and `..`, `BSHTTP.pm:423-428`), so result dirs are flattened; the
  only tree structure is the `.kiwitree` manifest.
* All cpio transfers are SVR4 "newc" (`070701`) without CRC (`BSCpio.pm`,
  `BSHTTP.pm:404-412`); directories only with `createsubdirs` (code update).

### 5.3 Authentication and trust boundaries

* Backend -> worker: none. Any client that can reach the worker port can
  `PUT /build` a job (the worker checks only that it is idle and that `jobid`s
  match later), `/kill`, `/discard`, read the log.
* Worker -> backend: IP ACL class `worker` (`BSConfig::ipaccess`), optionally a
  dedicated worker port (`workerreposerver`/`workersrcserver`), optionally HTTPS
  (`--proto https`, `BSSSL`, server-side `ssl_keyfile/certfile`; client certificates
  only via `subjectdnaccess` on the server). No per-job token: `putjob` is
  authorised by `(job, jobid = md5 of job XML, workerid, starttime)` consistency and
  optionally the peer IP (`bs_repserver:2336-2353`, `2515-2520`).
* Worker -> srcserver: same IP ACL; `/getsources` returns any revision of any
  package to any worker.
* Code integrity: the worker executes whatever `/getworkercode` and `/getbuildcode`
  deliver (md5 compared, not signed).
* Inside the sandbox: `secure` = xen/kvm/zvm (`Constraints.pm:34-41`); the VM has no
  network unless `--vm-network`; results come out through the swap device block
  list (`computeblocklists`/`extractbuild`), which the host re-validates only
  structurally.

### 5.4 Recovery scenarios

| Failure | Detection | Recovery |
|---|---|---|
| Repserver restarts | nothing is lost: all state is on disk (`workers/*`, `jobs/*`). Idle workers re-register every 5 min (`bs_worker:4741-4744`); workers that finish during the outage retry `putjob` once after 60s and otherwise **lose the result** (`:5250-5252`) -> job stays `building` until the warden probes the (now idle) worker: `/worker?jobid=` fails (`not building a job`, `:4486-4493`) -> status unlinked -> re-dispatch. Alternatively the worker's next `state=idle` with a stale `workers/building` entry triggers the repserver's own crash path (`bs_repserver:1773-1794`). |
| Dispatcher restarts | state dump/restore of badhost/infocache (`bs_dispatch:666-710`, `808-815`); jobs are re-discovered from disk; `:status` files written by the dead dispatcher stay (`dispatching` ones become `building`-less orphans cleaned by warden after 24h, `bs_warden:238-253`). |
| Worker dies mid-build | warden `/worker` probe fails -> unlink `:status`, worker -> `down` (`bs_warden:188-202`); or repserver `workerstate` when the worker comes back idle. The job is re-dispatched from scratch (no checkpointing). `%badhost` is **not** set in this case. |
| Worker reboots with new code | `startbuild` dies `rebooting...` -> dispatcher marks worker `down` (`bs_dispatch:322-324`); worker re-execs and re-registers. |
| Job cancelled (source change, delete) | scheduler `killjob` -> `GET uri/discard` -> worker state `discarded`, `build --kill`, no upload (`BSSched/BuildJob.pm:163-204`, `bs_worker:4624-4634`, `:5020-5039`). |
| API `abortbuild` | repserver `/kill` -> worker uploads `failed` with "Killed job" appended (`bs_repserver:3746-3760`, `bs_worker:4612-4622`). |
| API `restartbuild` | repserver `/discard` + unlink `:status` (`:3723-3744`). |
| Host unfit (`badhost`) | worker `code=badhost` -> `%badhost{prj/pkg/arch/worker}`, `{arch/job}`, `{arch/job/worker}` for 24h; after >5+20/nhosts attempts the job is dispatched with `nobadhost` and finally fails (`bs_dispatch:224-235`, `280-288`). |
| Clock skew > 1h | `putjob` dies `time mismatch`, worker treated as badhost (`bs_repserver:2398-2404`). |
| Scheduler restarts | `init_ourjobs` re-reads `jobs/<arch>` (`BSSched/BuildJob.pm:131-143`); events in `events/<arch>/` are durable. |
| Master/slave dispatcher split | slave re-syncs its job list with the srcserver every 600s and pushes/deletes jobs on the master (`bs_dispatch:454-597`); the owning repserver is told via `/workerdispatched` and `cmd=idleworker`. |

---

## 6. Project configuration (prjconf) semantics

### 6.1 How the config reaches the build path

* The srcserver assembles the effective config by concatenating the `_config` of every
  project in the repository path (last path element first) with
  `%define _project`, `%define _repository`, `_is_this_project`, `_is_in_project`
  markers, `#!!line` directives and `FromScratch:` resets, and strips `Macros:` blocks
  out of the middle (`BSRepServer/ProjPacks.pm:113-173` is the repserver copy of the
  same algorithm; `BSConfig::extraconfig` can append per-project snippets).
  `#!PrependConfigFile:` and `Build::combine_configs` (`build/Build.pm:295-325`)
  merge multiple `Macros:` sections.
* The worker fetches the text (`/getconfig`), writes `.build.config`, and parses it
  with `Build::read_config($arch, file)` (`bs_worker:4089`); obs-build re-reads it in
  `init_buildsystem`/`expanddeps`/`queryconfig --dist` and inside the VM as
  `/.build/build.dist` (`build/build-vm:993-998`).
* `Build::read_config` (`build/Build.pm:385-638`) first runs the text through the RPM
  spec parser `Build::Rpm::parse` with `parsing_config` set (`:415-419`), so
  `%if`, `%ifarch`, `%define`, macro expansion and `%(...)`-free expressions work in
  the config; `Macros:` ... `:Macros` blocks are kept verbatim (`:401-412`). Then
  every remaining line is `Keyword: args`.

### 6.2 Directive reference (as implemented in `Build::read_config`)

Keywords are case-insensitive (`lc`, `:452`). `!*` clears a list, `!name` removes an
entry; later lines win (`:471-481`). Unknown keywords only warn (`:555-557`).

| Directive | Stored as | Consumer and effect |
|---|---|---|
| `Preinstall: pkgs` | `preinstall[]` (`:471`) | Packages unpacked (no scripts) by `init_buildsystem` before the package manager exists (`build/init_buildsystem:915-918`); flagged `preinstall=1` in job bdeps (`BSSched/BuildJob.pm:1223`); always part of the build env (`get_build` `:775`); kept when subpackages are removed (`:769`); used to guess `type` (`:566-572`). |
| `VMInstall: pkgs` | `vminstall[]` | Additional preinstalls only for VM builds (mount, kernel modules, udev) (`init_buildsystem:926-932` with `--prepare`); `vminstall=1` bdep flag. |
| `Runscripts: pkgs` | `runscripts[]` | Preinstalled packages whose `%post` scripts are run right after preinstall (`init_buildsystem:898-900`, `941`); `runscripts=1` flag; defaults to preinstall list for old clients. |
| `Required: pkgs` | `required[]` | Always installed; changes trigger rebuilds (included in `get_deps`, `:887-898`). |
| `Support: pkgs` | `support[]` | Always installed but excluded from the rebuild fingerprint (`get_deps` strips them, `:903-910`); bdeps get `notmeta=1` (`BSSched/BuildJob.pm:1225`). |
| `Keep: pkgs` | `keep[]` | Normally all subpackages of the package being built are *excluded* (`-subpack`) from the env to avoid self-cycles; `Keep` whitelists some (`get_build:766-773`). Empty `Keep` switches to "filter subpacks from required/support" mode. |
| `Prefer: pkg` / `Prefer: -pkg` / `Prefer: A:pkg` | `prefer[]` -> `init_helper_hashes` | libsolv/Build::Expand choice resolution: when several providers satisfy a dep, pick the preferred one (`-` = de-prefer; `A:` = only when `A` caused the choice). Otherwise the expander errors "have choice". |
| `Ignore: pkg` / `Ignore: A:pkg` | `ignore[]` | Drop a dependency edge globally or only from `A` (`build_config.adoc:234-244`). `--ignoreignore--` in job deps (`BSSched/BuildJob.pm:1140`) disables ignores for sysdeps/simpleimage. |
| `Conflict: pkgs` / `Conflict: A:B` | `conflict[]` | Extra conflicts for the expander. |
| `Substitute: A B C...` / `Substitute: !A` / `Substitute: A A --enable-ccache` | `substitute{A}=[B,C]`, `substitute_vers` (`:482-491`, `:593-598`) | Direct BuildRequires replacement (`do_subst`, `:655-674`); also the namespace for `build-packages:<type>`, `system-packages:<type>` (`get_sysbuild` `:818-881`), `obs-service:<name>` (`BSSched/BuildJob.pm:1122-1131`), `sysroot-packages`, `kiwi-setup:*`, `mock:chroot_setup_cmd`. Default substitutes come from `%subst_defaults`. |
| `FileProvides: /path pkgs` | `fileprovides{}` | Declares which packages provide a file so file BuildRequires can be resolved without filelists. |
| `ExportFilter: regex arch...` | `exportfilter{}` | Scheduler/publisher: which built binaries are copied to other arches' `:full` (`BSSched/BuildResult.pm:136-241`). Not a worker concern. |
| `PublishFilter: regex...` | `publishfilter[]` | Publisher. |
| `OptFlags: arch flags` | `optflags{}` | Emitted into `.rpmmacros`/`optflags` by `queryconfig rawmacros`. |
| `Order: A:B` | `order{}` | `init_buildsystem` install ordering hints (`reorder`, `:325`). |
| `Repotype: rpm-md splitdebug:-debuginfo deltagen-mopt:N ...` | `repotype[]` | Publisher; `deltagen-mopt` read by the worker for `_delta` jobs (`bs_worker:4140-4144`). |
| `Patterntype: ymp comps` | `patterntype[]` | Publisher. |
| `Type: spec|dsc|kiwi|arch|livebuild|preinstallimage|productcompose|...` | `type` | Recipe type; selects `build-recipe-*`, `binarytype` defaults, result dirs, handler class in the scheduler. Guessed from preinstalls if absent (`:564-574`). |
| `BinaryType: rpm|deb|arch|apk|none` | `binarytype` | Package format of the pool (`BSSolv::pool->settype`, `BSRepServer/Checker.pm:218-226`) and of `pkg_*` functions in obs-build. |
| `BuildEngine: mock|debootstrap|debbuild|podman` | `buildengine` | Re-maps recipe type in obs-build (`build-recipe:148-159`); drives `system-packages:*` sysdeps. |
| `Release: <CI_CNT>.<B_CNT>` / `Release: X kiwi:Y` | `release`, `release@` (`:532-534`) | Scheduler computes `release` for the job (`BSSched/BuildJob.pm:975-998`); worker passes `--release`. Per-buildtype overrides via `type:value` tokens. |
| `CICntStart:`, `ReleaseProg:`, `ReleaseSuffix:` | | Scheduler/ srcserver release handling. |
| `Target: arch` / `ChangeTarget:` | `target` + `%_target_cpu` macro | obs-build `--target` (`build-recipe-spec:202-204`, productcompose `ABUILD_TARGET`). |
| `HostArch: arch` | `hostarch` | Cross build host; job gets `hostarch`/`crossarch`; dispatcher routes to that host arch. |
| `Constraint: selector value` | `constraint[]` (`:546-552`) | Dispatcher constraints (3.2k). |
| `SingleExport:` | `singleexport` | Maintenance export rule (scheduler). |
| `ExpandFlags: flag[:value]` | `expandflags[]` + `expandflags:<flag>` keys (`:614-621`) | Known flags in code: `preinstallexpand` (expand preinstall/vminstall transitively, `expandpreinstalls` `:696-724`), `module:NAME-STREAM` (collected into `modules[]`, `:617`, `:636`), `dorecommends`, `dosupplements`, `ignoreconflicts`, `keepfilerequires` (Build::Expand), `unorderedimagerepos`, `filterbasecontainerpkgs`, `macroserial`, `noautoenablemodule` (scheduler), `docker-nobasepackages`/`kiwi-nobasepackages` (`get_build:740-744`). |
| `BuildFlags: flag[:value]` | `buildflags[]` + `buildflags:<flag>` keys (`:622-628`) | Consumed by the build path: `vmfstype`, `vmfsoptions` (worker `--vmdisk-filesystem[-options]`, `bs_worker:4197-4198`; also `build-vm:734-750`), `maxjobs` (`bs_worker:4155`), `allowrootforbuild` (`:4253`), `nodisturl` (`:4147`), `setvcs` (`:4149`), `slsaversion` (`:4091`), `logidlelimit` (job field, `BSSched/BuildJob.pm:1291`), `useccache:<pkg>`, `ccachetype` (`:1150-1165`), `obsgendiff` (`:1293-1300`, `build/build-recipe:315-331`), `signflavor`, `genmetaalgo`, `forcebinaryidmeta`, `modulemdplatform`, `kiwiprofile`, `sbom:<fmt>`, `dockerarg KEY=VAL`, `container-build-format`, `container-compression-format`, `rpm-buildroot-override`, `kvm-max-cpu-compat-power8`, `nouseforbuild:<pkg>`, `excludebuild:<pkg>`, `onlybuild:<pkg>`, `norelsync`, `aggregate-checklocal`, `productcompose-onlydirectrepos`. |
| `PublishFlags: flag` | `publishflags[]` | Publisher only (`ympdist`, `create_empty`, `withreports`, `withsbom`, `nofailedpackages`, `noearlykiwipublish`, `archsync`, `artifacthub`, `cosignbundle`, `mandatorycontainerplatforms`, `nocontentchange`). |
| `RepoURL:`, `RegistryURL:`, `AssetsURL:` | lists | pbuild / source assets (not bs_worker). |
| `OnlyNative:`, `AlsoNative:` | lists | Cross-build dependency classification (`BSSched/BuildJob/Package.pm:427-...`). |
| `DistMacro: name value` | macro | Adds `%define` to the macro set (`:466-470`). |
| `Macros:` ... `:Macros` | `rawmacros` (`:457-465`) | Verbatim rpm macros written to `~/.rpmmacros` in the build root (`recipe_setup_macros`, `build-recipe:242-258`); `Macros:` with a leading `!` line resets. |
| `FromScratch:` | handled by the config assembler | discard everything inherited from projects earlier in the path. |

Derived fields: `type`, `binarytype` (`:562-589`), `modules`, the helper hashes for the
expander (`init_helper_hashes`), and the `expandflags:`/`buildflags:`/`publishflags:`
lookup keys. The config object is used by: scheduler (`get_build`, `get_deps`,
`get_sysbuild`, `get_preinstalls`, `get_vminstalls`, `get_runscripts`, release
computation), repserver (`_buildinfo`, YMP, fileinfo_ext), worker (flags above,
`readbuildenv` fixups), obs-build (`queryconfig`, `expanddeps`, `init_buildsystem`,
recipes), publisher.

`Build::get_build` (`:726-792`) is the canonical "what goes into the build root"
function: `deps + build-packages:<type> substitutes + preinstall + required + support
- ignores - subpackages (unless Keep) -> do_subst -> expand`. `get_sysbuild`
(`:818-881`) returns the separate *host* environment for engines/images (mock,
debootstrap, livebuild, kiwi, productcompose, docker/podman, fissile, deltarpm,
mkosi): `system-packages:<x>` substitutes + preinstall + required (+ support for kiwi)
expanded; packages in sysdeps but not in bdeps get `installonly=1`, packages in bdeps
but not in sysdeps get `noinstall=1` (`BSSched/BuildJob.pm:1226-1229`) -- the latter
are the image content, placed in `repos/` rather than installed.

---

## 7. Catalog: recipe types and VM backends

### 7.1 obs-build `build` script contract (as used by bs_worker)

`build/build` (2011 lines): sources `build-vm`, `build-recipe`, `build-pkg`,
`build-validate-params` (`:53-62`); parses options (`:1042-1365`), VM options via
`vm_parse_options` (`build-vm:127-312`), recipe options via `recipe_parse_options`
(`build-recipe:74-116`). Flow for a VM build:

1. `vm_verify_options`/`vm_set_defaults` (`:1385-1388`), `vm_setup` (create/mkfs/mount
   root image and swap, `build-vm:820-889`), `mkdir_build_root`, logging wrapper that
   prefixes `[  NNNs]` timestamps (`:1474-1500`), `setmemorylimit` (chroot only).
2. `recipe_set_buildtype` from the recipe filename (`build-recipe:119-160`); for
   `*.src.rpm` the srpm is unpacked.
3. `vm_first_stage` (`build-vm:925-1163`): `init_buildsystem --prepare ... --rpmlist
   <rpmlist> <recipe>` (`:932`) = preinstall + vminstall into the mounted image
   (without running scripts); copy `$BUILD_DIR` to `$BUILD_ROOT/.build`, sources to
   `.build-srcdir`, old packages to `.build.oldpackages`; write `.build/build.data`
   (all relevant variables, `:987-1057`); umount; copy guest kernel/initrd out of the
   image if the preinstall image shipped one (`.build.kernel.kvm`, `:1098-1127`);
   start the log **watchdog** (`background_watchdog`, `build-vm:528-549`: kills the VM
   if 300s pass between `### VM INTERACTION START ###` and `END` markers -- i.e.
   boot/shutdown hangs) when `--vm-watchdog`; `vm_startup` (boots the VM with
   `init=/.build/build`); afterwards read `BUILDSTATUS<n>` from the first 12 bytes of
   the swap device and, for 0/2/9, run `extractbuild --disk ROOT --input SWAP --skip
   512` to recover the result files from the block list written by the guest
   (`:1131-1160`); exit with that status.
4. Inside the guest, `vm_detect_2nd_stage` (`build-vm:596-732`) re-sources
   `build.data`, mounts proc/sys/devpts/shm, switches personality, sets up swap,
   starts the statistics monitor (`DO_STATISTICS`: max disk/mem sampled every 5s,
   `:495-526`), then the same script runs the "chroot" path with `BUILD_ROOT=/`:
   `init_buildsystem` (full install, scripts), `recipe_setup` (copy sources to
   `$TOPDIR/SOURCES`), unpack `*.obscpio`, `recipe_setup_macros`, build-time services
   (`runservices`), `recipe_prepare`, `recipe_build`, post hooks, rpmlint/checks,
   baselibs, `recipe_compare_oldpackages` (-> `same_result_marker`),
   `recipe_create_deltarpms`, then `vm_wrapup_build <resultdirs> OTHER` which writes
   `TIME_*`/`MAX_*`/`IO_*` statistics and the block list of all result files into the
   swap device (`build-vm:1165-1225`), and `cleanup_and_exit` -> `vm_shutdown`
   writes `BUILDSTATUS<n>` and powers off (`build-vm:355-377`).
5. `cleanup_and_exit` codes: 0, 1, 2 (unchanged), 3 (retry elsewhere), 4 (fatal
   host), 9 (dynamic buildrequires changed) (`build:457-461`, `:1867`).

The worker also calls `build --kill` (`bs_worker:225-244`; `kill_running_build`
`build:1172-1187` -> `vm_kill` e.g. `fuser -k` on the root image for kvm),
`build --sysrq X` (`:246-262`) and `build --wipe` (`:264-280`).

### 7.2 Recipe types (`build/build-recipe:35`, `:119-160`)

| Recipe file | BUILDTYPE | Engine variants | Build user | Result dirs sent (`bs_worker:5086-5097`) | Post-processing / special files |
|---|---|---|---|---|---|
| `*.spec`, `*.src.rpm` | `spec` (`build-recipe-spec`) | `BuildEngine: mock` -> `mock` (chroot `mock --buildsrpm`/`--rebuild` against a yum repo of `.build.binaries`, `build-recipe-mock:32-95`); `debbuild` -> `debbuild` (spec -> deb, `build-recipe-debbuild`) | abuild unless `# needsrootforbuild` | `RPMS/<arch>`, `SRPMS`, `OTHER` (debbuild: `DEBS/<arch>`, `SDEBS`) | `rpmbuild` with `--define _srcdefattr`, `--nosignature`, `--target`, `--buildroot` override, `_build_create_debug`, `disturl` (`:170-293`); `%generate_buildrequires` -> `OTHER/_generated_buildreqs` + exit 9 (`:245-271`, `:292`); rpmlint -> `OTHER/rpmlint.log`; `same-build-result.sh` vs `.build.oldpackages` -> `same_result_marker`; deltarpms; baselibs (`mkbaselibs`); `_modulemd.yaml` via `writemodulemd`; `.changes` -> `%changelog` (`--changelog`). |
| `*.dsc`, `debian.control`, `debian/control` | `dsc` (`build-recipe-dsc`) | `BuildEngine: debootstrap` -> `debootstrap` (builds inside a debootstrap chroot of `.build.binaries`, `build-recipe-debootstrap:32-117`) | abuild | `DEBS` | `dpkg-buildpackage` via `dsc_build`; `pristine-tar` orig regeneration; `*.changes` moved to `DEBS` (`:190-205`). |
| `PKGBUILD` | `arch` | | abuild | `ARCHPKGS` | `makepkg -ef` + `--allsource` with `PKGDEST=$TOPDIR/ARCHPKGS` (`build-recipe-arch:24-66`). |
| `APKBUILD` | (handled as `apk` in obs-build upstream; worker expects `APKS`) | | | `APKS` | |
| `build.collax` | `collax` | | | `DEBS` | runs `./build` in SOURCES (`build-recipe-collax`). |
| `*.kiwi` | `kiwi` (`build-recipe-kiwi`, 953 lines) | image vs product by `<type image="product">` | **root** for images, abuild for products (`build:1710-1713`) | `KIWI` (images, incl. subdirs via `.kiwitree`) ; products likewise `KIWI` | repos from `$TOPDIR/SOURCES/repos/<prj>/<repo>` get `createrepo`/`dpkg-scanpackages`/`repo-add` metadata (`:830-865`); `root.tar*` unpacked; `copytoimage`; kiwi (legacy `/usr/sbin/kiwi` or kiwi-ng) `perform_image_build` + `perform_image_bundle` per profile/type (`:654-680`); `kiwi_post_run` hook; for `docker` image types `postprocess_kiwi_containers` (`:690-754`): uncompress `.tar.xz`, write `.containerinfo` (`Build::Kiwi::showcontainerinfo`), SBOMs, `.pkgsummaries`, `.basepackages`; `.packages`, `.milestone`, `.sha256` files; products: `build_kiwi_product` + `perform_product_bundle`, `.report`, `_channel` created by the worker. |
| `*.productcompose` | `productcompose` | | abuild | `PRODUCT` (+ `.kiwitree`) | `product-composer build --release R --flavor F` (`build-recipe-productcompose:58-69`); worker `annotatereport(PRODUCT)`; `RPMHDRS_ONLY`-style header-only inputs supported via `view=cpioheaderchksums`. |
| `Dockerfile`, `Dockerfile.*` | `docker` (`build-recipe-docker`, 345 lines) | `BuildEngine: podman` -> `podman` (`call-podman --root`, `build-recipe-podman`) | **root** | `DOCKER` | base images loaded from `containers/` (`docker load` or `docker import`, `:148-164`); `createrepo`/`dpkg-scanpackages` over `SOURCES/repos` served to the container by `startdockerd --webserver` (`obs-docker-support` injected by `patchdockerfile`, `:231-235`); tags from `Build::Docker::show containertags` or a `TAG` file; `--build-arg` from `BuildFlags: dockerarg`; `--squash` unless `#!NoSquash`; `LABEL org.openbuildservice.disturl`; `docker save` (or `podman push oci-archive:` with zstd) -> `DOCKER/<name>.<arch>[-<release>].tar` + `.containerinfo` (`Build::Docker::showcontainerinfo`) + `.packages`/`.basepackages`/`.pkgsummaries` + SBOM `.spdx.json`/`.cdx.json` (`:278-323`); `Dockerfile.dapper` runs the image and copies out `DAPPER_OUTPUT` (`:69-118`). |
| `fissile.yml` | `fissile` | | root | `FISSILE` | docker load + `fissile build packages/images`, each image saved + `.containerinfo` (`build-recipe-fissile:45-128`). |
| `Chart.yaml` | `helm` | | abuild | `HELM` | `helm package` -> `HELM/<name>-<version>.tgz` + `<name>-<version>.helminfo` (`Build::Helm::show helminfo`) (`build-recipe-helm:62-85`); the scheduler/publisher later turn the chart into an OCI artifact (`BSContar::container_from_helm`). |
| `flatpak.yaml|yml|json` | `flatpak` | | | `OTHER` | unpack runtime tars from `/var/lib/flatpak/tars/*.tar.gz` into the flatpak runtime dir, rewrite URLs to file://, `call-flatpak-builder --root` (`build-recipe-flatpak:52-104`). |
| `appimage.yml` | `appimage` | | root | `OTHER` | needs `.build.binaries` (all deps as a local yum/apt repo, `recipe_needs_build_binaries`), `pkg2appimage` -> `*.AppImage[.digest|.zsync]` (`build-recipe-appimage:55-116`). |
| `snapcraft.yaml` | `snapcraft` | | root | `OTHER` | apt repo from `.build.binaries`, `snapcraft pull/build/snap` -> `*.snap` (`build-recipe-snapcraft:66-118`). |
| `*.livebuild` | `livebuild` | | root | `OTHER` | Debian live-build with local apt repo; results + `.sha256` + the recipe tarball (`build-recipe-livebuild:124-268`). |
| `mkosi.*` | `mkosi` | | | `OTHER` | local rpm/apt/arch repo from `.build.binaries`, `mkosi ... --output-dir OTHER --checksum --with-network=never --local-mirror file:///.build.binaries/` (`build-recipe-mkosi:43-119`); manifests gzipped. |
| `simpleimage` | `simpleimage` | | root | `OTHER` | run the `%build` section in the chroot, then tar/squashfs the root (`build-recipe-simpleimage:39-108`). |
| `_preinstallimage` | `preinstallimage` | | | `OTHER` | see 2.7. |
| worker-synthesised: `deltagen.spec` (`_delta`), `ptf.spec` (`_ptf`), `reproduciblecheck.spec` (`_reproduciblecheck`) | `spec` | | | `OTHER`/`RPMS` | `worker/worker-deltagen.spec` runs `makedeltarpm` over `*.old/*.new` pairs from the job data; reproducible check rebuilds and compares. |
| `_aggregate`, `_patchinfo`, `_channel` | not built on workers | | | | handled by the scheduler as fake jobs (`BSSched/BuildJob/Aggregate.pm:831`, `fakejobfinished_nouseforbuild`). |

`recipe_needs_build_binaries` (`build-recipe:292-313`): mock, debootstrap, livebuild,
snapcraft, appimage, docker, fissile, podman, mkosi and any recipe with
`# needsbinariesforbuild` get `init_buildsystem --create-build-binaries`, which
copies every dependency package into `$BUILD_ROOT/.build.binaries` so the recipe
can create a local repository; the worker in turn disables preinstall images and
collects `kiwiorigins` for the `.report` (`bs_worker:4032-4038`).

### 7.3 VM / sandbox backends (`build/build-vm:68-70`, option parsing `:143-172`)

| `--vm-type` | File | Host needs | Guest boot | Root/swap | Kill | Notes |
|---|---|---|---|---|---|---|
| `kvm` | `build-vm-kvm` (474) | `/dev/kvm`, `qemu-kvm`/`kvm`/`qemu-system-<arch>`, kernel+initrd (`vm_verify_options_kvm:63-278`), `fuser` | `qemu -nodefaults -no-reboot -nographic -vga none -cpu host -M pc,accel=kvm,usb=off,dump-guest-core=off,vmport=off -sandbox on [-bios qboot.rom] -kernel K -initrd I -append "root=/dev/disk/by-id/virtio-0 rootfstype=F rootflags=... kvmclock mitigations=off elevator=noop nmi_watchdog=0 rw rd.driver.pre=binfmt_misc oops=panic panic=1 quiet console=ttyS0 init=/.build/build" -m MEM -drive file=ROOT,format=raw,if=none,id=disk,cache=unsafe[,aio=io_uring] -device virtio-blk-pci[,iothread=io0],drive=disk,serial=0 -drive ...swap... -serial stdio -chardev socket,...monitor -smp JOBS[,threads=T] -net none [-runas qemu] -object rng-random,... -device virtio-rng-pci` (`:309-383`) | file or block device; virtio ids 0/1; ide fallback | `fuser -k -TERM ROOT` then `-k` (`:385-398`) | per-arch overrides for arm/aarch64 (`-M virt gic-version=host`, 32-bit guest kernels), ppc64 (`pseries`, `cap-ccf-assist`, `max-cpu-compat=power8`), s390x (ccw devices, `hvc0`), riscv64; hugetlbfs `-mem-prealloc -mem-path`; `--vm-network` adds SLIRP user net; sysrq via monitor socket `sendkey alt-print-X` (`:463-470`). "secure". |
| `qemu` | `build-vm-qemu` (274) | `qemu-system-<target arch>` for *foreign* arch TCG emulation, target kernel/initrd | same arg style, `-cpu cortex-a72/rv64/qemu64/...`, firmware for riscv/hppa/loongarch | | | slow; used for arches without native workers. |
| `xen` | `build-vm-xen` (143) | Xen dom0, `xl` (or `xm`), `vmlinuz-xen`/`initrd-xen` | `xl create -c <conf> name=build_<id> disk=[...] extra="... init=/.build/build console=hvc0"` (`:51-94`) | `file:`/`phy:` disks `xvda/xvdb` | `xl destroy` | "secure"; xenstore size check in worker. |
| `zvm` | `build-vm-zvm` (438) | s390x z/VM guest with `vmcp`, `iucvconn`, DASD volumes `ZVM_VOLUME_ROOT/SWAP`, worker guest names | `zvm_cp volume_attach/ipl`, console over IUCV (`:294-313`) | minidisks | `vmcp destroy` | "secure"; `--vm-worker NAME --vm-worker-nr N`. |
| `pvm` | `build-vm-pvm` (197) | IBM PowerVM: `pvmctl`, `hvcsadmin`, LVM `rootvg` | creates an LPAR per build, grub image generated at `vm_fixup_pvm` (`:160-196`), console via hvcs | LVs attached via vSCSI | `lpar power-off --hard` | |
| `openstack` | `build-vm-openstack` (282) | `nova`, `cinder` CLIs, `OS_*` env, cinder volumes for boot/root/swap, `--vm-server` (controller instance) | `nova boot --flavor F --block-device ...` x3, serial console via `openstack-console` websocket (`:253-281`) | cinder volumes attached to the controller for prepare/extract (`:48-120`) | `nova delete` | the worker itself runs in an OpenStack instance. |
| `ec2` | `build-vm-ec2` (251) | `ec2-*` tools, `AWS_ACCESS_KEY/SECRET`, EBS volumes, AKI | run-instance with root/swap snapshots | EBS | terminate | legacy. |
| `emulator` | `build-vm-emulator` (97) | `emulator/emulator.sh` (site-provided, e.g. aranym, qemu-system wrappers) | `emulator.sh [args] ROOT SWAP` (`:32-62`) | file image; if no swap, results are copied by loop-mounting the root (unsafe) | `fuser -k` | `--emulator-script`. |
| `uml` | `build-vm-uml` (74) | user-mode-linux kernel | | | | legacy. |
| `lxc` | `build-vm-lxc` (223) | `lxc-create/start/destroy` (standalone) or libvirt `virsh -c lxc:///` | container with `$BUILD_ROOT` bind-mounted, `lxc.conf` template (`:54-103`) | none (`VM_ROOT=`) | `lxc-stop` | result via `.build/_exitcode` file; not secure. |
| `docker` | `build-vm-docker` (83) | docker daemon | `docker run --rm --net=none [--privileged ...] --mount bind BUILD_ROOT:/mnt busybox chroot /mnt /.build/build` (`:30-44`) | none | `docker stop` | `docker:privileged` variant. |
| `podman` | `build-vm-podman` (90) | podman, `build-scratch:latest` image created on the fly (`:49-54`) | `podman run --rm --net=none --mount bind BUILD_ROOT:/ build-scratch /.build/build` | none | `podman stop` | rootless-capable (`podman unshare rm -rf` for wipe). |
| `nspawn` | `build-vm-nspawn` (83) | `systemd-nspawn`, `machinectl` | `systemd-nspawn -D BUILD_ROOT -M name --private-network [--pipe] [--privileged] /.build/build` | none | `machinectl terminate` | |
| (none) chroot | `build` itself | root, `chroot`, `killchroot` | `chroot $BUILD_ROOT su - $BUILD_USER -c /.build.command` | n/a | `killchroot -s 9` | no isolation; `setmemorylimit` ulimit to 2/3 of RAM+swap (`build:730-754`). |

Common VM mechanics (`build-vm`): root fs types ext2/3/4, xfs, btrfs, reiserfs with
tuned mkfs/mount options (`:423-493`, `:752-770`); `--vm-use-mkfs-copyin` populates the
image with `mkfs.ext4 -d` instead of mounting (needed for unprivileged hosts,
`:772-792`, `:853-873`); `--vmdisk-clean` recreates images each build; the guest
`/.build/build.data` carries every parameter (`:987-1057`); `BUILDSTATUS` protocol on
the swap device (`:315-321`, `:355-377`); result extraction via
`computeblocklists`/`extractbuild` (FIBMAP/FIEMAP block lists written into the swap
device, read back from the raw root image on the host without mounting it -- the
security boundary) (`build/computeblocklists:1-60`, `build/extractbuild`).

### 7.4 `init_buildsystem` and the rpmlist contract

`build/init_buildsystem` (1295 lines) is the only consumer of the worker-written
rpmlist. Key options (`:60-120`): `--prepare` (first stage, no scripts, used by VMs),
`--rpmlist FILE` (skip `expanddeps`, trust the list), `--create-build-binaries`,
`--clean`, `--configdir`, `--cachedir`, `--buildflavor`, `--obspackage`, `--define/
--with/--without`, `--repository` (zypp/http/dir repos for standalone use).
With `--rpmlist` the flow is: parse tags (`:725-793`); `preinstall_image` (unpack via
`bsdtar --chroot`, `:277-296`) and filter already-contained packages (`:903-909`);
`preinstall` each preinstall/vminstall package (`cpio`-level unpack, `:298-315`,
`pkg_preinstall`); `run_pkg_scripts`, `pkg_initdb` (`:941-942`); then the main
install loop (`:1091-1160`): compares `installed-pkg/<pkg>` ids with the package's
buildid (`getbuildids`), erases/re-installs changed packages, records `hdrmd5 pkgid`
into `.preinstall_image/<pkg>` for image creation; sysroot packages into
`/build-sysroot` (`:1160-1192`). `getbinaryid --createmeta` (`build/getbinaryid`)
is the standalone helper that computes the same hdrmd5 ids for non-OBS use.

---

## 8. Implications for a distributed redesign

### 8.1 What a worker really needs

Reading `bs_worker` end to end, the worker's dependencies on the backend reduce to:

1. **A job spec** (`$BSXML::buildinfo`): identity (prpa, package, job id), the recipe
   filename, the fully resolved dependency list with per-dep flags
   (`preinstall/vminstall/runscripts/notmeta/noinstall/installonly/sysroot`), the
   expanded project config text (or a URL for it), release/bcnt/disturl/vcs, limits
   (`logidlelimit/logsizelimit`), feature switches (`debuginfo, ccache, slsaprovenance,
   signflavor, module*, modularity_*`), and the *locations* (`path/syspath/
   containerpath` with `server`) from which inputs can be fetched. Everything the
   worker computes locally from this (`meta`, rpmlist, build args, SLSA materials) is a
   pure function of the spec plus fetched content.
2. **Content-addressed artifact fetch**: every binary input is identified by a
   content hash the worker verifies after download (`hdrmd5` = signature-header MD5
   for rpm, `leadsigmd5`, `md5sum` for non-package files, sha256 for `_blob.*`,
   `verifymd5` for the source tree). The current protocol however addresses inputs by
   *(prpa, name)* (`/getbinaries?binaries=name,...`) and uses a separate
   `/getbinaryversions` round-trip to learn the hashes -- a repo-state-dependent lookup
   that is a source of races (`sendbadpackagebinaryversionlist`, `scanrepo` events)
   and of the `now=` "waited N seconds" diagnostics. A redesign should put the hash in
   the job (`_buildinfo` already does for `dobuildinfo`: `bdep hdrmd5`) and let the
   worker fetch `GET /blob/<hash>` from any replica/CDN/proxy, with the `.meta`
   fingerprint file attached as another hashed blob (`metamd5` is already published).
   The worker's cache (`cachedir/<sha256(prpa/hdrmd5)>`) is already a CAS; dropping the
   `prpa` from the key makes it shareable across repositories.
3. **Sources**: `GET /getsources?project&package&srcmd5` is already content-addressed
   by `srcmd5` and verified by `verifymd5`.
4. **Result upload**: a single streamed multi-file upload with a result code and the
   job id as idempotency key; the repserver already treats `(job, jobid, workerid,
   starttime)` as the claim token. A redesign needs an explicit *lease* (see 8.3) and
   should make the upload resumable or at least retried beyond the current single
   60-second sleep (`bs_worker:5250-5252`).
5. **Control plane**: `start(job)`, `kill`, `discard`, `sysrq`, `logtail(start,end)`,
   `liveness`. Today these are inbound HTTP to the worker; an outbound long-poll or a
   bidirectional stream (gRPC) from the worker removes the need for routable worker
   IPs, the `uri` derived from the TCP peer address (`bs_repserver:1800`), and the
   `checkclientconnectivity` probe.
6. **Code distribution**: the worker self-updates `worker/` and `build/` from the
   backend by md5 (`/getworkercode`, `/getbuildcode`). A container image per obs-build
   version replaces this; the dispatcher's `buildcode` md5 becomes an image digest in
   the job spec.
7. **Config**: `BSConfig` values the worker reads: `buildlog_maxsize/maxidle`,
   `workerhostcheck`, `norootexceptions`, `localarch`, `nokiwitree`, `deltagen_mopt`,
   `srcserver`, `reposerver(s)`, `ssl_*` (`bs_worker:120-126`, `4253`, `5108`, `4141`).
   All of these can move into the job spec or agent config.

### 8.2 Where the shared filesystem is assumed

| Coupling | Files | Who |
|---|---|---|
| Job queue and lease | `jobs/<arch>/<job>`, `:status` (flock + link atomic create), `:dir`, `:cross` markers | scheduler writes/deletes; dispatcher claims; repserver completes; warden cleans; `/workerstatus` and `_result` read them for status |
| Worker registry | `workers/{idle,building,away,down,dead,disable}` | repserver writes; dispatcher reads idle + building, moves to building/down; warden moves to down; `/workerstatus` reads |
| Load model | `jobs/finished` (append), `jobs/load`, `jobs/mdload`, `jobs/dispatchprios` | repserver appends; dispatcher owns |
| Events | `events/<arch>/*`, `events/dispatch`, `events/signer`, `events/publish`, `events/repository`, `.ping` | repserver/dispatcher/scheduler/signer/publisher |
| Repository tree | `build/<prpa>/...` (`:full`, `:full.solv`, `:bininfo`, `:meta`, `:packstatus`, `:preinstallimages`, package dirs) | scheduler writes; repserver serves to workers/API; `putjob` hardlinks kiwitree files from it; SLSA store hardlinks from it |
| Result ingest | `rename(jobs/<arch>/.putjob.N, <job>:dir)` then scheduler `rename` into `build/...` -- relies on same filesystem for atomic moves and hardlinks | repserver -> scheduler |
| Daemon liveness | `run/*.lock` flock probing in `/workerstatus` | |
| Container blobs | `blobs/sha256/` hardlinked into `:full` and registry dirs | repserver DoD, publisher |
| Code | `worker/`, `build/` directories served by md5 | repserver, dispatcher (`getcodemd5` must see the same files) |

The dispatcher, warden and the repserver's job endpoints are the pieces that must be
re-homed first; the scheduler/publisher coupling to `build/` is outside this report
but note that the *only* contract between repserver and scheduler for job completion
is "`<job>:dir` exists with `.bininfo`, `:status` says finished, and an event file
appeared".

### 8.3 Dispatcher as a replicated / sharded service

What the algorithm needs as inputs (3.2):

* The set of ready jobs per arch with: `prpa`, `package`, `readytime`, `needed`,
  `reason` (source change vs rebuild), `revtime`, `hostarch`, constraints (3 sources),
  `file` type (`_preinstallimage` priority, `_aggregate` exclusion), job XML md5.
* The set of idle workers per host arch with their capability document, grouped by
  physical host (instance count idle/busy) for spreading.
* A per-prpa load figure with exponential decay fed by completed-build durations, a
  per-prpa "last build" time, per-project newest source change time, admin priority
  adjustments, and the badhost table (24h TTL) with attempt counts.
* An atomic claim on a (job, worker) pair and a way to tell the scheduler
  "dispatch details" / "fail job".

All of this is small (the dispatcher already keeps `%infocache` in memory and strips
`bdep`) and naturally lives in a database/KV with a lease: `jobs` table with
`state ∈ {ready, leased, running, finished, signing}`, `lease_owner`, `lease_expiry`;
`workers` table with `state`, `capabilities`, `host`, `last_heartbeat`; a `load`
table keyed by prpa. Sharding by **arch** is natural (jobs, workers and load are
all keyed by arch; the only cross-arch interaction is the `:cross` marker, i.e. a job
for arch A that must run on hostarch H -- place it in H's shard with `target_arch=A`,
exactly what `add_crossmarker` emulates). Replication within a shard needs only the
atomic claim (compare-and-swap on `state=ready`), which replaces `lockcreatexml`.

The **locality signal** the current system lacks: the worker's cache contents.
Options: (a) workers publish a Bloom filter / count of cached `hdrmd5`s with their
heartbeat and the dispatcher scores candidate workers by overlap with the job's
`bdep hdrmd5` list (the job already carries all hashes when generated with
`dobuildinfo`; the preinstall-image bitstring in 2.7 is exactly this idea applied
server-side); (b) sticky routing of a prpa to a worker pool (the per-host spreading
already approximates "don't put all jobs of a prpa on one host" -- the inverse); (c)
rely on a shared regional cache (`bs_getbinariesproxy` role) and ignore worker-local
cache in placement. The dominant win in practice is preinstall images plus a warm
regional CAS; worker-local affinity matters mostly for huge image builds.

Keep from the current design: source-change-first ordering, `needed` (blocked
count) priority, decayed per-prpa load with project-level smoothing and admin
adjustments, badhost quarantine with bounded retries and `nobadhost` fail-over,
constraint oracle semantics (including `secure` sandbox class and `nativeonly`),
`checkconstraints` feedback to users ("no compliant workers"), `powerpkgs`/
`powerhosts`, and the 50-assignments-per-60s pass limit (fairness under backlog).

### 8.4 A lightweight worker agent for autoscaled VMs/containers

The agent is the subset of `bs_worker` that is not about the Perl HTTP server and the
shared-code bootstrap:

1. Register with capabilities (reuse the probing in `bs_worker:788-902`; emit the
   same `worker` document, as JSON), then **pull** jobs (long-poll/stream) instead of
   waiting for a `PUT /build`.
2. For a job: fetch sources (by `srcmd5`), fetch inputs by hash through a CAS
   endpoint with a local LRU cache (the existing cache layout and verification logic
   can be ported 1:1), optionally fetch a preinstall image chosen by the server (the
   server has all the bitstrings; moving selection server-side removes two RPC
   round-trips from the worker), write `.build.config`, `.build.rpmlist`, `.build.meta`
   exactly as today, run `build`.
3. Stream the log (push chunks or serve a pull endpoint), accept `kill/discard/
   sysrq`, send heartbeats carrying the lease.
4. Collect results per 4.5.6 (including `.kiwitree`-style tree manifests and container
   normalisation, which should stay on the worker since it is CPU-heavy), compute the
   SLSA statement, upload as a multipart/tar stream keyed by lease, retry with
   backoff, then release.

**Can obs-build be used unchanged?** Yes for the build itself: the worker->build
contract is the command line in 4.5.4 plus the three files `.build.config`,
`.build.rpmlist`, `.build.meta` and the directory conventions
(`.build-srcdir`, `.pkgs`, `.build.oldpackages`, `.build.packages`, `.build.log`,
`.mount`), all of which an agent in any language can produce. obs-build also needs
`Build.pm` (for `queryhdrmd5`/`query` the worker currently imports Perl `Build`
modules; an agent would shell out to `getbinaryid`/`build/queryconfig` or
reimplement the rpm signature-header MD5 -- `bs_repserver:509-547` `getrpmheaders`
shows the format: lead 96 bytes, signature header, md5 at tag 1004). The parts of
`bs_worker` that are *not* obs-build and must be reimplemented are: cache, fetch
orchestration (`getbinaries*`, `getpreinstallimage`, product/image input trees),
meta generation (`BSBuild::gen_meta`, 120 lines), report/channel generation,
kiwitree, container normalisation (`BSContar`, 728 lines; or keep `obs-gzip-go` +
a small Go port), SLSA statement, result packaging. Everything in `build/` stays
as is.

**Host requirements for nested virtualization** (from 4.7 and 7.3): a KVM worker
inside a cloud VM needs nested virt exposed (`/dev/kvm` present and `kvm_intel/amd
nested=1` on the outer hypervisor), `qemu-kvm`, a guest kernel+initrd pair with
virtio-blk/virtio-rng/serial drivers (OBS ships `kernel-obs-build`), root privileges
(or at minimum CAP_SYS_ADMIN for mounts, loop devices or `mkfs -d` copy-in mode with
`--vm-use-mkfs-copyin`, which exists precisely for unprivileged hosts), `bsdtar`,
`fuser`, `mkfs.*`, enough disk for root+swap images per instance, and ideally
hugepages. Without nested virt, the choices are `podman`/`nspawn`/`docker` sandboxes
(insecure for untrusted content; constraints `sandbox=secure` would exclude them),
`qemu` TCG (slow), or running the worker agent *as* the VM: boot a cloud VM per job
whose init is the agent itself and which uses the chroot backend (`VM_TYPE=` empty) --
isolation then comes from the outer hypervisor, and `build-vm-openstack`/`ec2` show
the prior art of driving cloud volumes for root/swap. In that model the agent
replaces the swap-device result channel by a direct upload from inside the VM, which
is what `bs_worker --test`/`--build` plus `build` in chroot mode already do on a
single host.

### 8.5 Behaviours worth preserving as explicit requirements

* Job identity = hash of the full job spec; results are accepted only for the exact
  spec that was dispatched (`jobid` everywhere).
* A result must never be ingested twice or for a superseded job (`:status` jobid/
  workerid/starttime triple; `job_is_waiting`/`purgejob` ordering).
* `unchanged` is a first-class result that keeps the previous binaries and logfile.
* `badhost` vs `failed` separation with bounded retries and the "broken config, not
  broken host" heuristics (`Exec format error`).
* `genbuildreqs` loop for dynamic BuildRequires.
* Log size/idle limits enforced by the worker, log streaming while building.
* Sign-after-build hand-off (`signing` state) before the scheduler sees the result.
* Time-skew check (or better, server-assigned timestamps).
* Per-prpa fairness with decayed load, source-change priority, `needed` ordering.
* Preinstall images and the bitstring subset match.
* Worker capability model: `hostlabel`, `sandbox`, `linux`, `hardware` (+ synthetic
  x86-64 levels, `nativeonly`), and the `cando` host-arch -> target-arch table with
  personality helpers.
