# OBS Backend: Publishing, Signing and Distribution

Engineering analysis of the publisher / signer / registry subsystem of the Open Build Service
backend (`src/backend`), written to support a ground-up re-architecture as Go microservices.
Everything below was derived from reading the Perl sources in this checkout; line numbers refer
to the files as they are today. Where I describe the *external* `sign`/`signd` program (which is
not in this repository) I say so explicitly and separate "observed from the callers in this tree"
from "known about the obs-sign package".

All paths are relative to `/home/jeffm/src/open-build-service/src/backend` unless stated otherwise.

---

## 0. Scope and file inventory

| File | Lines | Role |
|---|---|---|
| `bs_publish` | 4061 | The publisher daemon: consumes `publish` events, assembles `repos/<prj>/<repo>/`, runs per-repotype metadata generators, signs, pushes containers, syncs to mirrors, updates the published DB, emits notifications |
| `BSPublisher/Container.pm` | 1149 | Container publishing orchestration: tag mapping, per-registry upload via `bs_regpush` (remote) or `BSPublisher::Registry` (local), cosign/notary glue, obsolete tag deletion |
| `BSPublisher/Registry.pm` | 988 | The OBS-local OCI registry writer (`$bsdir/registry/<repo>/...`): blobs, manifests, tags, cosign signatures/attestations, TUF (notary) metadata, atomic signatures |
| `BSPublisher/Containerinfo.pm` | 123 | Reconstruct a container tar from blobs; build per-image package lists for manifestinfos |
| `BSPublisher/Helm.pm` | 116 | helminfo parsing, `index.yaml` generation |
| `BSPublisher/Diffnotify.pm` | 268 | "what changed since last publish" JSON notification to an external HTTP endpoint (Airflow-style `dag_run_id`) |
| `BSPublisher/Blobstore.pm` | 38 | refcount-style GC of `$bsdir/blobs/sha256/xx/yyyy` |
| `BSPublisher/Util.pm` | 81 | `qsystem` (fork/exec with stdin/stdout redirection) |
| `bs_signer` | 1315 | The signer daemon: signs job results (rpm/deb/iso/apk/appx/AppImage/helm/SLSA) before the scheduler picks them up |
| `BSSigAuth.pm` | 105 | HTTP "Signature" authentication scheme using ssh signatures (used for outgoing RPCs, not for signd) |
| `BSSSHSign.pm` | 91 | Build an `SSHSIG` signature using the `sign` tool's raw RSA mode |
| `BSPGP.pm` | 310 | Minimal OpenPGP packet parser/encoder (pubkey algo/times/fingerprint, signature data, one-pass signed message) |
| `BSX509.pm`, `BSASN1.pm` | 402/357 | Minimal DER/X.509 pack/unpack (certs for TUF root, SubjectPublicKeyInfo conversion) |
| `BSConSign.pm` | 354 | Container signature payloads: atomic (OpenShift), cosign simplesigning, DSSE/in-toto attestations, sigstore bundle v0.3 |
| `BSTUF.pm` | 154 | The Update Framework metadata (root/targets/snapshot/timestamp) and self-signed root cert creation |
| `BSRekor.pm` | 232 | Sigstore Rekor transparency log client (rekord, hashedrekord v1/v2, intoto, dsse) |
| `bs_notar` | 438 | Docker Notary (TUF) uploader |
| `bs_regpush` | 1472 | Docker Registry v2 client: push images/manifest lists, cosign signatures & attestations, list/delete tags |
| `BSRegistryServer.pm` | 83 | Reply/err helpers for the registry-v2 routes served by `bs_repserver` |
| `bs_mkapkrepo`, `bs_mkarchrepo` | 98/123 | Pure-Perl APKINDEX and Arch `.db.tar.gz`/`.files.tar.gz` generators |
| `bs_productconvert`, `BSProductXML.pm`, `BSKiwiXML.pm` | 1653/439/462 | Product definition (`_product`) → kiwi/spec converter (build-time; not the publisher) |
| `bs_sourcepublish` | 258 | Optional daemon that rsyncs the *sources* of published binaries to a mirror |
| `bs_deltastore` | 105 | Source-side delta cpio generator for the source repo (not related to published deltarpms) |
| `BSUrlmapper.pm` | 152 | prp ↔ published directory ↔ download URL mapping (`publishredirect`, `prp_ext_map`, `repodownload`) |
| `BSSrcServer/SQLite.pm` | 728 | The sqlite-backed published DB (`repoinfo`, `binary`, `pattern` tables) and the source `linkinfo`/`scmsync` DB |
| `BSSrcServer/Signkey.pm` | ~250 | Project key creation/extension/cert creation via `sign -g/-x/-C` |
| `BSSched/PublishRepo.pm` | ~620 | Scheduler side: fills `<arch>/:repo` + `:repoinfo`, builds deltarpm jobs, sends publish events |
| `BSConfig.pm.template` | 377 | Config keys |

Relevant on-disk roots (`$BSConfig::bsdir`, default `/srv/obs`):

```
/srv/obs/build/<prj>/<repo>/<arch>/:repo/        scheduler output for publishing (hardlinks)
/srv/obs/build/<prj>/<repo>/<arch>/:repoinfo     Storable: {binaryorigins=>{file=>pkg}, conflicts=>...}
/srv/obs/build/<prj>/<repo>/<arch>/:repodone     scheduler "already published this state" marker
/srv/obs/build/<prj>/<repo>/:repoinfo            publisher's own state (Storable, see §1.3.9)
/srv/obs/build/<prj>/<repo>/.finishedlock        flock between scheduler(s) and publisher
/srv/obs/repos/<prj with : → :/>/<repo>/         THE published tree (served by apache as DocumentRoot)
/srv/obs/repos_sync/                             trigger files for stage servers
/srv/obs/events/publish/                         publisher event queue
/srv/obs/events/signer/                          signer event queue
/srv/obs/db/published.sync                       fallback spool if the published-DB RPC fails
/srv/obs/db/sqlite/published                     sqlite published DB (lives on the *source server*)
/srv/obs/registry/<regrepo>/                     OBS-local container registry
/srv/obs/blobs/sha256/xx/yyyy                    shared container blob store
/srv/obs/upload/publisher.<pid>*                 temp files (sign keys, pubkeys, digest files, blobdir)
/srv/obs/run/bs_publish{,.rescan,.exit,.restart} run lock + control files
```

---

## 1. The publisher loop

### 1.1 Event model: `BSRunner` / `BSStdRunner`

`bs_publish` is a `BSStdRunner` (`BSStdRunner.pm:200-260`). The runner:

* drops privileges to `$BSConfig::bsuser` (`BSStdRunner.pm:221`), takes a run lock
  `$rundir/bs_publish` (`:233`), opens a ping FIFO `$eventdir/publish/.ping` (`:236`).
* on start renames every `*::inprogress` event back to its plain name (`:254-256`) so crashed
  children are retried.
* loops in `BSRunner::run` (`BSRunner.pm:48-146`): `lsevents` → for each event `getevent` →
  optionally fork → `dispatch`. `filechecks` (`$rundir/bs_publish.exit|.restart|.rescan`) are
  polled between events; `.rescan` just unlinks itself and causes a re-list (`bs_publish:4035-4038`).
* `maxchild` (`$BSConfig::publish_maxchild`) and `maxchild_flavor`/`publish_getflavor`
  (`bs_publish:70-73, 4046-4059`) give per-"flavor" concurrency caps; the flavor function is
  site-provided.
* Waiting is `BSUtil::waitping` on the FIFO; writers call `BSUtil::ping(".ping")` after dropping
  an event file. Nothing else: no inotify, no DB queue.

**Event files.** The event directory is `/srv/obs/events/publish/`. Event file name is
`publish::<project>::<repository>` (`BSSched/EventSource/Directory.pm:151-161`, md5-hashed if
>200 chars, `:90`). Content is `$BSXML::event` XML (`BSXML.pm`, the `event` definition):

```xml
<event type="publish">
  <project>openSUSE:Factory</project>
  <repository>standard</repository>
  <time>1718000000</time>
</event>
```

`sendevent` keeps the original `time` if the file already exists (`Directory.pm:93-96`), so
repeated pings coalesce into one event whose `time` is the first request; `bs_publish` prints
`publish statistics: <prp>/- <code> <createtime>-<starttime>-<endtime> <flavor>` from it
(`bs_publish:3850-3859`).

Other event producers writing into `events/publish/`:
* `bs_repserver` `forwardevent(..., 'publish', ...)` (`bs_repserver:2924-2955`) for API-triggered
  `unpublish` / `wipepublishedlocked` (`:3223-3290`).
* `bs_publish` itself writes `publishtime::<prj>::<repo>` events (`:3766-3779`) — these are an
  in-band way for forked children to report back `(lasttime, duration)` or a retry due time to the
  parent (`:3948-3960`), because forked children cannot update the parent's `%publish_retry` /
  `%publish_lasttime` hashes.
* `configuration` events (`:3962-3967`) reload `BSConfiguration`.

**Dispatch table** (`bs_publish:4040-4044`):

```
'configuration'                          => configurationevent
'publish $project $repository'           => publishevent
'publishtime $project $repository $job'  => publishtimeevent
```

**Ordering** (`lsevents`, `:3969-3979`): `publishtime*` first, then `_*` (high priority), then
the rest in `sort` order of filename.

**Retry / back-off** (`:3796-3832`, `:3861-3946`):
* `getevent` returns "not due" if `%publish_retry{$evname} > now` (`:3984-3987`).
* An exception from `publish()` → `clear_repoinfo_state` (drops `state` so next run republishes,
  marks `code=failed`, stores `details`), then exponential back-off
  `1<<min(retrycount,5)` minutes (1,2,4,8,16,32) unless the error message is prefixed:
  * `CRITICAL_NORETRY:` / `NORETRY:` → no retry; event file is rewritten with `<details>` and
    chmod 0444 (`set_publish_error`, `:3821-3832`) and `getevent` skips publish events that have
    `details` (`:3999`) — i.e. the event sits in the queue as a visible "stuck" marker until an
    admin removes it or a new event overwrites it.
  * `CRITICAL:` → logged to `$logdir/publisher.crit.log` (`BSStdRunner::critlogger`), normal retry.
  * `RETRY_AFTER<n>:` → minimum back-off n (used by `bs_regpush` 429 handling, see §3.4).
* `publish_penalty_multiplier` (`:3878-3887`): if the last publish of this prp took >300 s, the next
  one is refused until `lasttime + duration*multiplier` (clamped to 1 h).
* A publish that returns `'delayed'` (archsync) leaves the event in place and returns success from
  the dispatcher's point of view (`:3918-3921`); a `recheck` event is sent to the lagging
  scheduler(s) (`writerecheckevent`, `:3781-3794`).

**In-progress marking.** With `inprogress => 1` the dispatcher renames `ev` → `ev::inprogress`
before running (`BSStdRunner.pm:79-83`) and unlinks it on success / renames it back on failure
(`:95-104`). If a child dies from a signal, `sigkill` renames it back (`:176-186`). New pings for
the same prp during processing create a fresh `publish::…` file next to the `::inprogress` one;
`lsevents` hides the plain file while an `::inprogress` twin exists (`BSStdRunner.pm:35-40`), so
it is processed on the next loop.

**Single-event mode.** `bs_publish --event <file>` runs one event in the foreground and exits
(`:4017-4033`); handy for a Go re-implementation's conformance tests.

### 1.2 What the scheduler leaves behind (`BSSched/PublishRepo.pm`)

The scheduler for architecture A decides a prp is "finished" (or kiwi/productcompose
"earlypublish", `bs_sched:1130-1142`) and calls `BSSched::Checker::publish`
(`BSSched/Checker.pm:1530-1640`), which:

1. Computes a `repodone` state hash from `patternmd5`, the list of publish-enabled packages, the
   unfinished set and the `excludepublish/onlypublish` flags (`:1577-1595`). If
   `<arch>/:repodone` holds the same hash nothing is done. "disabled:<hash>" means all packages
   have publishing disabled; `bs_repserver` maps that to the `unpublished` repo state
   (`bs_repserver:1982`).
2. Honours `publishflags:nofailedpackages` (refuse to publish if any package is failed/broken),
   `publishflags:keepobsolete` (keep binaries of packages that disappeared).
3. Calls `BSSched::PublishRepo::prpfinished` (`PublishRepo.pm:113-477`):
   * takes `flock(LOCK_EX)` on `$reporoot/$prp/.finishedlock` with a 300 s timeout when it has
     packages (`:128-139`), returns `delayed:lock timeout` otherwise. This is *the* lock shared
     with the publisher (§5).
   * If `$packs` is undef (arch no longer builds this repo) it wipes `<arch>/:repo` and
     `:repoinfo` and sends a publish event (`:141-158`).
   * `makedeltas` (`:481-640`) scans the already *published* tree of every prp in the search path
     (`BSUrlmapper::get_extrep($aprp)/<arch>/`) for older versions of each rpm and, for the newest
     candidate, creates a `_deltas` build job (deltas built on workers via
     `BSSched::BuildJob::DeltaRpm`) unless the delta already exists in `<arch>/_deltas/<md5id>`
     + `.dseq`. Returns `'building'` while delta jobs are outstanding, which keeps the prp in
     state `building` (`Checker.pm:1627`). So deltarpm generation blocks publishing.
   * Links every publishable file of every package dir `<arch>/<pkg>/` into `<arch>/:repo/`
     (hardlink or `linktree` for directories like product media) (`:365-437`). Filters:
     `publishfilter` regexes from project config, `publishflags:excludepublish:<pkg>` /
     `onlypublish:<pkg>`, `publishflags:singleexport` (first rpm name/arch wins),
     `noorphanedsrcrpms`, `.nosourceaccess` (drop src/debug), ignore list
     (`history logfile meta status reason rpmlint.log _buildenv _slsa_provenance.json …`, `:60-63`).
   * Naming inside `:repo`: files keep their basename, except Debian results, `updateinfo.xml`
     and `_modulemd.yaml`, which get `<packid>::` prefixed (`:321`) because they collide across
     packages. Deltas are `<rbin>::<deltaname>.drpm` and `::….dseq` (`publishdelta`, `:570-600`;
     name built by `mkdeltaname` → `foo-1.0_1.1-1_2.x86_64.drpm`). Provenance:
     `<bin-without-suffix>.slsa_provenance.json` (`:329-340, 606-620`).
   * Conflicts (two packages producing the same filename): first wins, recorded in
     `conflicts{file}=[pkgs]` (`:316-320`).
   * Writes `<arch>/:repoinfo` = `{'binaryorigins' => {file => packid}, 'conflicts' => {...}}`
     (`:447-449`).
   * Per-package `.checksums` (from the signer's `sign -S`, §2.4) are merged into
     `<arch>/:repo/.newchecksums` (`:451-461`) for `createrepo --checksum` reuse.
   * `publishflags:archsync`: writes `:repo/.archsync` = `{lastcheck, lastchange}` (`:464-471`)
     so the publisher can wait until all archs are in sync (§1.3.3).
   * Finally `sendpublishevent` if anything changed (`:475`).

Example `<arch>/:repoinfo` (Storable, shown as Perl):

```perl
{
  'binaryorigins' => {
    'foo-1.2-3.4.x86_64.rpm'                   => 'foo',
    'foo-1.2-3.4.src.rpm'                      => 'foo',
    'foo-debuginfo-1.2-3.4.x86_64.rpm'         => 'foo',
    'foo-1.2-3.4.x86_64.rpm::foo-1.1_1.2-2.1_3.4.x86_64.drpm' => 'foo',
    'foo-1.2-3.4.x86_64.rpm::foo-1.1_1.2-2.1_3.4.x86_64.dseq' => 'foo',
    'foo-1.2-3.4.x86_64.slsa_provenance.json'  => 'foo',
    'patchinfo.1234::updateinfo.xml'           => 'patchinfo.1234',
    'bar::bar_2.0-1_amd64.deb'                 => 'bar',          # Debian results are always '<packid>::' prefixed (PublishRepo.pm:321)
    'mycontainer-1.0-Build5.docker.tar'        => 'mycontainer',
    'mycontainer-1.0-Build5.containerinfo'     => 'mycontainer',
    '_blob.sha256:ab12…'                       => 'mycontainer',  # container layer blobs (hardlinks into $bsdir/blobs)
    'kiwi-product-Media1'                      => '_product:foo-cd-cd-x86_64',   # a directory (linktree)
  },
  'conflicts' => { 'README' => ['pkgA', 'pkgB'] },
}
```

### 1.3 `publish($projid, $repoid, $dbgsplit, $dbgpacktrack, $dbgsources)` walkthrough (`bs_publish:2260-3726`)

#### 1.3.1 Gathering project metadata
* `GET $srcserver/getprojpack?withrepos&expandedrepos&withsrcmd5&nopackages&project=&repository=`
  (`:2270`). If the project or repository is gone → `deleterepo` (§1.11). From the result it
  takes the expanded search path `repo->{path}`, `repo->{arch}`, `repo->{base}`, `proj->{kind}`,
  `proj->{title}`, `proj->{patternmd5}`.
* `GET $srcserver/getconfig?project=&repository=` → `Build::read_config('noarch', …)` (`:2290-2291`).
  `repotype:` and `patterntype:` lines of the project config drive everything. If no repotype
  is set it is derived from `binarytype` (rpm→rpm-md, deb→debian, arch→arch, default rpm-md)
  (`:2293-2301`).
* `repotype` tokens are `name[:opt[:opt…]]` → `%repotype{name}=[opts]` (`:2308-2315`).
  Pseudo-repotypes carry config instead of generators: `splitdebug:<suffix>[:mainupdateinfo]`,
  `archorder:<arch>:<arch>…`, `packagesubdir:<dir>`, `repotag:<tag>…`, `sleimage`,
  `sleimage-media`, `slepool:<name|nobuildid>`, `resarchhack`, `rpm-md-legacy`.
* Sign flavor: `buildflags:signflavor` from config, validated against `$BSConfig::sign_flavor`
  (`:2302-2306`).
* `packtrack` (binary release tracking) on if `$BSConfig::packtrack` matches the prp or project
  `kind eq 'maintenance_release'` (`:2320-2324`).
* `sourcepublish` on if `$BSConfig::sourcepublish_sync` and the optional
  `sourcepublish_filter` regex list matches (`:2326-2334`).
* `publish_diffnotify`: list of `(regex => {uri, statedir, user, password, …})` pairs
  (`:2336-2347`).
* `packagesubdir` → all binaries go under `<extrep>/<subdir>/` (`:2349-2354`).
* `BSUrlmapper::get_extrep_stageservers($prp)` → `($extrep, $stageservers, $triggerstring)`
  (`:2356`). Default `$extrep = $bsdir/repos/<prj with ':'→':/'>/<repo>` (`BSUrlmapper.pm:66-85`);
  `$BSConfig::publishredirect{prp}` may be a string or
  `[dir, localsubpath, downloadurl, stageserver(s), triggerstring]` or a code ref.
  `$dbgsplit` (e.g. `_debug`) is appended to both `$extrep` and `$prp_ext`.

#### 1.3.2 Locking
`open(F,'>',"$reporoot/$prp/.finishedlock"); flock(F, LOCK_EX)` (`:2363-2370`) — blocking. The
lock is held while reading all `<arch>/:repo` dirs and while hardlinking into `$extrep`; released
at `:3027` *before* container uploads, repo metadata generation and rsync. So the scheduler may
rewrite `:repo` while `createrepo` runs; that is fine because the publisher works on its own
hardlinks in `$extrep` and has already captured `bins_id` (mtime/size/inode triples).

#### 1.3.3 archsync wait
If `publishflags:archsync` and more than one arch (`:2374-2400`): read each
`<arch>/:repo/.archsync`, and if any arch's `lastcheck` is older than the newest other arch's
`lastchange` → touch `<arch>/:repo/.archsync.new` (a signal the scheduler reads at
`PublishRepo.pm:464`), unlink `<arch>/:repodone`, send a `recheck` event to that scheduler, and
return `'delayed'`.

#### 1.3.4 Architecture ordering and "last one wins"
Archs are processed in reverse lexical order by default (so `x86_64` beats `i586` beats `aarch64`
for noarch files) or per `repotype archorder:` (`:2425-2434`). When the same published path `$p`
is produced by two archs, the *later* arch wins unless no archorder is configured, in which case
the file already present in `$extrep` with the same id wins (`:2693-2703`, "keep old file"
FIXME). Drpms taken for a replaced rpm are dropped again (`:2704-2712`).

#### 1.3.5 Disappearing repos and empty repos
If no `<arch>/:repo` exists at all and `publishflags:createempty` is not set → `deleterepo`
(`:2436-2448`). `createempty` keeps an empty tree with metadata.

#### 1.3.6 Scanning `:repo` and mapping to published paths (`:2469-2765`)
For every arch, for every `$rbin` in `<arch>/:repo` (sorted), the publisher decides the
destination path `$p` inside `$extrep`. The complete classification (in code order) is:

| `:repo` entry pattern | Action / published path |
|---|---|
| `.checksums`, `.newchecksums`, `.archsync` (+ `.new`) | skipped (checksums merged into `@chksumfiles`, `:2477-2496`) |
| `*:updateinfo.xml` | parsed, `update` entries appended to `@updateinfos` (type `_internal` dropped); md5 into `$updateinfos_state` |
| `*:_modulemd.yaml` (<1 MiB) | concatenated into `$modulemds`; not published as a file |
| `*[-.]appdata.xml` | merged by `merge_package_appdata` into one AppStream document; md5 into state |
| `<rpm>::<drpm>` | only if the rpm itself was taken; added to `%deltas{rpmsrc}` |
| `_blob.sha256:<hex>` | hardlinked into a temp blobdir (`$uploaddir/publisher.$$.blobs/`), used to reconstruct containers |
| then `$bin =~ s/^.*?:://` strips the package prefix | |
| `name-ver-rel.<arch>.{rpm,drpm}` | `<arch>/<file>`; with `resarchhack`: `SRPMS/` or `RPMS/` |
| `name_ver_<arch>.{deb,udeb}` | `<arch>/<file>` |
| `*.AppImage`, `*.AppImage.zsync`, `*.snap`, `*.exe[.sha256[.asc]]`, `*.appx`, `*.wsl[.sha256[.asc]]` | top-level |
| other `*.rpm`/`*.deb` (legacy names) | `Build::query` for evra → `<arch>/<n-v-r.a.rpm>` |
| `*.pkg.tar.{gz,xz,zst}[.sig]` | `<schedarch>/<file>`, `i586`→`i686` |
| `*.apk` | `<schedarch>/<file>`, `i586`→`x86` |
| other `@BSOBS::binsufs` | `Build::query` arch → `<arch>/<file>` |
| `*.slsa_provenance.json` | skipped here, picked up with its binary |
| `*.iso[.shaN[.asc]]` | `iso/<file>`; `$kiwimedium` recorded; `mapsleimage` renaming if `sleimage*` |
| `*.{spdx,cdx}.json` next to an iso | `iso/<file>` |
| `*ONIE.bin…` | `onie/<file>` |
| `*.raw[.install][.gz…][.sha256…]`, `*.install.tar…` | top-level (kiwi appliance) |
| `*.tgz` with `.helminfo` | top-level; `BSPublisher::Helm::readhelminfo` → `%containers{$p}` (type `helm`) |
| `*.tgz.prov` | top-level if helminfo readable |
| mkosi artefacts `*.{manifest,metainfo.xml,img,efi,vmlinuz,initrd,cpio,verity,roothash[.p7s],usrhash[.p7s]}…`, `SHA*SUMS[.gpg|.asc]`, `mkosi.*` | top-level |
| `*.containerinfo` whose `file` is a `.tar` that does **not** exist (blob-only container) | `[<arch>/]<file>` virtual entry in `%containers`, reconstructed later from blobs |
| `*-Build<N>…[.docker].{tbz,tgz,tar,tar.gz,tar.bz2,tar.xz}[.sha256[.asc]]` (kiwi image) | top-level, or `<arch>/` when multi-arch and a `.containerinfo` exists |
| `*.tar[.gz|bz2|xz][.sha256[.asc]]` with `.containerinfo` (Dockerfile build) | same as above; `-appstream.tar` etc. dropped if not a container |
| `*.tgz`/`*.zip`[.sha256…] | top-level (Go builds) |
| `*.squashfs`, `*.diff.gz…`, `*.dsc…`, `*.orig.tar.*.asc` | top-level |
| `*.{box,json,ova,ovf,phar,qcow2,vdi,vhdfixed,vmx,vmdk,vhdx}[.xz][.shaN[.asc]]` | top-level (VM images) |
| `*.{cdx,spdx}.json` (other) | only if `publishflags:withsbom`; under `repo/` if next to a product dir |
| `*.packages` | only if `publishflags:withreports` (FIXME2.11) |
| `*.report` | collected into `%kiwireport` for packtrack/sourcepublish; published only with `withreports` |
| `*.index` | concatenated into `$kiwiindex` (virt-builder index) |
| `*.flatpak`, `ChangeLog.*` | top-level |
| a directory (product media) | `repo/<dir>`; with `slepool` → `mapslepool` names (`product`, `product_debug`, `product_source`, `product.license`, or `repo/<no-buildid>`) |
| anything else | ignored |

Each taken file records `$bins{$p}=<source path>`, `$bins_id{$p}="mtime/size/inode"`,
`$binaryorigins->{$p}=<packid>`, optional `%kiwimedium{$p}` (which `.report` belongs to the
file) and `%containers{$p}`. Drpms additionally parse the `.dseq` sidecar into `%deltainfos`
(`:2722-2740`, fields `Name Epoch Version Release Arch OldName OldEpoch OldVersion OldRelease
OldArch Seq`). For every binary, `<base>.slsa_provenance.json` is taken/dropped alongside
(`:2742-2764`).

`publishflags:nocontentchange` (`:2713-2720`, `:2769-2793`): if a published file already exists
with a different id, keep the *old* published file and ignore the new one (used for repos where
only additions are allowed); the old `binaryorigins` entry is carried over from the previous
`:repoinfo`.

`splitdebug` (`:2809-2824`): with `repotype splitdebug:_debug`, `publish()` is run twice: the
main pass drops `*-debug(info|source)-*.rpm` (and their provenance), and a recursive call with
`$dbgsplit='_debug'` (`:3528`) publishes only those into `<extrep>_debug` with repoid
`<repo>_debug`, sharing the packtrack/sources hashes.

#### 1.3.7 Container decision (`:2829-2890`)
`BSPublisher::Container::registries_for_prp` returns the registries configured for this prp
(`$BSConfig::publish_containers` list of `regex => [registry names]` evaluated against
`"$projid/$repoid"` then `$projid`; `Container.pm:50-77`). If any registry applies:
* every container's tar is **removed** from the published tree (`.sha256`, `.sha256.asc` too) and
  replaced by `<name>.registry.txt` ("This container can be pulled via: docker pull …"), except
  helm charts which stay (`:2836-2856`, `Container.pm:270-292`).
* the tar is hardlinked into the blobdir as `<arch>:<basename>` for upload.

If no registry applies, blob-only containers are reconstructed into the published tree
(`BSPublisher::Container::reconstruct_container` → `BSTar::writetarfile`, `:2858-2890`),
reusing the existing file when its sha256 matches `containerinfo->{tar_sha256sum}`.

#### 1.3.8 Updating `$extrep` (`:2899-3015`)
Walk the existing published tree (`<extrep>/<arch>/…`, skipping `repodata repocache media.1
descr boxes *.repo Packages* Sources* Release* <registry extras>`) and:
* `- $p` unlink/cleandir anything not in `%bins` (recorded in `@db_deleted` if binary suffix)
* `! $p` if id differs: unlink + `link($bins{$p}, …)` or `BSUtil::linktree` for directories
  (directories are compared with `BSUtil::treeinfo` first)
* `+ $p` for new paths: `mkdir_p` + link
Everything is **hardlinks** from `build/<prp>/<arch>/:repo/<file>` (which are hardlinks to
`build/<prp>/<arch>/<pkg>/<file>`) into `repos/…`. Same filesystem is mandatory. The log lines
`      + path` / `      ! path` / `      - path` are what admins grep for.

`@changed` (all added/changed paths) feeds the `publishedhook`; `@db_changed`/`@db_deleted`
(binaries only) feed the published DB.

#### 1.3.9 State hash and the publisher's `:repoinfo`
`$state = md5(patternmd5 \0 repotype-list \0 title \0 patterntype \0 searchpath \0
updateinfos_state \0 appdatas_state \0 deltainfos_state \0 container_state \0 modulemds_state)`
(`:3041-3054`). `container_state` = `multi//` + sorted list of `<containerid>/<registry>/<arch>/<tag>`
(`Container.pm:132-152`). If `!$changed` and the state equals the old one → `nothing changed`
→ return `'unchanged'` (`:3073-3078`), no metadata regeneration, no rsync.

The publisher's `$reporoot/$prp/:repoinfo` (Storable) is written **before** metadata generation
with `code => 'building'` (`:3255-3277`) and rewritten at the end with `code => 'succeeded'`,
`endtime` (`:3565-3573`); on error `clear_repoinfo_state` sets `code => 'failed'`, `details`.
This is what `/published/<prj>/<repo>?view=status` returns (`bs_repserver:3997-4006`,
`$BSXML::buildstatus`). Example:

```perl
{
  'prpsearchpath' => [ {project=>'openSUSE:Factory',repository=>'standard'}, ... ],
  'binaryorigins' => { 'x86_64/foo-1.2-3.4.x86_64.rpm' => 'foo', 'iso/…-Media1.iso' => '_product:…', ... },
  'title'         => 'openSUSE Factory (standard)',
  'state'         => 'd41d8cd98f00b204e9800998ecf8427e',
  'repotype'      => { 'rpm-md' => ['deltainfo','prestodelta'], 'staticlinks' => [], 'splitdebug' => ['_debug'] },
  'projectkind'   => 'maintenance_release',
  'arch'          => ['x86_64','i586','aarch64'],
  'splitdebug'    => '_debug',
  'subdir'        => '',
  'base'          => { project => '…', repository => '…' },
  'container_repositories' => { 'registry.example.com' => ['opensuse/factory/standard/foo'], 'local' => [...] },
  'publishid'     => 1718000001,       # monotonically increasing: old publishid+1 or starttime
  'code'          => 'succeeded', 'starttime' => 1718000000, 'endtime' => 1718000123,
  'trackercache'  => [ 'x86_64/foo-….rpm/mtime/size/inode', ['x86_64','foo',0,'1.2','3.4','obs://…','1718000000',undef,'<hdrmd5>'], … ],
  'trackercacheversion' => 3,
}
```

The published-DB copy of this (`repoinfo` table, §4) strips `binaryorigins state code
starttime endtime publishid` (`SQLite.pm:prpext2id`).

#### 1.3.10 Signing data, patterns, containers, packtrack
* `getsigndata($projid, $signflavor)` (`:337-388`, §2.2) → `($pubkey, $signargs)`;
  `$signargs` is `['-P', "$uploaddir/publisher.$$"]` (the project's encrypted private key blob
  written to a temp file) plus `-h sha256` for RSA keys.
* Patterns are fetched from the source server (`GET /source/<prj>/_pattern`, each
  `/_pattern/<name>?rev=<srcmd5>`, `:304-335`).
* Containers: `BSPublisher::Container::upload_all_containers` (§3) runs **before** repo metadata
  generation; its result (`{registry => [repositories]}`) is stored in `:repoinfo` so that a later
  delete can clean up registries.
* Packtrack collection (`:3125-3252`): for every published binary, `Build::query(evra buildtime
  disturl)` (cached via `trackercache` keyed by `path/mtime/size/inode`), producing
  `{project, repository, arch, package, name, epoch, version, release, binaryarch, binaryid=hdrmd5,
  disturl, buildtime, cpeid, id}`; containers via `containerinfo2nevra` (`name =
  "container:<name>"`, `binaryid = imageid`); kiwi images via their `.report`
  (`$BSXML::report`, with every contained binary recorded under `"<medium>/<binaryarch>/<name>/<e>/<v>/<r>"`
  with `medium` set); updateinfo attaches `patchinforef`, `updateinfoid`, `updateinfoversion`,
  `supportstatus`. Source publishing derives `"<prj>/<pkgname>/<srcmd5>"` from
  `disturl =~ obs://host/<prj>/<repo>/<srcmd5>-<pkg>`.

#### 1.3.11 Repo generation dispatch (`:3385-3467`)
A `$data` hash is built once and passed to each generator:

```perl
$data = { projid, repoid, xrepoid (repoid.dbgsplit), subdir, pubkey, signargs, repoinfo,
          updateinfos => [...], deltainfos => {...}, appdatas, patterns => [...], dbgsplit,
          packtrack, repotags => [...], chksumfiles => [...], config, patterninfo => {},
          modulemds, sigs => [], publishid, signflavor, time };
```

`repotags` (`:3372-3383`): `repotype repotag:<tag>…` entries (`%XX` unescaped) plus, unless
`-obsrepository` is present, `obsrepository://<obsname>/<prj>/<repo>` and
`obsbuildid:<publishid>`. These become `createrepo --repo <tag>` entries in `repomd.xml`.

Each repotype has a create/delete pair so a removed repotype cleans its artefacts:

```
rpm-md   → createrepo_rpmmd / deleterepo_rpmmd        (rpm-md-legacy = rpm-md with 'legacy' opt)
suse     → createrepo_susetags / deleterepo_susetags   (YaST "susetags": content, media.1, descr/)
hdlist2  → createrepo_hdlist2 / deleterepo_hdlist2     (Mandriva genhdlist2 per arch dir)
debian   → createrepo_debian / deleterepo_debian
arch     → createrepo_arch / deleterepo_arch
apk      → createrepo_apk / deleterepo_apk
vagrant  → createrepo_vagrant / deleterepo_vagrant
staticlinks → createrepo_staticlinks / deleterepo_staticlinks
zyppservice → createrepo_zyppservice / deleterepo_zyppservice
helm     → createrepo_helm                              (no delete!)
checksumsfile → createrepo_checksumsfile                (no delete!)
patterntype ymp     → createpatterns_ymp / deletepatterns_ymp
patterntype rpm-md  → createpatterns_rpmmd / deletepatterns_rpmmd
patterntype comps   → createpatterns_comps / deletepatterns_comps
(index file present) → createrepo_virtbuilder
```

`$BSConfig::publishprogram{$prp}` (`:3357-3370`) replaces the whole generation step by an
external program run in a **forked background** process holding `.pplock`; the parent returns
immediately and `check_publish_prg_running` refuses new publishes while it runs.

#### 1.3.12 Post steps
* `directory.yast` listing for `suse` repotype (`:3472-3481`).
* `sync_to_stage` (§1.8).
* `unpublishedhook` (`:3486-3501`, args `prp extrep deleted-files…`, failure only warns) and
  `publishedhook` (`:3503-3518`, args `prp extrep changed-files…`, failure **dies** → retry).
  Both support `*_use_regex` to match hook keys as anchored regexes (longest key first).
* `BSNotify::notify('REPO_PUBLISHED', {project, repo, buildid=>publishid})` (`:3520`).
* `splitdebug` recursion (`:3528`), trackercache update (`:3530-3545`), Diffnotify (`:3547-3550`),
  `PACKTRACK` notification with a Storable payload of all packtrack entries (`:3552-3557`),
  `sourcepublish` notification (`:3559-3563`), final `:repoinfo` (`:3565-3573`).
* `publishevent` (the dispatcher wrapper, `:3861-3946`) sends `REPO_PUBLISH_STATE`
  `{state => 'publishing'}` before and `{state => 'published'}` after, and `db_sync()` flushes
  the published-DB ops.

### 1.4 Per-repotype generators in detail

#### rpm-md (`createrepo_rpmmd`, `:651-939`)
* Cleans `repodata/{repomd.xml.asc,repomd.xml.key,latest-feed.xml,index.html,repoview,.olddata,patterns*}`, `.repodata`.
* `CREATEREPO_CHECKSUMS=$extrep/.chksums` env: concatenation of the signer-provided per-package
  checksum blobs so `createrepo_c` (SUSE-patched) can skip re-hashing rpms.
* Command line (`:691-742`):
  ```
  createrepo[_c] -q -c <extrep>/repocache [--no-database] [--update --discard-additional-metadata]
      --changelog-limit 20 --repo <tag>... [--content debug] [--content update]
      [--filelists-ext] --general-compress-type=gz|zstd
      --unique-md-filenames --checksum=sha256 | --simple-md-filenames --checksum=sha (legacy) | …sha512
      [--zck --zck-dict-dir=$bsdir/zchunk/<project>] --compress-type=gz|zstd <extrep>
  ```
  Options from `repotype rpm-md:<opt>`: `legacy`, `sha512`, `zchunk`, `compression-zstd`,
  `filelists-ext`/`filelists_ext`, `deltainfo`, `prestodelta`, `cmssign`, `rsyncable`,
  `sbom-spdx`, `sbom-cyclonedx`. If the `--update` run fails it retries without.
* `updateinfo.xml` (`$BSXML::updateinfo`, with `embargo_date`, `supportstatus`, `superseded_by`,
  `patchinforef`, `blocked_in_product` stripped; the max embargo date is written to
  `repodata/embargo_date` for mirror scripts) → `modifyrepo`.
* AppStream: `create_appdata_files` writes `appdata.xml` (+ `app-icons.tar`, icons extracted
  from base64 `filecontent`, 32/48/64/24 px preference) → `modifyrepo` as `appdata`,
  `app-icons`, and again as `appdata-icons`.
* `deltainfo.xml` / `prestodelta.xml` (`$BSXML::deltainfo`/`prestodelta`) from `%deltainfos`
  merged per `arch\0name\0epoch\0version\0release`, with size/checksum computed from the published
  drpm.
* modulemd: `writemodulemd --filter` (from obs-build) produces `modules.yaml` → `modifyrepo`.
* `repocache` pruned to 7 days. `repoview` if installed. `$BSConfig::createrepo_rpmmd_hook`
  code ref. `rezip_repo_rsyncable` if `rsyncable`.
* **Signing**: `sign [--project P] [--signflavor F] -P keyfile -h sha256 -d -4 repodata/repomd.xml`
  → `repomd.xml.asc` (detached, armored, v4 sig), `repomd.xml.key` = project pubkey,
  signature collected into `$data->{sigs}` for Rekor upload; optional CMS signature
  (`create_detached_cms_signature`, `:437-459`: `sign --cmssign --cert <certfile> --cms-nocerts
  -h sha512 repomd.xml` → `repomd.xml.p7s`-style output; cert from `/getsslcert?signtype=cms`).
* `<projid>.repo` file (`:866-886`):
  ```
  [openSUSE_Factory]            # ':' → '_', plus dbgsplit suffix
  name=<title>
  type=rpm-md
  baseurl=<downloadurl>
  gpgcheck=1
  gpgkey=<downloadurl>repodata/repomd.xml.key   (or $BSConfig::gpg_standard_key)
  enabled=1
  ```
* SBOM: `/usr/lib/build/generate_sbom --format spdx|cyclonedx --rpmmd <extrep>` →
  `repodata/sbom-{spdx,cdx}.json` + `.sha256` (`:888-918`), unless image builds already
  provided `*.spdx.json`.

#### susetags (`:1017-1072`)
Writes `media.1/media` (`Open Build Service\n<YYYYMMDDhhmmss>\n1`), `media.1/directory.yast`,
`content` (PRODUCT/VERSION/LABEL/VENDOR/ARCH.* table/DEFAULTBASE/DESCRDIR/DATADIR), runs
`create_package_descr -o descr -x /dev/null`, writes `descr/directory.yast`. Not signed here.

#### debian (`:1106-1212`)
Flat-repo layout, **not** `dists/<suite>/…`: `dpkg-scanpackages -m . /dev/null > Packages`
(with `Filename: ./` fixed to `Filename: `), `gzip -9n` copy as `Packages.gz`,
`dpkg-scansources . /dev/null > Sources` + `.gz`, `debian-installer/Packages` for udebs
(`-t udeb`). `Release`:
```
Archive: <repoid>
Codename: <repoid>
Origin: obs://<obsname>/<prj>/<repo>
Label: <prj>
Architectures: <Build::Deb::basearch of each arch>
Date: <asctime gmt>
Description: <title>
MD5Sum:/SHA1:/SHA256: " <sum> <size> <file>" for Packages Packages.gz Sources Sources.gz
```
Signing: `sign -d -4 Release` → renamed `Release.gpg`; `sign -c -4 InRelease.tmp` (clearsign)
→ `InRelease`; `Release.key` = pubkey. Clients use
`deb <downloadurl> /` (flat). No `Components`, no `Suite`, no by-hash.

#### arch (`:1233-1266`)
`bs_mkarchrepo <reponame> <extrep>/<arch>` per arch dir; `reponame` = project (`:`→`_`) plus
`_<repo>` unless `standard`. Produces `<rname>.db.tar.gz` and `<rname>.files.tar.gz` with
`desc`/`files` entries built from `Build::Arch::queryvars/queryfiles` (pure Perl, includes
`%PGPSIG%` from the `.sig` the signer created). Hardlinked as `<rname>.db`/`.files`. Signing:
`sign -D <file>` (detached binary sig → `.sig`) and `.key` pubkey.

#### apk (`:1281-1317`)
`bs_mkapkrepo --rewrite-arch <arch> <dir>` → `APKINDEX.tar.gz` (gzip'd empty signature tar +
gzip'd `APKINDEX` built from `Build::Apk::queryvars`, v2 only). Signing: RSA only; to-be-signed
data = `Build::Apk::gettbsdata`, `sign -O -h sha1|sha512` (raw PKCS#1), signature inserted by
`Build::Apk::replacesignature`, key name `<email>-<keycreate-hex>.rsa.pub`, PEM pubkey written
next to it.

#### staticlinks (`:1335-1425`)
Creates `name.rpm`, `name.deb`, `foo-latest-x86_64.AppImage`, `name.iso`, `name.qcow2`, etc.
symlinks (optionally `versioned`) pointing at the newest matching file; copies+patches `.sha256`
/`.sha512` files for the link name and re-signs them (`sign -d -4`). Relies on the webserver's
`FollowSymLinks`.

#### vagrant (`:1441-1521`)
Builds `boxes/<name>.json` catalogs (`{name, versions:[{version, status:active, providers:[{name,url}]}]}`)
from `*.vagrant.<provider>.box` files; atomic via `boxes.new` → rename.

#### zyppservice (`:588-640`)
`repo/repoindex.xml` listing every search-path repo as `<repo alias url enabled>`, signed with
`sign -d -4`.

#### helm (`:969-983`)
`index.yaml` from `helminfo` JSONs (`BSPublisher::Helm::mkindexentry` adds `digest`, `urls`,
`created`).

#### checksumsfile (`:1533-1568`)
Aggregates all `*.sha256` files into `SHA256SUMS`, signs detached (`.asc`) and optionally raw
(`rawsig` → `SHA256SUMS.gpg`).

#### virtbuilder (`:949-967`)
If an `index` was assembled from `*.index` parts: `sign -c <index>` (clearsign in place) +
`index.key`.

### 1.5 Patterns, ymp, comps
* `patterntype rpm-md` (`:1581-1630`): all patterns into one `repodata/patterns.xml`
  (`$BSXML::patterns`) → `modifyrepo`; then **re-signs** `repomd.xml` (and CMS if `cmssign`).
* `patterntype comps` (`:1640-1701`): converts patterns to comps `group.xml`
  (`requires`→mandatory, `recommends`→default, `suggests`→optional) → `modifyrepo`, re-sign.
* `patterntype ymp` (`:1712-1809`): one-click-install `<name>.ymp` per user-visible pattern,
  listing every search-path repo with its download URL (`BSUrlmapper::get_downloadurl`) and
  project title/description (`getprojpack?nopackages`), packages with summary/description
  queried from the published rpm/deb (`fillpkgdescription`). `publishflags ympdist:<dist>` adds
  one `<group distversion=…>` per dist. Each ymp is recorded in the published DB `pattern`
  table (`db_store($patterndb, "$prp_ext/$ympname", {type=>'ymp', name, summary, description,
  path})`). Note: `/published/<prj>/<repo>/<arch>/<binary>?view=ymp` on the repserver
  (`BSRepServer::YMP::makeymp`) generates ymps on the fly for single binaries; the publisher
  only pre-generates pattern ymps.

### 1.6 Flags reference

`publishflags:` tokens in the project config (`Build::read_config` turns `publishflags: a b:c`
into `$config->{'publishflags:a'}=1` plus the raw list in `$config->{publishflags}`):

| flag | consumer | effect |
|---|---|---|
| `archsync` | scheduler + publisher | wait until all architectures have been checked after the last change |
| `createempty` / `create_empty` | both | publish (and keep metadata for) an empty repo |
| `excludepublish:<pkg>`, `onlypublish:<pkg>` | scheduler | package black/whitelist |
| `nofailedpackages` | scheduler | don't publish if any package failed |
| `keepobsolete` | scheduler | keep binaries of removed packages when publishing is globally disabled |
| `noearlypublish` / `noearlykiwipublish` | scheduler | don't publish kiwi/productcompose repos before the whole prp is finished |
| `singleexport` | scheduler | one binary per name.arch |
| `noorphanedsrcrpms` | scheduler | drop src.rpm if no binary rpm was taken |
| `nocontentchange` | publisher | never replace an already published file |
| `withreports`, `withsbom` | publisher | publish `.report`/`.packages` and SBOM json files |
| `mandatorycontainerplatforms=a,b` | publisher | NORETRY error unless every tag has those platforms |
| `artifacthub`, `artifacthub:<gun/repo>:<data>` | publisher | push an `artifacthub.io` tag with repository metadata |
| `cosignbundle` | publisher (local registry) | use sigstore bundle v0.3 referrers instead of `.sig/.att` tags |
| `ympdist:<dist>` | publisher | ymp distversion list |
| `buildflags:signflavor` | signer + publisher | select a key flavor in the sign daemon |

Project `publish` enable/disable flags (per repo/arch, `$BSXML::proj publish` element) are only
evaluated by the scheduler (`Checker.pm:1549-1562`); package-level `publish` flags too.
"Disable publishing" therefore means: the scheduler never fills `:repo`, and marks `:repodone` as
`disabled:…`; the publisher never sees an event. `lock` flags likewise block publishing
(`:1545-1548`, `:1597-1600`).

`BSConfig` keys consumed by the publisher: `sign`, `sign_project`, `sign_type`, `sign_flavor`,
`sign_package` (signer only), `keyfile`, `certfile`, `gpg_standard_key`, `createrepo`,
`modifyrepo`, `createrepo_rpmmd_hook`, `publisher_compile_content_hook`, `publishprogram`,
`publishredirect` (+`publishedredirect_use_regex`), `prp_ext_map`, `repodownload`,
`stageserver`, `stageserver_sync`, `rsync_extra_options`, `publishedhook`,
`publishedhook_use_regex`, `unpublishedhook`, `unpublishedhook_use_regex`, `packtrack`,
`sourcepublish_sync`, `sourcepublish_filter`, `publish_diffnotify`, `publish_nodb`,
`publish_maxchild`, `publish_maxchild_flavor`, `publish_getflavor`, `publish_penalty_multiplier`,
`markfileorigins`, `obsname`, `container_registries`, `publish_containers`,
`local_registry_signatures_extension`, `extradeltarepos` (scheduler), `relsync_pool` (scheduler
release-number sync — only tangentially related: it decides which archs share a release counter,
so that `-Build<n>` numbers match across archs that are published together), `disable_data_sync`,
`published_db_sqlite`, `srcserver`, `reposerver`, `bsdir`, `rundir`, `logdir`.

### 1.7 Product publishing (kiwi / productcompose)
There is no separate "productrepo" generator. Product builds (`_product:` packages for kiwi
products, `productcompose` builds) deliver directory trees (`*-Media1`, `-Media2`, …) plus
`*.iso`, `.report`, `.packages`, `.sha256`, `ChangeLog.*`. The publisher:
* `linktree`s the media dirs to `repo/<dir>` (or `slepool` names), isos to `iso/`;
* `sleimage` / `sleimage-media` repotypes (`mapsleimage`, `:2059-2087`) rename
  `-Build<N>` → `-<milestone>` (from a `<name>.milestone` file, macro-expanded) and `-Media` →
  `-DVD`, and patch + re-sign the `.sha256` files (`fixsleimage`, `:2132-2208`);
* `staticlinks` makes `…-Media1.iso` → `…-Media1.iso` without build id links;
* scheduler `earlypublish` publishes kiwi/productcompose prps before all packages finish
  (`bs_sched:1130-1142`).
`bs_productconvert` is a *build-time* tool (converts `_product/*.product` XML into kiwi configs
and spec files; `BSProductXML.pm` = `$BSProductXML::productdesc` schema, `BSKiwiXML.pm` =
kiwi config schema) invoked by the source server / worker, not by the publisher.

### 1.8 Mirroring: `sync_to_stage` (`:1826-1921`)
Stage servers come from `publishredirect[3]` or `$BSConfig::stageserver` (string or
`[regex => server(s)]` list). Supported targets:
* `rsync://` / `rsync-ssl://`: two passes using `--files-from=-` with the prp's extdir:
  1. binaries only (`--include=*.rpm` etc. from `@BSOBS::binsufs`, `--fuzzy`, no delete) and
  2. everything with `--delete-after --exclude=repocache --delete-excluded`
  (timeout 7200 s). On delete the parent dir is synced.
* `script:/path`: `script <prp> <extdir>` (or just `<prp>` on delete).
* `rekor://host` → `https://host`: uploads every collected detached signature
  (`$data->{sigs}` = `{data, signature, pubkey, format=>'pgp'}`) as a Rekor `rekord`
  (`BSRekor::upload_rekord`).
Then, if `$BSConfig::stageserver_sync` is set, a trigger file `repos_sync/<prp with / → _>`
containing `<extdir>\0` is rsynced (120 s timeout) to tell the stage server to fan out to
mirrors. The pattern is: rsync to one "stage" server, trigger file tells it which prp changed.

### 1.9 Source publishing (`bs_sourcepublish`)
Separate `BSStdRunner` on `events/sourcepublish/`. The publisher sends a `sourcepublish`
notification (payload: `{ "<prj>/<pkg>/<srcmd5>" => [included "<prj>/<pkg>/<srcmd5>" …] }`) via
the source server (`BSNotify`), which the API/source server turns into
`sourcepublish $project $package $srcmd5 $included:*` events (`$BSXML::event included`
elements). The daemon exports the source revision (`BSRevision::getrev_local` /
`getrev_deleted_srcmd5`, `revcopyoutfile`) into `sources/:upload/sourcepublish<pid>/<prj>/<pkg>/<srcmd5>/`,
adds `INCLUDED/<prj>::<pkg>::<srcmd5>` symlinks for image-embedded packages, rsyncs
(`--copy-unsafe-links --no-munge-links --fuzzy`) to `$BSConfig::sourcepublish_sync`, and records
`(project,package,srcmd5)` in sqlite `db/sqlite/publishedsource` to avoid re-syncing.
Multibuild flavors are stripped; `sourcepublish_blocklist` exists in config but is enforced
elsewhere.

### 1.10 Notifications emitted
All go through `BSNotify::notify($type, {params}, payload)` → `POST $srcserver/notify/<type>`
(`BSNotify.pm:40-70`); the source server relays to the API (`/lastnotifications`) and plugins
(RabbitMQ etc.). Payloads are Storable (`nfreeze`) and fatal on failure; param-only notifications
only warn.

| type | params | payload |
|---|---|---|
| `REPO_PUBLISH_STATE` | project, repo, state=publishing/published | – |
| `REPO_PUBLISHED` | project, repo, buildid=publishid | – |
| `CONTAINER_PUBLISHED` | project, repo, buildid, container=`<gun>/<repo>:<tag>` | – |
| `PACKTRACK` | project, repo | Storable array of packtrack entries (binary release tracking; API fills `binary_releases`) |
| `sourcepublish` | project, repo | Storable `{sourceid => [included…]}` |
| Diffnotify | HTTP POST to `publish_diffnotify{uri}` with Bearer/Basic auth | JSON `{dag_run_id, logical_date, conf:{project, repository, binaries:[{path,name,binaryname,architecture,schedulerarch,binaryid,package,flavor,disturl,state=added/changed/removed/unchanged,platform,tags,visibletags,associated:[…]}], registries:{<gun>:{tags:[{tag,distmanifest,binaryid,state…}]}}}}`; previous state kept in `<statedir>/<prp>/state`, last report in `…/report` |
| `markfileorigins` | `HEAD $markfileorigins/<prp_ext>/<file>?cmd=setpackage&package=<pkg>` / `?cmd=deleted` | – (download statistics hook) |

### 1.11 Delete / unpublish flows
* **Repository or project removed** (`getprojpack` returns nothing) → `deleterepo`
  (`:1949-2057`): delete all container repositories from registries
  (`BSPublisher::Container::delete_container_repositories` → `upload_all_containers(undef)`,
  §3), unlink every binary (recording `@db_deleted`), update published DB (`binary` index
  removal, `repoinfo` row delete), `markfileorigins deleted`, delete ymps from DB, `rm -rf
  $extrep`, `sync_to_stage(..., isdelete=1)`, recurse for the `splitdebug` tree, `PACKTRACK`
  notification with an empty list, unlink `$reporoot/$prp/:repoinfo`, `rmdir` project dir in
  `repos/`.
* **All `:repo` dirs gone** (e.g. all archs removed or `unpublish`) → same `deleterepo`.
* **API `unpublish`** (`bs_repserver:3241-3289`): under `.finishedlock`, remove the listed
  packages' binaries from `<arch>/:repo` (using `:repoinfo` origins) or wipe `:repo` entirely,
  send `recheck` events to the schedulers and a `publish` event → publisher republishes
  (removing files) or deletes.
* **`wipepublishedlocked`** (`:3223-3237`): wipes all `:repo` dirs even when the project is
  locked, then publish event.
* Single-file removal is just the normal diff: the publisher removes anything in `$extrep` that
  is no longer in `%bins`.
* Registry side: obsolete tags are deleted by `bs_regpush -X -F digestfile` (delete-except) or
  for the local registry by the GC in `push_containers`; notary GUNs via `bs_notar -D`.

---

## 2. Signing

### 2.1 What gets signed, where, with which `sign` mode

| Artefact | Signed by | sign invocation | Output |
|---|---|---|---|
| `*.rpm`, `*.drpm` (job results) | `bs_signer` | `sign [--project P] [--package PKG] [--signflavor F] [-P keyfile] [-h sha256] [-S <jobdir>/.checksums] -r [-T <signtime>] file.rpm` (`bs_signer:1070-1100`) | rpm header/v4 signature in place (`-r` = rpm mode); `-T buildtime` for drpms; `.checksums` sidecar with digests for createrepo |
| aggregated rpms (`_aggregate`) | `bs_signer` | `sign --delsign -r file.rpm` then re-sign with the old signtime clamped to the key creation time (`:1023-1050`); if same issuer the old rpm is reused | |
| `*.pkg.tar.{gz,xz,zst}` (Arch) | `bs_signer` | `sign -D -4 file` | detached binary `.sig` (OpenPGP v4) |
| `*.AppImage` | `bs_signer` | `sign -a file` (+ regenerate `.zsync` with `zsyncmake`, `.digest`) | embedded signature |
| `*.sha256`, `*.sha512` (image checksums) | `bs_signer` | `sign -d file` | detached armored `.asc`; iso checksum lines patched first (`fixup_sha256_checksum`) |
| `*.iso` | `bs_signer` | `signisofiles` (`:160-230`): finds 2048-byte `sIGnMe!\n<len><sum>` placeholder files `*.asc` and 8192-byte `*.p7s` inside the ISO9660 directory tree, signs the referenced file's content with `sign -d` (asc) or `sign --cmssign --cert … --cms-nocerts -h sha512` (p7s) and writes the signature back in place (patching the directory record length), replaces `sIGnMeP\n` `*.key` placeholders with the pubkey (`sign -p`); then `retagiso` recomputes the md5/sha tag in the primary volume descriptor (`:263-360`) and `signisotag` fills the `;signature=<blk>` block with a detached signature of the PVD tag area (`:232-250`) | in-place |
| kiwi trees (`.kiwitree_tosign`) | `bs_signer` | per listed `*.asc`/`*.p7s`/`*.key` placeholder file in the job dir, same placeholder protocol (`:948-965, 1090-1110`) | |
| `*.appx` | `bs_signer` | `sign --appx --cert <certfile> [--othercerts othercerts.der] file.appx` with signtype `appx` cert (`:466-486`) | Authenticode in place |
| `*.apk` | `bs_signer` | `sign -O -h sha1|sha512` over `Build::Apk::gettbsdata`, then `Build::Apk::replacesignature` (`:488-507`) | in place |
| `*.helminfo` → `<chart>.tgz.prov` | `bs_signer` | `sign -c` (clearsign) of `config_yaml + "...\nfiles:\n  <chart>: sha256:<sum>\n"` (`:509-547`) | helm provenance |
| `*.rsasign`, `*.cpio.rsasign` (kernel/secure-boot followup) | `bs_signer` | `sign -O -h sha256 [--bulk-cpio] file` → `.sig` (`:362-405`); fallback: iterate cpio, sign each file, write a cpio of `.sig`s | raw PKCS#1 RSA signatures, followup build job integrates them |
| `_slsa_provenance.json` / `<bin>.slsa_provenance.json` | `bs_signer` | `signslsaprovenance` (`:633-710`): POSTs material references to `$srcserver/slsa?cmd=addrefs` (pinning all input binaries in the repserver's `_slsa` store), fixes subject digests to the *signed* rpms' sha256, then DSSE-signs with `sign -D` (raw OpenPGP detached) | `{payloadType:"application/vnd.in-toto+json", payload:b64, signatures:[{sig:b64}]}` |
| `repodata/repomd.xml`, `Release`, `InRelease`, `repoindex.xml`, `SHA256SUMS`, `index`, patched `.sha256` | `bs_publish` | `sign -d -4` / `sign -c -4` / `sign -c` (see §1.4) | `.asc`/`Release.gpg`/`InRelease` |
| `repomd.xml` CMS | `bs_publish` | `sign --cmssign --cert <projectcert> --cms-nocerts -h sha512` | PKCS#7 |
| `APKINDEX.tar.gz` | `bs_publish` | `sign -O -h sha1|sha512` | embedded |
| Arch `.db.tar.gz`/`.files.tar.gz` | `bs_publish` | `sign -D` | `.sig` |
| container cosign signature / attestations | `bs_regpush`, `BSPublisher::Registry` | `sign [--project P] [-P keyfile] -O -h sha256` over the simplesigning payload or the DSSE PAE | `dev.cosignproject.cosign/signature` annotation / DSSE `sig` |
| container atomic signature (OpenShift/`containers-signature.5`) | `BSPublisher::Registry::update_sigs` | `sign -D -h sha256` → `BSPGP::onepass_signed_message` + `compress_packets` | OpenPGP one-pass signed, compressed message |
| Notary/TUF root cert & metadata | `bs_notar`, `Registry::update_tuf` | `sign -O -h sha256` over DER tbsCertificate / canonical JSON | X.509 self-signed root, `rsapkcs1v15` TUF sigs |
| TUF timestamp role | `Registry::gen_timestampkey` | `sign --project :tmpkey -P tmpkey -g rsa@2048 800 'timestamp signing key' timestampsign@…` then `-O` with that key | per-registry-repo RSA-2048 key stored in `:tuf` |
| ssh signatures (`BSSSHSign`) | callers of `BSSSHSign::sshsign` | `sign -u <user> -h sha256|sha512 -O` over `SSHSIG`-framed hash | armored `SSH SIGNATURE` |

### 2.2 Key model

**Storage (source server)**: a project's keys live in the project meta revision as
`_pubkey`, `_signkey`, `_sslcert` files (`bs_srcserver:2148-2215`, `BSRevision::addrev_meta_replace`)
with working copies at `$projectsdir/<prj>.pkg/_pubkey|_signkey`. `_signkey` is the *encrypted*
private key blob as returned by `sign -g` — the sign daemon's master passphrase is needed to use
it; it is never decryptable on the OBS host. `_sslcert` is an X.509 cert generated by `sign -C`
from the same key (code-signing cert for CMS/appx/Notary root).

**Creation**: `POST /source/<prj>?cmd=createkey` → `BSSrcServer::Signkey::createkey`
(`Signkey.pm:36-60`):
```
sign [--project <prj>] -P <tmpfile> -g rsa@4096 800 "<prj> OBS Project" "<prj>@<obsname>"
```
stdout = armored pubkey; `-P` file receives the private key blob. Default algo `rsa@4096`, expiry
800 days. `extendkey` (`:62-80`): `sign -P <signkey> -x <days> <pubkeyfile>` → new pubkey.
`pubkey2sslcert` (`:88-105`): `sign [--signtype T] [--signflavor F] -P <signkey> -C <pubkeyfile>`
→ PEM cert. `getdefaultpubkey`/`getdefaultcert` (`:107-130`): `sign [--project P] -p` / `-C`
without `-P` → the daemon's *default* key (or the project-specific key the daemon has configured
for `--project`). If `$BSConfig::forceprojectkeys` (from API configuration
`enforce_project_keys`) a key is auto-created on project creation (`bs_srcserver:2258-2262`) and
`deletekey` refuses unless a parent project has one.

**Inheritance**: `findsignkey($projid)` (`bs_srcserver:6288-6297`) walks up the project
namespace (`home:foo:branches:bar` → `home:foo:branches` → `home:foo` → `home`) and returns the
first project with a `_signkey`. So sub-projects sign with their nearest ancestor's key.

**Sentinel "special" key**: a `_signkey` of length ≤ 2 (e.g. `"1"`) means "this project uses a
key that lives *inside signd*, selected by `--project`/`--signtype`/`--signflavor`" — the source
server then does not pass `-P` and asks `sign -p`/`-C` for pubkey/cert (`:6305-6314`,
`:6341-6347`, `Signkey.pm`, `bs_signer:457-468`). This is the HSM-ish path used for
official distributions.

**Auto-extend**: `autoextend_check` (`:6276-6286`): if `withpubkey&autoextend` and the pubkey
expires within 14 days, `extendkey` is run (by the source server) and the new pubkey returned.
Both the signer and the publisher pass `autoextend=1`.

**Fallbacks when no project key exists** (`bs_publish:337-388 getsigndata`,
`bs_signer:899-947 getsignkey`): `sign --project P -p` if `sign_project`; else
`$BSConfig::keyfile` (static armored pubkey); the repo file's `gpgkey=` falls back to
`$BSConfig::gpg_standard_key`. `sign` without `-P` then signs with the daemon's default key.

**Source server endpoints** (`bs_srcserver:8069-8077`):
```
GET /getsignkey?project=P[&withpubkey=1][&autoextend=1][&withalgo=1][&signtype=T][&signflavor=F]
    → text/plain: "<algo>:<signkeyblob>\n<armored pubkey>"   (algo '?' if unknown; empty body if no key)
GET /getpubkey?project=P[&autoextend][&signtype][&signflavor]   → armored pubkey (project or default)
GET /getsslcert?project=P[&autoextend][&signtype][&signflavor]  → PEM cert (cached in _sslcert, or generated)
GET /source/<prj>/_pubkey[?rev=]      → pubkey (remote projects proxied)   (getpubkey, :2217-2239)
GET /source/<prj>/_keyinfo[?withsslcert&autoextend&donotcreatecert&signtype] → <keyinfo><pubkey algo keysize userid keyid fingerprint expires/><sslcert serial begins expires subject issuer fingerprint keyid/></keyinfo>  ($BSXML::keyinfo, Signkey::pubkeyinfo/certinfo)
POST /source/<prj>?cmd=createkey|extendkey[&days=]|recreatesslcert
```
`$BSConfig::disable_sslcert_cache{$prj}` makes the cert be regenerated on every request instead
of being committed to `_sslcert` (needed when signtype/flavor vary).

**Sign type / flavor**: `--signtype <cms|appx|notary|helm|…>` is appended when
`$BSConfig::sign_type` is set (`setsigntype`, `bs_signer:949-960`, inserted after `--project`
/`--package`); `--signflavor <F>` when the project config has `buildflags:signflavor` and the
flavor is in `$BSConfig::sign_flavor`. `--package <pkg>` when `$BSConfig::sign_package`. These
only matter to the daemon's key selection policy.

### 2.3 Signer queue and job flow (`bs_signer`)

1. A worker uploads a finished job to `bs_repserver` (`putjob`, `bs_repserver:2333-2616`). If
   `$BSConfig::sign` is set and any uploaded file matches
   `_slsa_provenance.json|*.{rpm,drpm,sha256,sha512,iso,pkg.tar.*,AppImage,deb,appx,apk,helminfo,raw,efi,roothash,usrhash}`
   or the job has a `.kiwitree_tosign` list, the job status becomes `signing` instead of
   `finished` (`:2581-2594`), and the event is written to `events/signer/finished:<arch>:<job>`
   (`:2607-2615`) instead of `events/<arch>/finished:<job>`. `_reproduciblecheck` jobs skip
   signing. Uploaded builds (`uploadbuild`) and scheduler-created `_aggregate` jobs
   (`BSSched/BuildJob.pm:740-747`) go the same way.
2. `bs_signer` (`BSStdRunner` on `events/signer/`, `maxchild` 4 by default, flavors `iso` /
   `rsasign` via `getflavor`, `:1269-1300`) dispatches `built $job $arch` / `uploadbuild $job $arch`
   → `signevent` → `signjob` (`:981-1240`).
3. `signjob` locks `<jobsdir>/<arch>/<job>:status` (`lockopenxml`), requires `code` ∈
   {`finished`,`signing`}, reads the `buildinfo` (`$BSXML::buildinfo`) for project/package/
   repository/signflavor/buildtime, lists the job dir and selects `@signfiles`.
4. Key material: `getsignkey($projid,$packid,$needpubkey,$needcert,undef,$signflavor)` (RPC to
   source server, §2.2); pubkey needed for iso/apk/aggregate; cert for appx; CMS cert
   (`getsslcert(signtype=cms)`) for `.p7s` placeholders. The private key blob is written to
   `$uploaddir/signer.<pid>` and passed as `-P`.
5. Followup detection (`:1073-1085`): if `.rsasign` files exist with a `.spec`/`.dsc`, or a
   `*.followup.spec`, or `mkosi.*` files exist, only the rsasign step is done and the job is
   handed *back to the scheduler as a followup build* (`info->{followupfile}`,
   `followupsteps++` max 3, `:1188-1198`): the worker re-runs the build with the signatures
   available (secure boot / signed kernels / mkosi). Loops are capped by `max_followupsteps`.
6. `-S <jobdir>/.checksums` is passed for rpm signing when the daemon supports it
   (`check_sign_S`, `:76-88`, probed at startup with `sign -S /dev/null -k`); the file is
   later merged into `:repo/.newchecksums` by the scheduler for createrepo.
7. After signing, the job's `.bininfo` is rebuilt (`:1203-1228`) because signing changed the
   rpm/deb ids: `Build::query evra`, `leadsigmd5`, `hdrmd5`, filename, id.
8. Status is written back as `finished` (`:1231-1235`) and the event is forwarded to the
   scheduler as `events/<arch>/finished:<job>` (`signevent`, `:1250-1265`). Redis job status
   updated if configured.
9. Errors: message prefix `fatal:` → `result=failed` (log appended, job marked finished so the
   scheduler records a failure); `rebuild` (set when rpm `--checksig --nosignature` fails after
   a sign failure, or when `slsa addrefs` returns 404) → job dir wiped, status removed, scheduler
   will rebuild; anything else → `setdue(now+180)` retry (`:1246-1249`). Non-RSA keys for
   openssl modes produce the fixed message "Need an RSA key for openssl signing".

### 2.4 The `sign` client CLI as used by OBS (observed flags)

`$BSConfig::sign` (default `/usr/bin/sign` from the obs-signd package). All invocations in this
tree, with the flag semantics inferred from usage:

```
Global / key selection
  --project <prj>         key selection hint (only if $BSConfig::sign_project)
  --package <pkg>         (only if $BSConfig::sign_package)
  --signtype <type>       cms|appx|notary|helm (only if $BSConfig::sign_type)
  --signflavor <flavor>   from buildflags:signflavor
  -P <file>               use the (encrypted) private key blob in <file> instead of the daemon default
  -u <user>               key by user id (BSSSHSign)
  -h <hash>               sha1|sha256|sha512 digest algo
  -T <time|buildtime>     signature creation time (rpm mode)
  -S <file>               write per-file checksums for createrepo
  -k                      (used with -S /dev/null as capability probe)

Modes
  -g <algo@bits> <expiry-days> <name> <email>   generate key; pubkey on stdout, privkey to -P file
  -x <expiry-days> <pubkeyfile>                 extend pubkey expiry; new pubkey on stdout
  -p                                            print pubkey (armored) of the selected key
  -C [<pubkeyfile>]                             print X.509 cert (PEM) for the key
  -r <file.rpm>                                 sign rpm in place (rpm v3/v4 header sig)
  --delsign -r <file.rpm>                       remove rpm signatures
  -d <file>                                     detached armored signature → <file>.asc
  -d -4 <file>                                  detached armored v4 signature
  -D <file>                                     detached binary signature → <file>.sig
  -D -4 <file>                                  same, v4
  -D -h sha256  (stdin→stdout)                  detached binary signature of stdin (used for DSSE/atomic sigs)
  -c <file>  /  -c -4 <file>                    clearsign in place (InRelease, helm .prov via stdin→stdout, virt-builder index)
  -a <file.AppImage>                            AppImage embedded signature
  -O -h <hash>  (stdin→stdout | <file>→<file>.sig)   raw "openssl" PKCS#1 v1.5 RSA signature (no PGP framing)
  -O --bulk-cpio <file.cpio.rsasign>            sign every file in a cpio, writes <file>.sig cpio
  --cmssign --cert <pem> --cms-nocerts -h sha512 <file> | stdin   CMS/PKCS#7 detached signature (.p7s)
  --appx --cert <pem> [--othercerts <der>] <file.appx>            Authenticode for appx
```

Data flow conventions: when a filename is given the tool reads/writes files next to it; when
no filename is given (`BSUtil::xsystem($data, …)`) data goes via stdin and the signature comes
back on stdout (`BSUtil.pm:699-760`). Exit status ≠ 0 is treated as failure; stderr is captured
into the exception text.

### 2.5 The signd wire protocol (external `obs-signd`)

The daemon is **not** part of this repository (`dist/obs-server.spec:187` only
`Recommends: obs-signd`; `dist/install_rh.sh:74-96` and `dist/functions.setup-appliance.sh:580-620`
configure it). What this tree shows about it:

* `/etc/sign.conf` keys written by the appliance setup: `user: <keyid/email>` (default key),
  `server: 127.0.0.1` (where the client connects), `allowuser: obsrun` (uid allowed to use the
  client's privileged socket), `allow: 127.0.0.1` (daemon ACL), `phrases: <dir>` (directory of
  passphrase files named after key ids, e.g. `gnupg/phrases/defaultkey@localobs`). The daemon
  runs with `GNUPGHOME=/srv/obs/gnupg` (`install_rh.sh:77`); keys live in a gpg keyring owned by
  the daemon host.
* The service unit `obssigner.service` orders `After=signd.service`
  (`dist/systemd/obssigner.service`).
* `bs_signer` probes capabilities at startup by running `sign -S /dev/null -k`,
  `sign --bulk-cpio --hashfile /dev/null`, `sign --delsign --hashfile /dev/null`
  (`bs_signer:76-120`) — i.e. the client has a `--hashfile` mode where it computes hashes locally
  and only ships hashes to the daemon.

What is known about obs-sign (github.com/openSUSE/obs-sign; `sign.c` client, `signd` perl
daemon) — **from general knowledge of that project, not from this tree**; verify against the
version you deploy:

* The client is setuid-root (or uses a privileged port) and connects to the daemon on
  **TCP port 5167** ("signd") from a privileged source port; the daemon's `allow:` list is an
  IP ACL and the client's `allowuser:` restricts who may invoke it.
* The client never sees private keys. For rpm signing it computes the header+payload digests
  locally (`-S` checksums, `--hashfile`) and sends only the *hash* to the daemon; the daemon
  returns an OpenPGP signature packet which the client splices into the rpm signature header.
  For `-d/-D/-c/-O/--cmssign` the data (or its digest) is sent in the same way.
* Request framing: a length-prefixed sequence of NUL-separated argument strings
  (`user`, `hashalgo`, then the mode and its arguments such as the hex digest, signature time,
  `--project`/`-P` blobs), sent as `<u16 argc><u16 len>…`; reply is
  `<u32 status><u32 outlen><u32 errlen>` followed by stdout and stderr bytes. Status is the
  daemon's exit code. (Framing details per obs-sign `signd`/`sign.c`.)
* `-P <file>` transports an *encrypted* private key that only the daemon can decrypt (it was
  produced by the daemon's `-g` with the daemon's master phrase); this is what lets OBS keep
  per-project keys in the source tree without the signer host holding secrets.
* `--project/--signtype/--signflavor/--package` are forwarded verbatim so the daemon's config
  can map them to different keys (e.g. HSM-backed official keys for certain flavors).

For a Go reimplementation the right abstraction is therefore: a **signing service RPC** with
operations {`GenKey(algo, expiry, uid) → (pubkey, wrappedPrivKey)`, `ExtendKey`,
`GetPubKey(selector)`, `GetCert(selector, signtype)`, `SignDigest(selector, hashalgo, digest,
mode=pgp-v4|raw-pkcs1|cms, time)`, `SignRPM` (= digest of header+payload), `ClearSign`} where
`selector = {project, package, signtype, signflavor, wrappedPrivKey?}`. Everything the Perl
callers do can be expressed through those, and the rpm/iso/apk/appx container formats are
client-side splicing.

### 2.6 HTTP Signature auth (`BSSigAuth.pm`) and ssh signatures (`BSSSHSign.pm`)

`BSSigAuth::generate_authenticator($keyid, keyfile => …)` returns an authenticator for
`BSRPC` that answers a `WWW-Authenticate: Signature realm=…,headers="(created)",algorithm="ssh"`
challenge (`:62-100`) with
`Authorization: Signature keyId="…",algorithm="ssh",headers="(created)",created=<ts>,signature="<b64 SSHSIG>"`,
signing `"(created): <ts>"` with `ssh-keygen -Y sign -n <realm> -f <keyfile>` or a callback.
This is OBS's scheme for *outgoing* authenticated requests (e.g. to another OBS acting as
registry/notification target); it is unrelated to signd. `BSSSHSign::sshsign` (`:45-61`) builds
the same `SSHSIG` structure but gets the raw RSA signature from `sign -u <user> -h <hash> -O`,
i.e. ssh signatures with a signd-held RSA key; `keydata2pubkey` converts a PGP RSA key into an
`ssh-rsa` line.

### 2.7 Container, TUF, cosign, Rekor, DSSE, SLSA — the cryptographic formats

* **PGP → X.509/SSL key bridging**: `BSPGP::pk2keydata` extracts `{algo, mpis, keysize, curve}`
  from a PGP pubkey; `BSX509::keydata2pubkey` packs it as a DER SubjectPublicKeyInfo. All
  non-PGP signing (cosign, TUF, Rekor, ssh) reuses the project's *PGP RSA key* via `sign -O`
  (raw PKCS#1 v1.5) and presents the key as PEM `PUBLIC KEY`. Non-RSA project keys silently
  disable container signing (`Container.pm:103-107`, `Registry.pm:231`).
* **cosign cookie** (`BSConSign::create_cosign_cookie`, `:226-240`):
  `sha256("OBS/<pubkey fingerprint or sha256(spki)>/<gun>")`, stored as annotation
  `org.open-build-service.cosign.cookie` on every signature/attestation layer; `bs_regpush -l
  --cosign` reports `layers=N,cosigncookie=<c>` per `.sig/.att` tag so the publisher can tell
  whether the registry already holds signatures made with the current key for the current GUN
  (`Container.pm:add_cosign_tags_to_expected`, `compare_to_repostate`).
* **cosign simplesigning payload** (`BSConSign::create_signature_payload`, `:60-74`):
  `{"critical":{"identity":{"docker-reference":"<gun>"},"image":{"docker-manifest-digest":"sha256:…"},"type":"cosign container image signature"},"optional":{"creator":"OBS"}}`
  canonical JSON; signature = `sign -O -h sha256` over it, base64 in annotation
  `dev.cosignproject.cosign/signature`; stored as OCI manifest with an empty cosign config
  (`create_cosign_config_ent`) under tag `sha256-<digest>.sig`.
* **Attestations**: SLSA provenance, SPDX, CycloneDX and arbitrary `*.intoto.json` files next
  to the container are wrapped into in-toto v0.1/v1 statements with the subject rewritten to the
  pushed manifest digest (`fixup_intoto_attestation`), DSSE-signed
  (`dsse_pae("DSSEv1 <len> <type> <len> <payload>")`, `sign -O -h sha256`), layer mediaType
  `application/vnd.dsse.envelope.v1+json`, tag `sha256-<digest>.att`. With `cosignbundle`
  (sigstore bundle v0.3, `mt_cosign_bundle`), each attestation becomes its own OCI artifact
  manifest with `subject` (OCI referrers API) and `artifactType`, and the signature itself is
  just another bundle.
* **Rekor** (`BSRekor.pm`): `POST <server>/api/v1/log/entries` with `hashedrekord` (signature
  over sha256 hash, PEM pubkey), `rekord` (full content, used for repomd.xml.asc uploads),
  `intoto` 0.0.1 / `dsse` 0.0.1 (DSSE envelopes); `/api/v2` (rekor-tiles) uses
  `hashedRekordRequestV002` with `keyDetails` and DSSE is downgraded to hashedrekord. The
  returned entry (201/409) is turned into a cosign "bundle" annotation
  (`dev.sigstore.cosign/bundle` with `Payload{body,integratedTime,logIndex,logID}` +
  `SignedEntryTimestamp`) or into a `tlogEntries` entry of the v0.3 bundle
  (`rekorentry_to_tle`).
* **Atomic signatures** (`Registry::update_sigs`, `:401-440`): payload type
  `atomic container signature`, signed with `sign -D -h sha256`, wrapped by
  `BSPGP::onepass_signed_message` (one-pass sig packet + literal data packet named
  `rpmsig-req.bin` + signature packet) and deflate-compressed — served via
  `/registry/<repo>/signatures/<digest>` as `{"signatures":[{schemaVersion:2,type:atomic,name:"<digest>@<md5>",content:b64}]}`
  (OpenShift/`X-Registry-Supports-Signatures` extension, enabled by
  `$BSConfig::local_registry_signatures_extension`).
* **TUF / Notary** (`BSTUF.pm`, `Registry::update_tuf`, `bs_notar`): root key =
  `rsa-x509` self-signed cert (CN=GUN, validity = pubkey selfsig_create…key_expire+6 months,
  code-signing EKU) signed with `sign -O`; snapshot and targets signed with the same key; a
  separate RSA-2048 timestamp key generated through signd (`--project :tmpkey`, blob kept in
  `:tuf`) so the repserver can re-sign `timestamp.json` every 14 days without the project key
  (`BSRepServer::Registry::extend_timestamp`, served at
  `/registry/<gunprefix>/<repo>/_trust/tuf/{root,targets,snapshot,timestamp}.json[.<sha256>]` and
  `{snapshot,timestamp}.key`). Targets = `{tag => {hashes:{sha256:b64}, length}}`. Signatures use
  `method: rsapkcs1v15`, `keyid = sha256(canonical_json(key))`. `bs_notar` does the same against
  a remote Notary server (`/v2/<gun>/_trust/tuf/` GET/POST multipart/DELETE), reusing the
  server's timestamp key (`/_trust/tuf/timestamp.key`).
* **SLSA pinning**: `POST $srcserver/slsa?cmd=addrefs&prpa=<prpa>` with a Storable
  `{ "<prj>/<repo>/<arch>" => { <sha256> => <filename> }, "_config" => {sha256 => config} }`
  (forwarded to the owning repserver, `BSRepServer::SLSA::add_references`), which hardlinks the
  referenced binaries into `$bsdir/slsa/<prpa>/` and an sqlite `refs` table so provenance
  materials (`uri …/_slsa/<prj>/<repo>/<arch>/<file>/<digest>`) stay downloadable via
  `/slsa/<prj>/<repo>/<arch>/<file>/<digest>` even after the binaries are gone.

---

## 3. Container publishing

### 3.1 Where `containerinfo` comes from
The build (kiwi `docker`/`oci` image type, Dockerfile builds via obs-build, helm chart builds)
produces next to the image tar a `<name>.containerinfo` JSON (`BSRepServer::Containerinfo::readcontainerinfo`,
`BSRepServer/Containerinfo.pm:149-200` validates it). Observed/used keys:

```json
{
  "name": "opensuse/leap",            "tags": ["opensuse/leap:15.6", "opensuse/leap:latest"],
  "file": "opensuse-leap-15.6-Build7.3.docker.tar",
  "version": "15.6", "release": "7.3", "arch": "x86_64",
  "goarch": "amd64", "goos": "linux", "govariant": "v8",
  "disturl": "obs://build.opensuse.org/openSUSE:Containers:Leap:15.6/images/<srcmd5>-opensuse-leap-image",
  "buildtime": 1718000000, "imageid": "sha256:<config digest>", "source": "opensuse-leap-image",
  "repos": [...],
  "tar_manifest": "<manifest.json content>", "tar_mtime": 1718000000,
  "tar_blobids": ["sha256:…", "…"], "tar_blobcompression": ["gzip", null],
  "tar_sha256sum": "…", "tar_md5sum": "…",
  "layer_compression": ["gzip"], "config_mimetype": "application/vnd.oci.image.config.v1+json"
}
```
`tar_*` fields describe a *normalized* container whose layers were split off into the shared
blob store (`BSRepServer::Containertar::normalize_container`): the tar itself may not exist in
`:repo`; instead `_blob.sha256:<id>` hardlinks do, and the publisher reconstructs the tar
(`BSPublisher::Containerinfo::construct_container_tar`) only when it has to publish it as a file.
The publisher adds `arch`, `_id` (mtime/size/inode), `_origin` (package), `_p` (published path),
`publishfile` (path of the tar it may upload), `blobdir`, and the associated files
(`readcontainermetafiles`, `bs_publish:2218-2258`): `slsa_provenance_file`, `spdx_file`,
`cyclonedx_file`, `intoto_files[]`, `container_packages` (`.packages`: `name|epoch|version|release|arch|disturl|license` lines),
`container_basepackages`, `container_report`, all hardlinked into the blobdir.

Helm charts: `<chart>.helminfo` (`BSPublisher::Helm::readhelminfo`) with `name, version,
release, tags, config_json (Chart.yaml as JSON), config_yaml, chart_sha256, disturl,
buildtime`; treated as a container of `type => 'helm'`, pushed as an OCI artifact
(`BSContar::container_from_helm`) and also kept in the published tree with `index.yaml`.

### 3.2 Registry configuration
`$BSConfig::container_registries` = `{ name => { server, pushserver, user, password,
repository_base, repository_delimiter, mapper (code), notary, notary_gunprefix,
notary_pubkey_args, cosign (bool|code), cosign_attestation, cosign_nocheck, cosignbundle,
rekorserver, nodelete, repostate_unneeded, registrystate (dir), manifestinfos (dir),
container_tag_deletion_safeguard (1|2|code), use_head_for_list } }`
(`BSConfig.pm.template:288-333`, used throughout `Container.pm`). `pushserver => 'local:'`
selects the OBS-local registry. `$BSConfig::publish_containers` = ordered list of
`regex => [registry names]`; first match wins (`Container.pm:50-77`).

### 3.3 Tag computation
`default_container_mapper` (`Container.pm:113-125`): repository = lc(`<repository_base><prj>/<repo>`)
with `:` replaced by `repository_delimiter` (default `/`), tags = `containerinfo->{tags}`
each prefixed with that repository: e.g. project `openSUSE:Containers:Leap:15.6`, repo
`images`, tag `opensuse/leap:15.6` → `opensuse/containers/leap/15.6/images/opensuse/leap:15.6`.
A site `mapper` sub can return arbitrary `repo[:tag]` strings (the template shows one that
strips the project prefix for official projects). Tags without `:` get `:latest`.
`upload_all_containers` (`:162-300`) groups uploads as `uploads{reponame}{tag}{platformstr} = $p`;
tag conflicts between containers are resolved by `cmp_containerinfo` (highest version/release
wins, `:154-160`). `platformstr` = `BSContar::make_platformstr(goarch, govariant, goos)` (`any`
for helm/artifacthub).

### 3.4 Remote registries: `do_remote_uploads` + `bs_regpush`
Per repository (`Container.pm:1046-1147`):
1. Optional *tag deletion safeguard* (`:962-1009`): list the registry first (with
   `--listidx-no-info` so manifest-list platforms are known) and refuse (mode 1 → critical error)
   or warn (mode 2) if any existing non-cosign tag/platform would be removed.
2. Cosign set-up if the registry has `cosign` and we have a pubkey: cookie, `gun =
   <notary_gunprefix|server>/<repository>`, attestation on by default, rekor server.
3. `query_repostate` (`:468-540`): `bs_regpush --dest-creds - -l --cosign [--no-cosign-info]
   [--listidx-no-info] [--missingok] [-F tagsfile] [--old-listfile <registrystate>/<repo>/:oldlist]
   [--use-head-for-list] <server> <repository>` → lines `<tag> <digest> <type> [extra]` and
   `/<platform> <digest>` sub-lines; parsed into `repostate{tag}=digest` and
   `repostate{sha256-…sig}="layers=N,cosigncookie=…"`. The old list file enables
   `If-None-Match` revalidation to cut registry traffic.
4. Group tags by identical container set and call `upload_to_registry` (`:616-770`): first
   `compare_to_repostate` computes locally what the manifest digests *would* be
   (`create_container_dist_info` builds the exact distribution manifest from the tar:
   normalized layers → `BSContar::create_dist_manifest_data`; multi-arch → manifest list) and
   if every expected tag/digest and every expected cosign tag is already there, and the
   `manifestinfos` cache exists, nothing is pushed. Otherwise:
   ```
   bs_regpush --dest-creds - [-t tag]... [-m] [--oci] [-B blobdir]
       [--cosign --cosigncookie C [--cosignbundle] -p pubkeyfile -G gun [--project P] [-P keyfile] [--signflavor F]
        [--rekor URL [--force-rekor-upload]] [--slsaprovenance] [--sbom]]
       -F <containerdigests out> [--write-info <json out>] <server> <repository> <tar|.containerinfo|.helminfo|artifacthub:…>...
   ```
   Credentials `user:password\n` go in on stdin (`--dest-creds -`), never on the command line.
5. `bs_regpush` upload flow (`bs_regpush:1228-1472`): per tar → read manifest/config
   (`BSContar::get_manifest/get_config`), `blob_upload` config (HEAD `/v2/<repo>/blobs/<d>`;
   POST `/v2/<repo>/blobs/uploads/` → PUT `<location>?digest=` streaming the layer with
   `send_layer`, honouring a different auth realm for redirected blob stores, `:177-223`),
   `normalize_layer` (recompress to gzip/zstd per OCI/Docker rules) + upload each layer,
   `manifest_upload` (PUT `/v2/<repo>/manifests/<tag>` after `manifest_exists` HEAD/GET check;
   `Content-Type` = docker v2 or OCI manifest). Multi-arch: anonymous per-platform manifests
   then a manifest list / OCI index under the tags. Then cosign: `.sig` tag with the
   simplesigning layer (reused if `reuse_cosign_manifest` finds the same cookie and layer
   count), `.att` tag with one DSSE layer per provenance/SBOM/intoto file (or sigstore bundle
   artifacts with `subject` when `--cosignbundle`), Rekor upload when `--rekor`.
   Output: `-F` digest file lines `<digest> <size> [<tag>]` (anonymous images have no tag) and
   `--write-info` JSON `{images:[{file,imageid,goarch,goos,govariant,distmanifest,layers:[{blobid,blobsize,diffid}]}], tags, distmanifesttype:image|list, distmanifest}`.
   Auth: `BSBearer` (`BSBearer.pm:48-88`) handles `Basic` and `Bearer` challenges
   (token endpoint from `WWW-Authenticate: Bearer realm=…,service=…,scope=…`).
6. Exit codes (`process_regpush_error`, `Container.pm:452-466`; `bs_regpush:95-120`):
   `2` = configuration error (4xx other than 401/404/408/410/417/429) → `CRITICAL_NORETRY`;
   `10..19` = HTTP 429 with a sleep hint encoded as `10+n` → `RETRY_AFTER(<n→seconds>)`;
   anything else → generic retry.
7. `create_manifestinfo` (`:542-575`) writes `<manifestinfos>/<repository>/<distmanifest>` JSON
   (`{project, repository, arch, package, type, disturl, buildtime, version, release, imageid,
   goarch, goos, govariant, distmanifest, layers, packages:[{name,epoch,version,release,
   binaryarch,disturl,license,base,summary}], cosign_cookie, cosign_pubkey}`) — the data the
   repserver serves at `/registry/<gun>/<repo>/manifestinfos/<digest>` for external registries
   (`bs_repserver:4448-4490`) so tooling can map a published image digest back to its OBS origin
   and package list (used by security scanners / the API's image info).
8. `add_notary_upload` collects `<digest> <size> <tag>` lines per GUN; after all repositories
   `upload_to_notary` runs `bs_notar --dest-creds - [--project P] [-P keyfile] -p pubkeyfile -F digestfile <notaryserver> <gun>` (`:810-845`).
9. `delete_obsolete_tags_from_registry` (`:772-807`): `bs_regpush -X -F digestfile [--cosign
   [--cosignbundle]] <server> <repo>` deletes every tag not listed (keeping the matching
   `.sig`/`.att`/bundle tags); skipped when `nodelete`, or when the repostate shows nothing
   obsolete. `delete_tag` (`bs_regpush:659-700`) tries `DELETE /v2/<repo>/manifests/<tag>`
   and falls back to re-uploading a mangled manifest under the tag and deleting that digest
   (registries that only delete by digest).
10. Repositories that existed in the previous `:repoinfo` `container_repositories` but have no
    uploads now get `do_remote_uploads(…, undef)` → full tag deletion + notary GUN delete.

### 3.5 The OBS-local registry: `BSPublisher::Registry::push_containers`
Layout (`Registry.pm:51-57`, `$bsdir/registry/<regrepo>/`):
```
:repos                 (in $bsdir/registry) Storable { regrepo => "prj/repo" }  ownership map
<regrepo>/:info        Storable {project, repository, gun, tags => {tag => {distmanifesttype, distmanifest, images:[imginfo…]}}}
<regrepo>/:blobs/sha256:…        hardlinks (from blob store / written entries)
<regrepo>/:manifests/sha256:…    manifest JSON
<regrepo>/:tags/<tag>            hardlink to the manifest file
<regrepo>/:manifestinfos/sha256:…  per-image JSON (see §3.4 step 7)
<regrepo>/:referrers/<subject digest>  Storable [[digest, manifest_json, artifactType]…]
<regrepo>/:referrerinfo          Storable { referrer digest => subject digest }
<regrepo>/:cosign                Storable {pubkey fp, gun, creator, digests{d=>sigmanifest}, attestations{d=>attmanifest}}
<regrepo>/:sigs                  Storable atomic signatures {digests{d => [[tag, sig]…]}}
<regrepo>/:tuf, :tuf.old         Storable {root, targets, snapshot, timestamp (JSON strings), timestamp_privkey/pubkey, *_expires}
```
Ownership: `ownrepo` (`:59-99`) asks the source server `POST /ownregistryrepo?project&repository&regrepo`
(`bs_srcserver:7772-7780`, `BSSrcServer::Registry::ownrepo`, which keeps the authoritative
`:repos` map on the source server and is also used by `registry_forward` to route `/v2/`
requests to the owning repserver in partitioned setups); if another prp owns the repository
name the push is refused (`cannot push to … owned by …`). `disownrepo` on emptying.
`push_containers` (`:818-986`) mirrors `bs_regpush` but writes files: push blobs (hardlink when
the entry has a `blobfile`, else write), manifests, manifest list, `push_tag`, cosign
signature/attestation manifests (`update_cosign`, with referrers for bundle mode), then GC of
unreferenced tags/manifests/manifestinfos/blobs/referrers (`BSPublisher::Blobstore::blobstore_chk`
unlinks blob-store entries whose link count dropped to 1), `:info` with `CONTAINER_PUBLISHED`
notifications for changed tags, `update_tuf`, `update_sigs`. Whole-repository deletion =
`push_containers` with an empty tag set.

Served by `bs_repserver` (`:4776-4785`, with `BSRegistryServer` reply helpers adding
`Docker-Distribution-Api-Version: registry/2.0` and JSON `{"errors":[{code,message}]}`):
```
GET|HEAD /registry/<regrepo>/manifests/<tag|digest|sha256-…sig|sha256-…att|sha256-…>  (Accept negotiation; manifest list → amd64/linux image for clients that only accept single manifests)
GET|HEAD /registry/<regrepo>/blobs/<digest>        (Range support, Cache-Control 31 days)
GET|HEAD /registry/<regrepo>/tags/list?n&last      (paginated with Link header)
GET|HEAD /registry/<regrepo>/referrers/<digest>[?artifactType]   (OCI referrers API)
GET|HEAD /registry/<regrepo>/signatures/<digest>[@signature-N]   (atomic sigs extension)
GET|HEAD /registry/<regrepo>/manifestinfos/<digest>
GET|HEAD /registry/<regrepo>/info.json
GET|HEAD /registry/_catalog
GET|HEAD /registry/<gunprefix>/<regrepo>/_trust/tuf/<file>       (notary)
```
The web server maps `/v2/` to `/registry/` (the source server's `registry_forward` or an apache
rewrite); the source server also answers the `/v2/` version check.

### 3.6 ArtifactHub and multi-arch readme/symlinks
`publishflags:artifacthub` + `artifacthub:<gun/repo>:<data>` pushes an extra `artifacthub.io`
tag containing an `artifacthub-repo.yml` (`BSContar::container_from_artifacthub`) so ArtifactHub
can claim the repository. After uploads, for every container `<name>.registry.txt` is written
(or, without registries, a `<name>` symlink to the last arch's `<arch>/<name>` so a flat path
exists, `Container.pm:270-292`).

### 3.7 Diffnotify integration
`tagdata_cb`/`regdata_cb` (`Diffnotify.pm:36-62`) record per published container the
`platformstr`, all tags computed and the subset actually used (after conflict resolution), and
per registry/tag the resulting `distmanifest`/`imageid`, feeding the registry section of the
diff notification (§1.10).

---

## 4. The published database and search

### 4.1 What the publisher records
Three logical tables, all keyed by the *external* path form `prp_ext` (`openSUSE:/Factory/standard`,
i.e. every `:` followed by `/`) (`bs_publish:186-262, 265-301, 1712-1823, 1990-2003, 3304-3310, 3466`):

| op (`@db_sync` triplet) | key | value |
|---|---|---|
| `binary` updateindex_rel | `[removed keys]` | `[added keys]` — keys are `<prp_ext>/<arch>/<file>` for `*.rpm`, `*.deb`/`udeb`, `*.pkg.tar.*` only (`updatebinaryindex` derives the package *name* from the filename) |
| `repoinfo` store | `<prp>[<dbgsplit>]` | the publisher `:repoinfo` hash (or undef on delete) |
| `patterninfo` store | `<prp>[<dbgsplit>]` | `{ "<ymp file>" => {type:'ymp', name, summary, description, path} }` (or undef) |
| `pattern` store (legacy) | `<prp_ext>/<name>.ymp` | same pattern record |

They are batched and shipped as a Storable array by `db_sync` (`:218-262`) with
`POST $srcserver/search/published?cmd=updatedb` (`Content-Type: application/octet-stream`,
timeout ≥ 60 s scaled by op count). If the RPC fails the batch is spooled to
`$bsdir/db/published.sync` (locked append in forked children, `$db_sync_append`) and the
runner refuses to fork new children until the spool is drained (`getevent`, `:4001-4011`,
`syncdb`), guaranteeing ordering.

### 4.2 Source-server side (`search_published_updatedb`, `bs_srcserver:5554-5600`; `BSSrcServer/SQLite.pm`)
With `$BSConfig::published_db_sqlite` (default 1) the DB is `$bsdir/db/sqlite/published`:
```sql
CREATE TABLE repoinfo(id INTEGER PRIMARY KEY, path TEXT UNIQUE, json TEXT, project TEXT);
CREATE TABLE binary (repoinfo INTEGER REFERENCES repoinfo(id), name TEXT, path TEXT, package TEXT);
CREATE TABLE pattern(repoinfo INTEGER, path TEXT, package TEXT, json TEXT, name TEXT, summary TEXT, description TEXT, type TEXT);
-- indexes: repoinfo(path), repoinfo(project), binary(name), binary(repoinfo), pattern(name), pattern(repoinfo)
```
`updatedb_repoinfo` (`SQLite.pm:200-262`) upserts the repoinfo row (json = repoinfo minus
`binaryorigins state code starttime endtime publishid`), then diffs the `binary` rows against
`binaryorigins` (so the `binary` triplets from the publisher are ignored in sqlite mode — the
repoinfo's binaryorigins is the source of truth; `binarypath2name` recognises rpm/deb/arch
names). `updatedb_patterninfo` does the same for patterns. Legacy (non-sqlite) mode uses the
`BSDB` file-per-key DB in `$bsdir/db/published/`.

### 4.3 Queries served (`bs_srcserver:8047-8100`, API passes them through: `src/api/config/routes/api.rb:121-123`)
```
GET /search/published/binary/id?match=<xpath>[&limit=1000][&withdownloadurl][&withscmsyncurl]
GET /search/published/binary/name?match=…           → distinct names only
GET /search/published/pattern/id?match=…
GET /search/published/repoinfo/id?match=…[&withdownloadurl]
GET /published?view=collection&name=<bin>&project=<prj>…   (published_collection, the webui "binary search")
GET /published[/<prj>[/<repo>[/<arch>[/<file>[/<sub>]]]]][?view=ymp|fileinfo|status|publishedpath&medium=]
```
`match` is the OBS XPath dialect (`BSXPath`/`BSXPathKeys`) over virtual `<binary>` documents
built by `binary_key_to_data` (`:5826-5885`): attributes `name, version, release, versrel, arch,
type (rpm|deb|arch), project, repository, package, filename, filepath, baseproject,
path/project, path/repository, downloadurl, scmsyncurl`. Only `name`, `project`, `package`
(via `key2package`) are indexed; everything else is a filter over candidate keys, with
`413 refusing to get all keys` protecting against unanchored queries. Results are sorted by
version desc, capped by `limit` with `limited="true"`. The API's only in-tree consumer is
`Backend::Api::Search.published_binaries_for_package` (`@project= and @package=`) and
`Backend::Api::Search.binary` (`/published?view=collection&name=`), used by the webui package
"binaries" views and the `search/published/*` API pass-throughs for osc/users.

`/published/...` paths on the source server are proxied to the owning repserver
(`:6082-6145`; partitioning aware), which lists `$bsdir/repos` (`publisheddir`, decoding the
`:/` directory encoding back to project names, `bs_repserver:3860-3932`), serves files
(`publishedfile`, with `view=fileinfo` → `Build::query` metadata, `view=ymp` → on-the-fly
one-click-install) and `view=status` from the publisher's `:repoinfo`. `view=publishedpath`
(`published_path`, `:6159-6232`) computes the download URL/path for a binary or product
medium from `BSUrlmapper` + the repotype (`slepool` naming), used by the API's
`Backend::Api::BuildResults::Binaries.publishedpath` to give users a download link.

### 4.4 What a frontend needs from a redesigned published index
* `repoinfo` per prp: searchpath, title, repotype, archs, splitdebug suffix, base project,
  container repositories, publish status (`code/starttime/endtime/details/publishid`).
* binary rows: `(prp, arch, filename, name, version, release, type, package)` with indexes on
  `name`, `package`, `project`; download URL derivation.
* pattern rows for ymps.
* The *collection* query (`name IN (...) AND project IN (...)`) and the per-package query are
  the hot paths; the XPath engine is otherwise unused by the Rails app and can be replaced by
  a plain filter API.

---

## 5. Concurrency, atomicity and failure handling

### 5.1 Locks
* `$reporoot/$prp/.finishedlock` (`flock`): held by each arch scheduler while it rewrites
  `<arch>/:repo` (`PublishRepo.pm:128-139`, 300 s timeout with packages, blocking on wipe),
  by the publisher while it reads all `:repo`s and hardlinks into `$extrep`
  (`bs_publish:2363-2370, 3027`), by the repserver for `unpublish`/`wipepublishedlocked`
  (`bs_repserver:3228, 3247`). Deleted by the scheduler when a prp vanishes
  (`BSSched/Checker.pm:241`).
* `$reporoot/$prp/.pplock`: held by a background `publishprogram` child.
* `$rundir/bs_publish` run lock (single daemon per host); `events/publish/*::inprogress`
  marks per-prp in-flight work so two children never publish the same prp concurrently (the
  dispatcher does the rename before forking's work starts, and `lsevents` hides the twin).
* `$bsdir/registry/:repos` and `$registrydir/<repo>/:tuf`: `BSUtil::lockopen` append locks.
* `$extrepodb.sync`: locked append.
* No lock protects `$extrep` against the web server: readers may observe partially regenerated
  `repodata/` (createrepo writes to `.repodata/` then renames, which helps for rpm-md; Debian
  `Packages`/`Release` and the hardlink churn are not atomic as a set).

### 5.2 Atomic writes
`BSUtil::store/writestr/writexml` write `.tmp` then `rename` (with `fdatasync` unless
`disable_data_sync`); `:repoinfo`, `.archsync`, `.checksums`, `boxes/`, `.ymp`, `.repo`,
`directory.yast`, registry manifests/tags/info all use that. Hardlinks are installed with
`unlink + link` (not atomic rename), and `createrepo --update` uses the `repocache`. There is
**no whole-repository atomic swap**: a client fetching during a publish can see new
`repodata` referring to a not-yet-linked rpm or vice versa. Mirrors get the two-pass rsync
(binaries first, then metadata with deletes) to minimise that window.

### 5.3 Idempotence / change detection
Everything is driven by `(mtime, size, inode)` ids and content hashes: the scheduler only
re-links changed files; the publisher's `nothing changed` exit avoids metadata regeneration;
container uploads compare locally computed digests to the registry state; TUF/cosign/sigs
files are compared with `BSUtil::identical` before rewrite. Re-running a publish after a
crash is safe (the `::inprogress` rename-back, `clear_repoinfo_state` dropping `state`).

### 5.4 Failure modes and retries
See §1.1: exceptions → exponential back-off (1..32 min) with `NORETRY`/`CRITICAL`/`RETRY_AFTER`
prefixes; `publishedhook` failure retries, `unpublishedhook` failure only warns; rsync failure
dies (retry) after possibly having pushed binaries; signd failures die; registry 429 → timed
retry, 4xx config errors → stuck with `details`. The signer retries every 3 min, marks
`failed` on `fatal:` and triggers rebuilds for corrupt rpms. Published-DB RPC failure is
spooled, never fatal.

### 5.5 Shared-filesystem assumptions
The design assumes one POSIX filesystem (hardlinks!) shared by: scheduler(s) (`build/`),
repserver (`build/`, `registry/`, `blobs/`, `slsa/`, serves `/published` and `/registry`
from disk), publisher (`build/` → `repos/`, `registry/`, `blobs/`, `upload/`), signer
(`jobs/`), the web server (`repos/` as DocumentRoot with `Options Indexes FollowSymLinks`,
`dist/obs-apache24.conf:30-37`, port 82 = "download.opensuse.org style"), and optionally
`publishredirect` targets. Partitioning (`$BSConfig::partitioning`) replicates the whole set
per partition with the source server routing by project. The published DB is the one piece
that already crosses hosts (HTTP to the source server). Stage servers (rsync) and external
registries are the only "remote" sinks; `download.opensuse.org` is populated by the rsync
stage push + trigger file, not by serving `repos/` directly from the backend host.

---

## 6. Implications for a distributed redesign

### 6.1 Proposed decomposition

1. **Publish Orchestrator** (one service, horizontally scalable by prp partition):
   consumes `PublishRequested{prp}` events (today: event files), serialises per prp (the
   `.finishedlock` + `::inprogress` semantics become a per-prp lease/queue), materialises the
   *publish plan*: the input set (what the scheduler calls `:repo`, i.e. a list of
   `{arch, sourceobject, publishedpath, origin package, kind}`), the config
   (repotype/patterntype/publishflags/signflavor), and the previous repoinfo. Emits the
   state hash and decides "unchanged" early. Owns retry/back-off, status (`building/
   succeeded/failed/details/publishid`) and the notification fan-out
   (`REPO_PUBLISH_STATE`, `REPO_PUBLISHED`, `PACKTRACK`, diffnotify, sourcepublish).
2. **Scheduler → publisher contract** becomes a *manifest*, not a directory: the scheduler
   publishes an immutable "repo snapshot" (`binaryorigins` + object references by content
   hash) to object storage/metadata DB instead of hardlinking into `:repo`. Delta-rpm
   generation stays a build job but keyed by content hashes rather than `(mtime,size,inode)`.
3. **Repo Metadata Generators**, one worker type per repotype, each a pure function
   `(snapshot, options, signer) → set of metadata objects`: rpm-md (createrepo_c + modifyrepo
   or a Go port), susetags, debian, arch, apk, helm, vagrant, staticlinks, zyppservice,
   checksumsfile, patterns/ymp/comps, appdata, deltainfo/prestodelta, modulemd, SBOM. They
   only need read access to the binaries (for `createrepo`, `dpkg-scanpackages`,
   `Build::query`) — feasible via a local cache of objects pulled from object storage.
4. **Object-storage-backed published tree with versioned layouts**: write each publish as a
   new immutable prefix (`<prp>/<publishid>/…`, content-addressed objects deduplicated across
   publishes, mirroring today's hardlink dedup), then atomically flip a pointer
   (`<prp>/current → publishid`) that the HTTP front (CDN origin) resolves. This gives the
   atomic swap the current design lacks, trivially supports `nocontentchange`/rollback, and
   makes "mirror sync" = CDN invalidation or a list-diff rsync from the versioned prefix. The
   `staticlinks` symlinks become redirect rules or duplicate keys; `directory.yast`/
   index listings become generated objects. Keep the on-disk layout identical
   (`<prj with :/>/<repo>/<arch>/…`, `repodata/`, `Packages`, `Release`, `iso/`, `repo/`,
   `boxes/`) because zypper/apt/pacman/apk clients and `BSUrlmapper` URLs depend on it.
5. **Sign Service** with a KMS/HSM abstraction: the key store becomes
   `{project → key ref}` with inheritance (`findsignkey` walk), key refs pointing either at a
   wrapped private key (today's `_signkey` blob) or an HSM/KMS key id, plus flavors/types.
   API as in §2.5: generate/extend/pubkey/cert/sign-digest (PGP v3/v4 detached, clearsign,
   raw PKCS#1, CMS, Authenticode, rpm header). Client-side format splicing (rpm, iso
   placeholders, apk, appx, AppImage, DSSE, cosign, TUF, SSHSIG) can be Go libraries; the
   service should be reachable over mTLS/gRPC from signer workers, generators and the registry
   pusher, replacing the `sign` setuid binary + signd TCP protocol. Keep an adapter that
   speaks to a legacy `signd` for migration.
6. **Signer workers** (job result signing) stay a separate stage between "built" and
   "finished"; they need the job result objects and the Sign Service, and must write back
   updated bininfo/checksums. Followup builds (rsasign/mkosi) remain a scheduler concept.
7. **Registry Pusher**: a service that takes `(containerinfo + blobs + attestations, registry
   config, key ref)` and converges a remote repository (list → diff → push → cosign/rekor →
   delete obsolete → notary), exactly `do_remote_uploads`. `bs_regpush` is already almost that
   as a CLI; in Go use `go-containerregistry`/`oras` for the registry client,
   `sigstore/cosign` libraries for bundle formats, and the Sign Service as a `crypto.Signer`.
   The **local registry** becomes a thin OCI registry backed by the same object store
   (blobs are already content addressed), with the extras (manifestinfos, atomic signatures,
   TUF) served from its metadata DB; `ownregistryrepo` becomes a uniqueness constraint.
8. **Published Index** (replacing `db/sqlite/published` + XPath): a relational/elastic
   index populated from the orchestrator's final repoinfo; expose `search binaries by
   name/project/package`, `repoinfo`, `patterns`, download URL resolution. Keep an XPath
   compatibility shim for `/search/published/*` API routes if osc compatibility matters.
9. **Mirror/CDN sync**: replace `sync_to_stage` rsync + trigger file with object-storage
   replication/CDN invalidation; keep an rsync exporter for legacy mirrors; Rekor upload of
   repo signatures becomes a generator post-step.
10. **Hooks**: `publishedhook`/`unpublishedhook`/`publishprogram`/`createrepo_rpmmd_hook`/
    `publisher_compile_content_hook` are site-specific scripts with the `(prp, extrep,
    changed files)` contract — model them as webhook/event subscribers receiving the publish
    manifest.

### 6.2 External tooling vs. Go-native

| Function | Today | Realistic in Go? |
|---|---|---|
| rpm-md metadata (`primary/filelists/other`, `--update`, zchunk, modulemd) | `createrepo_c`, `modifyrepo_c`, `writemodulemd` | Keep `createrepo_c` (mature, zchunk, sqlite-free). A Go rpm header parser exists (`sassoftware/go-rpmutils`), but reproducing createrepo_c output (checksums of compressed files, unique md filenames, changelog limits) is sizeable; shell out, run inside the generator container |
| repoview, rezip_repo_rsyncable, generate_sbom | external | shell out (optional features) |
| susetags (`create_package_descr`) | external (obs-build) | Go-native is feasible but low value; shell out |
| Debian `Packages`/`Sources`/`Release` | `dpkg-scanpackages`, `dpkg-scansources`, gzip | Go-native (deb control parsing is simple; `pault.ag/go/debian`); Release/InRelease hashing trivial |
| Arch db/files tarballs | `bs_mkarchrepo` (Perl) | Go-native (tar + `.PKGINFO` parsing) |
| APKINDEX + signature | `bs_mkapkrepo` + `Build::Apk` (Perl) | Go-native (tar/gzip + RSA; v3 not implemented today either) |
| Helm index.yaml | Perl + YAML::XS | Go-native (`helm` `repo.IndexFile`) |
| vagrant, staticlinks, checksumsfile, zyppservice, ymp, comps, appdata merge, deltainfo xml | Perl | Go-native |
| deltarpm creation | worker job (`makedeltarpm`) | keep external |
| PGP signing/parsing | `sign` + `BSPGP` | Go: `ProtonMail/go-crypto/openpgp` for packets; signatures still come from the Sign Service (HSM); rpm signing header splice in Go (`go-rpmutils` has it) |
| ISO placeholder patching, apk/appx/AppImage signatures | Perl + `sign --appx` | ISO/apk/AppImage: Go-native byte patching; Authenticode (appx): shell out to `osslsigncode`/sign tool or implement PKCS#7 with the Sign Service providing the raw RSA op |
| CMS/PKCS#7 | `sign --cmssign` | Go `go.mozilla.org/pkcs7` + external signer |
| X.509 cert generation, TUF JSON, DSSE, cosign payloads, Rekor client | Perl (`BSX509/BSTUF/BSConSign/BSRekor`) | Go-native: `crypto/x509`, `theupdateframework/go-tuf`, `sigstore/sigstore` + `cosign` libs, `rekor` client |
| Registry v2 client / local registry | `bs_regpush` / `BSPublisher::Registry` + `bs_repserver` | Go-native: `go-containerregistry`, `distribution/distribution` storage drivers |
| Notary (v1) | `bs_notar` | legacy; keep as sidecar or drop (cosign supersedes) |
| rsync to stage servers | `rsync`/`rsync-ssl` | keep external or replace with object replication |
| Source publishing | Perl + rsync | Go-native export + rsync/object copy |
| XPath search | `BSXPath` | replace with SQL/REST; compatibility shim optional |

### 6.3 Behaviours worth preserving explicitly in the new design
* Arch precedence for noarch files (`archorder`, reverse-lexical default) and "same path from
  several archs" semantics.
* `publishid` monotonic per prp (used as `buildid` in notifications and `obsbuildid:` repo tag).
* The `.registry.txt` replacement of container tars and the exact tag mapping (consumers rely
  on `<prj>/<repo>/<name>:<tag>` lowercased paths).
* `nocontentchange`, `createempty`, `archsync`, `splitdebug` (`_debug` twin repos with
  `mainupdateinfo` handling), `packagesubdir`, `slepool/sleimage` renames, `repotag`
  (`obsrepository://` and `obsbuildid:`), `<project>.repo` file contents.
* Signature artefacts and names: `repomd.xml.asc/.key`, `Release.gpg`/`InRelease`/`Release.key`,
  `.sig` for arch, `.asc` for checksums, `SHA256SUMS.asc/.gpg`, `index.key`, cosign
  `sha256-<d>.sig/.att` tags with the OBS cookie annotation, atomic-signature extension,
  manifestinfos.
* The published-DB keys (`prp_ext` with `:/` encoding) and `/published` URL scheme.
* Retry semantics and the stuck-event-with-details signalling for configuration errors.
