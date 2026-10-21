# OBS Backend Scheduler (`bs_sched` + `BSSched::*`) — Engineering Analysis

Scope: `src/backend/bs_sched`, `src/backend/BSSched/**`, `BSBuild.pm`, `BSCando.pm`,
`bs_dodup`, the parts of `build/Build.pm` + `build/Build/Expand.pm` the scheduler calls,
and the surface of the external `perl-BSSolv` XS binding as used by the scheduler.
All paths below are relative to `src/backend/` unless absolute. Line numbers refer to the
checked-out tree at commit `db93ae665b`.

The purpose of this document is to give enough detail for a ground-up re-architecture
(Go microservices). It is organized as:

1. Process model, startup, in-memory state, on-disk state, checkpointing
2. Event model (sources, event types, queue semantics, priority / lookat scheduling)
3. The per-package check algorithm (state machines, dep expansion, cycles, rebuild/block
   modes, `:full` vs `:repo`, meta comparison, lastcheck cache)
4. Build-job creation and result collection (job file format, per-recipe handlers,
   jobhistory, dispatchdetails)
5. Repositories as build dependencies (search path, RepoCache, remote/interconnect,
   DoD, bininfo, publish trigger, relsync)
6. Multi-arch / cross-prp coupling
7. Concurrency, consistency and scaling limits
8. Implications for a distributed redesign

Quick inventory of the code read (line counts from `wc -l`):

| File | Lines | Role |
|---|---|---|
| `bs_sched` | 1254 | process entry, startup, main loop |
| `BSSched/Checker.pm` | 1874 | per-prp check pipeline: setup/preparepool/expandandsort/checkpkgs/publish |
| `BSSched/BuildJob.pm` | 1611 | job creation (`create`, `writejob`), job completion (`jobfinished`), meta diff |
| `BSSched/BuildJob/Package.pm` | 469 | default recipe handler (spec/dsc/arch/...): rebuild decision |
| `BSSched/BuildJob/{Aggregate,Channel,Patchinfo,KiwiImage,KiwiProduct,ProductCompose,Docker,Image,PreInstallImage,DeltaRpm,Import,Upload,BuildEnv,Reproduciblecheck,Unknown}.pm` | 945/387/793/355/697/791/462/106/192/206/157/96/172/426/77 | per-type handlers |
| `BSSched/BuildResult.pm` | 945 | moving results into `<prpa>/<pkg>` and `:full`; bininfo; wipe |
| `BSSched/BuildRepo.pm` | 972 | `:full` tree maintenance, `:full.solv`, useforbuild, scanning |
| `BSSched/ProjPacks.pm` | 1965 | projpack fetch (sync/async), postprocess: search paths, prp DAG, links |
| `BSSched/PublishRepo.pm` | 759 | `:repo` population, deltas, publish event |
| `BSSched/Remote.pm` | 807 | interconnect: remote watches, remote `:full`, remote bininfo/packstatus |
| `BSSched/RepoCache.pm` | 123 | in-memory cache of solv data per prpa |
| `BSSched/DoD.pm` | 483 | download-on-demand glue |
| `BSSched/EventHandler.pm` | 709 | all event handlers |
| `BSSched/EventQueue.pm` | 321 | ordering/processing of a batch of events |
| `BSSched/EventSource/{Directory,Retry,RemoteWatcher}.pm` | 199/102/219 | event sources |
| `BSSched/Lookat.pm` | 271 | the priority queues of prps to (re)check |
| `BSSched/RPC.pm` | 397 | async RPC multiplexer (`xrpc`) |
| `BSSched/Access.pm` | 117 | access/sourceaccess/role checks |
| `BSSched/Bininfo.pm`, `Blobstore.pm`, `Modulemd.pm` | 124/86/177 | helpers |
| `BSBuild.pm` | 159 | perl fallback for `add_meta`/`gen_meta`/`diffdepth` |
| `BSCando.pm` | 90 | host-arch -> buildable-arch table, `%knownarch` |
| `bs_dodup` | 1113 | DoD metadata updater daemon |

Terminology used throughout:

* **prp** = `project/repository` string, e.g. `openSUSE:Factory/standard`.
* **prpa** = `project/repository/arch`.
* **gdst** = `$reporoot/<prp>/<arch>` = `/srv/obs/build/<project>/<repo>/<arch>` (the
  per-prpa directory on the shared build filesystem).
* **projpacks** = the scheduler's in-memory copy of all project + package metadata
  for its architecture, obtained from `bs_srcserver` via `/getprojpack`.
* **gctx** = global context hash (one per process); **ctx** = per-prp check context
  (`BSSched::Checker` object); **ectx** = event-queue context; **fctx** = full-tree
  update context; **rctx** = async RPC context.

---

## 1. Process model

### 1.1 One process per architecture

`bs_sched` takes exactly one argument, the architecture (`bs_sched [--exit|--stop|--restart|--status] <arch>`,
`bs_sched:117-122`). The arch must exist in `%BSCando::knownarch` (`bs_sched:119`,
`BSCando.pm:78-88`). All of the state it owns is keyed by that arch:

* the lock file `$rundir/bs_sched.<arch>.lock` (flock LOCK_EX|LOCK_NB, `bs_sched:430-436`);
* the event directory `$bsdir/events/<arch>/` with a FIFO `.ping` inside it (`bs_sched:447-454`);
* the job directory `$bsdir/jobs/<arch>/` (`bs_sched:442-445`);
* every per-prpa directory `$bsdir/build/<proj>/<repo>/<arch>/`.

The `--status/--exit/--restart` modes do not start a scheduler: they check the flock or
write an `exitcomplete`/`restart` event into `events/<arch>/` and wait for the file to
disappear (`bs_sched:387-428`).

A "repo server" installation runs one `bs_sched` per arch, plus one `bs_dispatch`,
one `bs_publish`, one `bs_signer`, one `bs_repserver`, optionally one `bs_dodup`,
all sharing `$bsdir` (`/srv/obs`). `BSConfig::partition` allows several such
installations ("partitions") behind one `bs_srcserver` (`BSConfig.pm.template:242-250`).
Nothing in the scheduler talks to another scheduler by network: cross-arch and
cross-daemon communication is via files in the shared `$bsdir` (section 2.8, 6, 7.3).

Privileges: `BSUtil::drop_privs_to($bsuser,$bsgroup)` (`bs_sched:86`), and
`BSUtil::set_fdatasync_before_rename()` unless `$BSConfig::disable_data_sync`
(`bs_sched:88`) — every `writexml`/`store` of an important file does
`fdatasync` then `rename` (`BSUtil.pm:71-80, 562-581`).

### 1.2 Startup sequence

`bs_sched:384-795`, in order:

1. Create event dir + `.ping` FIFO; open the FIFO O_RDWR|O_NONBLOCK (`bs_sched:447-454`).
2. `BSSched::Remote::setup_authenticator()` for interconnect signature auth (`bs_sched:457`, `Remote.pm:797-805`).
3. Build `$gctx` (`bs_sched:460-561`, enumerated in 1.3).
4. Negotiate the "genmeta algorithm" between `BSBuild` and `BSSolv` (`bs_sched:563-587`):
   `$BSConfig::genmetaalgo` or the max both support. Dies if perl-BSSolv is too old.
5. Create `$rctx = BSSched::RPC->new(maxserverload=>$BSConfig::sched_maxserverload||1, wakeupfunction=>\&BSSched::Checker::setchanged)` (`bs_sched:590-594`).
6. **Warm restart**: if `$rundir/bs_sched.<arch>.state` exists (written by a previous
   `exitcomplete`/`restart`/`dumpstate`), retrieve it (Storable), unlink it, and restore
   (`bs_sched:603-734`):
   * `projpacks` + `remoteprojs` (if the dump was "complete"), then
     `get_projpacks_postprocess` (`bs_sched:610-630`);
   * `lookat_low/med/high` from `lookat/lookat_oob/lookat_oobhigh` (`bs_sched:638-640`);
   * `changed_low/med/high` entries that still refer to known prps/projects (`bs_sched:651-663`);
     any prp that is new since the dump is put in `changed_med` with level 2 (`bs_sched:634-649`);
   * `prpfinished`, `prpnotready` (dumped as `globalnotready`), `repounchanged`,
     `delayedfetchprojpacks`, `projsuspended`, `watchremote_start` (`bs_sched:671-710`);
   * packages marked `error => 'delayed startup'` schedule a delayed `/all` fetch (`bs_sched:713-718`);
   * `fetchprojpacks` requests that were in flight are re-issued synchronously (`bs_sched:720-723`);
   * `retryevents` are re-queued (`bs_sched:724-732`).
7. Clear `booting` in `info/schedulerinfo.<arch>` (`bs_sched:736-739`).
8. **Cold start** if there are no projpacks (`bs_sched:741-758`):
   * `sched_startupmode` 0 (default): `get_projpacks($gctx, undef)` — one giant synchronous
     `GET /getprojpack?withsrcmd5&withdeps&withrepos&withconfig&withremotemap&arch=<arch>[&partition=..]&view=storable`
     (`ProjPacks.pm:150-204`), then `get_projpacks_postprocess` (`ProjPacks.pm:713-726`).
   * mode 1: same but `noremote=1` — remote (interconnect) packages come back with an
     `interconnect error: noremote option` and are converted to `'delayed startup'`
     (`ProjPacks.pm:109-140, 1897-1939`).
   * mode 2: no `withsrcmd5/withdeps` at all; every package is `'delayed startup'`; per-project
     `/all` fetches are queued.
   * All projects and all prps are put into `lookat_low` (`bs_sched:755-757`), which is then
     moved into `lookat_next` (`bs_sched:779-782`) so the first "series" checks everything.
9. `bigdeps_unification` (`bs_sched:759`, `ProjPacks.pm:1941-1963`): any `info.dep` list with
   >100 entries is interned into `$gctx->{bigdeps}` keyed by `join("\n",@deps)` so identical
   huge BuildRequires lists share one array (memory saver).
10. Suspended projects' prps go to `lookat_med` to re-check their timeout (`bs_sched:762`).
11. `writeschedulerinfo` (`bs_sched:765`, `164-205`) — writes `info/schedulerinfo.<arch>` XML
    (`$BSXML::schedulerinfo`, `BSXML.pm:1536-1562`): queue lengths, avg/variance of per-prp
    check times, 10 worst prps, `buildavg`.
12. `BSSched::DoD::init_doddata` (`bs_sched:768-773`, `DoD.pm:365-398`): sync
    `$bsdir/dods/<proj>::<repo>::<arch>` XML files (`$BSXML::doddata`) with projpacks and touch
    `dods/.changed` so `bs_dodup` rescans.
13. `BSSched::BuildJob::init_ourjobs` (`bs_sched:775`, `BuildJob.pm:131-137`): scan
    `jobs/<arch>/` to fill the module-global `%ourjobs` (prefix -> {job=>1}) used by
    `killunwantedjobs`.
14. `unlink("$rundir/bs_sched.<arch>.dead")` (`bs_sched:777`): the "emergency dump" marker.
15. Enter the main loop (`bs_sched:806-1245`).

### 1.3 The global context (`$gctx`) — field inventory

Created at `bs_sched:460-561`; additional keys are added lazily by other modules. Grouped by
purpose (memory growth noted in section 7.4):

**Configuration / paths (constant)**

| key | value |
|---|---|
| `arch` | this scheduler's architecture |
| `reporoot` | `$bsdir/build` |
| `obsname` | `$BSConfig::obsname` (used in `disturl`) |
| `jobsdir`, `myjobsdir` | `$bsdir/jobs`, `$bsdir/jobs/<arch>` |
| `eventdir`, `myeventdir` | `$bsdir/events`, `$bsdir/events/<arch>` |
| `dodsdir` | `$bsdir/dods` |
| `rundir`, `infodir`, `blobdir`, `remotecache` | `$bsdir/run`, `$bsdir/info`, `$bsdir/blobs`, `$bsdir/remotecache` |
| `remoteproxy` | `$BSConfig::proxy` |
| `asyncmode` | `$BSConfig::sched_asyncmode` — enables async `xrpc` for projpack fetches and remote repo fetches |
| `critlogfile` | `$bsdir/log/scheduler_<arch>.crit.log` |
| `obssrcdir` | directory of `bs_sched` (templates for ptf/reproducible spec) |
| `genmetaalgo`, `maxgenmetaalgo` | meta algorithm version in use / supported |
| `rctx` | `BSSched::RPC` object (async RPC multiplexer) |
| `testmode` | set with `--testmode` |

**Project data (the big one)**

| key | contents |
|---|---|
| `projpacks` | `{projid => project}`; project = projpack `<project>` element minus `name`, with `package` turned into `{packid => pdata}` (`ProjPacks.pm:411-451`). Each `pdata` holds `rev, srcmd5, verifymd5, versrel, revtime, constraintsmd5, releasename, originproject, linked[], error, info[] (per-repo: name, file, dep[], prereq[], path[], containerpath[], extrasource[], imagetype, ...), aggregatelist, patchinfo, channel, modulemd, flags (build/publish/useforbuild/lock/...), bcntsynctag, hasbuildenv, hasbigdep`. Project-level: `kind, config (prjconf text), patternmd5, link[], remoteurl, remoteproject, scmsync, scminfo, flags, person/group, repository[] (name, rebuild, block, linkedbuild, download[], releasetarget[], path[], hostsystem[], arch[], status, crosshostarch), missingpackages`. |
| `channeldata` | `{md5 => channel}` deduplicated channel definitions (`ProjPacks.pm:404-410, 437-444`) |
| `channelids` | `{projid => [md5...]}` |
| `remoteprojs` | `{projid => remotemap entry}`: `remoteurl, remoteproject, root, remoteroot, partition, proto, config, repository[], link[], flags, error` (`Remote.pm:304-347`) |
| `remotemissing` | `{projid => 0 (fetch in progress) / 1 (does not exist)}` |
| `remoteprojs_changed` | set of remote projids whose data changed since last postprocess |
| `projsuspended` | `{projid => [ {job, time}, ... ]}` (from `suspendproject` events, `EventHandler.pm:536-557`) |
| `genbuildreqs` | `{prp => {packid => [md5, [deps], verifymd5]}}` — generated BuildRequires reported by the worker (`BuildJob.pm:414-440`), mirrored on disk as `<gdst>/:genbuildreqs` |
| `bigdeps` | interning table for large dep lists |

**Derived ("post-processed") project data** — rebuilt by `setup_projects` (`ProjPacks.pm:921-1265`)

| key | contents |
|---|---|
| `prps` | sorted list of all local prps for this arch, topologically ordered by `prpdeps` via `BSSolv::depsort` (`ProjPacks.pm:1229-1235`) |
| `project_prps` | `{projid => [prp...]}` |
| `prpsearchpath` | `{prp => [prp...]}` expanded path (own repo first) |
| `prpsearchpath_host` | `{prp => [hostarch, [prp...]]}` for cross builds |
| `prpdeps` | `{prp => [prp...]}` = searchpath ∪ aggregate sources ∪ kiwi/container paths ∪ channel targets (`ProjPacks.pm:1155-1189`) |
| `rprpdeps` | reverse of `prpdeps` (who depends on me); entries with only a self-dependency are deleted to save memory (`ProjPacks.pm:1238-1247`) |
| `relatedprpdeps`, `rrelatedprpdeps` | `prpdeps` filtered to "related" projects (same project, parent/child by `:` prefix, or `$BSConfig::related_projects`) and its reverse (`ProjPacks.pm:889-900, 1190-1192, 1250-1256`) |
| `expandedprojlink` | `{projid => [projid...]}` transitive `<link>` closure |
| `linked_projids` | `{projid => [projid...]}` projects our packages' `linked` entries point to |
| `projpacks_linked` | `{lprojid => [ {package, myproject, mypackage}... ]}` — reverse index of source links; `projpacks_linked_blks` records block sizes so a single project's entries can be spliced out on partial update (`ProjPacks.pm:969-993, 1019-1048`) |
| `alllocked` | `{prp => 1}` prps where project lock is on and no package is unlocked — they get an empty search path and are never expanded (`ProjPacks.pm:1126-1146`) |
| `dodprps` | `{prp => doddata}` (`DoD.pm:365-398`) |

**Repository data caches**

| key | contents |
|---|---|
| `repodatas` | `BSSched::RepoCache` object; `{"prp/arch" => {lastscan, random, solv (string) \| solvfile, error, isremote, meta => dep2meta cache, modules, "modules/<m>..." => {...}}}` (`RepoCache.pm`) |
| `remotegbininfos` | `{prpa => {lastfetch, cookie, error}}` remote bininfo fetch bookkeeping (`Remote.pm:604-614`) |
| `remotepackstatus` | `{prpa => packstatus + '/users'}`; `remotepackstatus_cleanup` `{prp => [prpa...]}` (`Remote.pm:545-582`) |
| `lastcheck` | `{prp => {packid => 128+ byte string}}` the per-package rebuild-decision cache (section 3.8); loaded from `<gdst>/:lastcheck`, dropped when prp finishes (`bs_sched:1158-1167`) |

**Scheduling triggers / queues**

| key | contents |
|---|---|
| `changed_low/med/high` | `{prp or projid => 1\|2}`; 1 = "something local changed", 2 = ":full changed as well" (`bs_sched:534-539`) |
| `changed_dirty` | `{prp => 1}` → touch `<gdst>/:schedulerstate.dirty` before next lookat (`bs_sched:913-921`) |
| `lookat_low/med/high` | ordered arrays of prps to check (section 2.6) |
| `lookat_next` | `{prp => 1}` the next low-priority "series" |
| `notlow`, `notmed` | starvation counters (section 2.6) |
| `nextmed`, `nexthigh` | `{prp => earliest time}` back-off after an expensive check (`Lookat.pm:252-269`) |
| `delayedfetchprojpacks` | `{projid => [packid.. \| '/all' \| '/dolink']}` package data fetches postponed until the prp is looked at (`ProjPacks.pm:1401-1442`) |
| `retryevents` | `BSSched::EventSource::Retry` queue (`addretryevent`, 60 s) |
| `prpcheckuseforbuild` | `{prp => 1}` useforbuild flags may have changed → re-run `checkuseforbuild` (`ProjPacks.pm:1793-1807`) |
| `prpfinished` | `{prp => 1}` prp was in state `finished` last time |
| `prpnotready` | `{prp => {srcname => 1}}` packages of *other* prps that are blocked/scheduled — the cross-prp "blocked" signal (`Checker.pm:1415-1420`, `Remote.pm:470-476`) |
| `repounchanged` | `{prp => 1 (unchanged) \| 2 (only bininfo changed)}`; absent = full tree changed → decides whether to emit a `repository` or `repoinfo` event (`bs_sched:358-369`) |
| `watchremote` | `{remoteurl => {"project/P" \| "package/P/pkg" \| "repository/P/R/arch" => local projid}}` (`Remote.pm:77-186`) |
| `watchremote_start` | `{remoteurl => last event sequence number}` |
| `get_projpacks_postprocess_needed` | flag set by `append_info_path` (Docker container paths) |

**Statistics**

`buildavg` (EWMA of job turnaround, seeded 1200 s, `BuildJob.pm:405-412`), `prpunfinished`
(`{prp => count}`), `prpchecktimes` (`{prp => seconds}`), `schedulerstart`, `slept`,
`prplastcheck` (unused).

### 1.4 The per-prp check context (`$ctx`, `BSSched::Checker`)

Constructed per lookat (`Checker.pm:90-104`) with `gctx, prp, project, repository, gdst,
changeprp, changetype, verbose, alllocked`; filled by the pipeline:

| set in | keys |
|---|---|
| `setup` (`Checker.pm:337-539`) | `proj, repo, prpsearchpath, conf (parsed prjconf), prptype, packs (sorted packids), lastcheck (ref into gctx), genmetaalgo, forcebinaryidmeta, excludebuild, onlybuild, genbuildreqs, isreprorepo/reprorepoid, prpsearchpath_host, conf_host, modularity_*` |
| `preparepool` (`Checker.pm:705-733`) | `pool (BSSolv::pool), dep2pkg, dep2src, depislocal, notready, subpacks, pool_host, dep2pkg_host, havedelayed` |
| main loop (`bs_sched:1079-1091`) | `expander (BSSolv::expander)` and `*Build::expand` is monkey-patched to it |
| `expandandsort` (`Checker.pm:776-991`) | `edeps {packid=>[expanded deps]}, experrors, pkg2buildtype, pkg2src, cycles, sccs, split_hostdeps, prpcheckstart` |
| `calcrelsynctrigger` (`Checker.pm:993-1039`) | `relsynctrigger, relsyncmax` |
| `checkpkgs` (`Checker.pm:1110-1506`) | `packstatus, cychash, cyclevel, incycle, nharder, building, unfinished, prpchecktime, sendunblockedevents, missingdodresources, doddownloads, blocked_cache, gbininfo_cache, packstatus_cache, alien_repo_cache, otherjobscache, rebuildpackage_needed, scheduler_buildtime, reprojob_written` |

Handlers frequently clone the ctx (`bless {%$ctx, ..., realctx => $ctx}`) to swap in a
different pool/config for a single package (e.g. `Package.pm:412`, `KiwiImage.pm:305,339`).

### 1.5 On-disk state per prpa (`<gdst>` = `$reporoot/<prp>/<arch>`)

This is the real "database". All of these are produced and consumed by the scheduler,
most are also read by `bs_repserver` (API views) and some by `bs_publish`.

| file | format | writer | purpose |
|---|---|---|---|
| `:schedulerstate` | Storable hash `{code, details, buildid, oldbuildid, repostateid, scmsync, scminfo}` (legacy: text `"code details"`) | `Checker::set_repo_state` (`Checker.pm:136-206`) | repository status shown by API; `buildid`/`REPO_BUILD_STARTED/FINISHED` notifications |
| `:schedulerstate.dirty` | empty marker | scheduler (`bs_sched:913-921`), `bs_repserver::dirty` (`bs_repserver:1925-1941`), other-arch schedulers | "a recheck is pending" flag; API reports `dirty="true"` |
| `:packstatus` | Storable `{packstatus => {packid=>code}, packerror => {packid=>text}, scmsync?, scminfo?}` | `checkpkgs` (`Checker.pm:1466-1471`); mtime bumped by `patchpackstatus` | per-package build status |
| `:packstatus.finished` | text lines `"<code> <packid>"` or `"scheduled <packid>/<job>/<details>"` | `patchpackstatus` (`BuildJob.pm:834-845`), `event_dispatchdetails` (`EventHandler.pm:687-698`); pruned by `prune_packstatus_finished` (`Checker.pm:1041-1058`) | overlay so the API shows `finished`/dispatch details between a job finishing and the next full check |
| `:repostate` | XML `<repositorystate><blocked>srcname</blocked>...</repositorystate>` (`BSXML.pm:1509-1512`) | `checkpkgs` (`Checker.pm:1426-1436`) | the list of not-ready source packages, served to *remote* schedulers with the repo (`Remote.pm:468-476`) |
| `:repodone` | text = md5 of (patternmd5, publish-enabled packids, unfinished, publish filters) | `Checker::publish` (`Checker.pm:1569-1635`) | "already published for this state" memo; unlinked on any result change |
| `:full/` | directory of hardlinks `<binname>.<suf>` + `<binname>.meta` | `BuildRepo::fctx_add_binary_to_full/del` | the per-prpa pool of binaries usable as build deps |
| `:full.solv` | libsolv solv file | `BuildRepo::writesolv` (`BuildRepo.pm:788-802`) | pre-parsed index of `:full`, loaded by `pool->repofromfile` |
| `:full.useforbuild` | Storable `[packid...]` | `checkuseforbuild` | which packages' binaries are allowed in `:full` |
| `:full.metacache`, `:full.metacache.merge` | Storable `{binname => [metaid, metamd5]}` | `fctx_move_into_full`, merged by `mergemetacachefile` (`bs_sched:327-346`) | cache of `.meta` ids for `:full` entries |
| `:full.subdirs`, `:full/doddata*` | text / Storable | `bs_dodup` | DoD metadata (section 5.3) |
| `:bininfo`, `:bininfo.merge` | Storable `{packid => {filename => {name,epoch,version,release,arch,hdrmd5,leadsigmd5,md5sum,id,source,filename}}}` (`/outdated` marker possible) | `BuildResult::update_bininfo_merge/read_gbininfo` | "gbininfo": what binaries each package has (consumed by aggregates, kiwi products, API `_jobhistory`/`binaryversions`, remote schedulers) |
| `<packid>/.bininfo` | Storable (same as one value above) | `read_bininfo` / `create_bininfo` (`Bininfo.pm:187-245`) | per-package bininfo |
| `:meta/<packid>` | text, line 1 `"<verifymd5>  <packid>"`, then `"<hdrmd5>  <path/of/dep>"` lines (section 3.10) | `jobfinished` (`BuildJob.pm:606,636,676`) | the "meta" of the last build: inputs that produced it |
| `<packid>/.meta.success` | hardlink to the meta of the last *succeeded* build | `update_dst_full` (`BuildResult.pm:525-530`) | needed when rebuilding `:full/*.meta` |
| `:logfiles.success/<packid>`, `:logfiles.fail/<packid>` | hardlinked logfiles | `jobfinished` | existence of `:logfiles.fail/<p>` = package is `failed` (`Checker.pm:1397-1400`) |
| `<packid>/{status,reason,logfile,history,.stats,.failhistory,_statistics,_ccache.tar,.nouseforbuild,.nosourceaccess,.preinstallimage.*,.reprojob,...}` | XML/Storable/BSFileDB | `writejob`, `jobfinished` | per package result dir |
| `:jobhistory` | BSFileDB (`$BSXML::jobhistlay`: package rev srcmd5 versrel bcnt readytime starttime endtime code uri workerid hostarch reason) | `addjobhist` (`BuildJob.pm:879-892`) | API `_jobhistory` |
| `:relsync`, `:relsync.merge`, `:relsync.sent`, `:relsync.max` | Storable `{packid => "versrel.bcnt"}` / `{"tag/versrel" => maxbcnt}` | `jobfinished`, `sendrelsyncupdate`, `updaterelsyncmax` (`bs_sched:208-325`) | release-counter sync across archs (section 5.6) |
| `:lastcheck` | Storable `{packid => string}` | `checkpkgs` / main loop | persisted rebuild-decision cache |
| `:depends` | Storable `{pkgdeps, subpacks, pkg2src, cycles, sccs}` | `expandandsort` (`Checker.pm:974-981`) | API `_builddepinfo` |
| `:genbuildreqs` | Storable | `set_genbuildreqs` | generated build requires |
| `:preinstallimages` | Storable list | `update_preinstallimage` (`PreInstallImage.pm:131-190`) | preinstall images available to the dispatcher |
| `:repo/`, `:repoinfo` | hardlink tree, Storable `{binaryorigins, conflicts}` | `PublishRepo::prpfinished` | staging area for `bs_publish` |
| `_deltas/`, `_volatile/` | dirs | `PublishRepo::makedeltas`, `BuildRepo::fctx_migrate_full` | deltarpm cache; binaries in `:full` not owned by any package |
| `<packid>/.waiting_for_<arch>` | marker | kiwi product / patchinfo / productcompose handlers | cross-arch "I am waiting for you" marker (section 6.2) |
| `$reporoot/<prp>/.finishedlock` | flock file | `PublishRepo` | mutual exclusion between scheduler publish and `bs_publish` |
| `$reporoot/_repoid` | 9-char id | `checkbuildrepoid` (`ProjPacks.pm:87-101`) | guards that this build tree belongs to this srcserver |

Process-level files: `run/bs_sched.<arch>.{lock,state,dead,pmat}`, `info/schedulerinfo.<arch>`,
`log/scheduler_<arch>.crit.log`, `jobs/<arch>/*` (section 4), `events/<arch>/*` (section 2),
`dods/*` (section 5.3), `remotecache/<xx>/<md5>.{solv,bininfo}` (section 5.2).

### 1.6 Main loop structure

`bs_sched:806-1245`, paraphrased:

```
loop forever:
  if any changed_{low,med,high} non-empty:  Lookat::changed2lookat(gctx); continue     # (1)
  reconcile %remotewatchers with gctx->{watchremote} (drop obsolete, create missing)     # (2)
  ev_queue = EventQueue->new
  add due retry events                                                                    # (3)
  select_read(0, ping, remote watchers, async RPC handles)                                # (4)
     finished RPC -> rctx->xrpc_resume(handle)  (runs its _resume callback)  ; gotevent=1
     ping readable -> gotevent=1
     remote watcher finished -> ev_queue->add_events(watcher->getevents(...)) ; gotevent=1
  if gotevent: drain ping, readevents(events/<arch>) -> ev_queue                           # (5)
  if ev_queue non-empty: ev_queue->process_events (die -> emergencydump + exit 1); continue   # (6)
  initialstartup = undef
  touch :schedulerstate.dirty for every prp in changed_dirty                              # (7)
  (lookattype, prp) = Lookat::nextlookat(gctx)                                            # (8)
  if prp and rctx->xrpc_busy(projid): xrpc_addwakeup; continue                            # (9)
  every 4h: print_project_stats + bigdeps_unification                                     # (10)
  if no prp: writeschedulerinfo (<=1/min); select_read(3600, ping, watchers, retry, rpc); continue   # (11)
  Lookat::lookatprp (remove from other queues, bump starvation counters, log)             # (12)
  if prp is a bare project id: delete repositories on disk that no longer exist; continue # (13)
  do delayed projpack fetches for projid (async -> continue)                              # (14)
  ctx = Checker->new(gctx, prp, changeprp=>prp, changetype=>lookattype, verbose=>1)
  merge :bininfo.merge / :relsync.merge / :full.metacache.merge if present               # (15)
  (state, details) = ctx->setup            ; if not 'scheduling' -> set_repo_state, sendrepochangeevent, continue
  if prpcheckuseforbuild{prp}: BuildRepo::checkuseforbuild                                # (16)
  ctx->wipeobsolete                                                                       # (17)
  ctx->preparepool                         ; if not 'scheduling' -> set_repo_state (unless delayed), continue
  set_repo_state('scheduling')
  ctx->{expander} = BSSolv::expander->new(pool, conf); monkeypatch Build::expand          # (18)
  ctx->expandandsort                       ; if not 'scheduling' -> continue
  ctx->calcrelsynctrigger
  (state, details) = ctx->checkpkgs                                                        # (19)
  sendrepochangeevent; printstats; DoD::dodfetch if downloads needed
  sendrelsyncupdate if :relsync changed since :relsync.sent
  if finished or earlypublish(kiwi/productcompose): ctx->publish                            # (20)
  Build::forgetdeps(conf); unpreparepool; free_caches                                      # (21)
  reprorepo follow-up; prpfinished bookkeeping; drop lastcheck+meta cache when finished;
  free repodata if leaf prp whose deps are all finished                                    # (22)
  notify related prps (rrelatedprpdeps) via changed_med                                    # (23)
  set_repo_state(state, details); update prpunfinished/prpchecktimes; Lookat::setdelayed   # (24)
  writeschedulerinfo (<=1/min); get_projpacks_postprocess_projects if flagged
```

Important properties:

* Steps (1)-(6) always win over checking: as long as events keep arriving, the scheduler
  only ingests them (the `continue`s at (1) and (6)). This is why a burst of `built`
  events delays all checks, and why `dstcache` batching exists (section 2.4).
* A single prp check is atomic w.r.t. events: nothing is read from the event dir while
  `checkpkgs` runs, except that async RPC callbacks only fire in step (4).
* The loop sleeps only in (11) (`select` with 1 h timeout on the FIFO + sockets). The
  FIFO `.ping` is how any other process wakes it (`BSUtil::ping`, `BSUtil.pm:615-622`).

### 1.7 Checkpointing

There is no continuous checkpoint of in-memory scheduling state. Durable state is the
per-prpa file set of 1.5 (written during checks and on job completion). Memory-only state
is dumped as one Storable blob only on `exit`/`exitcomplete`/`restart`/`dumpstate`
events and on any uncaught exception (`emergencydump` → `run/bs_sched.<arch>.dead`,
`EventHandler.pm:464-519`, `bs_sched:898-905, 1247-1252`). The dump contains:

`projpacks`+`remoteprojs` (only for complete dumps), `prps`, `changed_{low,med,high}`,
`lookat` (= `lookat_low` + `lookat_next` members in prp order), `lookat_oob` (med),
`lookat_oobhigh` (high), `prpfinished`, `globalnotready` (= `prpnotready`),
`repounchanged`, `projsuspended`, `delayedfetchprojpacks`, `watchremote_start`,
`fetchprojpacks` (pending + in-flight async fetches via `runningfetchprojpacks`,
`ProjPacks.pm:1815-1844`), `retryevents`.

Not dumped (recomputed): all derived project data (`setup_projects`), `repodatas`,
`lastcheck` (persisted separately per prp when it finishes or the check took >2 min and
>10 "hard" lookups, `Checker.pm:1492-1495`, `bs_sched:1158-1167`), remote caches
(persisted in `remotecache/`), `genbuildreqs` (on disk), `%ourjobs` (rescanned).

A crash without a dump (kill -9) loses `changed_*`/`lookat_*` and `prpnotready`; recovery
relies on (a) the `:schedulerstate.dirty` markers and (b) the fact that a cold start puts
every prp into `lookat_next` so everything is rechecked once. Because all decisions are
recomputed from `:meta`/`:full`/`projpacks`, a lost queue only costs a full pass, not
correctness. The exceptions are mid-write multi-file updates (section 7.5).

---

## 2. Event model

### 2.1 Event sources

| source | mechanism | code |
|---|---|---|
| local event directory `events/<arch>/` | one XML file per event (`$BSXML::event`, `BSXML.pm:771-794`); writers `ping` the FIFO `events/<arch>/.ping` afterwards; the scheduler reads the dir sorted by name, special-casing names `finished:<job>` as `{type=>'built', job=>...}` without parsing (`EventSource/Directory.pm:53-75`) | `bs_sched:884-893` |
| remote watchers | one long-poll HTTP request per remote OBS (or per partition via `$srcserver/lasteventsproxy`): `POST <remoteurl>/lastevents?filter=...&start=<seq>` returning `$BSXML::events` (`next`, `sync`, `event[]`); each returned event is mapped back to the *local* projid that watches it (`EventSource/RemoteWatcher.pm:337-511`) | `bs_sched:814-876` |
| retry queue | in-memory events with `retry = now+60` deduplicated by (type, project, repository[, arch/package]) (`EventSource/Retry.pm:250-266`); used for transient interconnect errors, delayed publishes, missing packages | `bs_sched:840-847` |
| async RPC completion | `BSSched::RPC` handles have sockets; when readable, `xrpc_resume` calls the stored `_resume` callback which usually calls `Lookat::setchanged` → effectively a synthetic "recheck prp" event | `bs_sched:853-861`, `RPC.pm:321-382` |
| self-generated | handlers call `Lookat::setchanged`, write to `changed_*` directly, or write event files for *other* daemons/archs (2.8) | — |

Who writes into `events/<arch>/`:

* `bs_srcserver` (through its notification to the repo server; file names like
  `package:<proj>::<pkg>`, `project:<proj>`, see `bs_repserver:2941-2952` for the naming
  helper) — `package`/`project`/`lowprioproject`/`configuration`/`wipe`/`rebuild`...;
* `bs_repserver` after a worker uploads a job result: `finished:<job>` (`bs_repserver:2605-2617`),
  `suspendproject:<job>`/`resumeproject:<job>` (`:2834-2871`), `uploadbuild:<job>` (`:2919`),
  `scanrepo:<proj>::<repo>` (`:1707-1710`);
* `bs_dispatch`: `dispatchdetails:<job>` (`bs_dispatch:1478-1496`);
* `bs_signer`: `finished:<job>` after signing (`bs_signer:1273`);
* `bs_dodup`: `scanrepo:<proj>::<repo>` (`bs_dodup:847-858`);
* other schedulers: `unblocked::P::R`, `lowunblocked::P::R`, `relsync:P::R`, `import.<job>`,
  `rebuild_full_tree:P:R` (self, as a crash marker, `BuildRepo.pm:676-683`);
* `bs_sched --exit/--restart`: `exitcomplete::` / `restart::`.

Real fixture (`t/data/0018/events/`):

```xml
<!-- events/x86_64/finished:devel:languages:perl::perl-Test-Finished-Package  (content is ignored, only the name counts) -->
<event type="built">
	<project>devel:languages:perl</project>
	<package>perl-Test-Finished-Package</package>
	<job>blah-blah-nix</job>
</event>

<!-- events/x86_64/package:devel:languages:perl::perl-Devel-Cover -->
<event type="package">
	<project>devel:languages:perl</project>
	<package>perl-Devel-Cover</package>
</event>
```

Event file names are de-duplicating keys: a second `package:P::pkg` event overwrites the
first (atomic `writexml` to `.name$$` then rename, `Directory.pm:87-99`), so a flood of
source changes to one package collapses to one event. Names over 200 chars are replaced
by `<type>:::<md5>`.

### 2.2 Full event type table

`%BSSched::EventHandler::event_handlers` (`EventHandler.pm:40-78`):

| type | handler | effect |
|---|---|---|
| `built` | `event_built` (`:98-135`) | lock+read `jobs/<arch>/<job>:status`; must be `code=finished`; read job buildinfo; `BSSched::BuildJob::jobfinished` (section 4.5); `purgejob` unless `job_is_waiting` |
| `uploadbuild` | `event_built` → `BuildJob::Upload::jobfinished` | API "upload build result" (`bs_repserver:2919`): integrate uploaded files as if built |
| `import` | `event_built` → `BuildJob::Import::jobfinished` | binaries exported from another arch (exportfilter) are integrated as `::import::<arch>::<file>` (section 6.1) |
| `srcevent`, `package` | `event_package` (`:143-157`) | queue `fetchprojpacks{projid} += packid` (undef packid = whole project + `deepcheck`); resumes a suspended project if packid undef |
| `project`, `projevent`, `lowprioproject` | `event_project` (`:165-174`) | queue `fetchprojpacks{projid} += undef`; `lowprioproject` marks `lowprioproject{projid}` |
| `repository` | `event_repository` (`:182-201`) | `changed_med{prp}=2`; drop `repodatas` cache for `prp/arch`; delete `repounchanged{prp}` (if own arch) — "another arch/remote's `:full` changed" |
| `repoinfo` | same | only `repounchanged{prp}=2` (bininfo changed, full tree not) |
| `rebuild`, `recheck` | `event_check` (`:209-249`) | no project → every prp `changed_low=1`; project+repo → `changed_high{prp}||=1`, `changed_dirty`; project only → all its prps high |
| `admincheck` | same + jump to the head of `lookat_high`, reset back-off and starvation counters | |
| `unblocked`, `relsync` | `event_check_med` (`:257-268`) | `changed_med{prp}||=1` |
| `lowunblocked` | same | `changed_low{prp}||=1` |
| `scanrepo` | `event_scanrepo` (`:276-299`) | no project → drop whole `repodatas`; else drop cache for prp, rescan `:full` (`addrepo`), `changed_high{prp}=2`, delete `repounchanged` |
| `scanprjbinaries` | `event_scanprjbinaries` (`:307-343`) | delete remote gbininfo cache; for local: unlink `<pkg>/.bininfo`(s), `rebuild_gbininfo`, `changed_high{prp}=1` |
| `dumprepo` | `event_dumprepo` | debug: Data::Dumper of repodata to `/tmp/repodump` |
| `wipenotyet` | `event_wipenotyet` | clear back-off (`setdelayed(prp,0)`) |
| `wipe`, `wipeallarch` | `event_wipe` (`:384-410`) | `BuildResult::wipe` (remove result + `:full` entries; `wipeallarch` also removes imports), clear genbuildreqs, all prps of the project `changed_high=2` |
| `exit`, `exitcomplete`, `restart`, `dumpstate` | `event_exit` (`:464-519`) | dump state (1.7); exit / `exec $0` |
| `useforbuild` | `event_useforbuild` (`:418-456`) | package given → `update_dst_full` with `jobdir == dst` ("refresh"); no package → `forcefullrebuild`; all prps of project `changed_high=2` |
| `configuration` | `event_configuration` | re-read `BSConfig` (`obsname`, `proxy`) |
| `suspendproject` | `event_suspendproject` (`:536-557`) | push `{job,time}` on `projsuspended{projid}`; set repo state `blocked` immediately |
| `resumeproject` | `event_resumeproject` (`:559-589`) | pop matching job; when empty → all prps `changed_high` |
| `memstats`, `dumppmat`, `projectstats` | debug | malloc stats / Devel::MAT dump / project stats |
| `dispatchdetails` | `event_dispatchdetails` (`:687-698`) | append `scheduled <pkg>/<job>/<details>` to `:packstatus.finished` (so the API can show why a job is not dispatched) |
| `force_publish` | `event_force_publish` | `Checker::publish(1)` for a repo where publishing is disabled |
| `emergencydump` | internal | state dump to `.dead` |

Note the conspicuous absence of an "arch-sync"/"noarch" event: cross-arch binary sharing is
done with `import` jobs, and cross-arch build ordering with `unblocked` events (section 6).

### 2.3 Event queue semantics (`BSSched::EventQueue`)

* All events collected in one loop iteration are processed as a batch. `order()` sorts
  them: `exit/exitcomplete/restart` first (-2), `suspendproject` (-1), everything else 0,
  `uploadbuild/import` last (+1); ties by type, project, job (`EventQueue.pm:240-257`).
* `uploadbuild`/`import` events are **delayed** (pushed back and re-run once) if a projpack
  fetch for that project/package is pending or in flight (`event_uploadbuildimport_delay`,
  `EventHandler.pm:597-629`), so that results are never integrated against stale pdata.
* `dstcache` (`{fullcache=>{}, bininfocache=>{}}`) is switched on when two consecutive
  events are `built`/`wipe` (`EventQueue.pm:144-156`). It lets `update_dst_full` keep the
  `:full.solv` repo object, the `oldids` map, the metacache and the `:bininfo.merge` hash in
  memory across many job completions for the *same* prp, writing them once when the prp
  changes or the batch ends (`BuildResult::set_dstcache_prp`, `BuildRepo::sync_fullcache`,
  `BuildResult::sync_bininfocache`). If more than 20 integrations hit one prp in a batch it
  switches to "rebuild full tree later" mode and writes a `rebuild_full_tree:P:R` event
  file as a crash guard (`BuildRepo.pm:671-688, 774-778`).
* After the batch, `process_events` calls `ProjPacks::do_fetchprojpacks` once with the
  accumulated `fetchprojpacks`, `fetchprojpacks_nodelay`, `deepcheck`, `lowprioproject`
  sets (`EventQueue.pm:166-182`).
* `exit`/`exitcomplete` seen in the very first batch after startup are ignored
  (`EventQueue.pm:208-211`) — guards against a stale exit event from a previous shutdown.

### 2.4 Projpack fetching (how source changes become scheduler state)

`do_fetchprojpacks` (`ProjPacks.pm:1450-1628`) has four passes:

* pass 0 — *delay if possible*: if an entry is only package-level (no `undef`/`/all`) and
  not in `fetchprojpacks_nodelay`, it is moved to `delayedfetchprojpacks{projid}` and the
  project's prps get `changed_high=1`+`dirty`. The fetch then happens in the main loop just
  before the prp is checked (`bs_sched:1004-1013` → `do_delayedprojpackfetches`). This
  prevents fetching package data for projects that are never looked at.
* pass 1 — *fetch*: project-level changes first try the cheap
  `update_project_meta` (`nopackages&withrepos&withconfig&withsrcmd5`) and only if
  `update_project_meta_check` detects a *critical* change (build/lock/link/repository
  structure, or a prjconf change altering macros/`type`/`macroserial`/`dockerarg:` buildflags,
  `ProjPacks.pm:517-568`) does it refetch all packages (`get_projpacks`).
  Package-level changes fetch just those packages (+ their local linked packages in async
  mode). In async mode every fetch returns immediately with an `_resume` callback
  (`get_projpacks_resume`, `update_project_meta_resume`).
* pass 2 — `get_projpacks_postprocess`; set `changed_high` (level 2 for project changes,
  1 for package changes; `changed_low` for `lowprioproject` unless deep check).
* pass 3 — *follow source links*: `find_linked_sources` (using `projpacks_linked`) finds
  packages in other projects that `<link>` to the changed ones and fetches them too, with
  `med` priority if it was a source change, `low` otherwise.

`get_projpacks_resume` (`ProjPacks.pm:272-369`) is where the async bottom half lives: it
checks for critical config changes (`trigger_auto_deep_checks` → delayed `/all` fetches
for dependent projects if `$BSConfig::deep_check_dependent_projects_on_macro_change` or
`expandflags:macroserial` changed), commits with `update_projpacks`, runs
`get_projpacks_postprocess_projects` only when `postprocess_needed_check` says something
other than `rev/srcmd5/versrel/verifymd5/revtime/dep/prereq/file/name/error/build/publish/useforbuild/scmsync`
changed (`ProjPacks.pm:678-703`), optionally follows links (`_dolink`), and finally
`Lookat::setchanged(projid, changetype, changelevel)`.

`update_projpacks` (`ProjPacks.pm:378-471`) is a replace-in-place: for whole-project
answers the old project entry is deleted first; for package answers only the listed
packages (and their multibuild flavors `pkg:flavor`) are replaced; `missingpackages` is
preserved; `remotemap` entries are merged via `remotemap2remoteprojs`; `checkbuildrepoid`
verifies `$reporoot/_repoid` matches the srcserver's `repoid`.

### 2.5 `changed_*` → `lookat_*`: the priority scheduler (`BSSched::Lookat`)

Three explicit priority levels plus a "next series":

| level | meaning (comment at `bs_sched:541-544`) | typical producers |
|---|---|---|
| `high` | user interaction, do ASAP | source/project changes, rebuild/wipe/useforbuild events, async RPC completions (default changetype) |
| `med` | builds finished here | `built` events (`jobfinished` sets `changed_med{prp}`), `unblocked`, `relsync`, `repository` events, related-prp follow-ups |
| `low` | not so important | `lowunblocked`, `lowprioproject`, link follow-ups, `rebuild` without project |
| `next` | the next full series of low checks | dependents of anything in `changed_med/low` |

`setchanged(gctx, changeprp, changetype, changelevel)` (`Lookat.pm:31-87`): for a prp,
remove it from the queue and `unshift` it to the **front** of `lookat_<type>`, set
`changed_<type>{prp}` (level 2 overrides 1), and `changed_dirty{prp}`. For `low` it
instead marks all prps whose `prpdeps` contain it. For a projid, all prps of the project
(plus all prps in `rprpdeps` of that project) are unshifted.

`changed2lookat` (`Lookat.pm:96-143`), run at the top of every loop iteration when any
`changed_*` is non-empty: high entries are copied into med so that "simpler" logic can treat
them uniformly, then appended to `lookat_high`/`lookat_med` (deduplicated, high removed from
med); `changed_low` keys and `changed_med` keys **plus all their reverse dependents
(`rprpdeps`, excluding `alllocked`)** are added to `lookat_next`. Then all three `changed_*`
hashes are cleared. This is the transitive "a repo changed, so everything that uses it must be
rechecked eventually" fan-out — it is *one* level deep here; `extend_lookat_next`
(`Lookat.pm:145-163`) makes it transitive when a new series starts.

`nextlookat` (`Lookat.pm:171-225`):

```
if lookat_low empty and lookat_next non-empty:
    extend_lookat_next  (transitive closure over rprpdeps, chunked by 100)
    lookat_low = prps in global topological order that are in lookat_next; clear lookat_next
try in order:
  low   if notlow  > 10      (starvation guard: after 10 non-low picks, force a low one)
  med   if notmed  > 2
  high
  med
  low
  high (ignoring back-off), med (ignoring back-off)
```

`check_queue` skips prps whose `nexthigh`/`nextmed` back-off time is in the future
("not yet"), re-inserting them at the front. `lookatprp` (`Lookat.pm:227-250`) removes
the chosen prp from the *lower*-priority queues (a high pick also clears it from med; a
low pick clears high and med), bumps `notlow`/`notmed`, and resets the one it served.

`setdelayed` (`Lookat.pm:252-269`), called after every check with
`delay = prpchecktime - scheduler_buildtime` (the time spent checking, minus time spent
building aggregates/channels in-process): sets `nextmed{prp} = now + medfactor*delay`,
`nexthigh{prp} = now + highfactor*delay` where `medfactor = min(4, 1+|lookat_med|/100) * 10`
(`$BSConfig::delayscaler_med`) and `highfactor = min(4, 1+|lookat_high|/10) * 1`. So a prp
that takes 60 s to check will not be re-checked at med priority for ≥ 10 min — this is the
scheduler's main defense against expensive repos hogging the single thread.

The `low` series is always walked in **global topological prp order** (`$gctx->{prps}`,
sorted by `BSSolv::depsort` over `prpdeps`), so in a full pass base repos are checked
before the repos that use them.

### 2.6 "lastcheck" vs "lookat"/"lastcheck time"

Two unrelated things share the word:

* `$gctx->{lastcheck}{prp}{packid}` / `<gdst>/:lastcheck` is the per-package *rebuild
  decision cache* (section 3.8). It has nothing to do with time.
* `$gctx->{prplastcheck}{prp}` is a timestamp that is written but never read
  (`bs_sched:560, 1229`). `prpchecktimes{prp}` (duration) and `nextmed/nexthigh`
  (back-off) are what actually influence scheduling.

### 2.7 Suspension and "delayed" states

* `projsuspended`: set by `suspendproject` events sent by the repserver while a
  "copybuild"/release job is moving results (`bs_repserver:2834-2871`). While suspended,
  `setup` returns `('blocked', "<jobs>")` (`Checker.pm:363-365`) and suspensions older than
  8 h are dropped (`timeout_suspends`, `Checker.pm:308-335`).
* `havedelayed`: set during a check whenever something that is needed is "in progress"
  (async remote repo fetch, remote bininfo fetch, missing remotemap, `delayed startup`
  package, interconnect errors). In that case the prp ends in `blocked` and the repo state
  is *not* downgraded by `preparepool` errors (`bs_sched:1070`); a retry event or the RPC
  completion will cause a re-look.

### 2.8 Outgoing events

| function | target dir | name | consumer |
|---|---|---|---|
| `sendrepochangeevent(gctx, prp[, 'repoinfo'])` (`Directory.pm:109-121`, `bs_sched:358-369`) | `events/repository/` | `repository::P::R::<arch>` / `repoinfo::...` | `bs_dispatch`/repserver forwards to srcserver → remote schedulers watching this prp get `repository`/`repoinfo` events via `lastevents` ("we don't directly send it to the src server, as this would slow down the scheduler too much", comment at `Directory.pm:101-107`) |
| `sendunblockedevent(gctx, prp, arch, type)` (`:130-143`) | `events/<otherarch>/` | `unblocked::P::R`, `lowunblocked::P::R` | the other arch's scheduler (section 6.2) |
| `sendpublishevent(gctx, prp)` (`:151-161`) | `events/publish/` | `P::R` (type `publish`, `time` preserved from existing file) | `bs_publish` |
| `sendimportevent(gctx, job, arch)` (`:170-178`) | `events/<otherarch>/` | `import.<job>` | other arch's scheduler (exportfilter) |
| `sendrelsyncevent(gctx, prp, arch)` (`:187-197`) | `events/<otherarch>/` | `relsync:P::R` | other arch (section 5.6) |
| `fakejobfinished` (`BuildJob.pm:714-749`) | `events/<arch>/` or `events/signer/` | `finished:<job>` / `finished:<arch>:<job>` | self or signer (aggregates that need signing) |
| `Checker::notify` (`Checker.pm:119-130`) | RabbitMQ/Redis forwarder | `REPO_BUILD_STARTED`, `REPO_BUILD_FINISHED` with `buildid` | API/notification bus |
| Redis (`BSRedisnotify::updateresult/updateoneresult/updatejobstatus/deleteresult`) | redis | `<prp>/<arch>` hash of packstatus | `bs_repserver` `_result` fast path when `$BSConfig::redisserver` |

---

## 3. The per-package check algorithm

### 3.1 Pipeline for one prp

```
setup()                 -> 'scheduling' | repo-level terminal state (broken/disabled/blocked/excluded)
wipeobsolete()          -> delete results of packages that vanished / became excluded
preparepool()           -> BSSolv pool over the search path; dep2pkg/dep2src/depislocal/notready/subpacks
expandandsort()         -> per-package expanded build deps (edeps), build type, topological sort, cycles; writes :depends
calcrelsynctrigger()    -> packages whose bcnt must be bumped to sync with other archs
checkpkgs()             -> per-package status; writes :packstatus, :repostate, :lastcheck; creates jobs; kills obsolete jobs
publish()               -> if finished (or early publish for kiwi/productcompose): update :repo, send publish event
```

### 3.2 `setup` — repository-level outcomes (`Checker.pm:337-539`)

| condition | result (code, details) |
|---|---|
| project unknown | `(0, 'project does not exist')` — repo is skipped silently, state file untouched |
| repo not in project | `(0, 'repo does not exist')` |
| `proj.error` =~ `service in progress` | `('blocked', error)` |
| other `proj.error` | `('broken', error)` |
| `repo.status eq 'disabled'` | `('disabled')` |
| project suspended | `('blocked', joblist)` |
| `_reproduciblecheck` repo whose base repo lacks our arch | `('broken', ...)` |
| `alllocked` | `('scheduling')` with empty config/pool — only wipe/publish paths run |
| no `prpsearchpath` | `(0, 'no prpsearchpath?')` |
| `getconfig` returns undef (a remote path element has an error) | `(0, "<proj>: <error>")` + retry event if interconnect/5xx; else `('broken', "no config (<lastproj>)")` |
| prjconf parse error | `('broken', "project config: ...")` unless `$BSConfig::ignore_project_config_errors` |
| bad `hostarch` | `('broken', ...)` |
| no build `type` (and no channel packages) | `('broken', "no build type (<proj>)")` |
| type `excluded`/`disabled` | `(type)` |
| genmetaalgo unsupported | `('broken', 'unsupported genmetaalgo')` |
| crosshostarch mismatch | re-postprocess project, `setchanged`, `(0, 'crosshostarch mismatch')` |
| cross: host config problems | `('broken', ...)` |
| modulemd problems | `('broken', ...)` |
| scmsync project with `rebuild=local` and source updates running | `('blocked', 'source update in progress for: ...')` |
| otherwise | `('scheduling')` |

`getconfig` (`ProjPacks.pm:1639-1690`) concatenates the `config` text of every project in
the search path **in reverse order** (base first), stripping `Macros:` sections, injecting
`%define _project/_repository/_is_this_project/_is_in_project`, honouring `FromScratch:`,
appending `$BSConfig::extraconfig` by project regex, then `Build::read_config($arch, ...)`
(`build/Build.pm:385-...`). The result `$bconf` contains the parsed prjconf: `type`,
`binarytype`, `repotype`, `preinstall/vminstall/required/support/runscripts/keep`,
`substitute`, `ignore`, `prefer`, `conflict`, `fileprovides`, `exportfilter`,
`publishfilter`, `buildflags`, `publishflags`, `expandflags`, `macros`, `modules`,
`hostarch`, `constraint`, `release`, `debuginfo`, ... Note "all the macro code stripped as we
do not need any macros in the scheduler" — macro *evaluation* happens on the worker; the
scheduler only needs the dependency-affecting parts.

### 3.3 `preparepool` (`Checker.pm:653-733`)

`createpool` builds a `BSSolv::pool` (type `deb`/`arch`/`apk` per `binarytype`, modules
set for modular repos) and calls `ctx->addrepo(pool, aprp)` for every prp in
`prpsearchpath` in order (section 5.2). Access is checked with `checkprpaccess`
(`Access.pm:430-448`). A repo returning `0` (async fetch in progress) marks the pool
`delayed`; `undef` (hard error) makes the prp `broken: repository 'X' is unavailable`.
Missing modules (`repo->missingmodules`) are reported as `broken`. Finally
`pool->createwhatprovides()`.

`preparehashes` (C in BSSolv if available, else `Checker.pm:613-639`) walks
`pool->consideredpackages()` (the "best" package per name after repo-order shadowing) and
produces:

* `dep2pkg{name} = solvable id`, `dep2src{name} = source package name` (container
  packages have `container:` stripped),
* `depislocal{name}=1` if the providing repo *is this prp* (so its `.meta` is on disk
  locally and is used for deep meta comparison),
* `notready{srcname}=2` if the providing repo `rprp` ≠ this prp and
  `prpnotready{rprp}{srcname}` is set — i.e. the dependency's source package is
  blocked/building in its own repo; the value 2 distinguishes "global" from "local"
  not-ready, see `Checker.pm:1410-1413`,
* `subpacks{srcname} = [binary names...]`.

`prpnotready` is ignored entirely when the repo has `block="local"` (`Checker.pm:717-718`).

### 3.4 `expandandsort` (`Checker.pm:776-991`)

For every package (sorted packid order) determine the build type:

```
aggregatelist      -> 'aggregate'
patchinfo          -> 'patchinfo'
channel            -> 'channel'
modulemd           -> 'modulemd'
info.file =~ .(spec|dsc|kiwi|livebuild|productcompose)$ -> that; kiwi split into kiwi-product/kiwi-image by info.imagetype
else Build::recipe2buildtype(info.file)  (PKGBUILD->arch, Dockerfile->docker, _preinstallimage, simpleimage, Chart.yaml->helm, mkosi.*, flatpak, snapcraft, appimage, ...)  (build/Build.pm:1269-1289)
```

then compute `edeps{packid}` (the *expanded* build dependencies — the closure of
BuildRequires over the pool, which is what the meta is made of) through the type handler's
`expand`:

| handler | `expand` |
|---|---|
| Package (default), PreInstallImage | `Build::get_deps($bconf, $subpacks, @deps)` (`build/Build.pm:883-916`): remove own subpackages (unless in `keep`), add `required`, apply `substitute`, run `Build::expand`, then drop `preinstall`/`support` packages from the result (they don't influence meta) |
| Image (mkosi/appimage/livebuild/simpleimage) | `Build::get_build(..., '--ignoreignore--')` (full build env expansion) |
| KiwiImage, Docker | only expand `container:` deps "for sorting purposes"; real expansion happens in `check` with the right pool |
| Aggregate, Channel, Patchinfo, ProductCompose, KiwiProduct, Reproduciblecheck, Unknown | identity (`return 1, @deps`) |
| cross-build package without special handler | `Package::expand_sysroot` → `Build::get_sysroot` (split into sysroot deps and native deps via `onlynative`/`alsonative`) |

`Build::expand` is the libsolv-backed expander (`$ctx->{expander} = BSSolv::expander->new($pool,$bconf)`,
`bs_sched:1081-1090`); the pure-perl `Build::Expand::expand` (`build/Build/Expand.pm:375-807`)
is the reference semantics (direct deps added even if provided, `prefer`, `ignore`,
`conflict`, rich deps, recommends/supplements for choices, `--directdepsend--`,
`--ignoreignore--`, `!conflict` and `!!exact-conflict` syntax). Expansion failures become
`experrors{packid}` and the package later ends as `unresolvable`; the unexpanded `@deps`
are used for sorting.

`genbuildreqs` (BuildRequires the worker generated, e.g. from `%generate_buildrequires`)
are appended if their `verifymd5` still matches (`Checker.pm:901-907`).

Sorting: `BSSolv::depsort2(\%pdeps, $dep2src, \%pkg2src, \@cycles, @packs)` with
`setdepsortsccs(2)` returns the packages in dependency order and the list of cycles; with
sccs mode, `@cycles` is `[scc..., [], cycle...]` and the SCCs are separated out
(`Checker.pm:935-951`). Patchinfos are moved to the end. `:depends` is written for the API
(disabled packages keep their previous entry).

### 3.5 `checkpkgs` — the per-package decision procedure (`Checker.pm:1110-1506`)

Preparation: `projbuildenabled` (project `build` flag, default off for
`kind=maintenance_release`), `projlocked`, `prjuseforbuildenabled`; `cychash` built from
the SCCs/cycles (each cycle member maps to the sorted member list via `cycsort`,
`Checker.pm:1090-1108`, which drops bidirectional edges and depsorts the rest). If the
project has `missingpackages` (srcserver could not deliver everything), old `:packstatus`
entries for unknown packages are carried over and a retry event is queued.

Then, walking `@cpacks` (the sorted packages, mutated by cycle handling), the first matching
rule wins:

| # | rule | status | error text |
|---|---|---|---|
| 1 | in a cycle: `handlecycle` may re-order / skip (3.9) | — | |
| 2 | packid `_product` | `excluded` | |
| 3 | package `lock` flag (or project locked and no package flag) | `locked` | |
| 4 | prjconf `buildflags:excludebuild:<pkg|releasename>` | `excluded` | `package blacklist` |
| 5 | prjconf `buildflags:onlybuild:` not matching | `excluded` | `package whitelist` |
| 6 | package comes from a project link and repo `linkedbuild` not in `localdep/all/alldirect/alldirect_or_localdep` (or `alldirect` and origin not a direct link) | `excluded` | `project link` |
| 7 | `pdata.error` ∈ {disabled, locked, excluded} | that code | |
| 8 | `pdata.error` =~ `download in progress` / `source update running` / `service in progress` | `blocked` | the error |
| 9 | `pdata.error` = `delayed startup` or interconnect/5xx | `blocked` (+ retry event, `havedelayed`) | |
| 10 | other `pdata.error` (e.g. "bad link", "no source") | `broken` | the error |
| 11 | package `build` flag disabled (or project build disabled and no package flag) | `disabled` | |
| 12 | `info.error` ∈ {disabled, locked, excluded} | that code | |
| 13 | `info.error` interconnect/5xx | `blocked` (+retry) | |
| 14 | other `info.error` (recipe parse error etc.) | `broken` | |
| 15 | buildtype `unknown` | `broken` | `no recipe file` |
| 16 | reproduciblecheck repo and type isn't reproduciblecheck | `excluded` | |
| 17 | type `modulemd` | `excluded` | |
| 18 | a job file `jobs/<arch>/<jobname>-<srcmd5>` exists ("speedup hack") | `scheduled`; `building{packid}=job`; `notready{srcname}=1` if useforbuild; `unfinished{srcname}=1` | |
| 19 | `experrors{packid}` | `unresolvable` | expansion errors |
| 20 | needed DoD container resources missing | `blocked` | `waiting for dod resources to appear: ...` |
| 21 | handler `check(ctx, packid, pdata, info, buildtype, edeps)` → see 3.6 | | |

Handler results are mapped (`Checker.pm:1370-1407`):

| handler returns | mapped status |
|---|---|
| `('scheduled', data)` | call `handler->build(...)`: → `('scheduled', job)` → `building{packid}=job`; `('delayed', msg)` → `blocked` "delayed: msg" + `havedelayed`; `('broken'/'unresolvable'/'failed'/'blocked', msg)` as returned; always `unlink :repodone` |
| `('delayed', msg)` | status copied from the **old** `:packstatus` (or `blocked 'delayed'`), `havedelayed`, `unfinished` |
| `('building', job)` | `scheduled` |
| `('done')` | `succeeded` or `failed` depending on `-e :logfiles.fail/<packid>` |
| `blocked`, `excluded`, `broken`, `unresolvable`, `failed` | as is |

For `blocked`/`scheduled` results the package's *source name* is added to `notready` (only
if useforbuild is enabled, i.e. only if its binaries could ever be in `:full`) and
`unfinished`.

Post-processing (`Checker.pm:1410-1505`):

1. remove `notready` entries with value 2 (those came from other prps);
2. `prpnotready{prp} = notready` if some other prp depends on this one (`rprpdeps`), else
   delete — this is what makes *other* prps block on us;
3. write `:repostate` (`<blocked>` list) and clear `repounchanged` if the list changed;
4. `prpchecktime = now - prpcheckstart`;
5. send `unblocked`/`lowunblocked` events collected in `sendunblockedevents` and, if this
   arch is the cross host arch, to all other archs of the repo;
6. `killunwantedjobs` — kill jobs of packages now `disabled/excluded/locked`; kill
   not-yet-dispatched jobs of `blocked/unresolvable/broken` packages (`BuildJob.pm:256-284`);
7. write `:packstatus`; prune/unlink `:packstatus.finished`; push to redis;
8. write `:lastcheck` if the check was expensive (>120 s and >10 hard lookups);
9. repo state: `building` if any `building`, `blocked` if `havedelayed` or any `unfinished`,
   else `finished`.

### 3.6 Package status state machine

The scheduler never stores transitions, only recomputes the status on each check. The
"state machine" is therefore the set of codes and the inputs that select them:

| code | written where | meaning / inputs |
|---|---|---|
| `excluded` | checkpkgs rules 2,4,5,6,7,12,16,17; handlers (`'excluded'` from KiwiProduct/ProductCompose/Patchinfo "is built in architecture X", Channel "not target of channel", Package "project link, only depends on non-local packages") | will never build in this prpa by configuration |
| `disabled` | rules 7,11,12 | build flag off |
| `locked` | rules 3,7,12 | lock flag on: result frozen, never wiped, never rebuilt, published as-is |
| `broken` | rules 10,14,15; handlers (missing repos, no config, bad recipe, bad patchinfo...) | needs human action; retried on every check |
| `unresolvable` | rule 19; `BuildJob::create` (`'unresolvable', missing pre/vminstalls`); handlers | dep expansion failed against the current pool; retried on every check |
| `blocked` | rules 8,9,13,20; handlers; `create` DoD download; `'delayed'` | waiting on something: a dep's source package is `notready` (scheduled/blocked) in this or a dependent repo, a remote fetch, a DoD download, a `.waiting_for_<arch>` marker, a service run |
| `scheduled` | rule 18; handler `scheduled`+`build` ok; `building` | a job file exists in `jobs/<arch>/`; the dispatcher/worker own it now. The API refines this into `scheduled` (no `:status` file) / `building` / `finished` / `signing` / `dispatching` from the job status file (`bs_repserver:1293-1361`) — those are **not scheduler states** |
| `succeeded` | `'done'` and no `:logfiles.fail/<p>` | last build succeeded and nothing changed since |
| `failed` | `'done'` and `:logfiles.fail/<p>` exists; `Package.pm:205` buildsystem setup failure (fake meta <4 h old) | last build failed and nothing changed since (the scheduler does not retry failed builds until an input changes, except the 4-hour "fake meta" retry) |
| `finished` / `signing` / `dispatching` / `unknown` | *API-only*: `:packstatus.finished` overlay written by `patchpackstatus`/`dispatchdetails`, and `bs_repserver` reading `jobs/<arch>/<job>:status` | the result is being integrated or the job is in flight |

Transitions, informally (every arrow is "next check computes"):

```
                     +-------------------------------------------------------------+
                     |                                                             v
 (new/changed source, meta change, relsync trigger) --> scheduled --(built ok)--> succeeded
            ^   |                                           |  \--(built fail)--> failed
            |   +--(dep notready)--> blocked ---------------+   \--(unchanged)-->  succeeded (meta replaced, :full untouched)
            |                                                    \--(genbuildreqs)-> (re-check with new deps)
 unresolvable <--(expansion fails)   broken <--(config/recipe error)   excluded/disabled/locked <--(flags)
```

Every code except `scheduled` is "stable" (recomputed identically) until an input changes;
`scheduled` persists only while the job file exists.

### 3.7 Repository (`:schedulerstate`) codes

| code | when |
|---|---|
| `scheduling` | set at the start of a check (`bs_sched:1076`); also what the API reports as "dirty" |
| `building` | ≥1 package `scheduled` |
| `blocked` | no builds, but `unfinished` non-empty or `havedelayed`; also suspended project / service in progress |
| `finished` | nothing building/blocked; triggers publish, `REPO_BUILD_FINISHED`, drops lastcheck cache |
| `broken` | config/path errors (3.2), or `publishflags:nofailedpackages` with failed packages, or publish error |
| `disabled`, `excluded` | repo status / build type |
| `published` | *not set by the scheduler*: `bs_publish` rewrites the state after publishing |

`set_repo_state` also maintains `oldbuildid`/`buildid`/`repostateid` (md5 of the stat ids
of `:full.solv` and `:bininfo`) to emit exactly one `REPO_BUILD_STARTED` per build cycle and
a `REPO_BUILD_FINISHED` with the same id, including synthetic starts when a finished repo's
content changed without passing through `building` (`Checker.pm:157-193`).

### 3.8 The default package handler (`BSSched::BuildJob::Package::check`, `Package.pm:75-359`)

This is the heart of "do we need to rebuild". Inputs: `edeps` (expanded deps),
`notready`, `dep2src`, `depislocal`, `pool`, `lastcheck`, `:meta/<packid>`.

1. **buildinfo shortcut**: when running inside `bs_repserver` for `_buildinfo`
   (`ctx->{isreposerver}`), always answer `scheduled` (`Package.pm:91`).
2. **localdep links**: a linked package in a `linkedbuild=localdep` repo is `excluded` unless
   at least one expanded dep is local (`Package.pm:94-104`).
3. **blocked computation**: `@blocked = grep {$notready->{$dep2src->{$_}}} @$edeps` — any
   expanded dependency whose *source* is not ready. `block="never"` empties the list. Inside a
   cycle on pass ≥2, members with a lower `cyclevel` are not considered blocking
   (`Package.pm:113-121`, see 3.9). Unless `rebuild=local` (which postpones the blocked
   check), return `('blocked', "dep1, dep2, ... (max 10)")`.
4. cross-build: expand host deps against `pool_host` (`Package.pm:131-144`).
5. **lastcheck string** (`Package.pm:147-173`): `lastcheck{packid}` is
   `srcmd5(32) . metamd5(32) . hdrmetamd5(32) . "mtime/size/inode of :meta/<packid>"`.
   If absent or the `:meta` stat changed, it is rebuilt from the `:meta` file: `srcmd5` =
   first 32 chars of line 1, `metamd5` = md5 of the whole file (or the literal
   `fakefake...` for a 2-line "fake" meta written after a setup failure), `hdrmetamd5` =
   `xxxx...` placeholder.
6. If blocked and `rebuild=local` and the last build (per lastcheck) matches the current
   source and is not fake and no relsync trigger → `done` (`Package.pm:175-184`): local
   rebuild mode ignores dependency readiness once a build for the current source exists.
7. no meta at all → `scheduled` (`new build`).
8. `srcmd5` in meta ≠ `verifymd5||srcmd5` of the package → `scheduled` (`source change`,
   with `oldsource`).
9. fake meta: if younger than 4 h (14400 s) → `failed` (shown as "buildsystem setup failure"),
   else `scheduled` (`retrying bad build`).
10. `rebuild=local` or `hasbuildenv` → skip to the relsync check (rebuild only on source change).
11. **cheap dep check** (`Package.pm:219-236`): `check = md5(metamd5 . genmetaalgo .
    forcebinaryidmeta . modularity_meta . rebuildmethod . pkgid of every edep (sorted) .
    pkgid of every host dep)`. `pkg2pkgid` is the hdrmd5 (or hdrid) of the solvable. If this
    equals the stored `hdrmetamd5` → nothing in the *direct* dependency set changed → relsync
    check. This is the fast path for the overwhelming majority of packages.
12. **expensive meta regeneration** (`Package.pm:237-302`): for every edep build its
    contribution:
    * local dep (`depislocal`) and not `forcebinaryidmeta`: read the dep's own `.meta` from
      `:full/<name>.meta` (cached in `repodatas{prpa}{meta}` keyed by `/<stat id>` and name),
      and `add_meta` it: every line of the dep's meta gets `  <bin>/` prefixed to its path
      (`BSBuild::add_meta`, `BSBuild.pm:29-45`; C version `BSSolv::add_meta`), except lines
      referring to our own package (so a package never depends on its own old build);
    * non-local dep: just `"<hdrmd5>  <bin>"`;
    * host deps: `"<hdrmd5>  <hostarch>:<bin>"`; modularity meta first.
    Then `BSSolv::gen_meta(subpacks, @new_meta)` (`BSBuild::gen_meta`, `BSBuild.pm:47-113`):
    sort by depth (number of `/`), then path; drop deeper duplicates of the same `md5  lastpkg`;
    drop entries whose path passes through one of *our* subpackages (cycle); with
    `genmetaalgo=1` only prune cycle entries deeper than the first sighting. Prepend
    `"<verifymd5>  <packid>"`. If `md5(join "\n")` equals the stored `metamd5` → "nothing
    changed (looked harder)", bump `nharder`, update lastcheck with the new `hdrmetamd5`.
13. otherwise read the old meta; in cycle pass 1 compute `cyclevel = diffdepth(new, old)`
    (`BSBuild::diffdepth`, `BSBuild.pm:127-157`: 1 = source line differs, n+2 = first
    differing line has n slashes) and if >1 return `done` (postpone to pass 2).
14. `rebuild=direct`: strip all lines containing `/` from both metas (only direct deps count);
    if equal → done.
15. `diffsortedmd5(old,new)` (`BuildJob.pm:1477-1504`) produces `+path`, `-path`, `!path`
    entries → `sortedmd5toreason` → `scheduled` (`meta change`, with `packagechange` list).
16. `relsynccheck`: if `relsynctrigger{packid}` → `scheduled` (`rebuild counter sync`),
    else `done`.

Rebuild modes (`<repository rebuild="...">`, `BSXML.pm:71`):

| mode | behaviour |
|---|---|
| `transitive` (default) | full meta comparison: a rebuild is triggered by any change in the closure of binary inputs (every `.meta` line, any depth) |
| `direct` | only the top-level lines of the meta (direct deps' hdrmd5) are compared |
| `local` | only source changes (and relsync) trigger builds; dependency readiness is ignored once built |

Block modes (`<repository block="...">`):

| mode | behaviour |
|---|---|
| `all` (default) | `notready` includes local not-ready sources and `prpnotready` of all repos in the path |
| `local` | `prpnotready` ignored (`Checker.pm:718`); only packages in this prp block |
| `never` | never blocked (`Package.pm:111`, `KiwiImage.pm:96`, `Docker.pm:181`, `Image.pm:76`) |

### 3.9 Cycles

* Detection: `BSSolv::depsort2` returns cycles (or SCCs when `setdepsortsccs(2)`). The
  scheduler prefers SCCs (`ctx->{sccs}`), falling back to raw cycles.
* `cychash{packid} = [sorted members]` for every member (`Checker.pm:1152-1165`, printed as
  "cycle components").
* `handlecycle` (`Checker.pm:1060-1088`) implements a **three-pass** walk over each cycle
  within the main package loop:
  * pass 1: all members are unshifted onto `@cpacks` (plus the first one again as an end
    marker); each member is checked normally but `Package::check` computes `cyclevel` via
    `diffdepth` and returns `done` for packages whose meta differs only at depth >1 (i.e. not
    because of their own source or direct deps) — so that only the "root cause" packages are
    scheduled in pass 1.
  * pass 2: members are re-queued sorted by `cyclevel` ascending; `@blocked` is pruned of
    cycle members whose level is lower (`Package.pm:113-121`), so a package is allowed to
    build against a cycle-mate that will itself be rebuilt later, and the "postponed" ones
    now get scheduled.
  * pass 3: if any member is `notready`, all members are marked `notready` (`Checker.pm:1079-1085`)
    so that consumers outside the cycle wait for the whole cycle.
  * packages already decided (status not `done/succeeded/failed`) are not re-evaluated
    (`Checker.pm:1189`).
* `gen_meta` prunes paths through own subpackages (self-cycles) so that A→B→A does not make
  A's meta depend on A's previous build.
* At the prp level, `BSSolv::depsort` over `prpdeps` prints "cycle: ..." for repo-level
  cycles but otherwise just picks an order (`ProjPacks.pm:1230-1234`); `bs_sched:1198-1214`
  avoids ping-pong between related prps that depend on each other.

### 3.10 The meta file

Example (constructed from the format rules; `:meta/<packid>` and `<packid>/.meta.success`):

```
3b1f0c...e2 (verifymd5 of source)  mypackage
9a7d...11  gcc
5c2e...70  glibc-devel
4411...0f  glibc-devel/glibc
77aa...c3  libfoo1
77aa...c3  libfoo1/libfoo-devel        <- pruned by gen_meta: same md5+lastpkg at greater depth
```

* line 1: `"<verifymd5|srcmd5>  <packid>"` (two spaces).
* following lines: `"<hdrmd5>  <path>"` where `<path>` is `bin` for a direct dependency and
  `bin/dep/dep2...` for transitive ones (the dependency's own meta, re-rooted). Non-local
  deps (from other repos) contribute only their top line (their hdrmd5) — transitive
  information across repositories is *not* tracked; a rebuild of `glibc` in the base repo
  changes its hdrmd5, which changes the meta of every direct consumer, which changes their
  hdrmd5 after rebuild, and so on. That is how rebuild propagation across repos works even
  though only direct lines cross repo boundaries.
* `metacheck` (`BuildJob.pm:1387-1451`) is the generic version used by non-package handlers:
  compare the whole old/new meta textually; `no former build`, `src change`, fake-meta retry,
  `rebuild counter sync` (not for patchinfo/aggregate/channel/preinstallimage), `rebuild=local`
  short-circuit for image types, else `meta change` with a diff.
* The worker writes the meta it actually used (`.build.meta`, `bs_worker:4076`), which
  the repserver stores as `meta` in the job dir; `jobfinished` moves it to `:meta/<packid>`.
  Thus the comparison is always "what the scheduler would use now" vs "what the worker
  actually used last time".

### 3.11 `:full` vs `:repo` and how `:full` is updated

* `:full/` is the **build pool** of the prpa: for every binary *name* exactly one file
  `<name>.<suf>` (rpm/deb/pkg.tar.*/apk/obsbinlnk...) plus `<name>.meta`, hardlinked from
  the owning package dir `<gdst>/<packid>/<file>`. Only binaries of packages with
  `useforbuild` enabled, and only one version per name (first in `orderpackids` order
  wins; `_volatile` last; for maintenance projects the highest incident number wins;
  `volatile_cmp` prefers non-imported and higher EVR, `BuildRepo.pm:238-259, 306-342`).
  The `exportfilter` (prjconf, default per arch in `BuildResult.pm:74-127`, e.g. x86_64
  drops `-debuginfo-`/`-debugsource-`; i586 sends `.x86_64.rpm` to the x86_64 scheduler)
  decides which files go into `:full` at all and which are *exported* to other archs
  (section 6.1).
* `:repo/` is the **publish staging** tree: *all* files of all publish-enabled packages
  (not just one per name; subject to `publishfilter`, `singleexport`, `nosourceaccess`,
  `excludepublish/onlypublish`), named `<file>` or `<packid>::<file>` for debian/updateinfo,
  plus deltas and provenance, with `:repoinfo` recording origins/conflicts
  (`PublishRepo.pm:114-473`). `bs_publish` consumes it.
* `update_dst_full` (`BuildResult.pm:379-606`) is the single entry point for integrating a
  job result (or a wipe, or a `useforbuild` refresh, or an import): part 1 moves job files
  into `<gdst>/<packid>` (keeping `::import::` files, blobstore-linking `_blob.*`, writing
  `.meta.success`, `.nosourceaccess`, `.bininfo`, updating `:bininfo.merge`); part 2
  computes old/new filtered sets and calls `BuildRepo::fctx_move_into_full` unless
  useforbuild is disabled. It returns whether `:full` changed, which decides
  `changed_med{prp}=2` and a `repository` (vs `repoinfo`) event.
* `fctx_integrate_package_into_full` (`BuildRepo.pm:524-650`) has a fast path ("just same
  names with new versions and all old files seen": hardlink new over old), a medium path
  (read gbininfo, recompute the whole `:full` map for the *interesting* names only, verify
  by `stat` ids), and the slow path `fctx_rebuild_full` (`:352-424`) that stats every file in
  `:full`, deletes strays, re-adds missing, and if any package dir is out of sync with its
  `.bininfo`, rebuilds the gbininfo and recurses. `fctx_migrate_full` moves unknown
  binaries into `_volatile`.
* After integration the solv index is updated incrementally: `satrepo->updatefrombins(":full", %oldids)`
  (or `repofrombins`) and `writesolv(":full.solv")` (`BuildRepo.pm:726-737`), and the in-memory
  cached solv string for this prpa is dropped so the next check reloads it.
* `checkuseforbuild` (`BuildRepo.pm:811-883`) re-derives the allowed package set from
  flags + `buildflags:nouseforbuild:<pkg>` and, if it differs from `:full.useforbuild`,
  runs the integration with `olduseforbuild/newuseforbuild` to add/remove whole packages.
  It is triggered via `prpcheckuseforbuild{prp}` which is set on *every* project update
  (`update_prpcheckuseforbuild`) and evaluated once per check (`bs_sched:1054-1062`).
* "dirty": the main loop touches `:schedulerstate.dirty` for all prps in `changed_dirty`
  *before* picking the next prp (`bs_sched:913-921`); the repserver does the same when a job
  finishes (`bs_repserver::dirty`). The file is removed when a check starts, or the repo
  goes `broken`/`disabled` (`Checker.pm:205`). The API exposes it so clients can tell
  "finished but stale" from "finished and current".
* `unchanged` results (worker found the new binaries byte-identical to the previous ones
  modulo build dates — `bs_worker:5180-5183`, suppressed by `nounchanged`) only replace the
  meta and logfile, bump `changed_med{prp}=1` (not 2), and do not touch `:full`
  (`BuildJob.pm:588-630`). This is why a rebuild of a leaf package does not cascade.

### 3.12 Release-counter sync (`relsync`)

`<gdst>/:relsync` maps `packid => "versrel.bcnt"` of the last successful build
(`BuildJob.pm:688-697`, written to `:relsync.merge` when the file is >8 KiB to avoid
rewriting a big Storable file on every job). After each check, if `:relsync` differs from
`:relsync.sent` (stat id compare), `sendrelsyncupdate` (`bs_sched:234-309`) converts it to
`"<bcntsynctag|packid>/<versrel>" => maxbcnt`, merges into every sibling arch's
`:relsync.max` under a lock (`updaterelsyncmax`, `bs_sched:208-232`; archs can be grouped
with `$BSConfig::relsync_pool`), and sends them a `relsync` event (or POSTs to
`$srcserver/relsync` if project data looks wrong). `calcrelsynctrigger` (`Checker.pm:993-1039`)
marks packages whose own bcnt is below `:relsync.max` for their versrel (and that are not
currently failed) so they get a `rebuild counter sync` build; `nextbcnt` (`BuildJob.pm:913-935`)
uses `max(history bcnt, relsync.max - 1) + 1`. Net effect: the same `name-version-release`
gets the same build counter on every arch.

### 3.13 Generated build requires

A worker may answer with result `genbuildreqs` and a `_generated_buildreqs` file;
`set_genbuildreqs` stores `[md5, [deps], verifymd5]` in `$gctx->{genbuildreqs}{prp}{packid}`
and `<gdst>/:genbuildreqs` (`BuildJob.pm:414-440, 646-654`); the next check appends them to
the package's deps (`Checker.pm:901-907`) and `create` puts their md5 in the job
(`BuildJob.pm:1292`) so the worker can detect a change and loop until stable.

---

## 4. Build job creation and result collection (`BSSched::BuildJob`)

### 4.1 Job identity and files in `jobs/<arch>/`

`jobname(prp, packid)` (`BuildJob.pm:1341-1347`) = `"<proj>::<repo>::<packid>"`
(`/` → `::`), or `":<md5(prp)>::<packid>"` / `":<md5(prp)>:::<md5(packid)>"` when longer than
200 chars. The full job name is `<jobname>-<srcmd5>` (`create`, `BuildJob.pm:1089-1090`),
so a new source revision automatically gets a new job while the old one is killed
(`kill_otherjobs`, `:371-379`). Aggregates/channels/patchinfos (non-ptf) use the bare
jobname without srcmd5 (`Aggregate.pm:426`, `Channel.pm:317`, `Patchinfo.pm:453-454`);
delta jobs use `<prp>::_deltas[-<n>]` (`DeltaRpm.pm:67-68`).

Per job, in `jobs/<arch>/`:

| file | writer | content |
|---|---|---|
| `<job>` | scheduler `writejob` (`:319-350`) via `writexml(".<job>", "<job>")` | `$BSXML::buildinfo` XML (4.3) |
| `<job>:status` | `bs_dispatch` when dispatching (`lockcreatexml`, `bs_dispatch:245`); `bs_repserver` on worker upload (`code=finished|signing`, `result=...`); `bs_signer`; scheduler `killjob` (`code=deleting`) | `$BSXML::jobstatus` (`code, result, details, starttime, endtime, workerid, hostarch, uri, arch, job, jobid, attempt`) |
| `<job>:dir/` | `bs_repserver` (renamed from a tmpdir, `bs_repserver:2528`) | uploaded result files: binaries, `meta`, `logfile`, `_statistics`, `.bininfo`, `_generated_buildreqs`, ... |
| `<job>:<myarch>:cross` | `add_crossmarker` (`:387-397`) in `jobs/<hostarch>/` | marker so the *host* arch dispatcher sees cross jobs |
| `.dispatch.$$`, `.sched.$$`, `.<job>` | temporaries for the lock-create trick | |

The job state machine from the scheduler's point of view is file existence:
`<job>` without `:status` = *scheduled* (dispatcher may take it); `<job>` + `:status{code=building}` =
*building*; `:status{code=finished}` + `events/<arch>/finished:<job>` = ready to collect;
`:status{code=signing}` = signer owns it; `:status{code=deleting}` = scheduler is killing it.
Ownership is passed with `lockcreatexml` (write temp, flock it, `link` it to `<job>:status`
— link fails if someone else already created it, `BSUtil.pm:494-508`) and `lockopenxml`
(flock + re-stat to detect unlink races, `BSUtil.pm:444-492`).

`%ourjobs` (module global, `BuildJob.pm:119`) indexes jobs by prp prefix for
`killunwantedjobs`; `otherjobscache` in the ctx caches one `ls` of the job dir per check.

### 4.2 `create` (`BuildJob.pm:1065-1328`)

Given `ctx, packid, pdata, info, subpacks, edeps, reason, needed`:

1. If `jobs/<arch>/<jobprefix>-<srcmd5>` already exists → `('scheduled', job)` (no rewrite).
2. Build type from `pdata.buildtype || recipe2buildtype(info.file)`; `kiwimode` for
   kiwi-image/kiwi-product/docker/fissile/productcompose.
3. Paths: `path = path2buildinfopath(prpsearchpath)` (each element gets a `server` URL:
   the worker repo server, or the worker src server / partition server / remote URL for
   remote projects, `:1356-1378`); for kiwimode the image's own repositories
   (`expandkiwipath`, `_obsrepositories/` expands to the prp search path) become `path`
   and the prp search path becomes `syspath` (the build *system* repos); cross builds get
   `syspath` from `prpsearchpath_host`.
4. `@btdeps`: `obs-service-<name>` packages for `buildtimeservice` entries (via
   `substitute obs-service:<name>`).
5. `@sysdeps = Build::get_sysbuild($bconf, $buildtype, [...])` (`build/Build.pm:818-880`):
   the packages for the build *environment* when it differs from the package's own deps
   (mock, debootstrap, livebuild, kiwi, productcompose, docker/podman, deltarpm, mkosi)
   = `system-packages:<type>` substitutes + preinstall + required (+support for kiwi),
   expanded. Cached per buildtype in `ctx->{sysbuild_<type>}` when there are no extra deps.
6. `@bdeps = Build::get_build($bconf, $subpacks, info.dep, btdeps, extradeps, genbuildreqs,
   '--directdepsend--', prereq (only `/` paths with fileprovides), ['--ignoreignore--'])`
   (`build/Build.pm:726-784`): the full expanded install set for the build root
   (preinstall + required + support + deps, substitutes, own subpackages removed unless
   `keep`), or `(1, @edeps)` for kiwimode/buildenv. `--enable-ccache` / `buildflags:useccache:<pkg>`
   adds the ccache package and sets `ccache` in the job.
7. Errors → `('unresolvable', ...)`.
8. `@pdeps = get_preinstalls`, `@vmdeps = get_vminstalls`.
9. **DoD check** (`DoD::dodcheck`, `DoD.pm:189-217`): any needed package whose `pkg2pkgid`
   is the DoD placeholder `d0d0d0d0...` is recorded in `ctx->{doddownloads}{prpa[/modules]}{pkg}`
   and the package returns `('blocked', "downloading N dod packages")`; after the check
   `dodfetch` asks the repserver for `view=binaryversions&binary=...` which makes the
   repserver download the files into `:full` (`DoD.pm:246-283`), then `dodfetch_resume`
   drops the repo cache and sets the prp changed.
10. Missing pre/vminstalls → `('unresolvable', 'missing pre/vminstalls: ...')`.
11. `kill_otherjobs(jobprefix)`.
12. Build the `bdep` list: union of pdeps, vmdeps, edeps, bdeps, sysdeps with flags
    `preinstall`, `vminstall`, `runscripts`, `notmeta` (not in `edeps` → does not influence
    the meta), `installonly` (sysdep not in bdeps, non-kiwi), `noinstall` (bdep not in
    sysdeps/vm/pre — only downloaded, e.g. image content); `project/repository` of origin
    when `dobuildinfo` (buildinfo API) or `unorderedrepos`; `epoch/version/release/arch/hdrmd5`
    when `dobuildinfo`. `extrasource` entries become `repoarch=src` bdeps; `extrabdeps`
    (containers, kiwi image packages, sysroot) are prepended.
13. `create_jobdata` (`:943-1015`) fills identity fields, computes `bcnt` via `nextbcnt`,
    `release` from prjconf `Release:` with `<CI_CNT>`/`<B_CNT>` substitution, `debuginfo`
    flag, `disturl = obs://<obsname>/<proj>/<repo>/<srcmd5>-<packid>`, modularity fields.
14. Extra fields: `needed` (how many packages' edeps mention this source, from
    `create_rebuildpackage_needed`, `Checker.pm:1863-1872` — the dispatcher uses it for
    priority), `constraintsmd5`, `prjconfconstraint`, `constraint`, `ccache`, `releasename`
    (maintenance incidents), `revtime` (max over inter-project links), `logidlelimit`,
    `genbuildreqs`, `obsgendiff` release target, `slsaprovenance`/`slsabuilder`/`slsadownloadurl`,
    `signflavor`, `nouseforbuild`, `reprorepoid`/`reprojobid`.
15. `writejob` (`:319-350`): sets `job`, `readytime`, `reason` (explain string), `srcserver`,
    `reposerver` (worker-facing URLs), `genmetaalgo`, `forcebinaryidmeta`; writes
    `<gdst>/<packid>/status` (`{status=>'scheduled', readytime, job}`) and
    `<gdst>/<packid>/reason` (`$BSXML::buildreason`: `explain`, `time`, `oldsource`,
    `packagechange[]`); writes the job XML; adds the cross marker; updates `%ourjobs`;
    writes a copy to the `_reproduciblecheck` repo if configured.

### 4.3 Job XML (`$BSXML::buildinfo`, `BSXML.pm:563-694`) — representative example

Assembled from the field assignments in `create_jobdata`/`create`/`writejob` for a plain
spec build of `openSUSE:Factory/standard/x86_64` package `zlib`:

```xml
<buildinfo project="openSUSE:Factory" repository="standard" package="zlib"
           srcserver="http://backend:5353" reposerver="http://backend:5253">
  <job>openSUSE:Factory::standard::zlib-3b1f0c7e9a4d2f6b8c1e5a7d9f0b2c4e</job>
  <arch>x86_64</arch>
  <srcmd5>3b1f0c7e9a4d2f6b8c1e5a7d9f0b2c4e</srcmd5>
  <verifymd5>3b1f0c7e9a4d2f6b8c1e5a7d9f0b2c4e</verifymd5>
  <rev>42</rev>
  <disturl>obs://build.opensuse.org/openSUSE:Factory/standard/3b1f0c7e9a4d2f6b8c1e5a7d9f0b2c4e-zlib</disturl>
  <reason>meta change</reason>
  <needed>137</needed>
  <revtime>1760000000</revtime>
  <readytime>1760012345</readytime>
  <file>zlib.spec</file>
  <versrel>1.3.1-2</versrel>
  <bcnt>5</bcnt>
  <release>2.5</release>
  <debuginfo>1</debuginfo>
  <subpack>zlib-devel</subpack>
  <subpack>libz1</subpack>
  <subpack>zlib-devel-static</subpack>
  <genmetaalgo>1</genmetaalgo>
  <bdep name="aaa_base" preinstall="1" runscripts="1" notmeta="1"/>
  <bdep name="bash" preinstall="1" runscripts="1" notmeta="1"/>
  <bdep name="rpm" preinstall="1" notmeta="1"/>
  <bdep name="kernel-default" vminstall="1" notmeta="1"/>
  <bdep name="gcc"/>
  <bdep name="glibc-devel"/>
  <bdep name="pkgconf-pkg-config"/>
  <bdep name="build" notmeta="1"/>
  <bdep name="rpm-build" notmeta="1"/>
  <path project="openSUSE:Factory" repository="standard" server="http://backend:5253"/>
  <path project="openSUSE:Factory:Rings:1-MinimalX" repository="standard" server="http://backend:5253"/>
</buildinfo>
```

Field semantics the worker relies on: `bdep` with `notmeta` are installed but not part of
the meta; bdeps without any flag are the "edeps" that *do* form the meta (the worker
writes `<hdrmd5>  <name>` lines for them into `.build.meta`); `path` is the repository
list the worker fetches binaries from, in search-path order; `syspath` (image types)
is where the *build system* packages come from; `needed`/`readytime` drive dispatcher
priority; `nounchanged` disables the "unchanged" short-cut on the worker.

### 4.4 Per-recipe-type handler summary

Each handler implements `expand` (for sorting), `check` (returns status or
`('scheduled', data)`), `build` (creates the job from `data`), and some implement
`jobfinished`. `%handlers` in `Checker.pm:66-84`: `productcompose, kiwi-product,
kiwi-image, docker, fissile(=Docker), patchinfo, aggregate, preinstallimage, channel,
mkosi/appimage/livebuild/simpleimage(=Image), reproduciblecheck, unknown, default(=Package)`.
(`spec`, `dsc`, `arch`, `helm`, `flatpak`, `snapcraft`, `collax`, `buildenv` fall through
to `Package`; `BuildEnv.pm` is used only by the repserver's buildinfo path.)

| type | `check` essentials | `build` / result | worker job? |
|---|---|---|---|
| **Package** (`Package.pm`) | 3.8 | `BuildJob::create` with edeps; cross builds split sysroot/host deps and build against `pool_host` | yes |
| **PreInstallImage** (`PreInstallImage.pm:72-122`) | blocked on notready; meta = `gen_meta` of the *full* `get_build` set + pre/vminstalls (the image contains the whole build env) | `create`; result processed by `update_preinstallimage` (`:131-190`) into `<gdst>/:preinstallimages` (bitstring index over hdrmd5s) for the dispatcher/worker to pick a matching preinstall image | yes |
| **Image** (mkosi/appimage/livebuild/simpleimage) | blocked on notready; meta from `pkg2pkgid` of edeps; `forcebinaryidmeta` | `create` | yes |
| **KiwiImage** (`KiwiImage.pm:84-353`) | own repo list from the kiwi file (`expandkiwipath`, `_obsrepositories/` → prp search path), own config (`getconfig` over that path, `type=kiwi`), possibly `unorderedrepos` with priorities (`createextrapool`), container deps resolved in a container pool with `containerpath`, annotation of the base container; blocked via `notready`/`prpnotready` per repo of each expanded package; meta = `"<hdrmd5>  <prp>/<bin>"` sorted + container metas + extrasource; `metacheck`; DoD check | `create` with image packages as `edeps` (or as `noinstall` extrabdeps for buildinfo/unordered), `syspath` = prp search path, container bdeps/`containerpath`/`containerannotation` | yes |
| **Docker** (`Docker.pm:169-460`) | like KiwiImage; additionally the base container's annotation `repo` URLs are mapped to prps (`getpathfromannotation`, `maptoremote` for partitions, `BSUrlmapper`) and appended to `info.path` via `append_info_path` (triggers a project postprocess and may return `delayed 'remotemap entry missing'` while the remote project is fetched); `filterbasecontainerpkgs` turns installed packages of the base image into `-pkg` exclusions | `create` | yes |
| **KiwiProduct** (`KiwiProduct.pm:78-653`) | one *build arch* (first repo arch, or `local`); other archs only check readiness and report via `.waiting_for_<arch>` markers + `unblocked` events, returning `excluded "is built in architecture X"`; on the build arch: sysbuild expansion against `localbuildarch` pool, then walk every path prp × every arch's gbininfo (`read_gbininfo`, remote via cookie cache) selecting rpms by the product's package list (`BSSolv::kiwiproductcheck` fast path), blocked if any needed package is scheduled/blocked/finished in that arch's `:packstatus`; meta = `"<hdrmd5>  <prp>/<arch>/<pkg>/<name>.<arch>"` per rpm; `metacheck` | `create` with all rpms as `extrabdeps` (`project/repository/repoarch/package/binary`), sysdeps from the kiwi pool | yes |
| **ProductCompose** (`ProductCompose.pm:78-747`) | same structure as KiwiProduct with `imagearch` mapping, versioned deps (`name >= ver`, `Build::matchsingledep`), `--unorderedproductrepos`, patchinfos (`updateinfo.xml`) and `_modulemd.yaml` included, `buildflags:productcompose-onlydirectrepos` | same | yes |
| **Aggregate** (`Aggregate.pm:144-834`) | for each `<aggregate>` element: source project/repos (target mapping), package list (explicit, or derived from gbininfo + binary/binaryarch filters), blocked if any source package is scheduled/blocked/finished (local `:packstatus` or remote packstatus), broken if a source prp is missing/inaccessible, `delayed` while remote gbininfo is fetched; meta = md5 of file list+stat ids (local) or hdrmd5s (remote) per `<prp>/<arch>/<pkg>`; `_repository` aggregates pull from `:full` | **built in-process**: copies/links the binaries (remote via `view=cpio`), containers with blobs, helm charts, product `.report` subsets, provenance, into `jobs/<arch>/<job>:dir`, writes `meta`+`logfile`, then `fakejobfinished` → `finished` event (via signer if resign) → `Aggregate::jobfinished` integrates with `update_dst_full` | no (fake) |
| **Channel** (`Channel.pm:73-385`) | target matching via release targets, binary filter per repo (`filter{"<repo>/<name>"}`, `packagefilter`, `importedfrom`), blocked on scheduled/blocked/finished/unknown of any package in the project, broken if a filtered binary is missing; meta = md5(stat id) per selected binary | in-process: links binaries + appdata + provenance + `.checksums` into job dir, writes `.channelinfo`, `fakejobfinished_nouseforbuild` → result goes straight to `<gdst>/<packid>` with `.nouseforbuild` (never in `:full`) | no |
| **Patchinfo** (`Patchinfo.pm:111-791`) | one build arch (first repo arch or own arch with `seperate_build_arch`); other archs maintain `.waiting_for_` markers; blocked until every package of the project is done on every arch (`:packstatus` + `:logfiles.success/fail`); `ptype` `binary` (maintenance incident: meta = stat ids of all rpms per arch/package) or `local` (meta = own srcmd5 only) | in-process: links all rpms/containers/charts of all archs, reuses old packages per `.updateinfodata`, fetches issue trackers from the srcserver (synchronous RPC!), writes `updateinfo.xml` (`$BSXML::updateinfo`), `.updateinfodata`; `category=ptf` instead creates a real worker job with a generated `ptf.spec` (`build_ptf_job`) | no (ptf: yes) |
| **DeltaRpm** (`DeltaRpm.pm:58-204`) | invoked by `PublishRepo::makedeltas` (not by checkpkgs): batches of `[old,new,deltaid]` ≤ 500 MB, ≤ 100 running jobs | job `file=_delta`, bdeps = `deltarpm` expansion; `jobfinished` moves `<deltaid>{,.dseq}` into `<gdst>/_deltas/` | yes |
| **Import** (`Import.pm:45-155`) | n/a | `createexportjob` writes a pre-finished job into **another arch's** `jobs/<arch>/` with the exported binaries hardlinked and `replaced.xml`, sends `import.<job>`; the receiving scheduler's `jobfinished` integrates them as `::import::<fromarch>::<file>` with `importarch` | no |
| **Upload** (`Upload.pm:38-94`) | n/a | API-uploaded build results (`bs_repserver:2919`) integrated like a build | no |
| **Reproduciblecheck** (`Reproduciblecheck.pm:68-424`) | in a `<repo>_reproduciblecheck` repository: replays the saved `.reprojob` of the base repo's build (same `buildtime`), then a second "compare" job with `reproduciblecheck.spec` comparing `a_*`/`b_*` files; result into `:reproduciblecheck.fail/` and the base package dir | yes (2 jobs) |
| **BuildEnv** (`BuildEnv.pm:67-170`) | repserver-only (`_buildinfo` with a supplied `_buildenv`): resolves each bdep by `name.hdrmd5` across pool and gbininfos | `create` with `extrabdeps` | — |
| **Unknown** | `('broken', 'unknown package type')` | | |

### 4.5 Collecting results: `jobfinished` (`BuildJob.pm:498-704`)

Triggered by `event_built` after locking `<job>:status` (must be `finished`) and reading
the buildinfo (`EventHandler.pm:98-135`). Dispatch: `reprojobid` → Reproduciblecheck;
`file=_aggregate` → Aggregate; `file=_delta` → DeltaRpm; else:

1. Sanity: project/package still exist; `<gdst>/<packid>/status.job` must equal this job
   (otherwise "outdated", result discarded — a newer job superseded it).
2. `update_buildavg(now - readytime)`.
3. `code = js.result` ∈ {succeeded, unchanged, genbuildreqs} else `failed`.
4. No `meta` file + success → discard; no meta + failure → write a 2-line fake meta
   `"<verifymd5>  <packid>\nfake to detect source changes...  fake"` (this is what makes
   `Package::check` report `failed` for 4 h then retry).
5. `patchpackstatus` (append to `:packstatus.finished`, touch `:packstatus`, redis).
6. `unchanged`: reuse last success logfile, append a note, `addjobhist('unchanged')`, move
   meta, `changed_med{prp}||=1`; if `nouseforbuild` flag flipped, refresh bininfo/`:full`.
7. `failed`: logfile → `<pkg>/logfile` and `:logfiles.fail/<pkg>`, meta → `:meta/<pkg>`,
   `status=failed`, `addjobhist('failed')` (+ `.failhistory`), `changed_med{prp}||=1`.
8. `genbuildreqs`: store, `changed_med{prp}||=1` (next check creates a new job).
9. `succeeded`: mark `.nouseforbuild`/`.preinstallimage` in job dir, `addbuildstats`
   (`_statistics` → `.stats` BSFileDB), **`update_dst_full`** (3.11) → `changed_med{prp}=2`
   if `:full` changed and `repounchanged` cleared/downgraded, meta → `:meta/<pkg>`,
   `status=succeeded`, `addjobhist('succeeded')`, `addsucceededhist` (`<pkg>/history`
   BSFileDB: versrel bcnt srcmd5 rev time duration), `:relsync` update, logfile →
   `<pkg>/logfile` + `:logfiles.success/<pkg>`, remove `:logfiles.fail/<pkg>`.
10. `event_built` then `purgejob` (remove `<job>`, `<job>:status`, `<job>:dir`).

Note that `jobfinished` does **not** re-check the package: it only integrates the result
and marks the prp changed at med priority; the next `checkpkgs` recomputes everything,
which is why `:packstatus.finished` exists to hide the gap.

### 4.6 History and diagnostics

* `:jobhistory` (`addjobhist`, `BuildJob.pm:879-892`): one BSFileDB line per finished job
  (`$BSXML::jobhistlay`), consumed by the API `_jobhistory`; `<pkg>/.failhistory` keeps a
  fail counter.
* `<pkg>/history`: successful builds (`addsucceededhist`), the source of `nextbcnt`.
* `<pkg>/.stats`: flattened `_statistics` (`flat_hash`) per build.
* `<pkg>/reason`: why the last job was created (`explain` ∈ {new build, source change,
  meta change, rebuild counter sync, retrying bad build, buildinfo generation}, plus
  `packagechange` list from `diffsortedmd5`).
* `dispatchdetails`: the dispatcher cannot find a worker (constraints, no free worker of
  the arch, ...) → it writes an event with `details`; the scheduler appends
  `scheduled <pkg>/<job>/<details>` to `:packstatus.finished` so the API can display
  the reason; `prune_packstatus_finished` keeps only lines whose job is still the current
  one (`Checker.pm:1041-1058`).
* `writeschedulerinfo` (1.2 step 11) is the only scheduler self-monitoring output;
  `memstats`/`dumppmat`/`projectstats` events give ad-hoc introspection.

---

## 5. Repositories as build dependencies

### 5.1 Search path expansion and the prp dependency graph

`expandsearchpath(gctx, projid, repo[, 'hostsystem'])` (`ProjPacks.pm:828-858`): start with
the repo's own prp, then its `<path>` elements in order; only the **last** element of the
path is expanded recursively (its own project's `<path>` list is appended), with
de-duplication. This is OBS's "the last path entry inherits its paths" rule. Remote
projects are looked up in `remoteprojs` (their `repository[].path` came with the remotemap).

`setup_projects` (`ProjPacks.pm:921-1265`) then derives, per prp of this arch:
`prpsearchpath` (the above), `prpdeps` = searchpath ∪ aggregate source prps (obeying
`<repository target= source=>` mappings) ∪ kiwi/container `path`/`containerpath` prps ∪
(for channel repos) all non-channel repos of the project; `relatedprpdeps`; cross-build
`prpsearchpath_host`. Finally `prps` is topologically sorted with `BSSolv::depsort(prpdeps)`
and `rprpdeps`/`rrelatedprpdeps` are inverted. Incremental mode (`projids_todo`, used by
`get_projpacks_postprocess_projects`) splices out and recomputes only the given projects
plus everything that depends on them via `rprpdeps`/`expandedprojlink`; above 1000
projects it falls back to a full recompute (`ProjPacks.pm:929`).

The build config of a prp is the concatenation of the configs along `prpsearchpath`
(3.2), so a prjconf change in a base project changes `bconf` of all dependents without
any explicit event: dependents are rechecked because the base project's change puts
its prps in `changed_high`, whose `rprpdeps` land in `lookat_next`.

### 5.2 Loading repositories into the pool: `addrepo` and `RepoCache`

`Checker::addrepo(pool, prp, arch)` (`Checker.pm:1692-1725`):

1. `repodatas->addrepo(pool, prp, arch)` (`RepoCache.pm:48-96`): if the cache entry is
   younger than 24 h (+ random ≤30 min jitter) it returns a repo from the in-memory solv
   string (`solv`), or from a `solvfile` (remote cache file, atime bumped), or from
   `<reporoot>/<prp>/<arch>/:full.solv`; an `error` entry short-circuits to "unavailable".
   Modular repos are keyed additionally by the module set.
2. Cache miss, remote project → `Remote::addrepo_remote` (5.2.1).
3. Cache miss, other arch (`arch ne myarch`, e.g. kiwi product on the build arch reading
   another arch's `:full`) → `BuildRepo::addrepo_scan` with a per-check `alien_repo_cache`
   of solv strings (not kept across checks: "we don't cache local alien repos",
   `RepoCache.pm:41-45`).
4. Cache miss, local → `addrepo_scan` (`BuildRepo.pm:902-970`): load `:full.solv` if present
   (external/DoD-only repos return immediately), merge DoD metadata (`put_doddata_in_cache`),
   `readdir(":full")` for binaries (or `:full.subdirs`), stat each one, `updatefrombins`
   (libsolv re-reads only files whose `mtime/size/inode` id changed) and rewrite
   `:full.solv` if dirty; `setcache(prp, arch)` records `lastscan`.
5. `checkdodresources` (`Checker.pm:1670-1690`) for registry DoD repos: compares the
   `container:` deps needed by this prp against `repo->dodresources()` and, if some are
   missing, `DoD::signalmissing` writes `<prp>/<arch>/:full/doddata.needed` (locked
   Storable `{ "<arch>" => [resources], '' => {arch=>time} }`) or POSTs
   `cmd=missingdodresources` to the srcserver for partitioned DoD repos.

Consequences: a prp's `:full.solv` is the unit of exchange between (a) the scheduler that
owns it, (b) other local schedulers reading it as an alien arch, (c) `bs_repserver`
serving it to remote schedulers (`view=solvstate`), and (d) workers (indirectly: the
worker fetches binaries by name from `<server>/build/<prp>/<arch>/_repository`). It is
rebuilt incrementally after every result integration (`fctx_move_into_full`) and
validated by directory scan at most once per 24 h.

#### 5.2.1 Remote repositories (interconnect)

`addrepo_remote` (`Remote.pm:362-412`): `GET <remoteurl>/build/<remoteproject>/<repo>/<arch>/_repository?view=solvstate`
(partition or `$BSConfig::usesolvstate`; else `view=cache` with modules) received as cpio
with members `repositorysolv` (a solv file), `repositorycache` (thawed BSSolv cache),
`repositorystate` (the remote `:repostate` XML → `prpnotready{prp}`); async when
`asyncmode`, in which case `addrepo` returns `0` and the prp is `delayed` until
`addrepo_remote_resume` fires `setchanged_unless_othersinprogress`. The solv is written to
`remotecache/<xx>/<md5(prp/arch[/modules])>.solv` and cached with `solvfile`; on transient
errors the stale cache file is used and a `repository` retry event queued
(`addrepo_remote_unpackcpio`, `:432-538`).

Remote bininfo/packstatus (needed by aggregates, kiwi products, buildenv):
`read_gbininfo_remote` (`:621-704`) with a 1 h cache in `remotecache/<md5>.bininfo`,
cookie-validated against `view=packstatus&withbininfocookie=1` for partitions,
otherwise `view=gbininfocode` (partition) or `view=binaryversionscode` (foreign OBS).

Watches (`setup_watches`, `Remote.pm:77-186`): for every remote project referenced by
`projpacks_linked` (source links), `expandedprojlink`, `rprpdeps` and
`prpsearchpath_host`, build a filter list `project/<P>`, `package/<P>[/<pkg>]`,
`repository/<P>/<R>/<arch>` per remote URL; `RemoteWatcher` turns that into the
`lastevents` long-poll (more than 3 package filters for one project collapse to the whole
project). Remote events are translated to local `project`/`package`/`repository`/`repoinfo`
events with the *local* (possibly prefixed, `remoteprojid`, `:246-296`) project id. A lost
sequence (`sync=lost`) regenerates synthetic events for every watch (`create_resync_events`).

### 5.3 Download on demand (DoD)

Declaration: `<repository><download arch= repotype= url= archfilter= master= pubkey=>` in
project meta (`BSXML.pm:46-58`). The scheduler only (a) mirrors these into
`$bsdir/dods/<proj>::<repo>::<arch>` XML files (`DoD::update_doddata`/`init_doddata`) and
touches `dods/.changed`, (b) consumes the parsed metadata the `bs_dodup` daemon drops into
`<prpa>/:full/doddata` (+ `doddata.cookie`), and (c) triggers downloads.

`bs_dodup` (`bs_dodup:1015-1067` daemon loop): every 10 s rescan `dods/` if `.changed`,
and for each prpa whose `lastcheck + 1h` (10 min after errors, or immediately if
`doddata.needed` is newer) has passed, call the repotype handler (`dod_rpmmd`,
`dod_susetags`, `dod_deb`, `dod_arch`, `dod_apk`, `dod_mdk`, `dod_registry`,
`:267-764`): fetch the index (with master mirror check, GPG verification, checksum
verification), compute a cookie, and if changed parse it with `Build::Repo::parse` into a
Storable `{ "name.arch" => {name, epoch, version, release, arch, path (location), hdrid, provides, requires, ...}, '/url' => baseurl, '/moduleinfo' }`
keeping only the highest version per name/arch (`addpkg`, `:792-827`), write it to
`:full/doddata`, update the cookie and send a `scanrepo` event to the arch's scheduler.
Registry DoD (`dod_registry`, `:646-764`) resolves only the `container:<repo>:<tag>`
resources listed in `doddata.needed`, fetching manifests (HEAD with digest short-cut),
fat manifests per arch, and the CycloneDX attestation to fill `annotation.installed`.

On the scheduler side `readparsed` (`DoD.pm:48-77`) gives every entry `hdrmd5 =
'd0d0d0d0...'` and `id='dod'`, `put_doddata_in_cache` injects/updates them into the solv
repo (`repo->updatedoddata`), and `clean_obsolete_dodpackages` removes downloaded files
from `:full` that the new metadata no longer lists. Once a real file is downloaded by the
repserver (on `dodfetch`), the scan replaces the placeholder with the real hdrmd5, so
the meta of consumers changes and they rebuild exactly once.

### 5.4 Binary indexes

Three indexes exist per prpa, serving different consumers:

| index | key | consumer |
|---|---|---|
| `:full.solv` (libsolv) | binary name → provides/requires/evr/hdrmd5/path | dependency expansion (this and other schedulers via cache/remote) |
| `:bininfo` (+ `:bininfo.merge`) = "gbininfo" | packid → filename → nevra/hdrmd5/leadsigmd5/md5sum/stat id/source | aggregates, kiwi products, productcompose, buildenv, channel; API `view=binaryversions`/`gbininfocode`; remote schedulers |
| `<packid>/.bininfo` | filename → … | the per-package source of truth for gbininfo (rebuilt by `create_bininfo` querying each rpm/deb when missing) |

`:bininfo.merge` is a write-ahead overlay: `update_bininfo_merge` (`BuildResult.pm:243-310`)
writes `{packid => bininfo|undef}` entries there (with a `/outdated` marker while the
`dstcache` holds unflushed entries) and `read_gbininfo` folds it into `:bininfo` on the next
read; merges over 100 KB are folded eagerly. Corruption or an `/outdated` marker after a
crash triggers `rebuild_gbininfo` (stat every package dir).

### 5.5 Publishing hand-off

`Checker::publish` (`Checker.pm:1523-1636`) runs when the prp is `finished` (or always
for kiwi/productcompose unless `publishflags:noearlypublish|noearlykiwipublish|archsync`).
It computes `repodonestate` = md5 of (patternmd5, publish-enabled packids, unfinished
packids, exclude/only publish flags); if `:repodone` holds the same value nothing is done.
Otherwise `PublishRepo::prpfinished` (`PublishRepo.pm:114-473`):

1. take `$reporoot/<prp>/.finishedlock` (flock; wait ≤300 s when called from a normal
   check, forever for repo deletion; timeout → `delayed:lock timeout` → retry event);
2. `makedeltas` if `repotype` has `deltainfo`/`prestodelta`: compare against the
   *published* trees of all search-path prps (`BSUrlmapper::get_extrep`), create
   `_delta` jobs for missing deltas, return `building` while they run;
3. walk packages in `orderpackids` order, hardlink files into `:repo` (first origin wins,
   conflicts recorded), apply filters, link deltas/provenance, copy `.checksums`;
4. delete obsolete files from `:repo`; write `:repoinfo` `{binaryorigins, conflicts}`;
   `.newchecksums`; `.archsync` bookkeeping (`publishflags:archsync` lets `bs_publish`
   wait until all archs are current);
5. release the lock; `sendpublishevent` → `events/publish/<P>::<R>` if anything changed.

`:repodone` is written only when `unfinished` is empty, so a `blocked` repo keeps
republishing partial state on every check (kiwi early publish), and is unlinked by every
result integration (`unlink :repodone` in `jobfinished` and friends) so the next
`finished` republishes.

`bs_publish` (out of scope) reads `:repo`/`:repoinfo`, creates repository metadata, moves
it to the public tree and writes `:schedulerstate` = `published`.

### 5.6 `:relsync` — see 3.12.

---

## 6. Multi-arch and cross-prp coupling

### 6.1 noarch and binary export between archs (`exportfilter`)

Every arch builds *all* packages independently (there is no "noarch builds once" rule
in the scheduler). Sharing is explicit: the prjconf `ExportFilter:` (or the per-arch
defaults in `BuildResult.pm:74-127`) maps filename regexes to a list of target archs
(`.` = keep locally). In `update_dst_full` part 2, matching files are passed to
`Import::createexportjob` (`BuildResult.pm:574-586`, `Import.pm:45-109`), which writes a
pre-finished job directly into `jobs/<targetarch>/` (status `finished/succeeded`, files
hardlinked, `replaced.xml` listing the previous export's ids) and sends `import.<job>` to
that arch. The target scheduler integrates them as `::import::<srcarch>::<file>` in the
package dir and (if useforbuild) in its `:full`; imported files sort last in `:full`
precedence (`volatile_cmp`) and are never re-exported or published
(`PublishRepo.pm:307`). This is how i586 gets x86_64's `-32bit` baselibs, how
`local`/`x86_64` product builds obtain other archs' rpms when the exportfilter says so,
and in general the only inter-arch binary channel besides the kiwi/product handlers
reading other archs' trees directly.

`wipe`/`wipeobsolete` keep imports unless `wipeallarch` (`BuildResult.pm:810-833, 877-911`).

### 6.2 Cross-arch build ordering ("arch sync")

* **`unblocked`/`lowunblocked` events**: a scheduler collects `sendunblockedevents{"<prp>/<arch>"}`
  during a check (KiwiProduct/ProductCompose/Patchinfo: "I, non-build arch, am now ready"
  or "please recheck", Aggregate with `arch=` mapping, cross-build host arch) and sends
  them after `checkpkgs` (`Checker.pm:1441-1460`). The receiver just marks the prp changed
  (med/low). Nothing is carried in the event; the receiver re-reads the other arch's
  `:packstatus`/`:logfiles.*`/`:bininfo` from the shared filesystem.
* **`.waiting_for_<arch>` markers** in `<buildarch>/<packid>/`: written by the non-build
  arch when it is blocked (so the build arch's API status shows the wait) and by the
  build arch per blocked arch; removed when unblocked (`KiwiProduct.pm:230-239, 529-567`,
  `Patchinfo.pm:167-220, 322-327`).
* **`:schedulerstate.dirty` of another arch** is touched to make that arch look stale
  (`Patchinfo.pm:212`, `bs_sched:1207-1208`).
* **`relsync`** (3.12) synchronises build counters.
* **Cross builds** (`hostsystem`): the target arch scheduler expands sysroot deps against
  its own pool and host deps against `pool_host` built from `prpsearchpath_host` for
  `crosshostarch` (`Checker.pm:474-509, 721-729`); the job gets `hostarch`, `crossarch`,
  `syspath`, sysroot bdeps with `sysroot="1"`, and a cross marker in `jobs/<hostarch>/`
  so the host arch's dispatcher dispatches it; the host arch scheduler sends `unblocked`
  to all other archs after every check (`Checker.pm:1452-1460`).

### 6.3 Blocking across projects/repositories

The only cross-prp "blocked" signal is `prpnotready{prp} = {srcname => 1}` (3.5 step 2),
kept in the memory of the scheduler for local prps and refreshed on every check of that
prp, and for remote prps from the `repositorystate` cpio member (`Remote.pm:470-476`),
i.e. the remote scheduler's `:repostate` file. Since `prpnotready` is only populated for
prps that have dependents (`rprpdeps`) and is cleared on a cold start (restored from the
state dump on warm restarts), a cold-started scheduler may let dependents build against
a base repo that is actually mid-rebuild until the base prp is checked once — the
topological order of the first series mitigates this.

A package blocked by a dependency in *another* prp is shown as `blocked` with the
dependency names; when the base package finishes, the base prp's check clears its
`notready`, `changed2lookat` puts all `rprpdeps` into `lookat_next`, and the dependent is
rechecked in the next low series (or sooner, via med if the base check marked it).

### 6.4 Aggregates and products spanning projects

Aggregates read other projects' package dirs (`<reporoot>/<aprp>/<arch>/<apackid>` or
`:full` for `_repository`) directly, remote ones via `view=cpio`. Kiwi products /
productcompose read `<aprp>/<arch>/:packstatus` and gbininfo of **every** arch of every
path prp. Both register the source prps in `prpdeps` (5.1) so they are rechecked when the
sources change, and both report `delayed` while remote data is in flight.

### 6.5 Access control

`BSSched::Access` (`Access.pm`): `checkaccess(type, projid, packid, repoid)` evaluates the
`access`/`sourceaccess`/`publish`/... flags with `BSUtil::enabled` (repo/arch-specific
enable/disable scoring); `checkprpaccess(aprp, prp)` (`:430-448`) decides whether `prp` may
use `aprp`'s binaries: allowed if `aprp` is public, or both are protected and in the same
project, or publish flags are compatible and every maintainer/reader of `prp` has at
least reader role in `aprp` (`checkroles`, including parent projects by `:` prefix). Used
in `createpool`/`createextrapool`, aggregates (`checkprojectaccess`) and products. The
`sourceaccess` flag controls `.nosourceaccess` markers (no src/debuginfo in `:repo`, no
sources in aggregates). Note "remote project access checks are handled by the remote
server" (`Access.pm:372-373`): foreign OBS projects are treated as accessible.

### 6.6 Repository change notification to other instances

`repounchanged` tracks whether a prp's `:full` (deleted = changed), only its bininfo (2),
or nothing (1) changed since the last `sendrepochangeevent`; the main loop sends
`repository`/`repoinfo` events after every check (`bs_sched:1049, 1073, 1114`) into
`events/repository/`, which the repserver/dispatcher forwards to the srcserver's event
log so that remote schedulers (and partition siblings) polling `lastevents` re-fetch the
solv/bininfo (5.2.1).

---

## 7. Concurrency, consistency and scaling limits

### 7.1 Single-threaded with cooperative async I/O

One Perl process per arch, no forks (except `bs_dodup`'s decompressors), no threads. The
only concurrency inside the process is the `BSSched::RPC` multiplexer (`RPC.pm`): an
`xrpc` call with `param->{async}` starts a non-blocking `BSRPC::rpc` and returns a handle;
the main loop `select()`s on all handle sockets together with the event FIFO
(`bs_sched:853-861, 961-962`) and, when one is readable and finished, calls its `_resume`
callback. Per *resource* (`"<projid>"` for projpack fetches, `"repository/<prpa>"`,
`"bininfo/<prpa>"`, `"dodfetch/<prpa>"`, `"missingdodresources/<prp>"`) only one request
is in flight; duplicates become wakeups (`xrpc_addwakeup`) or are queued in order
(`_nextxrpc`); per server at most `maxserverload` (default 1!) requests run, the rest wait
in `iswaiting_serverload{server}` with a 2:1 high:low drain ratio (`RPC.pm:86-137, 335-356`).
While the scheduler is checking a prp nothing is read, so a 10-minute check means a
10-minute latency for every event and RPC completion. In sync mode (`sched_asyncmode`
unset) every projpack fetch and remote repo fetch blocks the loop (with 60 s sleeps and
retries, `ProjPacks.pm:181-203`).

Aggregates, channels and patchinfos are *built inside* this thread (copying files, remote
`view=cpio` downloads, a synchronous `issue_trackers` RPC in `Patchinfo.pm:719-727`), which is
why `scheduler_buildtime` is subtracted from `prpchecktime` before computing back-off.

### 7.2 Atomicity and locking patterns

* **write-temp + fdatasync + rename** for every XML/Storable/solv file (`BSUtil::writexml`,
  `BSUtil::store`, `BuildRepo::writesolv`; temp name `.<file>` or `<file>.new` in the same
  directory). Readers therefore see either the old or the new file, never a torn one.
* **Hardlink-based moves**: binaries and metas are shared between the package dir,
  `:full`, `:repo`, job dirs and other archs' job dirs via `link` (+ `.dup`/rename dance to
  overwrite, `BuildRepo.pm:166-173`). `fctx_check_linkedmeta` caps the link count
  (`$BSConfig::maxmetahardlink`, btrfs limitation) by copying the meta.
* **Identity by `mtime/size/inode`** (`"$s[9]/$s[7]/$s[1]"`): used everywhere as a cheap
  content id — `:full` entries vs package files, `.bininfo.id`, `:relsync.sent`,
  `repostateid`, `:lastcheck` validity, meta caches, DoD cookies, channel/aggregate metas.
  This ties the design to a single POSIX filesystem: inode numbers are part of the state.
* **flock-based ownership**: job `:status` (`lockcreatexml` = create-if-absent via `link`,
  then `lockopenxml` by whoever processes it: dispatcher, repserver, signer, scheduler),
  `<prp>/.finishedlock` (scheduler vs publisher), `:relsync.max` (schedulers of all archs
  of one prp), `:full/doddata.needed` (schedulers vs `bs_dodup`), `run/bs_sched.<arch>.lock`.
* **Last-writer-wins files without locks**: `:schedulerstate.dirty` (touch by anyone),
  `:packstatus.finished` (append by scheduler only, but read racily by repserver),
  `events/*` (per-name overwrite), `:full.metacache.merge`/`:relsync.merge`/`:bininfo.merge`
  (scheduler-only overlays merged at the start of the next check, `bs_sched:1024-1037`).
* **Crash markers**: `:bininfo.merge` `/outdated`, `rebuild_full_tree:P:R` event, the
  `.dead` state dump, `delayed startup` package errors, `:schedulerstate.dirty`.

### 7.3 Shared-filesystem assumptions

Everything below `$bsdir` must be one coherent filesystem visible to: all `bs_sched`
instances (they read each other's `:packstatus`, `:logfiles.*`, `:bininfo`, `:full`,
`:full.solv`, `:meta`, package dirs and write into each other's `jobs/<arch>/`,
`events/<arch>/`, `:relsync.max`, `.waiting_for_*`, `:schedulerstate.dirty`), `bs_dispatch`
(reads `jobs/*`, writes `:status`, reads `:preinstallimages`), `bs_repserver` (serves
`:full`, package dirs, `:packstatus`, `:schedulerstate`, `jobs/*`; writes job dirs and
events), `bs_publish` (reads `:repo`, writes `:schedulerstate`), `bs_signer` (job dirs),
`bs_dodup` (`:full/doddata*`, `dods/`). Workers do **not** touch the filesystem; they
talk HTTP to the repserver/srcserver URLs embedded in the job. Partitions
(`$BSConfig::partition`) are the existing mechanism to split this into several such
filesystems; interconnect across partitions goes through the srcserver (`remotemap`,
`lasteventsproxy`, `view=solvstate`).

### 7.4 What grows with N (memory and time)

Let P = projects, K = packages (all projects, this arch), R = prps for this arch,
D = average path depth, B = binaries in a repository, E = average expanded deps per package.

| structure | size | notes |
|---|---|---|
| `projpacks` | O(K × (info fields + deps)) | the dominant resident set: every package's `dep[]` (BuildRequires) for every repository it builds in; `bigdeps` interning only helps for lists >100 entries; `channeldata` deduplicated by md5 |
| `projpacks_linked` | O(number of source links) | plus `_blks` |
| `prpsearchpath`, `prpdeps` | O(R × D) | |
| `rprpdeps`, `rrelatedprpdeps` | O(R × D) | reverse maps; `extend_lookat_next` walks them transitively |
| `prps` sort | O(R × D) time per `setup_projects` (`BSSolv::depsort`) | full recompute for >1000 changed projects |
| `lastcheck` | O(K_active × ~130 B) | only for prps currently being watched (dropped when finished) |
| `repodatas` solv cache | O(R_touched × B) strings | 24 h TTL; dropped for leaf prps when finished (`bs_sched:1171-1192`); remote ones live in `remotecache/` files |
| `prpnotready` | O(R_nonleaf × blocked sources) | |
| `remotepackstatus`, `remotegbininfos` | O(remote prpas × packages) | 1 h TTL |
| `lookat_*`, `changed_*` | O(R) | |
| `%ourjobs` | O(running jobs) | |
| `watchremote` | O(remote projects × packages watched) | collapsed to project granularity above 3 packages |

Per-check costs (`checkpkgs` for one prp): pool creation O(Σ B over the search path)
*unless* cached as a solv string (libsolv load is fast but memory-heavy; a long search
path of large repos is the common "worst prp"); `preparehashes` O(B); expansion
O(K_prp × E × solver work), done for **every** package on **every** check regardless of
whether anything changed (there is no per-package "inputs unchanged" short-cut before
expansion — the `lastcheck` cache only avoids the meta regeneration); `depsort2`
O(K_prp + edges); the cheap hdrmd5 check O(E) per package; the expensive meta
regeneration O(Σ meta lines of local deps) with file reads for metas not in the per-prpa
`meta` cache. `prpchecktimes` and the `worst` list in `schedulerinfo` quantify this in
production; the `setdelayed` back-off (2.5) is the throttle. `writeschedulerinfo`'s
`avg`/`variance` are the operational KPIs.

Event ingestion: O(1) per `built` event for the job status read, but `update_dst_full`
is O(files in package) plus solv update; without `dstcache` every job rewrites
`:full.solv` (O(B)) and `:bininfo.merge`.

Startup: one `getprojpack` for the whole instance (hundreds of MB of Storable on large
instances — hence `sched_startupmode` 1/2), `setup_projects` O(P + R × D), and a full
first series over all R prps.

### 7.5 Known race/consistency hazards in the current design

* A job result and the scheduler's view of sources can diverge: `jobfinished` validates
  `status.job == job` and `verifymd5` is in the meta, so a stale result is dropped
  rather than integrated. Upload/import events are delayed while projpack fetches are in
  flight for the same reason (2.3).
* `prpnotready` is memory-only (except warm-restart dumps); after a crash dependents
  may build against a partially rebuilt base repo once.
* `:full` consistency depends on `:full.useforbuild` + `.bininfo` ids matching the
  directory; mismatches are self-healed by `fctx_rebuild_full`/`rebuild_gbininfo`, which
  are O(B) stats.
* Other archs' files are read without any lock (`read_packstatus`, `:logfiles.*`,
  gbininfo); the only guarantee is per-file atomicity via rename.
* `:packstatus.finished` is an unsynchronised overlay; API consumers may briefly see a
  package as `finished` with stale details.
* The scheduler assumes nobody else modifies `:full`, `:meta`, package dirs of its arch
  except through job dirs and the documented events (`useforbuild`, `scanrepo`,
  `scanprjbinaries`, `wipe`).

---

## 8. Implications for a distributed redesign

### 8.1 What is naturally per-prpa (shardable)

Almost all durable state is already keyed by prpa (section 1.5) and every check operates
on one prpa with read-only access to others. A "prpa shard" would own:

* the package status table (`:packstatus` + `packerror` + the `finished` overlay),
* the build-input record per package (`:meta` + `.meta.success`), the rebuild-decision
  cache (`:lastcheck`), `:depends`, `:genbuildreqs`,
* the binary pool: `:full` membership (`name → file, meta id`), `:full.useforbuild`,
  the solv index, the metacache,
* per-package result metadata (`.bininfo`, `history`, `.stats`, `.failhistory`, `status`,
  `reason`) and the global `:bininfo`,
* `:relsync` (own counters), `:repostate` (blocked sources), `:schedulerstate`
  (+`buildid`), `:repodone`, `:preinstallimages`, DoD metadata and `doddata.needed`,
* the job slots for packages of the prpa (today `jobs/<arch>/<prp>::...`),
* the per-prpa scheduling fields that today live in `gctx`: `prpfinished`, `prpnotready`,
  `repounchanged`, `prpcheckuseforbuild`, `prpunfinished`, `prpchecktimes`, back-off
  times, `lastcheck`, the solv/meta caches.

The check pipeline (`setup → preparepool → expandandsort → checkpkgs → publish`) is a pure
function of (project metadata of the prp and its path, the pools of the path prpas, the
blocked-source sets of the path prpas, the prpa's own result state, the set of in-flight
jobs). It can run anywhere that can fetch those inputs.

### 8.2 What needs global or cross-shard coordination

1. **Project/package metadata** (`projpacks`): today fully replicated into every arch
   scheduler. It is naturally a shared, versioned read model (per project revision)
   fed by the source server; shards subscribe to the projects on their paths
   (`prpdeps` + source links + project links) exactly the way remote watches do today
   (`setup_watches` is a ready-made "interest set" computation).
2. **The prp dependency graph** (`prpdeps`/`rprpdeps`, path expansion with the
   last-element recursion, aggregate/kiwi/channel extra deps, `alllocked`, related prps)
   and the topological order used for the low series: a global, cheaply recomputable
   derived structure (O(R × D)); compute it in one place and distribute, or compute per
   shard for its own prps plus reverse edges (needs all projects' repo definitions, which
   are small compared with package data).
3. **Blocked propagation**: `prpnotready` of a path prpa must be visible to dependents
   before they decide; today it is a file (`:repostate`) + memory. As a shard-owned,
   versioned "repo state" record `{prpa, blocked: set<srcname>, fullversion, bininfoversion}`
   with change notifications it maps directly onto the existing `repository`/`repoinfo`
   events.
4. **Binary pool exchange**: dependents load `:full.solv` of every path prpa; remote mode
   already does this over HTTP (`view=solvstate` + `repositorystate`) with a local cache
   keyed by content. Treat the solv blob as a content-addressed artifact published by the
   owning shard (version = today's `repostateid`-like id), and the pool-build step as a
   fetch of N such blobs.
5. **Cross-arch coupling** (6.2): kiwi products/productcompose/patchinfo read other archs'
   packstatus and bininfo; exportfilter writes jobs into other archs; relsync merges
   counters across archs. All three are small messages (status tables, file lists with
   ids, counter maxima) — they become RPC/stream reads of sibling shards plus a
   "transfer binaries" operation instead of hardlinks into another arch's job dir.
6. **Job lifecycle** with dispatcher/repserver/signer/publisher: the `jobs/<arch>` directory
   protocol (create → `:status` lock → finished event → purge) is a queue with ownership
   hand-off; it becomes a job table/queue service. `needed`, `readytime`, constraints,
   preinstallimage lists, `dispatchdetails` are its attributes.
7. **Publishing**: `:repo` staging + `.finishedlock` + `publish` event become "publish
   snapshot N of prpa" messages; deltas are just more jobs.
8. **Global counters**: `bcnt` per (tag, versrel) across archs (`relsync.max`) is the one
   true cross-shard atomic counter; a small KV with max-merge suffices (today's
   `updaterelsyncmax` is exactly a max-merge under a lock).
9. **Remote OBS instances / partitions**: keep as today (poll `lastevents`, fetch solv
   state), they are already the "distributed" version of the above.

### 8.3 What is naturally event-sourced

* Source side: `package`/`project` events carry (project, package) and the shard fetches
  the new revision — i.e. the source server is the log, projpacks is a materialised view.
* Result side: `built`(job, result code, files) is the only event that changes a prpa's
  result state; everything else (`:packstatus`, `:full`, bininfo, repostate, publish
  state) is derived by re-running the check. A redesign can make "job finished" and
  "source revision changed" the two primary event streams per prpa and treat every
  check as a deterministic reducer over (last decision cache + inputs).
* Operational events (`rebuild`, `wipe`, `useforbuild`, `suspend/resume`, `scanrepo`,
  `admincheck`, `force_publish`) are commands on a prpa and map to API calls on its shard.
* Derived cross-shard notifications (`repository`/`repoinfo`/`unblocked`/`relsync`/`publish`)
  are exactly the change-feed of the per-prpa state record.

### 8.4 Expensive computations and how to cache them

| computation | today | cacheable key |
|---|---|---|
| prjconf merge + `read_config` | per check (`getconfig`) | hash of the ordered list of (project, config revision) along the path |
| pool creation | per check (solv strings cached 24 h) | set of (prpa, fullversion[, modules]) — a pool is a pure function of it; keep a warm pool per shard per distinct path signature |
| `preparehashes` (dep2pkg/dep2src/depislocal/subpacks) | per check | same key as the pool |
| per-package expansion (`get_deps`) | every package, every check | (config hash, pool key, sorted deps + genbuildreqs) → result; most packages have identical inputs between checks; today only the *meta* step is memoised (`lastcheck`), not the expansion |
| cheap rebuild check | per package | already O(E): `md5(metamd5 + pkgid of each edep)`; keep |
| meta regeneration | only when the cheap check fails | `.meta` of local deps keyed by their ids (today `repodatas{prpa}{meta}`, dropped when the prp finishes); in a store, metas are immutable blobs addressed by (prpa, pkg, buildid) |
| `depsort2`/SCC | per check | key: the `edeps` map; unchanged if no expansion result changed |
| `kiwiproductcheck` / aggregate scans | per check, walks gbininfo of all path prpas × archs | key: (bininfoversion of each prpa/arch, product dep list) |
| `:full` integration / `fctx_rebuild_full` | per job; rebuild is O(B) stats | with the pool as a versioned map `name → (file id, meta id)` the rebuild becomes a map diff, no stats |
| path expansion / prp sort | per project change (incremental) | per repo-definition revision |

The single most valuable change is making expansion incremental: record per package the
tuple (config hash, pool key, deps hash) → (edeps, experror) and only re-expand when the
tuple changes; the pool key changes whenever any path prpa publishes a new solv, which is
frequent for big base repos — so the second lever is to version the pool at the
*solvable* level (a pool key that is the set of solvable ids actually chosen by this
package's last expansion, checked against the new pool by `pkg2pkgid` lookups, which is
what the cheap meta check already does).

### 8.5 Mapping the state machine onto a persistent store

Suggested record model (one logical database per prpa shard; all records versioned):

```
PrpaState        {prpa, code (scheduling|building|blocked|finished|broken|disabled|excluded|published),
                  details, buildid, oldbuildid, repostateid, dirty, lastchecktime, checkduration,
                  blocked: set<srcname>          -- replaces :repostate / prpnotready
                  fullversion, bininfoversion}   -- replaces repounchanged + stat ids
PackageState     {prpa, packid, code, error, job?, dispatchdetails?,  -- :packstatus + overlay
                  lastinputs: {srcmd5, metamd5, hdrmetamd5, metaversion},  -- :lastcheck
                  depends: {edeps[], experror, pkg2src, cyclevel}}        -- :depends
BuildRecord      {prpa, packid, buildid, srcmd5/verifymd5, versrel, bcnt, result (succeeded|failed|unchanged|genbuildreqs),
                  meta blob id, files[] {name, id/hash, nevra, hdrmd5, leadsigmd5, source, flags},
                  logfile ids, statistics, reason, readytime/starttime/endtime, workerid}  -- package dir + jobhistory + history
Pool             {prpa, version, entries: name -> {buildid, filename, meta blob id, imported_from?},
                  useforbuild: set<packid>, solv blob id}                       -- :full + :full.solv + metacache
GBininfo         derived view over BuildRecord (latest per packid)             -- :bininfo
Job              {job, prpa, packid, srcmd5, buildinfo blob, state (scheduled|building|finished|signing|deleting),
                  worker, times, result}                                       -- jobs/<arch>
RelsyncCounters  {prp, "tag/versrel" -> maxbcnt}  (cross-arch, max-merge)      -- :relsync.max
PublishSnapshot  {prpa, repodonestate, entries name -> {buildid, filename}, conflicts, version} -- :repo/:repoinfo/:repodone
DoDState         {prpa, cookie, needed resources, parsed metadata blob}         -- :full/doddata*
Suspension       {projid, jobs[] with time}                                    -- projsuspended
ScheduleQueue    {prpa, priority (high|med|low|next), level (1|2), dirty, notbefore_med, notbefore_high}  -- changed_*/lookat_*/next*
```

Immutable blobs (binaries, metas, logfiles, solv files) go to a content-addressed object
store; the `mtime/size/inode` ids become content hashes or build ids. The per-package
`PackageState.code` transitions (3.6) are computed by the checker and written in one
transaction together with `PrpaState` and the created/killed `Job` rows, which gives the
atomicity that today is approximated by writing `:packstatus` last.

### 8.6 Service decomposition sketch

* **projpack-view service**: materialises source-server project/package metadata per
  revision; serves "project P at revision r" and change feeds; computes the global prp
  graph and the interest sets (replaces `ProjPacks.pm` + `Remote::setup_watches`).
* **prpa-scheduler workers** (stateless, N per arch): take a prpa from the queue service
  (respecting the priority/back-off/topological-series rules of `Lookat.pm`), run the
  check pipeline against the store, emit jobs and state updates. One prpa at a time per
  prpa (lease), but many prpas in parallel — the current single-thread limit disappears.
  The `BSSolv` expander becomes a Go/CGO (libsolv) component; the perl `Build::Expand`
  semantics (direct deps, prefer/ignore/conflict, rich deps, `--directdepsend--`) and the
  `gen_meta`/`add_meta`/`diffdepth` algorithms (`BSBuild.pm`) must be ported exactly, since
  the meta string format is what decides rebuilds.
* **pool service**: owns `Pool` records and solv blobs; applies `BuildRecord`
  integrations (the `update_dst_full`/`fctx_*` logic: useforbuild, exportfilter,
  precedence by `orderpackids`/`volatile_cmp`, imports) and publishes new versions.
* **job service**: the queue between scheduler, dispatcher, workers, signer; owns
  `Job`; emits `built` events.
* **publish service**: consumes `PublishSnapshot` requests (`PublishRepo` logic + deltas).
* **dod service**: `bs_dodup` as is, writing `DoDState` and emitting `scanrepo`.
* **relsync counter service**: max-merge KV.
* **notification bus**: `repository`/`repoinfo`/`unblocked`/`relsync`/`publish` become
  topics keyed by prpa; the API's redis fast path becomes a subscription.

### 8.7 Semantics that must be preserved (checklist for the port)

1. Search-path expansion rule (only the last path element recurses) and reverse-order
   config concatenation with `FromScratch:`/`_is_this_project` macros.
2. Package visibility rules: `linkedbuild` modes, `excludebuild`/`onlybuild`, lock/build
   flag scoring (`BSUtil::enabled`), `_product` excluded, `maintenance_release` build-off
   default, `missingpackages` carry-over.
3. The exact meta algorithm: line format, `add_meta` re-rooting, `gen_meta` pruning rules
   per `genmetaalgo`, non-local deps contribute one line, `forcebinaryidmeta`,
   `modularity_meta`, host deps `<hostarch>:` prefix, the fake-meta 4 h retry, `unchanged`
   results not touching the pool.
4. Rebuild modes `transitive|direct|local`, block modes `all|local|never`, the
   `notready` value-2 distinction, `useforbuild` gating of `notready`.
5. Cycle handling: SCC-based three-pass walk, `cyclevel` via `diffdepth`, pass-3 "all or
   nothing" notready marking, self-cycle pruning in `gen_meta`.
6. Job naming with `srcmd5`, superseding/killing of older jobs, `needed` counts, `bcnt`
   computation with relsync maxima, `release` template substitution.
7. `:full` precedence (`orderpackids`, maintenance incident ordering, `_volatile` last,
   imports last, `volatile_cmp`), exportfilter semantics including `.` (keep) and
   multi-target export, `nouseforbuild` for channels/patchinfos/reproduciblecheck.
8. Publish state hashing (`repodonestate`) and the "don't publish while unfinished unless
   early-publish type" rule; `nofailedpackages`, `keepobsolete`, `archsync`,
   `singleexport`, `noorphanedsrcrpms`, `excludepublish/onlypublish`.
9. Cross-arch product/patchinfo protocol (build arch selection, `local` arch handling,
   waiting markers → unblocked events) and aggregate semantics (first origin wins,
   `_repository` sources, binary/binaryarch filters, nosources/noupdateinfo, resign).
10. Back-off and starvation rules (`setdelayed` factors, `notlow>10`, `notmed>2`,
    topological low series, `admincheck` jump-the-queue) — these shape perceived latency
    on large instances and should be reproduced before being redesigned.
11. Interconnect compatibility: `lastevents` filters/sequence numbers, `view=solvstate`
    cpio members, `view=gbininfocode`/`binaryversionscode`, `remotemap` with partitions.
