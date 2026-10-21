# OBS Source Server (`bs_srcserver`) — Engineering Analysis for a Go Re-architecture

All paths are relative to `src/backend/` unless stated otherwise. Line numbers refer to the
checked-out tree at commit `db93ae665b`. `$bsdir` is `$BSConfig::bsdir` (`/srv/obs` in
production). Variable naming conventions used below (from `DESIGN`): `$projid` project name,
`$packid` package name, `$prp` = `project/repository`, `$prpa` = `project/repository/arch`.

The source server is a forking HTTP server (`BSServer.pm`) with an attached single-process
event-driven "AJAX" helper (`BSServerEvents.pm` + `BSWatcher.pm`) reachable over a Unix socket,
to which long-running or blocking requests are handed off. It is **the only backend component the
API talks to for sources**, and it is also the **proxy/front door for all `/build`, `/published`,
`/registry`, worker and search traffic**, forwarding those to the repository server(s) chosen by
the partition map (`BSSrcServer/Partition.pm`).

---

## 0. Component map

| File | Role |
|---|---|
| `bs_srcserver` (8255 l.) | route table (l.7970–8206), all handlers, main() (l.8210–8255) |
| `BSSrcrep.pm` | content-addressed file store + tree (MD5SUMS) store; link/service markers; obscpio/deltastore access |
| `BSRevision.pm` | `.rev`/`.mrev` revision log access, project/package meta read, delete/undelete |
| `BSFileDB.pm` | the pipe-separated append-only "file database" used for `.rev`, `.mrev`, `lastevents`, `lastnotifications` |
| `BSSrcServer/Link.pm` | `_link` expansion (`handlelinks`, `applylink`), `keeplink`, `integratelink`, branch 3-way merge |
| `BSSrcServer/Projlink.pm` | project-level links (`<link project=.../>` in project meta), frozen links, vrev derivation |
| `BSSrcServer/Service.pm` | source services: servicemarks, service runs, obs-scm bridge, dispatch events |
| `BSSrcServer/Scmsync.pm`, `ScmsyncDB.pm` | git-backed ("scmsync") projects/packages and their sqlite index |
| `BSSrcServer/Remote.pm` | interconnect: remote project resolution, remote getrev, binary caching |
| `BSSrcServer/Partition.pm` | project → repository-server mapping (sharding of the *binary* side) |
| `BSSrcServer/Access.pm` | `access`/`sourceaccess` flag evaluation, role merge |
| `BSSrcServer/Multibuild.pm` | `_multibuild` cache (`:multibuild` Storable file per project) |
| `BSSrcServer/Product.pm` | `_product` → `_product:*` expansion via `bs_productconvert` |
| `BSSrcServer/Signkey.pm` | wraps external `sign` binary: create/extend key, X.509 cert |
| `BSSrcServer/SQLite.pm`, `BSSQLite.pm`, `BSDB.pm`, `BSDBIndex.pm` | index databases (linkinfo, scmsync, published binary/pattern/repoinfo) |
| `BSSrcServer/LinkinfoDB.pm` | who-links-to-whom reverse index |
| `BSSrcServer/Srcdiff.pm`, `Blame.pm` | diff and blame implementation |
| `BSSrcServer/Registry.pm`, `Modulemd.pm`, `Notify.pm` | container registry owner map, modulemd parsing, notification plugins |
| `bs_deltastore` | background daemon: converts `*.obscpio` into delta-compressed form |
| `bs_sourcepublish` | background daemon: rsyncs published sources to an external server |
| `bs_mergechanges` | helper binary: 3-way merge of `.changes` files (used by branch link merge) |
| `bs_servicedispatch` | background daemon consuming `events/servicedispatch/*` (lives outside the srcserver process but shares its modules) |
| `bs_notifyforward`, `BSRedisnotify.pm`, `BSNotify.pm` | notification forwarding from repserver/scheduler/publisher into srcserver |

---

## 1. On-disk data model under `$bsdir`

### 1.1 Directory map (what the srcserver touches)

```
$bsdir/
  projects/                       # project + package meta, revision logs  (BSRevision.pm:34)
    _repoid                       # random 9-digit id of this data store (bs_srcserver:8247-8252)
    <projid>.xml                  # current project meta (BSXML::proj)       [authoritative, mirrored into mrev]
    <projid>.conf                 # current project config (prjconf text)   [authoritative, mirrored into _project.rev]
    <projid>.pkg/                 # per-project directory
      <packid>.xml                # current package meta (BSXML::pack)
      <packid>.rev                # source revision log (BSFileDB)          [authoritative]
      <packid>.mrev               # meta revision log (BSFileDB)            [authoritative]
      <packid>.rev.del / .mrev.del# logs of deleted packages (for undelete)
      <packid>.upload-MD5SUMS     # the "upload" pseudo-revision tree
      pattern-MD5SUMS             # legacy "_pattern" pseudo-package tree
      _project.rev / _project.mrev# project-level source (config, _service, ...) and meta revisions
      _pubkey, _signkey           # extracted copies of the signing key pair (meta rev files _pubkey/_signkey)
      :multibuild                 # Storable cache of parsed _multibuild per package (Multibuild.pm:53,62)
    _deleted/<projid>.pkg/        # revision logs of deleted projects (delproject, bs_srcserver:2314-2335)
  sources/                        # content-addressed blob store ("srcrep") (BSSrcrep.pm:35)
    <packid>/<md5>-<filename>     # blobs, keyed by package name (NOT project)   [authoritative]
    <packid>/<srcmd5>-MD5SUMS     # trees when $nosharedtrees==0 (or legacy for ==2)
    <packid>/<srcmd5>-_linkerror  # link expansion error log (BSSrcrep.pm:424-444)
    <packid>/deltastore           # delta store for *.obscpio of this package (bs_deltastore:53)
    :upload/                      # scratch dir for in-flight uploads/tmp files (BSSrcrep.pm:40)
  trees/<projid>/<packid>/<srcmd5>-MD5SUMS      # trees when $nosharedtrees (BSSrcrep.pm:36,282)
  trees/<projid>/<packid>/<servicemark>-_serviceerror  # service state marker (BSSrcrep.pm:498-588)
  events/
    lastevents                    # BSFileDB ring log for interconnect/long-poll (bs_srcserver:6745-6768)
    lastnotifications             # BSFileDB log consumed by API (bs_srcserver:165-181)
    undelivered/<type>:<proj>[::pkg]   # queued repserver notifications after RPC failure (bs_srcserver:334-375)
    deltastore/<evname>, .ping    # events for bs_deltastore (BSSrcrep.pm:252-260)
    servicedispatch/<evname>, .ping # events for bs_servicedispatch (Service.pm:326-351)
    sourcepublish/<evname>, .ping # events for bs_sourcepublish (bs_srcserver:7473-7525)
    redis/queue, .ping            # BSFileQueue for bs_redis (bs_srcserver:7398-7404; ScmsyncDB.pm:36-43)
    service/<proj>::<pkg>         # old-style service lock files (Service.pm:516-520)
    <arch>/...                    # NOT written by srcserver; written by bs_repserver::forwardevent (bs_repserver:2924-2988)
  db/
    source/                       # BSDB linkinfo index (pre-sqlite) (LinkinfoDB.pm:33)
    published/                    # BSDB binary/pattern/repoinfo index (pre-sqlite) (bs_srcserver:145)
    sqlite/source                 # sqlite: tables linkinfo, scmsync (SQLite.pm:32,106-149)
    sqlite/published              # sqlite: tables repoinfo, binary, pattern (SQLite.pm:53-104)
    sqlite/publishedsource        # bs_sourcepublish's own table (bs_sourcepublish:54-72)
    request/                      # unused by current code paths (bs_srcserver:144)
  diffcache/<xx>/<cacheid>[.run]  # sourcediff result cache (bs_srcserver:141,3298-3331)
  remotecache/<xx>/<sha>, lock    # cached binaries from remote OBS instances (Remote.pm:34,800-852)
  registry/:repos                 # Storable map regrepo → prp owner (Registry.pm:25)
  notificationpayload/<time>.<md5># JSON payloads referenced from lastnotifications (bs_srcserver:148,165-177)
  requests/, requests.old/        # legacy request XML (only /request/_lastid, /request/$id) (bs_srcserver:138-139)
  configuration.xml               # copy of API configuration (bs_srcserver:7260-7290)
  issuetrackers.xml               # issue tracker definitions (bs_srcserver:7294-7308)
  run/bs_srcserver.ajax[N]        # Unix socket for AJAX helper(s) (bs_srcserver:155)
  run/*.lock,*.status,*.exit,*.restart # daemon control files (BSStdServer.pm:159-201)
```

### 1.2 `projects/` — meta, config and revision logs

* `projects/<projid>.xml` is the current project meta (`BSXML::proj`, BSXML.pm:137-161). It is
  rewritten atomically whenever a new **meta revision** is added (`addrev_replace_common`,
  BSRevision.pm:226-306: the new `.xml` is `rename()`d into place *after* the `.mrev` line is
  appended, inside the `.mrev` flock).
* `projects/<projid>.conf` is the current project build config (free text). Same mechanism via
  `_project.rev` with file `_config` (`putprojectconfig`, bs_srcserver:2846-2861).
* `projects/<projid>.pkg/<packid>.xml` is current package meta (`BSXML::pack`, BSXML.pm:163-181).
* Existence of `<projid>.xml` *is* project existence (`lsprojects_local`, BSRevision.pm:473-485);
  existence of `<packid>.xml` *is* package existence (`lspackages_local`, BSRevision.pm:487-505;
  `getrev_local` returns undef without it, BSRevision.pm:82).
* `_project` is a pseudo-package holding project-level *source* files (`_config`, `_service`,
  `_frozenlinks`?no—that one is meta) and project-level *meta* files (`_meta`, `_pubkey`,
  `_signkey`, `_sslcert`, `_attribute`, `_frozenlinks`).

### 1.3 `sources/` — the content-addressed blob store (BSSrcrep.pm)

* Path: `sources/<packid>/<md5>-<filename>` (BSSrcrep.pm:59,88). **Keyed by package name only,
  not project.** Two packages with the same name in different projects share blobs (and hard-links
  are used to "copy" between package dirs: `copyfiles`, BSSrcrep.pm:173-196).
* `addfile` (BSSrcrep.pm:133-168): compute md5 if not given; `rename(tmp, dst)` (atomic); if dst
  already exists, compare SHA-256 of both and `diecritical` on mismatch — i.e. an md5 collision
  with different content is treated as an attack/corruption, server-fatal.
* `copyonefile`/`copyfiles` use `link()` and fall back to content comparison (`identicalfile`) if
  the inode differs (BSSrcrep.pm:185-194, 239-248).
* `.obscpio` files (service-generated source archives) are special: stored possibly in delta form,
  read through `BSSolv::obscpioopen(file, "<pkgdir>/deltastore", fd, uploaddir)` and expanded via
  `BSSolv::expandobscpio` (BSSrcrep.pm:58,65,204). `addfile` refuses to accept an already-delta
  obscpio (BSSrcrep.pm:136) and emits a `deltastore` event for `bs_deltastore` to compress it later
  (BSSrcrep.pm:155, 252-260). `bs_deltastore` (l.49-92) hard-links the blob to a tmp name, runs
  `BSSolv::makeobscpio(in, store, out)`, re-expands and verifies the md5 (l.72-84), then
  `rename()`s the delta form over the original. **So a blob's bytes-on-disk may change while its
  logical content (and md5) stays identical**; all readers must go through `fileopen`.
* `_serviceerror` is the one "file" whose blob lives in the trees dir instead (BSSrcrep.pm:83-87).
* `sources/:upload/` is the scratch area; tmp names are `$$`-based (`$uploaddir/$$`,
  `addmeta$$`, `applylink$$`, `srcdiff$$`, `runservice$$`, ...). Because of this, **a single
  request process must never run two of those concurrently**, and crashes leave litter.

Example: a package `foo` with `foo.spec` (md5 `1a2b…`) and `foo-1.0.tar.gz` (md5 `c3d4…`) has
```
sources/foo/1a2b...-foo.spec
sources/foo/c3d4...-foo-1.0.tar.gz
```

### 1.4 Trees (`MD5SUMS` files) and the `srcmd5`

A **tree** is the sorted list of `(md5, filename)` pairs for one revision. Its identifier
`srcmd5` is the md5 of its canonical serialization (`calcsrcmd5`/`addmeta`, BSSrcrep.pm:311-332):

```
meta = "" ; for f in sort(keys files): meta .= "<md5>  <name>\n"     # two spaces
srcmd5 = md5_hex(meta)
```
and the file written is exactly `meta` (so the tree file is self-describing and `srcmd5 ==
md5(file content)`). Example `trees/home:u/foo/9f…-MD5SUMS`:
```
c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8  foo-1.0.tar.gz
1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d  foo.spec
```
The empty tree has the well-known `srcmd5 = d41d8cd98f00b204e9800998ecf8427e` (md5 of "",
`$emptysrcmd5`, BSSrcrep.pm:42) and is never written (BSSrcrep.pm:270-272).

Tree location depends on `$BSConfig::nosharedtrees` (BSConfig.pm.template:123-128):
* `0`: `sources/<packid>/<srcmd5>-MD5SUMS` — shared across projects (older installs).
* `1`: `trees/<projid>/<packid>/<srcmd5>-MD5SUMS` — per project.
* `2` (package default): write to `trees/…`, but on read fall back to `sources/<packid>/…`
  (BSSrcrep.pm:283-285, 338-341, 355-358, 376).
Per-project trees exist **for access control**: a tree in `trees/A/pkg/` proves that project `A`
legitimately obtained that revision (e.g. via link expansion); `lsfiles` dies otherwise
(BSSrcrep.pm:289). Project copies/links therefore call `copytree` (BSSrcrep.pm:362-387) to
"import" trees (and hard-link blobs) into the destination project before using them
(Projlink.pm:98, bs_srcserver:3854, 4193, 4321).

**Pseudo-trees inside MD5SUMS**: tree files written by link expansion or service runs carry
extra pseudo-entries that are *not* part of the srcmd5 calculation of the plain tree but *are*
part of the stored file (so these trees are stored under a "wrong", derived md5):

| pseudo-name | meaning | written by |
|---|---|---|
| `/LINK` | srcmd5 of the (expanded) link target tree | `addmeta_link` BSSrcrep.pm:409-422 |
| `/LOCAL` | srcmd5 of the unexpanded local tree (the one with `_link`) | same |
| `/SERVICE` | servicemark: id of the pending/finished service run for this commit | `addrev` bs_srcserver:757 |
| `/LSERVICE` | srcmd5 of the tree the service was run on | `addmeta_service` BSSrcrep.pm:452-469 |
| `/LXSERVICE` | expanded-link srcmd5 the service was run on (links+services) | same |

`lsfiles` strips them and returns them via the `$linkinfo` out-param as `lsrcmd5`, `srcmd5`,
`xservicemd5`, `lservicemd5`, `lxservicemd5` (BSSrcrep.pm:296-308). **These names are how the
server distinguishes an "expanded" tree from a "committed" tree** — see §2.

Trees are immutable once written (`addmeta` only writes if absent, BSSrcrep.pm:326-330;
`addmeta_service` verifies identical content if present, BSSrcrep.pm:459-463).

### 1.5 `.rev` / `.mrev` revision logs (BSFileDB.pm)

Layout `$srcrevlay = [rev vrev srcmd5 version time user comment requestid]` (BSRevision.pm:38).
One record per line, fields joined by `|`, with `%XX` escaping of control chars, `%`, `|`, `=`,
and bytes 0x7f–0x9f (`encode_field`, BSFileDB.pm:54-70). Example `projects/home:u.pkg/foo.rev`:
```
1|1|9f1e4c…|1.0|1700000000|user1|initial commit|
2|2|a07b22…|1.0|1700003600|user1|fix build|12345
3|1|b11c00…|1.1|1700007200|buildservice-autocommit|auto commit by copy to link target|
```
* `rev` is assigned by `fdb_add_i` = last record's first field + 1, computed under `flock` by
  reading the tail of the file (BSFileDB.pm:262-300). The file is append-only (`+>>`).
* `vrev` is assigned by `fdb_add_i2` (BSFileDB.pm:304-334): find the **last record with the same
  `version`**, take its `vrev`, increment the trailing integer, and use `max(that, requested)`.
  This is the "version-release counter": vrev restarts at 1 for a new version and increases for
  every commit of the same version. A vrev can have a non-numeric prefix (`"12.1"`,
  `"<n>.1"` extended forms from `vrevmode=extend`, `makeolder`) — only the trailing digits are
  incremented.
* `fdb_getlast` reads the tail in 1k chunks; `fdb_getmatch` searches the last 4k first then the
  whole file (BSFileDB.pm:72-165) — i.e. the design assumes recent revisions are hot.
* The `.mrev` log has the same layout; `version`/`vrev` stay empty, `srcmd5` is the tree of
  `{_meta[, _pubkey, _signkey, _sslcert, _attribute, _frozenlinks]}` stored in `sources/<packid>/`
  (for `_project`: `sources/_project/`), BSRevision.pm:226-306.
* `events/lastevents` uses layout `[number time type project package repository arch]`
  (bs_srcserver:151); `events/lastnotifications` uses `[number type time data[]]` where the
  trailing `[]` collects all remaining fields into an array (`decode_line`, BSFileDB.pm:41-52).

### 1.6 Link-error and service-state markers

* `sources/<packid>/<srcmd5>-_linkerror`: hard link to the `.log` of a failed `applylink`
  (BSSrcrep.pm:424-430). `lsfiles` returns `{_linkerror => srcmd5}` if the tree is missing but
  the error file exists (BSSrcrep.pm:287), so an expanded-but-broken link has a stable id.
* `trees/<proj>/<pkg>/<servicemark>-_serviceerror`: either `"service in progress\n"` (with an
  optional `key=value` block for obs-scm runs, BSSrcrep.pm:574-588) or the error text. It doubles
  as the **lock** that serializes service runs (`addmeta_serialize_servicerun`, BSSrcrep.pm:498-512
  uses `flock` on it). Deleted by `addmeta_service` on success (BSSrcrep.pm:468).

### 1.7 `events/`

The srcserver *writes* only to: `lastevents`, `lastnotifications`, `undelivered/`,
`deltastore/`, `servicedispatch/`, `sourcepublish/`, `redis/`, `service/` (old-style). Per-arch
scheduler event dirs (`events/<arch>/`) are written by **bs_repserver** when it receives the
srcserver's `POST /event` (bs_repserver:4733, 2924-2988). Event files are XML
(`BSXML::event`, BSXML.pm:771-794), written atomically (`writexml(".name.$$", "name")`), and the
consumer is woken by a write to the dir's `.ping` FIFO (`BSUtil::ping`, BSUtil.pm:615-622). The
generic consumer loop is `BSRunner::run` (BSRunner.pm:61-140) + `BSStdRunner::dispatch`
(BSStdRunner.pm:71-130): list dir, skip `::inprogress`, rename to `::inprogress` while running,
unlink on success, rename back (or rewrite with `due=` for retry, `setdue`,
BSStdRunner.pm:158-168) on failure.

### 1.8 Index databases (`db/`) — what is indexed and why

Two generations coexist, selected by `$BSConfig::source_db_sqlite` / `published_db_sqlite`
(default `1`, BSConfig.pm.template:133-135).

**BSDB/BSDBIndex** (legacy): a filesystem "relational" store. Each (table, key) is a Storable
array file at `db/<name>/<md5[0:2]>/<md5[2:]>` where the file name is md5 of `"table\0key"`
(BSDBIndex.pm:36-53). `modify()` locks all touched files (sorted, to avoid deadlock),
read-modify-writes them with rename, and propagates existence up one level ("next level")
(BSDBIndex.pm:77-220). BSDB builds inverted indexes by flattening values into `path\0value`
relations (`torel`, BSDB.pm:41-68).

**SQLite** (`db/sqlite/source`, `db/sqlite/published`; SQLite.pm):
```
linkinfo(sourceproject, sourcepackage, project, package, rev)  UNIQUE(sourceproject,sourcepackage)
scmsync(project, package, scmsync_repo, scmsync_branch, scmsync_trackingbranch) UNIQUE(project,package)
repoinfo(id, path, json, project) UNIQUE(path)        -- path = prp with ':' → ':/' (prp_ext)
binary(repoinfo, name, path, package)
pattern(repoinfo, path, package, json, name, summary, description, type)
```
(SQLite.pm:106-149, 53-104). Indexed and why:
* **linkinfo** — "which packages link to X" (`getlinkers`, `getlocallinks`), maintained from
  every commit (`updatelinkinfodb`, BSRevision.pm:318-334 reads `_link` of the new revision) and
  cleared on delete. Used by `/search/package` fast paths and `locallink=1` result queries
  (bs_srcserver:5492-5516, 4507-4509). Pure derived data: can be rebuilt by scanning all
  latest revisions.
* **scmsync** — reverse map git-URL(+branch) → packages so a webhook can trigger syncs
  (`POST /source?cmd=triggerscmsync`, bs_srcserver:400-430; `/build/_result?scmrepository=`,
  bs_srcserver:4601-4649). Derived from package/project meta `<scmsync>`.
* **published repoinfo/binary/pattern** — fed by the *publisher* through `POST
  /search/published cmd=updatedb` (bs_srcserver:5554-5601) and queried by
  `/search/published/*` and `/published view=collection`. Derived from published repos.
* **publishedsource** (bs_sourcepublish) — dedup table of already-rsynced source trees.
* **`BSDB` BSXPath search** (`/search/package`, bs_srcserver:5473-5542) is a generic XPath over
  the package XMLs with `indexfunc` callbacks that enumerate `projects/` directly — an
  O(all packages) scan, deliberately short-circuited for the two queries the API actually issues.

### 1.9 Other derived/cache state

* `projects/<projid>.pkg/:multibuild` — Storable `{packid => parsed _multibuild + _md5}` per
  project (Multibuild.pm:44-66). Updated lazily on `getfilelist`/`sourceinfo`/`getprojpack` of the
  latest revision (bs_srcserver:3073,3103,7057,1672) and pruned by getprojpack (1373). It is
  authoritative in the sense that **the scheduler and `/build` paths only know flavors that are in
  this cache** (`ensure_known_flavor`, bs_srcserver:4983-5001).
* `projects/<projid>.pkg/<packid>.upload-MD5SUMS` — tree of the `rev=upload` working copy
  (`addmeta_upload`, bs_srcserver:632-643). Removed on real commit (BSRevision.pm:360,377).
* `diffcache/` — rendered diffs keyed by md5 of a cache id string (bs_srcserver:3434-3484);
  `.run` lock files mark in-progress generation.
* `remotecache/` — remote binaries keyed by sha256("prp/arch[/modules]/bin"), validated by
  (mtime,size) (Remote.pm:816-852); random-slot expiry after 24h idle (800-814).
* `registry/:repos` — Storable `{regrepo => "proj/repo"}` owner map (Registry.pm).
* `notificationpayload/` — JSON blobs (e.g. PACKTRACK) referenced by key in a notification.
* `$BSConfig::bsquotafile` XML — package-count quota per project tree (bs_srcserver:1061-1075).

### 1.10 Authoritative vs derived

| Authoritative (must be preserved exactly) | Derived / cache (rebuildable) |
|---|---|
| `sources/<pkg>/<md5>-<file>` blobs | `sources/<pkg>/deltastore` (delta form of blobs; logically identical) |
| trees `*-MD5SUMS` (incl. pseudo-entry trees, which encode link/service lineage) | `*-_linkerror` logs (recomputable by re-running applylink) |
| `.rev`, `.mrev`, `.rev.del`, `.mrev.del`, `_deleted/` logs | `<projid>.xml`, `<projid>.conf`, `<packid>.xml` (copies of the latest mrev/rev `_meta`/`_config` — but treated as primary by all readers) |
| `_pubkey`, `_signkey` (also in mrev tree) | `:multibuild`, `db/*`, `diffcache`, `remotecache` |
| `*.upload-MD5SUMS` (uncommitted working copy) | `*-_serviceerror` (transient state) |
| `events/lastnotifications`, `notificationpayload/` (until consumed by API) | `events/lastevents` (ring buffer; losing it only causes `sync=lost`) |
| `registry/:repos`, `configuration.xml`, `issuetrackers.xml` | `events/undelivered` (retry queue) |
| `projects/_repoid` (identity; schedulers compare it) | |

---

## 2. Revision model

### 2.1 The revision object

A `$rev` hash carries: `project`, `package`, `rev` (integer, or a 32-hex srcmd5, or
`upload`/`repository`/`pattern`/`obsscm`), `srcmd5`, `vrev`, `version`, `time`, `user`,
`comment`, `requestid`, plus transient keys used by algorithms: `linkrev`, `originproject`
(package came via project link), `originpackage` (multibuild flavor), `ignoreserviceerrors`,
`keepsignkey`, `acceptinfo`, `cpiofd`/`run`/`_service_info` (obs-scm).

**Resolution of a revision identifier** (`getrev`, bs_srcserver:594-630 → `BSRevision::getrev_local`
BSRevision.pm:57-107):
1. Multibuild name `pkg:flavor` → resolve `pkg`, set `originpackage` (594-602).
2. Local: `latest`/`build`/undef → last `.rev` line (or empty tree if none); `N` → exact
   match on `rev`; 32-hex → `{srcmd5 => id, rev => id}` without existence check; `upload` →
   `upload-MD5SUMS` if it exists else latest; `repository` → empty tree; `0` → empty tree.
3. Not local but project has `<link>` → `Projlink::getrev_projlink` (§2.6).
4. 32-hex id of a *deleted* package → allowed if tree exists and a deletion record exists
   (`getrev_deleted_srcmd5`, BSRevision.pm:44-52).
5. Remote project → `Remote::getrev_remote` (§3.9).
6. `_pattern` pseudo package; `missingok` → empty tree (with vrev from project-link chain).
7. else 404.

For `_project` with no `.rev` but an existing `.conf`, a synthetic "initial commit" is created
on first read (`retrofit`, BSRevision.pm:86-89,146-155) — lazy migration of pre-revision data.

### 2.2 `version`/`vrev` derivation (`getcommitinfo`, bs_srcserver:546-589)

On commit, the server parses the build recipe (`findfile` chooses `<pkg>.spec`, `.dsc`,
`.kiwi`, `Dockerfile`, `.productcompose`…, bs_srcserver:981-1037) with `Build::parse_typed`
using the project config, extracts `version` and the leading integer of `release`
(honouring `<B_CNT>`/`<RELEASE>` placeholders, 571-582). A link always yields
`('unknown', '0')` (552-556). `cicntstart` from prjconf overrides the release start (583-587).
`vrev` is then finalized by `fdb_add_i2` (§1.5) unless the caller passed an explicit `vrev`
(`addrev_local`, BSRevision.pm:363-367). Special fix-ups on known↔unknown version transitions
keep vrev monotonic (bs_srcserver:817-830).

### 2.3 Commit algorithm (`addrev`, bs_srcserver:683-864)

Input: `$files = {name => md5}` whose blobs already exist in `sources/<packid>/`.
1. Validate package name (read-only for multibuild/`_product:*` when a `_product` exists), no
   `_link` in `_project`, user/comment utf8-sanitized.
2. **Servicemark**: unless old-style services, strip `_service:*`/`_service_*` files from the
   commit and compute `servicemark = genservicemark(...)` if any server-side service is active
   (Service.pm:76-117; §6.1). With `noservice=1` a fake service result is written instead
   (`servicemark_noservice`, Service.pm:609-632).
3. `_pattern` auto-creates its package meta (712-723). Package must exist (724).
4. `target` variants: `upload` → write `upload-MD5SUMS`, notify `SRCSRV_UPLOAD`, return
   `rev=upload`; `repository` → return empty rev; any other defined target → write tree only and
   return `rev=srcmd5` ("internal" revision used by `emptylink`, 3066).
5. Sanity: no `/` in names, all values 32-hex (739-742). Package not `lock`ed, not scmsync
   unless `commitobsscm` (744-748). `_patchinfo` containers may only contain `_patchinfo`
   (+`_scmsync.obsinfo`) (750-754).
6. Write tree with `/SERVICE => servicemark` (757-759) — **the servicemark is baked into the
   committed srcmd5**, so a commit that needs a service run has a different srcmd5 than the
   identical file set without one.
7. `_product` → `Product::expandproduct` synchronously (763-765, §2.8).
8. version/vrev via `getcommitinfo` (767-771), overridable by cgi `version`/`vrev`/`time`
   (782-787; `time` must not go backwards).
9. `acceptinfo` (for request acceptance): record `osrcmd5` (old), `oxsrcmd5` (old expanded),
   later `rev`,`srcmd5`,`xsrcmd5` (789-800, 835-848).
10. Append to `.rev` (`addrev_local`), update linkinfo DB, delete `upload-MD5SUMS`
    (BSRevision.pm:350-381).
11. Notifications: `SRCSRV_COMMIT`, `SRCSRV_VERSION_CHANGE`; `notify_repservers('package')`
    unless obsscm (850-859). For `_project`, `notify_repservers('project')` only if `_config`
    changed (809-814).
12. Put `/SERVICE` back into `$files` so the caller (`addrev_runservice`, 866-871) can start the
    service run after the commit.

`block_identical_commits` (774-776) short-circuits obs-scm no-op commits.

### 2.4 Meta revisions (`.mrev`)

`addrev_replace_common` (BSRevision.pm:226-306) takes a list of `[tmpfile, livefile, revname]`
triples: blobs are added to `sources/<pkg>/`, the previous mrev tree is read, entries
replaced/removed, a new tree written, a new `.mrev` line appended **under the `.mrev` flock**, and
only then the live files (`<pkg>.xml`, `_pubkey`, …) are `rename()`d into place (290-301). Readers
of `<pkg>.xml` therefore may momentarily see the previous meta after the mrev line exists. The
`_signkey` blob is chmod 0600 (243) and hidden from `lsrev` unless `keepsignkey` (BSRevision.pm:557).
Legacy data is retrofitted on first touch (`retrofit_old_meta`, 157-188).

Meta operations that create mrev entries: `putproject`, `putpackage`, `createkey`, `extendkey`,
`recreatesslcert`, `deletekey`, `freezeprojectlink`, `putfile meta=1` (`_attribute`,
`_frozenlinks`), `lock`/`unlock`, delete (an empty-change record "package was deleted").

### 2.5 `_link` handling

**Schema** (`BSXML::link`, BSXML.pm:960-972; `linkpatch` 941-958):
```xml
<link project="P" package="K" rev="R" vrev="V" cicount="add|copy|local" baserev="SRCMD5" missingok="true">
  <patches>
    <apply name="x.diff"/>  <add name="y.patch" type="" after="" popt="" dir=""/>
    <delete name="f"/>  <topadd>text</topadd>  <branch/>
  </patches>
</link>
```
Verification: `BSVerify::verify_link` (BSVerify.pm:324-355): at least one of project/package/rev;
`<branch/>` requires `baserev` and must be the only patch. Project/package default to the link's
own project/package (Link.pm:574-575). `rev` may be an integer, a srcmd5, or omitted (= latest).

**Expanded vs unexpanded srcmd5.** The committed tree (contains `_link`) is the
*unexpanded* tree, its id is `lsrcmd5` ("local"). Expansion produces a new tree whose id
`xsrcmd5` is **not** `calcsrcmd5(files)` but is derived from lineage:

```
xsrcmd5 = md5_hex( "<target expanded srcmd5>  /LINK\n<local srcmd5>  /LOCAL\n" )
```
(Link.pm:644-645) computed **per hop from the far end of the chain** (638-656), and stored with
`addmeta_link` as a MD5SUMS file containing the real files plus `/LINK` and `/LOCAL` entries
(BSSrcrep.pm:409-422). Consequently:
* The same expanded file set reached from a different local or target tree has a different
  xsrcmd5 — xsrcmd5 identifies "this link at this target revision", not the content.
* `verifymd5` (= `calcsrcmd5` of the expanded file list, bs_srcserver:1664-1666, 7048-7050) is the
  content id the scheduler uses to decide whether a rebuild is needed.
* If the expanded tree file exists, expansion is skipped entirely (Link.pm:646) — the tree store
  is also the expansion cache. A broken link stores a `<xsrcmd5>-_linkerror` instead, which is
  also cached (Link.pm:157-160; BSSrcrep.pm:424-444).

**`handlelinks($rev, $files, $li)`** (Link.pm:480-659) — iterative expansion:
1. Push `{project,package,srcmd5,rev}` of the local tree onto `@linkinfo`.
2. While the current tree has `_link`: parse+verify; access check if the link omits `package`
   but points to another project (Link.pm:555-562); fill defaults; if `$rev->{linkrev}` is set,
   override the first hop's `rev` (`base` → use `baserev`) (576-577); detect cycles via
   `%seen{"proj/pkg/rev"}` (582); record `linked` for the scheduler (585); `checksourceaccess` on
   `missingok` targets (587-590); resolve target via `$getrev` (may be remote or via project link);
   target must not be empty (599-601); fetch the target's **service-expanded** file list
   (`$lsrev_linktarget` = `lsrev_service`, bs_srcserver:97, so links see service results, 606-619;
   `ignoreserviceerrors` falls back to raw tree).
3. **vrev propagation** (620-632): with `cicount=add` (default) `vrev = target.vrev + local.vrev`
   (numerically on the trailing digits); `copy` uses the target's vrev; `local` stops propagation;
   an explicit `vrev=` on the link freezes it.
4. Walk `@linkinfo` from the end: for each hop compute xsrcmd5 and, if the tree is missing,
   `applylink(xsrcmd5, target, local)`.
5. Return `lsrev($rev)` of the final tree; `$li` gets `srcmd5` (target) and `lsrcmd5` (local)
   from the `/LINK` `/LOCAL` entries.

**`applylink($md5, $lsrc, $llnk)`** (Link.pm:209-525):
* If the link's tree has any `_service*` file, drop all `_service:*` files from the target (219-223).
* Classify patches. **Simple link** (only `delete` patches or none, and `$md5` given): expanded
  files = target files ∪ local files (local wins), minus deletes, minus `_link` (251-268);
  blobs are hard-linked across package dirs via `copyfiles`.
* Otherwise materialize in `sources/:upload/applylink$$/`: copy all target files not overridden
  locally, then local files (except patches to apply and `_link`) (286-297); then apply in order:
  - `delete`: unlink.
  - `branch`: **three-way merge** (305-396): base = tree `baserev` of the *target package*
    (must exist and not be a link); for every file name in `fsrc ∪ flnk`: if `base==mine`, take
    `new`; if `base==new` or `mine==new`, take `mine`; else if any side missing → conflict;
    `.obscpio` → conflict; non-ASCII → conflict; else run `bs_mergechanges old new mine` for
    `*.changes` (falls back to diff3 on failure), and `diff3 -m -E mine old new` for everything
    else; non-zero exit → `"conflict in file F"`.
  - `add`: insert `PatchN: name` into the preamble of every `*.spec` and a `%patchN` line into
    `%prep` (`patchspec`, Link.pm:57-183 — position after the last lower-numbered patch, honoring
    pushd/popd blocks, `after=`, `popt=`, `dir=`).
  - `topadd`: prepend text to every `.spec` (`topaddspec`, 186-201).
  - `apply`: `patch --no-backup-if-mismatch -g 0 -f < patchfile` in the tmpdir (430-448); afterwards
    every produced file must be a regular file with a legal name, never `_link`
    (449-468), and permissions are normalized.
* On failure: append to `.log`, store it as `<xsrcmd5>-_linkerror`, return the error string.
* On success: md5 every result file (reusing the source md5 when the inode is unchanged,
  484-499), `addfile` them into `sources/<local pkg>/`, write the tree with `/LINK` `/LOCAL`
  (`addmeta_link`). With `$md5` undefined (used by `keeplink` repair) the files are left in the
  tmpdir renamed to `md5-name` and the hash is returned (502-509).

`bs_mergechanges` (l.87-216): parses each `.changes` file into dated entries (`"-----"` separator,
`<date> - <email>` header), groups entries by timestamp, uses word-overlap similarity to match
entries across files, and emits the union sorted newest first; dies on conflicting same-time entries.

**`keeplink($cgi, projid, packid, files, orev)`** (Link.pm:696-911) — used by `commit`,
`commitfilelist`, `putfile`, `delfile`, `copy` with `keeplink=1`: the caller has an *expanded*
file set and wants to turn it back into a `_link` + local overlay:
* For a branch link: keep `_link`, update `baserev` to the current target srcmd5 (or `linkrev`)
  and optionally `rev=setrev` (781-803) — a branch stores full files plus `_link`.
* For a patch link: expand the old link (or re-apply with `repairlink`), diff the expanded old
  files against the new files for ASCII files (`diff -up`, `rundiff` 661-676), collect into a new
  `projectN.diff` (`findprojectpatchname`), record deletes as `<delete>` patches, store non-ASCII
  or non-diffable changes as local full files, drop local files that now equal the target, bump
  `baserev`. `missingok` is removed once the target exists (774-780).

**`integratelink`** (Link.pm:914-1009) — when copying a link *onto its own target* which is itself
a link: append the source link's patches to the target link (renaming colliding
`projectN.diff`), convert branch→patch link if needed (`convertbranchtopatch`), update baserev.

**`linkinfo_addtarget`** (Link.pm:1012-1037) fills `project/package/rev/baserev/missingok` for
the `<linkinfo>` element of a directory listing from the `_link` of the `lsrcmd5` tree.

**`findlastworkinglink`** (Link.pm:1039-1087): scans all known trees of the package for ones
whose `/LOCAL` equals the current local srcmd5 and picks the newest working expansion (used by
`getfilelist lastworking=1`).

**Frozen links** (project level): `_frozenlinks` in the project mrev lists `(package, srcmd5,
vrev)` per linked project (or `/all`); `getrev_projlink` resolves to those srcmd5s instead of
latest (Projlink.pm:39-59, 72-78). Created by `POST /source/$project?cmd=freezelink`
(bs_srcserver:3971-3992) which expands every package and records the result.

**`linkrev`** request parameter: forces the first hop's target revision (`base` = the link's
baserev); used by diff/blame/getfilelist to show "what did my patches do relative to the base".

### 2.6 Project links (`Projlink.pm`)

Project meta `<link project="X" vrevmode="extend|unextend"/>` (BSXML.pm:147-150) makes packages
of X appear in this project ("virtual" packages, `originproject=X` in listings).
* `getrev_projlink` (61-110): iterate links in order, skip already-visited `(proj,pkg)`, respect
  frozen links, `checksourceaccess` on the source, **`copytree` the revision into this project's
  tree dir**, set `originproject`, and rewrite `rev->{project}` to *this* project. With
  `vrevmode`, vrev becomes `(leading int)+1` and `.1` is appended for `extend` (101-105).
* `findpackages_projlink` (112-163): breadth-first over the link graph; `maintenance_release`
  projects hide `*.<incident>` duplicates (135-146); multibuild names are stripped (154);
  `_product` packages suppressed if the local project has `_product` (152).
* `readpackage_projlink` (165-184): first project in BFS order that has the package meta.
* `getnewvrev` (194-215): for a *new* package in a project with vrevmode links, the initial vrev
  is the link depth (+`.1`).
* Project links are transitive (links of linked projects are followed), remote projects may be
  link targets (bs_srcserver:1316-1329 fills remotemap for them).

### 2.7 Source service revisions (new style)

Terminology (all are 32-hex ids used as tree names of the *same* package):
* `lsrcmd5`-of-service = the committed tree id (contains `/SERVICE`).
* `servicemark` (`xservicemd5`) = `md5("sourceservice/P/K/<time>\n<oldrev><sorted file list>")`
  (Service.pm:103-113); unique per commit by construction.
* On success the service result tree is stored **under the servicemark** with `/LSERVICE` =
  committed srcmd5 and `/LXSERVICE` = expanded link srcmd5 it was computed from
  (`addmeta_service`, BSSrcrep.pm:452-469). The result tree contains the original files plus
  `_service:<name>:<file>` outputs (and `_service_error`? no — errors go to the marker file).
* `handleservice($rev, files, servicemark)` (Service.pm:637-680): if tree `servicemark` exists
  return it (sets `$rev->{srcmd5}=servicemark`); if the package is not local (project link/remote)
  resolve via `$getrev`; otherwise consult the `_serviceerror` marker: "service in progress" →
  die `service in progress`, error → die `service error: …`, none → *start the service run* and die
  `service in progress`. **Reads can trigger service runs** (lazy re-execution after a crash).
* `lsrev_service` (bs_srcserver:875-881) = `lsrev` + `handleservice` if `/SERVICE` present;
  `lsrev_expanded` (883-892) = `lsrev_service` + `handlelinks`. The scheduler-facing `srcmd5` in
  `getprojpack` is therefore the servicemark (or the xsrcmd5 for links) and `verifymd5` the content
  hash (bs_srcserver:1602-1667).
* Old style (`$BSConfig::old_style_services`): service results are committed as a *new* `.rev`
  entry by user `_service` with `_service_error` as a file (Service.pm:185-197).
* `update_link_in_service` (bs_srcserver:3997-4020): when the server auto-rewrites a `_link` of a
  package that has a service result, it fabricates a new service result tree with the new `_link`
  so the servicemark stays consistent (`fake_service_run`, Service.pm:247-267).

### 2.8 Other special files

* `_aggregate` (`BSXML::aggregatelist`, BSXML.pm:227-245): binary aggregation spec, verified and
  passed through to the scheduler in `getprojpack` (bs_srcserver:1685-1727); referenced projects
  are pulled into the remotemap.
* `_multibuild` (BSXML.pm:2133-2138, `<flavor>`/legacy `<package>`, `buildemptyflavor`):
  cached per project (§1.9); flavors become pseudo-packages `pkg:flavor` which are read-only
  (`addrev` 689) and resolve to the master package (§2.1). Service-generated
  `_service:*:_multibuild` is honoured (`find_mbname`, Multibuild.pm:68-79).
* `_product` (Product.pm:40-166): on every commit of `_product`, `bs_productconvert` is run on
  the (link-expanded) files; each output dir becomes package `_product:<name>` with its own
  `.rev` (vrev forced to 1, no `.mrev`, no linkinfo) and `.xml` (written directly, not via mrev);
  stale `_product:*` packages are deleted by unlinking their files. `_product:*` and `_product`
  are mutually exclusive (bs_srcserver:2426-2433, 686-688).
* `_patchinfo` (BSXML.pm:249-281): container may hold only `_patchinfo`; verified at commit and in
  getprojpack (1728-1755).
* `_channel` (BSXML.pm:285-316): parsed once per md5 and shipped as `channeldata` (1756-1786, 2080-2085).
* `_constraints`: only its md5 is exported (`constraintsmd5`, 1812) and its content forwarded to
  the dispatcher for `checkconstraints` (7607-7633).
* `_buildenv[.repo.arch]`: `hasbuildenv` flag / content for buildinfo (1077-1089, 1813-1816);
  created by `cmd=collectbuildenv` which fetches `_buildenv` from the repserver (4355-4421).
* `_modulemd.yaml`: parsed (`Modulemd.pm`) into `modulemd` for the scheduler; the `modulemd`
  package is parsed first because its macros feed all other packages' build configs (1453-1472, 1802).
* `_pattern` pseudo-package (legacy `pattern-MD5SUMS` or real package): `patternmd5` for publisher.
* `_service` (BSXML.pm:1622-1632) and project-level `_service` with `<link project>`
  (`services_link`, 1635-1641) collected by `getprojectservices` (Service.pm:684-746).
* `_serviceerror`, `_service_error`, `_linkerror` are *virtual* file names in listings.
* Meta-side files: `_meta`, `_pubkey`, `_signkey`, `_sslcert`, `_attribute`, `_frozenlinks`.

### 2.9 Delete / undelete

* Package delete (`delpackage`, bs_srcserver:2459-2488): remove `upload-MD5SUMS`; add a
  "deleted" record to both logs; `delete_rev` appends the whole `.rev` to `.rev.del` (and `.mrev`
  to `.mrev.del`), removes `.xml` and the logs (BSRevision.pm:430-456); clears linkinfo and
  scmsync; `_product` → remove generated packages; notify.
* Package undelete (2490-2515 → `undelete_rev`, BSRevision.pm:388-428): take the **last block**
  of the `.del` file (from the last `rev=1` to the end), re-append it to `.rev`/`.mrev`, add a
  new "undeleted" record (rev+1, vrev bumped, optional `time`), re-extract live files.
  Only the latest deletion generation is recoverable through the API.
* Project delete (2293-2337): after replying OK, moves all `.rev/.mrev/.del` files to
  `_deleted/<proj>.pkg/` (older generation first appended into `.del` there, `delete_deleted`),
  removes `.xml`/`.conf`. Undelete (2339-2365) reverses it.
* `getrev(..., deleted=1)` and `_history deleted=1` read from the `.del`/`_deleted` logs
  (BSRevision.pm:63-81, 119-127).

### 2.10 Copy / branch semantics (the "request acceptance" primitives)

* **`cmd=branch`** (4279-4330): creates `_link` to `(oproject,opackage[,orev])` with
  `baserev = expanded srcmd5 of the origin` and `<branch/>` (unless `simplelink`), copies the
  expanded files into the new package (so the branch is self-contained), optional
  `missingok`, `extendvrev` (`vrev = (int+1).1`), `keepcontent`. The link omits `package`/`project`
  attributes when equal — but if the origin project has `sourceaccess` restrictions the package
  name is forced in (4303-4306).
* **`cmd=linktobranch`** (4332-4353): converts a patch link into a branch link (full files + baserev).
* **`cmd=copy`** (4078-4277) — the heavy one:
  - resolves origin (`orev`, `olinkrev`), expands links if `expand=1`, optionally preserves
    service results (`noservice` → copies the service tree and reuses the servicemark via
    `servicemark_noservice`).
  - **auto-simplify**: if the origin is a link whose target is the destination (i.e. "accept
    submit request into target"), then after the commit the *origin* is rewritten: a branch link
    gets a new `baserev` (so it shows no diff), a patch link gets its patches removed
    (`sourcecopy_update_baserev` / `sourcecopy_update_link`, 4023-4076, committed as user
    `buildservice-autocommit`). `dontupdatesource=1` disables this.
  - `withvrev` copies the origin's vrev bumped by `vrevbump` (default 1, 2 for
    `makeoriginolder`); `makeoriginolder` additionally appends a dummy commit to the origin with
    vrev `(n+1).1` so the origin keeps building "older" (4215-4227).
  - `freezelink` pins the origin's link `rev=` to the current expanded srcmd5.
  - `instantiate` copies a project-linked (virtual) package into the local project with the
    virtual vrev (4087-4097).
  - returns `revision_acceptinfo` (BSXML.pm:816-827).
* **`cmd=copy` at project level** (`copyproject`, 3784-3969): copies `_project` sources, creates
  packages, copies either the last revision or the full history (renumbering revs, `copytree` each),
  always creates one new commit with bumped vrev (`makeolder`: `(n+1).1`), optionally copies
  binaries via repserver `cmd=copy`, brackets the operation with `suspendproject`/resume events.
* **`cmd=move`** (3735-3781, admin): `rename()` of `.xml/.pkg/.conf`, `copyprojectrees`,
  linkinfo/scmsync moves, repserver `cmd=move`. Refuses if partitions differ.

---

## 3. HTTP routes

Dispatch grammar (BSDispatch.pm:36-45): `METHOD:/path/$positional ...`, `name:type?` optional,
`name:type*` list, `name` required; `$` = also passed positionally; `cmd=x` literal selects the
handler; a `!- ` prefix marks a route that needs no `rw` authorization (see §5.1 for the
`!rw :` / `!- GET:` defaults). Unknown query
parameters are a 400 (BSDispatch.pm:127). Types: `bool`, `num`, `intnum`, `md5`, `rev`,
`linkrev`, `project`, `package`, `repository`, `filename`, `prp`, `resultview` map to
`BSVerify::verify_*` (BSVerify.pm:28-230).

Legend: **R** = proxied to repserver by partition, **X** = interconnect-aware (remote project
handling), **A** = may hand off to the AJAX process, **E** = emits repserver event (`notify_repservers`),
**N** = emits API notification (`notify()`).

### 3.1 Projects

| Route | Handler / semantics |
|---|---|
| `GET /source [deleted] [project*]` | `getprojectlist` 2092-2106: list `projects/*.xml` (or `_deleted/`). |
| `GET /source/$project [deleted] [expand] [noorigins]` | `getpackagelist` 2369-2381: local packages + project-link packages (only when `expand=1`; the handler passes `noexpand = !expand`) + multibuild flavors; `originproject`/`originpackage` attrs. **X** (remote listing). |
| `GET /source/$project view=info …` | `getprojectsourceinfo` 7083-7137: per-package `sourceinfo` (srcmd5/lsrcmd5/verifymd5/recipe/parsed deps). Batches remote getrevs (`collect_remote_getrev`). **X** |
| `GET /source/$project/_meta [rev] [withconfig] [interconnect]` | `getproject` 2108-2131. `rev` reads mrev tree. `interconnect=1` strips `<scmsync>`. **X** |
| `PUT /source/$project/_meta [user comment requestid lowprio]` | `putproject` 2243-2291: verify, mrev commit, auto-create key if `forceprojectkeys`, **E** (`project` or `lowprioproject`, to *all* repservers if remote), scmsync DB + trigger. **N** `SRCSRV_CREATE/UPDATE_PROJECT`. |
| `DELETE /source/$project` | `delproject` 2293-2337 (§2.9). **E N** |
| `POST /source/$project cmd=undelete` | `undeleteproject` 2339-2365. **E N** |
| `POST cmd=copy oproject …` | `copyproject` (§2.10). **E** (suspend/resume, package) |
| `POST cmd=move oproject` | `moveproject` (§2.10). **R E** |
| `POST cmd=freezelink` | `freezeprojectlink` 3971-3992. |
| `POST cmd=createkey/extendkey/recreatesslcert` | §5.3. |
| `POST cmd=notifypackagechange` | `notify_repservers('package', proj)` — used by `bs_servicedispatch` to signal completion (bs_servicedispatch:42-49). **E** |
| `GET /source/$project/_config [rev]` | `getprojectconfig` 2827-2844. **X** |
| `PUT/DELETE /source/$project/_config` | `putprojectconfig`/`delprojectconfig` 2846-2871: `_project.rev` commit of `_config`; refused for scmsync projects. **E N** |
| `GET /source/$project/_pubkey [rev]` / `DELETE` | `getpubkey` 2217-2239 (**X**), `deletekey` 2198-2215. |
| `GET /source/$project/_keyinfo …` | `getkeyinfo` 6388-6440: pubkey+cert info, walks up the project hierarchy for the signing project, may auto-extend/auto-create cert. **X** |
| `GET /source/$project/_history …` | `getpackagehistory` with `_project`. |

### 3.2 Packages and files

| Route | Handler / semantics |
|---|---|
| `GET /source/$project/$package [rev linkrev expand emptylink deleted view extension lastworking withlinked meta product]` | `getfilelist` 2983-3174. Default rev is `upload` (falls back to latest). Returns `BSXML::dir` with `srcmd5`, `rev`, `vrev`, `<linkinfo>` (project/package/rev/baserev/missingok/srcmd5/lsrcmd5/xsrcmd5/error/lastworking/linked), `<serviceinfo code=running|failed|succeeded xsrcmd5 lsrcmd5 error>`, entries with md5/size/mtime. `expand=1` returns the expanded tree and sets `rev=xsrcmd5`. `view=cpio` streams a cpio (hands off to AJAX if the client is another OBS — detected via `User-Agent: BSRPC`, 3118-3123). `view=products|productrepositories`. `emptylink=1` creates an internal revision with the patches stripped. Updates the multibuild cache as a side effect (3073,3103). **A X** |
| `GET …/$package view=info …` | `getpackagesourceinfo` 7139-7157. |
| `POST …/$package view=info [filename]` | `getpackagesourceinfo_post` 7235-7256: parse an *uploaded* recipe or cpio (used by API for `_service`-less previews). |
| `GET …/$package view=getmultibuild` | `getmultibuildpackages` 5180-5189 (from cache). |
| `GET …/$package/_history [rev meta deleted limit startbefore]` | `getpackagehistory` 2548-2609: reads `.rev`/`.mrev` backwards; `rev=<srcmd5>` filters by srcmd5; follows project links; proxies remote. **X** |
| `GET …/$package/_meta [rev expand meta deleted view interconnect]` | `getpackage` 2383-2411: exact `_meta` string for rev/deleted/_project, else merged via project links. **X** |
| `PUT …/$package/_meta` | `putpackage` 2413-2457: verify, `_product` exclusivity, mrev, **E** if build-relevant fields changed (`title description devel person group url` ignored), scmsync trigger. **N** |
| `DELETE …/$package` | `delpackage` 2459-2488. **E N** |
| `GET …/$package/$filename [rev expand meta deleted view noajax]` | `getfile` 3176-3249: streams a blob; `view=blame` renders blame (Blame.pm); remote-OBS downloads hand off to AJAX. For scmsync packages `_serviceerror` is looked up under the obsscm servicemark (3228-3234). **A** |
| `PUT …/$package/$filename [rev user comment keeplink force meta]` | `putfile` 3251-3296: `meta=1` → only `_attribute`/`_frozenlinks` (mrev, **E** for frozenlinks). Else `addfile` + new revision over the current `rev` (default `upload`), `keeplink` re-links, service run. Writing `_service:*` needs `force`. **N** (meta) |
| `DELETE …/$package/$filename` | `delfile` 4434-4463 (same pattern). |
| `POST …/$package cmd=commit [rev keeplink repairlink linkrev setrev noservice]` | `sourcecommit` 3624-3633: turns the `upload` (or given) tree into a real revision. |
| `POST cmd=commitfilelist [withvalidate servicemark time version vrev …]` body=`<directory>` | `sourcecommitfilelist` 3635-3697: client sends `(name, md5[, hash=sha256:…])`; server replies with `error="missing"` listing files it lacks (and `hash=new/missing` with `withvalidate`); otherwise commits (auto-committing a stale `upload` rev first) and returns the new file list. This is how `osc commit` works. |
| `POST cmd=commitobsscm servicemark run [srcmd5 user comment info]` | `sourcecommitobsscm` 3699-3732: called by `bs_servicedispatch` to finish an obs-scm run (package: tree by srcmd5; project: cpio body → `Scmsync::sync_project`). |
| `POST cmd=copy …` / `cmd=branch …` / `cmd=linktobranch` | §2.10. |
| `POST cmd=collectbuildenv` | 4355-4421. **R** |
| `POST cmd=diff/linkdiff/servicediff …` | `sourcediff` 3356-3589: resolves both revisions (defaults: `orev = rev-1`), optional link expansion with `linkrev`/`olinkrev`, caches result in `diffcache` keyed by (srcmd5s, options), concurrent generation waits in AJAX (`/sourcediffcache`). Views: plain, `unified`, `xml` (`BSXML::sourcediff`, with `withissues` issue extraction). `linkdiff` = diff against link target; `servicediff` = diff committed vs service result. **A** |
| `POST cmd=runservice` | `triggerservicerun` 432-471: new commit with a fresh servicemark (or obs-scm run). |
| `POST cmd=waitservice` | `waitservicerun` 473-505: blocks (via AJAX file watcher on the `_serviceerror` marker) until the run finishes. **A** |
| `POST cmd=mergeservice` | `mergeservicerun` 507-541: copies `_service:x:f` results to `f`, keeps only `buildtime` services, commits. |
| `POST cmd=getprojectservices` | project-level service list (Service.pm:684-746). |
| `POST cmd=lock/unlock` | 2517-2546: sets `<lock><enable/></lock>` in package meta. **E** |
| `POST cmd=undelete [time]` / `cmd=deleteuploadrev` | §2.9 / 4423-4427. |
| `PUT …/$package cmd:` | obsolete alias of commitfilelist. |

### 3.3 Scheduler/worker/publisher-facing

| Route | Handler |
|---|---|
| `GET /getprojpack [project* repository* package* arch withrepos withsrcmd5 withdeps withconfig expandedrepos ignoredisable nopackages withremotemap remotemaponly noremote parseremote buildinfo partition view=storable]` | `getprojpack` 1108-2090 — **the scheduler's bulk view of the world**: for every (selected) project: flags, repos (optionally with expanded search path and `base`), config, packages with `srcmd5`/`verifymd5`/`rev`/`versrel`/`linked`/`originproject`, per-repo parsed recipe info (`BSXML::packinfo`: name, deps, prereqs, exclarch, path, buildtimeservice…), aggregates/patchinfo/channel/modulemd, errors (`disabled`, `locked`, `excluded`, `broken link…`, `source update running`, `no source uploaded`), and a `remotemap` (remote projects + projects on *other partitions*, Partition.pm:68-105). Evaluates build/lock flags per repo/arch (`BSUtil::enabled`, BSUtil.pm:523-560), quotas, `limit_projects`. Two-pass remote resolution (`collect_remote_getrev`, 1482-1489). `buildinfo=1` is the "osc buildinfo" variant. |
| `GET /getsources $project $package $srcmd5` | cpio of a tree (2875-2883). Worker. |
| `GET /getconfig $project $repository [path*]` | `worker_getbuildconfig` 2812-2824 → `concatconfigs` 2657-2767 (merge of prjconfs along the expanded path, `%define _project/_repository`, `FromScratch:`, macros sections, `%vendor obs://<obsname>/<signproject>`). Remote parts handed to AJAX with result sharing (`serialize`, 2769-2797). **A X** |
| `GET /getbinaries`, `/getbinaryversions`, `/getobsgendiffdata` | worker proxies to remote OBS with local cache (`Remote.pm:613-921`), chunked in <1900-byte URL groups; always on AJAX. **A X** |
| `GET /getsslcert`, `/getsignkey`, `/getpubkey` | §5.3. |
| `GET /getmodulemd` | 6543-6568. |
| `POST /relsync`, `GET /relsync` | release-number sync across archs of a repo (6870-6906). **R** |
| `GET /worker/$id`, `POST /worker cmd=checkconstraints` | dispatcher proxies (7594-7633). **R** |
| `GET /jobs/$arch [partition*]` | masterdispatcher job list filtered by partition (7559-7582). |
| `POST /search/published cmd=updatedb` (Storable body) | publisher feeds the published index (5554-5601). |
| `POST /notify/$type` | `external_notification` 7527-7539: from repserver/publisher/scheduler via `bs_notifyforward`; `redis` and `sourcepublish` types are handled specially (§4.3), `PACKTRACK` carries a Storable payload stored as JSON. |
| `POST /notify_plugins/$type` (JSON body) | `BSSrcServer::Notify::notify_plugins` — API-triggered plugin fan-out (hermes/rabbitmq). |
| `POST /event type project [package repository arch job]` | `newevent` 6770-6778: append to `lastevents` (used by repservers to publish repo events for interconnect). |
| `POST /ownregistryrepo`, `/disownregistryrepo` | publisher registers container repo ownership (7772-7786). |
| `POST /slsa cmd=addrefs`, `GET /slsa/…` | fan-out of SLSA references to the owning repservers (7896-7939). **R** |
| `PUT/GET /configuration` | store `configuration.xml`, push to all repservers + service server (7260-7290). |
| `PUT/GET /issue_trackers`, `POST cmd=issues` | 7294-7320. |

### 3.4 `/build` namespace (proxy to repservers; **all R**, most **X**)

`checkprojrepoarch` (4713-4726) validates project/repo/arch against the *project meta* before
forwarding. Partition routing: `projid2reposerver` (Partition.pm:26-43).

| Route | Notes |
|---|---|
| `/build`, `/build/$project`, `/build/$project/$repository` | lists from project meta (remote → forwarded). |
| `/build/_workerstatus` | aggregates `workerstatus` from all partition repservers + local daemon states via lock files (5260-5374). |
| `PUT/GET /build/_dispatchprios` | masterdispatcher. |
| `/build/_result scmrepository=…` | `getscmresult` 4601-4649: results for all packages tied to a git repo. |
| `/build/$project/_result […]` | `getresult` 4495-4599: expands `multibuild`, `locallink` (linkinfo DB), validates package names via `findpackages`, forwards `prpa=` list; `oldstate` long-poll handled by AJAX (4510-4513). Computes `summary`. **A** |
| `POST /build/$project cmd=…` | `docommand` 4651-4711: `rebuild/abort/wipe/restartbuild/unpublish/…`; `suspendproject`/`resumeproject` go straight to events. |
| `/build/$project/$repository/_buildconfig` | `getbuildconfig`. |
| `/build/$project/$repository/$arch [view=…]` | `getpackagelist_build` 4775-4808: package list or repserver views (`binaryversions`, `binarychecksums`, …), remote via `Remote::getpackagebinaryversionlist`. **A** |
| `/_builddepinfo`, `/_jobhistory`, `_repository` POSTs, `cmd=copy` (copybuild, ensures multibuild flavor known), `uploadbuild`, `getbinarylist`, `getbuildinfo[_post]`, `_jobstatus`, `_log` (streams; checks for link/service errors first, 5067-5108), `_reason`, `_status`, `_history`, `_buildstats`, `getbinary` (`view=fileinfo_ext` ships a projpack to the repserver; `view=publishedpath`), `putbinary`, `delbinary` | thin forwards with body streaming (`BSServer::forward_sender`). |
| `/build/$project/_availablebinaries` | 7659-7768: fans out to every reposerver/remote OBS on the path. |

### 3.5 `/published`, `/search`, `/registry`

* `/published…` (6082-6145): forwarded to the owning repserver; the root listing merges all
  partitions; `view=ymp` attaches a projpack so the repserver need not call back; `view=publishedpath`
  (6159-6232) computes download paths from `BSUrlmapper` and prjconf `repotype` (slepool media
  naming); `view=status`; `view=collection` queries the published binary index.
* `/search/project[/id]`, `/search/package[/id]` (5403-5550): XPath (`BSXPath`) over all
  project/package XML read from disk; special-cased regexes for the API's hot queries
  (`starts-with(@name)`, `linkinfo/@package=… and linkinfo/@project=…`).
* `/search/published/{binary,pattern,repoinfo}/{id,name}` (5846-5997): XPath over the published
  index with `limit`, `withdownloadurl`, `withscmsyncurl`.
* `/search $in $match [return values]` (6001-6038): generic.
* `/registry/…`, `/sigstore/…` (7788-7892): OCI registry facade. `registry/:repos` maps a
  registry repo name to the owning prp, which selects the repserver to forward to (incl. headers
  `Range`/`Accept`, `BSRegistryServer::errreply` error format). `_catalog` served locally.
  `manifestinfos` may probe all partitions.

### 3.6 Events / long-poll / status

* `/lastevents [filter* start obsname client block]` (6596-6695), `/public/lastevents`,
  `/lasteventsproxy` (6785-6866), `/lastnotifications` (209-291), `/notificationpayload/$key`
  (GET/DELETE) — §4.
* `/ajaxstatus [aidx]`, `/serverstatus` — introspection of AJAX jobs / forked children.
* `/service` — lists services from the service server. `/request/_lastid`, `/request/$id` — legacy.

### 3.7 AJAX-only routes (`$dispatches_ajax`, 8188-8206)

`/build/$project/_result` (oldstate long-poll), `/build/…/$arch` views, `_log` streaming,
binary lists, `/getbinaries`, `/getbinaryversions`, `/getconfig`, `/lastevents`,
`/lasteventsproxy`, `/lastnotifications`, `cmd=waitservice`, `/source/$project/$package rev view=cpio|notify`
(`getfilelist_ajax`, 2887-2930: background download of a remote tree with "master" dedup),
`/source/$project/$package/$filename`, `/request/$id`, `/sourcediffcache/$cacheid`.

### 3.8 AJAX partitions

`$BSConfig::srcserver_ajaxpartitions` splits the AJAX process into several processes keyed by
job class: `interconnect_in`, `interconnect_out`, `interconnect_in_lastevents`,
`interconnect_out_lastevents`, `lastevents`, `lastnotifications` (names used in
`BSHandoff::handoff_part` calls, e.g. 3121, 3242, 4791, 6489, 6613-6615, 6789). The mapping is
built in `BSStdServer::server` (BSStdServer.pm:558-570).

### 3.9 Remote (interconnect) projects — `Remote.pm`

* A *remote project* is a local project meta with `<remoteurl>` (+ optional `<remoteproject>`).
  `remoteprojid($projid)` (Remote.pm:51-94) resolves `A:B:C` by stripping `:`-components until a
  local project with `remoteurl` is found; the remainder becomes the remote name
  (`remoteproject = <remoteroot>:<rest>`). The result is `{name, root, remoteroot, remoteurl,
  remoteproject, remoteproxy}`. `maptoremote` maps names *back* (returns `_unavailable` if the
  remote name is outside the mounted subtree, 127-133). Remote data is rewritten so that path
  elements, release targets, project links and devel projects refer to local names (143-157).
* `getrev_remote` (370-506): `GET <remote>/source/<proj>/<pkg>?expand=1[&rev=][&withlinked]`;
  the remote's **expanded** listing is downloaded file-by-file (`rev=srcmd5` to pin), stored into
  the local `sources/<pkg>/` and a local tree written (`addmeta`) — remote packages therefore
  materialize as ordinary local trees, keyed by the *local* project name. `srcmd5` is checked
  against the remote's claim (497). Large trees during scheduler queries (`$linked` set,
  size>8192) are delegated to AJAX (`/source/P/K?rev=&view=notify`, 439-443) and the scheduler
  gets `download in progress`; after download `notify_all_repservers` re-triggers it
  (bs_srcserver:2898-2928). A per-request cache (`%remote_getrev_cache`) plus a bulk prefetch via
  `view=info&nofilename=1` on up to ~1900 bytes of package names (`fill_remote_getrev_cache`,
  298-368) reduces round trips in `getprojpack`.
* Remote project meta/config are fetched into the `remotemap` of a `getprojpack` reply
  (`fetchremoteproj`/`fetchremoteconfig`, 159-236; `proto=1` marks placeholders); person/group
  and flags are stripped from remote package meta (288-294).
* Binaries: `getremotebinarylist` (names via `view=names`), `getremotebinaryversions`,
  `getpackagebinaryversionlist`, `getremotebinaries` (cpio fetch in ≤50-package chunks into
  `remotecache`, serialized per prpa) — all AJAX-side with `BSWatcher::serialize` so concurrent
  identical requests share one upstream call (`getserialkey`, optional hashing into
  `interconnect_serialize_slots`).
* Events from remote instances: `lasteventsproxy` (§4.2).
* Access: `interconnect=1` on `_meta` strips `scmsync`; remote package meta loses roles/flags.

---

## 4. Event and notification model

### 4.1 Scheduler notification (`notify_repservers`, bs_srcserver:308-332)

```
notify_repservers(type, projid[, packid[, job]]):
   addevent({type, project[, package]})             # into events/lastevents (unless suspendproject)
   POST <reposerver(projid)>/event?type=&project=[&package=][&job=]   (background rpc)
   on failure for type in {project, package, lowprioproject}: write events/undelivered/<type>:<proj>[::pkg]
```
`periodic()` (7951-7968) forks a drainer for `undelivered/` every second if present. Event
types emitted by the srcserver: `project`, `lowprioproject`, `package`, `suspendproject`,
`resumeproject` (with `job=` comment). The repserver (`forwardevent`, bs_repserver:2924-2988)
writes one XML file per scheduler arch into `events/<arch>/<type>:<proj>[::pkg]` (name
truncated to an md5 if >200 chars), marks the prp dirty, and pings. Event files are
idempotent/coalescing by name (rewriting the same name = one event), except suspend/resume which
are randomized so they stack. **There is no acknowledgement**: delivery is at-most-once over
HTTP with a file-based retry only for the three types above.

Remote projects: `notify_all_repservers` (381-398) posts to every partition server since the
owning partition is unknown.

### 4.2 `lastevents` — the interconnect/long-poll stream

* `addevent` (6745-6768) appends `{number,time,type,project,package,repository,arch}` to
  `events/lastevents` (**skipped if the project has `access` disabled**, 6752-6757) and prunes the
  file to keep ≥256k/≥30min, pruning when >512k and older than 1h (`prunelastevents`, 6697-6739).
  Repservers also post repo events here via `POST /event` (`repository`, `repoinfo`, …).
* `GET /lastevents?start=N&filter=package/P/K|project/P|repository/P/R[/A]` returns events with
  `number >= N` (filtered, `filterlastevents` 6576-6594), `next` = next number, or
  `sync=lost` if N is older than the retained window. `start` absent → just the current number.
  Blocking mode (default) hands off to AJAX, which `addfilewatcher`s the file (stat-polled every
  1 s, `BSWatcher::filewatcher_handler`, BSWatcher.pm:127-151; 120 s periodic wake) and re-runs
  the handler when it changes (`redo_request`); a heartbeat reply is forced every 100 events or 5
  minutes (6674-6683). A small in-process cache of the tail avoids re-reading (6573-6630).
* `lasteventsproxy` (6785-6866): one AJAX job per `remoteurl` multiplexes all local schedulers'
  filters into a single upstream `POST /lastevents` (filters merged and aged out after 1000 s),
  using `joinable` RPCs so concurrent callers share the connection.
* Consumers: schedulers of other OBS instances (interconnect), `bs_dodup`, and API `public` route.

### 4.3 `lastnotifications` — the API event feed

* `notify(type, params[, payload])` (165-181) appends `{number,type,time,data=[k=v…]}` to
  `events/lastnotifications` (BSFileDB, `fdb_add_i` under lock). A payload (only `PACKTRACK`) is
  written to `notificationpayload/<time>.<md5>` and referenced as `payload=`.
* Types emitted: `SRCSRV_COMMIT`, `SRCSRV_VERSION_CHANGE`, `SRCSRV_UPLOAD`,
  `SRCSRV_CREATE_PROJECT`, `SRCSRV_UPDATE_PROJECT`, `SRCSRV_DELETE_PROJECT`,
  `SRCSRV_UNDELETE_PROJECT`, `SRCSRV_UPDATE_PROJECT_CONFIG`, `SRCSRV_CREATE_PACKAGE`,
  `SRCSRV_UPDATE_PACKAGE`, `SRCSRV_DELETE_PACKAGE`, `SRCSRV_UNDELETE_PACKAGE`,
  `SRCSRV_SERVICE_SUCCESS`, `SRCSRV_SERVICE_FAIL` (Service.pm:122-141), plus everything relayed
  from other daemons through `POST /notify/$type` (`BUILD_SUCCESS`, `BUILD_FAIL`, `REPO_PUBLISHED`,
  `PACKTRACK`, … — originate in scheduler/publisher via `BSNotify::notify` → `bs_notifyforward`
  queue → srcserver).
* `GET /lastnotifications?start=N[&block=1][&view=json]` (209-291): returns up to 1000
  notifications ≥ N (`limit_reached=1` if more), `next`, `sync=lost` if N fell off; the file is
  pruned to keep 500 entries behind the consumer once it is >1000 behind (`prune_lastnotifications`,
  192-207, rewrite+rename under lock). The Rails side (`UpdateNotificationEvents`,
  src/api/app/models/update_notification_events.rb) polls with its stored `lastnotification_nr`
  and converts each record into an `Event::*` row. **Single consumer semantics** (cursor stored in
  the API).
* `bs_notifyforward` (l.92-235) drains `events/notifyforward/queue` (BSFileQueue) written by
  `BSRedisnotify` (scheduler result updates) and `BSNotify`, batching `redis` jobs (≤32) into
  `POST /notify/redis` (Storable body, keep-alive child process `start_redis_keepalive_process`,
  7412-7436) which the srcserver re-queues into `events/redis/queue` for `bs_redis`. So the
  srcserver is a hub even for scheduler→redis status fan-out.
* `sourcepublish` notifications (7473-7525) become `events/sourcepublish/<proj>::<pkg>::<md5>[::<md5 of included>]`
  for `bs_sourcepublish`, after access checks and block lists.

### 4.4 Service and deltastore events

`events/servicedispatch/servicedispatch:<proj>::<pkg>::<srcmd5|run>::<servicemark>` (Service.pm:326-351;
fields `srcmd5 rev linksrcmd5 projectservicesmd5 oldsrcmd5 time`), consumed by
`bs_servicedispatch` (`maxchild` 4, per-project `limitinprogress`, retry via `due` +10 min).
`events/deltastore/deltastore:<proj>::<pkg>::<md5-file>` for `bs_deltastore`.

---

## 5. Access control and trust

### 5.1 Who is trusted

* Transport-level: `$BSConfig::ipaccess` (regex → comma-separated auth classes; template:
  localhost and the API host are `rw`, everyone else `worker`, BSConfig.pm.template:34-46) and/or
  `$subjectdnaccess` for TLS client certs, evaluated by `BSStdServer::authorize`
  (BSStdServer.pm:77-106). The dispatch table line `'!rw :' => undef` (7973) sets the default
  required class for every route to `rw`; `'!- GET:'`/`'!- HEAD:'` (7974-7975) drop the
  requirement for all GET/HEAD; individual `!- ` prefixed routes (`/lastevents`,
  `/lasteventsproxy`, `_builddepinfo`) are open regardless of method. Workers reach
  `/getsources`, `/getconfig`, `/getbinaries`… via GET, optionally on a second port
  (`$workersrcserver`, 8233-8240; `port2`/`maxchild2`). `trusted_peerip_header`/`X-Peer` carry
  the original client through the AJAX handoff.
* **There is no per-user authentication or authorization in the backend.** `user=`, `comment=`,
  `requestid=` are plain query parameters recorded verbatim into `.rev` (BSRevision.pm:280-282;
  bs_srcserver:691-698). The Rails API is the policy decision point; it passes `user: user_login`
  on every mutating call (src/api/app/lib/backend/api/sources/package.rb:22-174). Anyone with
  `rw` network access can commit as anyone.
* Signature auth (`setup_authenticator`, BSStdServer.pm:365-373) is for *outgoing* RPCs to other
  OBS instances (`$signature_auth_keyfile`).

### 5.2 Flags the backend does enforce

* `access` / `sourceaccess` (`BSXML::flags`, BSXML.pm:108-117; evaluated with
  `BSUtil::enabled`): `Access::checksourceaccess` (Access.pm:25-40) dies 403 if sourceaccess is
  disabled at project or package level, 404 if `access` is disabled. It is applied only at
  **cross-project boundaries** that could leak content: link targets without a package attribute
  or with `missingok` (Link.pm:555-562, 587-590), project-link resolution (Projlink.pm:96), kiwi
  boot callbacks (1047), branch creation (4304), sourcepublish events (7439-7444), freezelink.
  `addevent` suppresses `lastevents` for `access`-disabled projects (6752-6757). `getprojpack`
  ships `person`/`group` roles for access-protected projects so the *scheduler* can do its own
  check (1381-1386, `mergeroles` walks the project hierarchy).
* `lock`: package `<lock><enable/></lock>` blocks `addrev` (746) and scmsync updates; project/package
  lock flags produce `locked` errors in getprojpack (1407-1418, 1524-1530).
* `downloadbinaries`/`binarydownload`, `publish`, `debuginfo`, `useforbuild` are only passed
  through to scheduler/publisher.
* `scmsync` packages/projects refuse direct writes (747, 2425, 2463, 2849).
* Multibuild and `_product:*` packages are read-only.
* Quota (`$bsquotafile`) is enforced in getprojpack (packages beyond quota get `quota exceeded`).

### 5.3 Signing keys (`Signkey.pm`, bs_srcserver:2135-2239, 6276-6440)

* Keys are generated by the external `$BSConfig::sign` program (`-g algo expiry name email`,
  default `rsa@4096`, 800 days) and stored as meta files `_pubkey` + `_signkey` (0600) of
  `_project` (mrev) and extracted to `projects/<proj>.pkg/`. A 1–2 byte `_signkey`/`_pubkey`
  means "use the default key" (special-case throughout, e.g. 2234, 6308, 6354).
* Lookup walks up the project name hierarchy (`findsignkey`, 6288-6297) — `home:a:b` inherits
  `home:a`'s key. `%vendor` in prjconf is derived from the signing project (2666-2674).
* `extendkey` (`sign -x days`) is called automatically when the key expires in <14 days and a
  consumer asked with `autoextend=1` (6276-6286). `_sslcert` is derived (`sign -C`) and cached in
  the mrev (`projid2sslcert`, 6345-6366), optionally bypassed via `disable_sslcert_cache`.
* `forceprojectkeys` auto-creates a key on project creation (2258-2261) and forbids deleting the
  last key in a hierarchy (2202-2208).
* Consumers: `bs_signer`/publisher (`/getsignkey`, `/getpubkey`), workers (`/getsslcert`), API
  (`_keyinfo`, `_pubkey`). `signtype`/`signflavor` parameters pass through to `sign`.

---

## 6. Source services, scmsync and partitioning

### 6.1 Service run life cycle (new style, Service.pm)

```
commit (addrev)                    tree T = files + "/SERVICE sm"   (.rev gets srcmd5=T)
  └ runservice(cgi, rev, files)    Service.pm:470-603
      ├ addmeta_serialize_servicerun: flock+create <sm>-_serviceerror = "service in progress"
      ├ oldfiles := _service:* of the previous revision's service result (sent so services can be incremental)
      ├ projectservices := getprojectservices (project _service + links + package link chain)
      ├ no _service and no project services → addrev_service(empty) → result tree == T minus service files
      ├ links: sendfiles := handlelinks(rev, ignoreserviceerrors) ; lxservicemd5 := expanded id
      ├ if $servicedispatch: writeservicedispatchevent → return        (daemon does the rest)
      └ else fork child: doservicerpc → addrev_service → exit
doservicerpc(rev, files, send)     Service.pm:383-467
      POST <serviceserver>/sourceupdate/<proj>/<pkg>?timeout=N  body: cpio of files (+ _serviceproject + old _service:*)
      reply: cpio into sources/:upload/runservice$$/ ; ".errors" file => error
      result files must be named _service:* (or _service_error); obs-scm (noprefix) may return
      plain names but never _link/_meta (441) ; _service_<x> small metadata is captured into $rev
      transient errors (5xx, ENOSPC, timeouts) → die "Transient error" → retry (dispatcher)
addrev_service(rev, sm, files, error, lxservicemd5)   Service.pm:202-241
      ok   → addmeta_service: tree <sm>-MD5SUMS = result files + /LSERVICE=T + /LXSERVICE ; unlink marker
      fail → marker file := error text
      notify SRCSRV_SERVICE_{SUCCESS,FAIL}; notify_repservers(package)
```
Readers (`handleservice`) map `T → sm` when tree `sm` exists; `getfilelist` exposes
`serviceinfo code=running|failed|succeeded`. `waitservice` long-polls the marker file.
`bs_servicedispatch` (l.105-155) re-derives everything from the event fields and finishes with
`addrev_service` *in its own process* then tells the srcserver via `cmd=notifypackagechange`
(it shares the filesystem — this is a load-bearing shared-FS assumption).

Service modes (`BSXML::services` comment 1626; Service.pm:92): `localonly`, `disabled`, `manual`,
`buildtime` are not run server side; `trylocal`/default are. `buildtime` services are listed to
the scheduler (`buildtimeservice`, 1990-1997).

### 6.2 obs-scm bridge (`scmsync`)

* `<scmsync>URL[?trackingbranch=…][#branch]</scmsync>` on a package or project meta
  (BSXML.pm:145,171). On meta change the URL is indexed (`ScmsyncDB::storescmsync`) and a run
  is triggered (`runservice_obsscm`, Service.pm:294-324): servicemark is the *constant*
  `md5("obsscm/P/K")` (269-272), a random `run` id is written into the marker file
  (`writeobsscmdata`), and either a `servicedispatch` event with `rev=obsscm`, `srcmd5=<run>` is
  emitted or a child runs the `obs_scm_bridge` service directly (`generate_obs_scm_bridge_service`
  274-285).
* The service output is committed through `commitobsscm` (Service.pm:143-183): lock the marker,
  verify the `run` id still matches (stale runs are dropped), then for a package `addrev` with
  `commitobsscm=1`, `block_identical_commits`, comment suffixed `[info=<git sha>]`; for a project
  `Scmsync::sync_project` (Scmsync.pm:247-341) reads the returned cpio (`<pkg>.xml`, `<pkg>.link`,
  `<pkg>.info`, `_config`, `_scmsync.obsinfo`), brackets the update with
  suspend/resume events, creates/updates/undeletes/deletes packages (`sync_package`, 135-213),
  syncs `_link` files, config, and records the project commit id in `_project.rev`'s comment.
  From the dispatcher the commit goes over HTTP (`cmd=commitobsscm`, bs_servicedispatch:51-77).
* `scminfo` (last git sha) is exported to the scheduler in getprojpack (1235-1239).
* Webhook entry: `POST /source?cmd=triggerscmsync&scmrepository=&scmbranch=[&isdefaultbranch]`
  (400-430) → every package/project mapped to that repo/branch is re-run. Requires sqlite.

### 6.3 Partitioning (`Partition.pm`, BSConfig.pm.template:234-252)

This is the existing sharding mechanism and it shards **only the binary side**:

```perl
our $partition = 'home';                                  # name of the partition this *repserver host* serves
our $partitioning = [ 'home:' => 'home', '.*' => 'main' ];# ordered (regex-prefix => partition)
our $partitionservers = { home => 'http://home-backend:5252', main => 'http://main-backend:5252' };
our $workerpartitionservers = { ... };                    # optional alternative URLs for workers
```
* `projid2partition` / `projid2reposerver` (Partition.pm:26-66): first regex matching `^$re`
  wins; fallback `$partition`; error if unknown. **Every** `/build`, `/published`, event,
  registry, SLSA, relsync, copy and move operation in bs_srcserver resolves the target repserver
  with this (dozens of call sites of the form
  `$BSConfig::partitioning ? projid2reposerver($projid) : $BSConfig::reposerver`).
* Schedulers call `getprojpack?partition=<mine>` and get only projects of their partition; other
  local projects referenced from search paths, project links, aggregates or package links are
  returned in the `remotemap` with `partition=<other>` and `remoteurl=<that repserver>` plus
  repository list/flags/roles/config (`checkpartition`, Partition.pm:68-105; bs_srcserver:1149-1172,
  1264-1289, 1326, 1641-1654, 1712, 2640-2651, 2697-2700). The scheduler then treats them like
  interconnect projects for binaries.
* Constraints: project move/copy-with-binaries across partitions is refused (3744-3746,
  3795-3797); `getworkerstatus`, `published` root, `putconfiguration`, `registry_manifestinfo`
  fan out to all partition servers; `notify_all_repservers` broadcasts.
* Sources are **not** partitioned: "There is still just one source server" (template:237).

---

## 7. Concurrency and consistency

### 7.1 Process model

* Main server: pre-fork-per-connection (`BSServer::server`, BSServer.pm:328-500): `select` on the
  listen socket(s), `fork()` per accepted connection, `maxchild` 20 (configurable, 8216, 8230),
  separate `maxchild2` for the worker port. Each child handles **one request** (plus optional
  keep-alive for `/notify/redis`) and exits. No shared memory → every per-process cache
  (`%packagequota`, `@lastev_cache`, `$readproj_local_cache`, `%remote_getrev_cache`,
  `%prp_to_repoinfo`, `$multibuildcache`, frozenlinks cache) lives only for one request.
* AJAX helper: 1 + N single-threaded event-loop processes (`BSEvents::schedule`, `select`-based,
  BSEvents.pm:120-183) listening on `run/bs_srcserver.ajax[N]`. Children hand their client socket
  over via `SCM_RIGHTS` (`BSHandoff`, BSHandoff.pm:32-99) and exit; the AJAX process then owns the
  connection. Handlers in AJAX are written in a **re-entrant continuation style**: any blocking
  need (`BSWatcher::rpc`, `addfilewatcher`, `serialize`) returns `undef` and the whole handler is
  re-invoked later with the same `$cgi` (`redo_request`, BSWatcher.pm:83-101), memoizing results
  in the job event hash (`$jev->{...}`, e.g. `fetchremoteproj_$projid`, `rpcdone`). State that
  must survive re-invocation is stuffed into `$cgi` (e.g. `start_orig`, `req_time`, 6677-6678).
  `BSWatcher::background` detaches a job from its client (2907).
* Detached workers: `xfork()` children for service runs (Service.pm:313, 584), product conversion
  (`exec bs_productconvert`), patch/diff3/merge (`exec`), undelivered-event drainer (7957), redis
  keep-alive (7412-7423). Project delete replies before deleting (2311-2312).
* External daemons sharing the tree: `bs_servicedispatch`, `bs_deltastore`, `bs_sourcepublish`,
  `bs_redis`, `bs_notifyforward`, and bs_repserver/scheduler/publisher on the *same host* in the
  single-partition setup (repserver writes `events/<arch>` into the same `$bsdir`).

### 7.2 Atomicity and locking primitives

* **Atomic file replace**: `writestr(tmp, final)`/`writexml` = write tmp, optional `fdatasync`
  (unless `disable_data_sync`, 133), `rename` (BSUtil.pm:71-96). Used for trees, meta XML,
  event files, caches.
* **Append-only logs with flock**: `.rev`/`.mrev`/`lastevents`/`lastnotifications` via
  `BSFileDB::lockopen` (`+>>`, `LOCK_EX`, re-open loop if the inode vanished, BSFileDB.pm:29-39).
  `fdb_add_i` computes the next number inside the lock. Pruning rewrites the file via rename
  under the same lock (`prune_lastnotifications`, 192-207; `prunelastevents`).
* **Lock files**: `BSUtil::lockopen`/`lockcheck` (BSUtil.pm:444-477) on: `.mrev` (meta
  commit critical section, BSRevision.pm:250-302), `:multibuild` (Multibuild.pm:125,147,172),
  `<sm>-_serviceerror` (service serialization), `diffcache/<id>.run`, `remotecache/lock`,
  `registry/:repos`, `events/undelivered/.lock`, BSDBIndex per-file locks, daemon run locks.
  **The `.rev` append for ordinary commits is only protected by the `fdb_add_i` flock on the file
  itself**; the tree write (`addmeta`) and the blob adds happen *before* and are idempotent, so
  two concurrent commits to one package both succeed with consecutive revs (last writer's tree
  becomes "latest").
* **Idempotent content stores**: blobs/trees are write-once by name; concurrent writers race
  benignly (`rename` of identical content; `addmeta_service` verifies equality).
* **Hard links** (`link()`) are used for copies between package dirs, for `applylink` inputs,
  deltastore staging, and link error logs — **requires one filesystem** for `sources/`,
  `trees/`, `:upload/`.
* `$$`-based temp names assume one in-flight operation per PID.

### 7.3 What assumes a single node / POSIX FS

* `bs_servicedispatch`, `bs_deltastore`, `bs_sourcepublish`, the product converter and (in the
  simple setup) the repserver/scheduler read and write `$bsdir` directly.
* `flock`, `rename`, `link`, `.ping` FIFOs, `stat`-polling file watchers (`addfilewatcher`
  compares `mtime/size/inode` every second).
* `BSHandoff` passes **file descriptors** over a Unix socket between the main children and AJAX.
* `projects/_repoid` identifies the data store to schedulers (`repoid` in projpack, 2062).
* The hot path "is this package local?" is `-e projects/<proj>.pkg/<pkg>.xml`
  (BSRevision.pm:82, Service.pm:648, dozens of places).
* `getprojpack` without `project=` enumerates **all** projects by `readdir` (1148) and reads every
  package meta/rev — cold-start of a scheduler is an O(instance) scan.

### 7.4 Consistency hazards present today (must be understood, not necessarily reproduced)

* Meta XML is renamed into place *after* the mrev record; readers of `<pkg>.xml` can lag.
* `putproject` decides whether to notify the scheduler by comparing old vs new meta minus
  cosmetic fields (2263-2275) — a concurrent `putproject` may lose a notification.
* `getrev_local` with a 32-hex id does not check tree existence (BSRevision.pm:91-93) — later
  `lsrev` dies with a 500-ish "not in repository" message.
* `addrev` reads `rev_old` *before* appending (773), so `acceptinfo.osrcmd5` and
  `SRCSRV_VERSION_CHANGE` are computed against a possibly stale revision under concurrency.
* `copyproject withhistory` is explicitly "FIXME: races ahead" (3841).
* Service runs started from a *read* path (`handleservice`) may run in a request child; if the
  child is killed the marker stays "service in progress" until a re-trigger.
* `undelivered/` retry covers only `project|package|lowprioproject` events.

---

## 8. Invariants and gotchas a reimplementer must preserve

1. **Blob identity = md5 of content; blob path = `sources/<packid>/<md5>-<name>`** (package
   scoped, not project scoped). md5 collisions with different content are fatal; SHA-256 is used
   only as the collision detector (BSSrcrep.pm:158-167) and in `commitfilelist` `hash=` validation.
2. **srcmd5 = md5 of `"<md5>  <name>\n"` lines sorted by name (byte order, two spaces)**.
   Empty tree = `d41d8cd98f00b204e9800998ecf8427e`. Clients (osc) compute the same value.
3. **Expanded link trees are stored under a lineage hash, not a content hash**:
   `xsrcmd5 = md5("<target>  /LINK\n<local>  /LOCAL\n")`, chained from the far end; the tree
   file includes `/LINK`,`/LOCAL`. `verifymd5` = content hash of the expanded list. Service result
   trees are stored under the servicemark with `/LSERVICE` (+`/LXSERVICE`). Committed trees that
   need a service carry `/SERVICE <sm>` inside the hashed content. Any reimplementation must
   reproduce these ids exactly or every scheduler/osc/interconnect peer will see "new sources".
4. **The servicemark is `md5("sourceservice/P/K/<unixtime>\n<prev rev no><file list>")`** —
   intentionally unique per commit; obs-scm uses the constant `md5("obsscm/P/K")`.
5. `.rev` numbering is dense 1..N per package; `vrev` is per-`version` monotonic with trailing
   integer arithmetic and optional non-numeric prefixes (`"7.1"`, extend-mode `"N.1"`);
   version/vrev of a link revision are `unknown`/`0` locally and computed at expansion time
   (`cicount` rules). `time` must be non-decreasing when supplied.
6. Special names: `_project`, `_pattern`, `_product`, `_patchinfo`, `_product:<x>`,
   `_patchinfo:<x>`, `<pkg>:<flavor>` (multibuild, read-only, flavor `[^_./:]…`), `_repository`
   (binary namespace). Project names: `[^/\0-\37]`, no component starting with `_` or `.`, ≤200
   chars; package names additionally no `:`; file names: no `/`, no control chars, not starting
   with `.`. (BSVerify.pm:28-61,127-132). Revision ids: integer, 32-hex, 40-hex (git), `upload`,
   `build`, `latest`, `repository`; linkrev additionally `base` (152-165).
7. Reserved file names inside a package: `_link`, `_service`, `_service:*`, `_service_*`,
   `_serviceerror`, `_service_error`, `_linkerror`, `_aggregate`, `_multibuild`, `_patchinfo`,
   `_channel`, `_constraints`, `_buildenv*`, `_modulemd.yaml`, `_preinstallimage`, `_scmsync.obsinfo`,
   `*.product`, `*.obscpio` (delta-compressible), `/SERVICE` & co. (never real files).
   Services may not produce `_link`/`_meta`; patches may not create `_link`.
8. Meta files are versioned separately (`.mrev`) and `_meta` for a package is *the* package
   XML (name attribute must match, `verify_pack`).
9. Project links are resolved in order, transitively, with frozenlinks overriding; packages
   obtained that way get `originproject` and their trees are imported into the consuming project.
10. Access checks happen at specific cross-project edges (§5.2); a redesign that centralizes
    authorization must still produce `403 source access denied` / `404` at those points because
    osc and the API rely on the distinction.
11. `getprojpack` output (`BSXML::projpack`) is the scheduler contract: package `srcmd5`
    (servicemark/xsrcmd5), `verifymd5`, `rev`, `versrel = "<version>-<vrev>"`, `linked`,
    `originproject`, per-repo `info` with `error` codes (`disabled`,`excluded`,`locked`,
    `broken link…`), `remotemap` with `partition`, `channeldata`, `repoid`. Storable transport
    (`view=storable`) is the default used by schedulers.
12. `lastevents`/`lastnotifications` are monotonically numbered streams with explicit
    `sync=lost` semantics and a single API-side cursor; the heartbeat every 100 events / 5 min is
    relied upon by schedulers to prune dead peers.
13. Deletion keeps full history in `.del` files and undelete restores *the last generation*
    with a new revision number continuing the old sequence.
14. `upload` is a per-package *single* uncommitted working tree (not per user).
15. Notifications to repservers are fire-and-forget HTTP with a file retry for three types;
    schedulers are expected to tolerate duplicates and to rescan on `project` events.
16. `_repoid` must stay stable across restarts; schedulers detect a changed source store by it.
17. Diffs are produced by external `diff`/`diff3`/`patch` binaries with specific flags
    (`diff -up --label`, `diff3 -m -E`, `patch --no-backup-if-mismatch -g 0 -f`); output
    formats of `sourcediff` are consumed by the API/web UI.

---

## 9. Implications for a distributed (Go microservice) redesign

### 9.1 Natural ownership boundaries

**Per-package (finest shard)**: blobs (`sources/<pkg>`), trees, `.rev`/`.mrev`, upload tree,
service markers/results, link error logs, `:multibuild` entry, deltastore. Everything in
`addrev`, `putfile`, `commitfilelist`, service runs, `getfilelist` (unexpanded) is package-local.
Note blobs are currently shared across *projects* by package name; a per-project shard must
either keep a global blob store (content-addressed, so trivially global and dedup-friendly) or
accept blob copies on cross-project operations (the code already models "copy into my project"
via `copyfiles`/`copytree`).

**Per-project**: project meta/config/mrev, `_project` sources (`_service`, `_config`),
frozenlinks, keys/certs, package listing, multibuild cache, scmsync entries, quota.

**Global (today) and inherently cross-cutting**:
* `lastevents`, `lastnotifications`, `notificationpayload` (single sequence numbers; the API has
  one cursor) — becomes a global ordered log/stream service.
* `db/sqlite/source` (linkinfo reverse index, scmsync URL index) and the published index — global
  secondary indexes; rebuildable, so they can be a separate indexing service fed by change events.
* `registry/:repos`, `configuration.xml`, `issuetrackers.xml`, `_repoid`, partition map,
  quota file — small global config.
* `remotecache`, `diffcache` — caches, shardable by key.
* The project namespace itself (`findprojects`, search by XPath over all meta) — needs a catalog.

### 9.2 Cross-project operations (the hard part)

| Operation | Reads from other shards | Writes to other shards |
|---|---|---|
| `_link` expansion (`handlelinks`) | target package's (service-expanded) tree + blobs, transitively, possibly remote | writes the expanded tree + hard-linked blobs **into the linking package** |
| project links (`getrev_projlink`, `findpackages`) | package list/meta/rev of linked projects (transitive) | `copytree` into the consuming project |
| `keeplink`/`integratelink`/`sourcecopy` auto-simplify | target package history | **commits to the origin package** (`buildservice-autocommit`), possibly in another project |
| `branch`/`copy`/`copyproject` | origin trees/blobs/history | commits in destination (+ origin for makeoriginolder/freezelink) |
| `getprojectservices` | `_service` of linked projects and link-target packages | — |
| `getprojpack` | *all* projects of a partition + search-path projects + link/aggregate targets | updates `:multibuild` caches of every project it touches |
| `concatconfigs`/`expandsearchpath` | prjconf of every project on a repository path (incl. remote/other partition) | — |
| `_aggregate`, `_channel`, kiwi/product paths | existence/meta of referenced projects | — |
| `sourcediff` | two arbitrary package revisions | diffcache |
| signing key lookup | parent projects' `_signkey` | auto-extend writes to the *parent* project's mrev |
| `search/package` (linkinfo), `getresult locallink` | global reverse index | — |
| `scmsync` project sync | — | creates/deletes packages in bulk (bracketed by suspend/resume) |
| `delproject` | — | moves many package logs; must be atomic from the scheduler's view |

Design consequences:
* Link expansion needs a read API on the target shard that returns "service-expanded tree +
  blob access" and an access decision; the expanded result is cached in the *linking* shard.
  Chained links and project links make this recursive and potentially remote; the existing
  `collect_remote_getrev` two-pass batching in `getprojpack` shows the latency problem is already
  real.
* The auto-simplify / makeoriginolder / freezelink write-backs to the origin package are
  cross-shard **writes inside one API call**. They are "best effort" today (`eval`+warn,
  4262-4267); they could become asynchronous follow-up commands, but the request-acceptance flow
  in the API expects the origin to look clean immediately afterwards.
* `getprojpack` is a fan-in over an entire partition; a distributed design should turn it into
  per-project materialized views pushed to schedulers (the `remotemap`+`partition` mechanism is
  effectively that already for cross-partition references).

### 9.3 Where the shared filesystem is load-bearing

1. `bs_servicedispatch`/`bs_deltastore`/`bs_sourcepublish` and `bs_productconvert` operate on the
   store directly and finish transactions (`addrev_service`, tree writes) without going through
   the server. The obs-scm path already shows the HTTP alternative (`cmd=commitobsscm`).
2. Repserver → scheduler delivery via `events/<arch>` files and `.ping` (same `$bsdir` in the
   simple layout); the srcserver → repserver edge is already HTTP.
3. AJAX handoff via Unix socket + fd passing; file watchers (`addfilewatcher`) implement
   long-poll by polling `stat` on `lastevents`, `lastnotifications`, `_serviceerror`,
   `diffcache`, `projects/<proj>.xml`.
4. Hard links between `sources/<pkgA>` and `sources/<pkgB>`, and `trees/` vs `sources/`.
5. Locks: `flock` on `.mrev`, `.rev`, `:multibuild`, service markers, diffcache `.run`,
   registry map, BSDB files.
6. `-e <file>` as the existence oracle for projects/packages/trees/blobs everywhere, including in
   modules used by other daemons (`BSRevision`, `BSSrcrep`).
7. `:upload` scratch space shared between request children, AJAX, service RPC receivers and the
   dispatcher.

### 9.4 Suggested decomposition (derived from the code, not prescriptive)

* **Blob service** (global, content-addressed by md5, sha256 side-check, obscpio delta encoding
  as an internal storage detail, cpio streaming for `getsources`/`view=cpio`).
* **Source shard service** (per project or project-range): trees, revisions, meta revisions,
  upload tree, link expansion cache, service markers, multibuild cache; exposes
  `getrev`/`lsrev`/`lsrev_expanded`/`addrev`/history/undelete. Must reproduce ids from §8 exactly.
* **Link/expansion orchestrator** living with the *linking* shard, calling target shards (or
  interconnect) for "expanded tree + blob handles" — mirroring `$getrev`/`$lsrev_linktarget`
  indirection points that the current code already has (`BSSrcServer::Link::getrev`,
  `lsrev_linktarget`, `Projlink::getrev`, `Service::getrev`, `Remote`).
* **Service runner** (replace `bs_servicedispatch` + `bs_service` direct FS access with
  commit-over-RPC as obs-scm already does).
* **Catalog/index service**: project/package listing, linkinfo reverse index, scmsync index,
  published index, XPath search, quota.
* **Event/notification streams**: `lastevents` (filterable, numbered, bounded retention) and
  `lastnotifications` (API cursor, payload store), scheduler notifications with durable retry
  (today's `undelivered/`), redis/sourcepublish fan-out.
* **Build gateway** = today's `/build`, `/published`, `/registry`, worker routes: pure routing by
  partition map plus the interconnect binary cache and request coalescing (`serialize`).
* **Keys/signing**: per-project key material with hierarchy lookup and auto-extend; needs the
  external `sign` daemon.

The config-derived behaviours to carry over: `nosharedtrees` (per-project tree namespacing =
authorization witness), `old_style_services` (legacy path, probably droppable), `servicedispatch`,
`partitioning`, `ipaccess`, `forceprojectkeys`, `disable_data_sync`, `limit_projects`,
`sourcepublish_*`, `extramacros`/`extraconfig`, `srcserver_ajaxpartitions`, `obsname`.
