# 04 — Storage

Depends on: `02-domain-model.md`, `03-architecture.md`.

Two stores, one rule: **object storage holds content, PostgreSQL holds
state and every pointer to content.** Nothing in object storage is mutated;
every logical change is a new object plus a row update in one transaction.

## 1. Object storage

### 1.1 Buckets and prefixes

One logical namespace with prefixes (buckets may be split per prefix by the
operator for lifecycle or regional placement):

| Prefix | Content | Immutable? | Lifecycle |
|---|---|---|---|
| `blobs/sha256/<2>/<2>/<sha256>` | source file content | yes | refcounted GC (§ 1.4) |
| `blobs/md5/<md5>` | **pointer object** (small JSON `{sha256, size}`) for md5 lookups used by `commitfilelist` / legacy ids | yes | GC with blob |
| `trees/<srcmd5>` | canonical tree text (incl. pseudo entries) | yes | GC when no revision/expansion references it |
| `obscpio/<sha256>.delta` | delta-compressed form of a `.obscpio` blob (internal optimisation, see § 1.5) | yes | with blob |
| `bins/sha256/<2>/<2>/<sha256>` | built binaries (rpm, deb, …) and any job result file | yes | refcounted GC |
| `jobs/<job-id>/result.tar` | raw result bundle as uploaded by a worker (unpacked into `bins/` by artifact-svc, then deleted) | yes | 7 days |
| `logs/<prpa>/<package>/<job-id>.log[.zst]` | build logs | yes | per-instance retention (default: keep last success and last failure forever, others 90 days) |
| `pools/<shard>/<prpa>/full/<version>.solv` | solv index of `:full` at pool version | yes | keep last 3 versions |
| `pools/<shard>/<prpa>/full/<version>.json` | `:full` membership (name → bin sha256, meta sha256) | yes | keep last 3 |
| `meta/<prpa>/<package>/<metamd5>` | `.meta` files | yes | keep referenced |
| `published/<prp>/<version>/...` | one published tree version (metadata, signatures, links to bins by digest) | yes | keep last 2 + pinned |
| `published-live/<prp>` | **pointer** to the current published version (small JSON) | overwritten | — |
| `registry/blobs/<digest>`, `registry/manifests/<repo>/<digest>` | OCI content | yes | GC by tag refs |
| `dod/<repo-id>/<version>/...` | mirrored DoD metadata and cached packages | yes | keep last 2 |
| `cache/remote/<sha256>` | binaries fetched from remote OBS instances | yes | LRU by last access row |
| `diffcache/<cacheid>` | rendered source diffs | yes | 7 days |

Published trees are served to the public through a CDN or a thin
`published-gw` that resolves `published-live/<prp>` and 302s or proxies to the
versioned path, so mirrors and clients see an atomically switched repository.

### 1.2 Write protocol

1. Writer uploads the object (multipart for > 64 MiB). PUTs are idempotent by
   key because keys are digests.
2. Writer verifies the digest of what it uploaded (S3 `ETag` is not trusted
   for multipart; the writer computes sha256 while streaming).
3. Writer commits the PostgreSQL row(s) that reference the object.
4. If step 3 fails, the object is orphaned and reclaimed by GC.

Readers never see a row that references an absent object.

### 1.3 Presigned access

Workers, Rails (through compat-edge), and publishers obtain time-limited
presigned GET/PUT URLs from the owning service. No service proxies large
bodies when a presigned URL will do. compat-edge proxies bodies only when the
legacy API requires a direct response (for example `GET /source/P/K/file`),
and streams them.

### 1.4 Garbage collection

`gc-svc` (part of `artifact-svc` and `source-svc`) runs mark-and-sweep against
PostgreSQL reference tables (`blob_ref`, `bin_ref`) on a schedule. Objects
younger than 24 h are never swept (write-protocol grace). Sweeps are
rate-limited and resumable.

### 1.5 Large source archives

`.obscpio` service outputs are stored as full objects. The delta-store
optimisation of `bs_deltastore` is replaced by object-storage economics plus
optional zstd dictionary compression inside `source-svc` (an implementation
detail invisible to clients). The importer expands any delta-stored obscpio
it finds.

## 2. PostgreSQL

### 2.1 Conventions

- All tables carry `created_at`, `updated_at`.
- IDs are `bigint` identity except where the natural key is the identifier.
- `shard_id` is denormalised onto every scheduler-owned table so a shard's
  state can be selected and moved with one predicate.
- Partitioning: scheduler tables are `PARTITION BY HASH (shard_id)`;
  source tables by `project_id`. Operators may map partitions to Citus or
  CockroachDB without schema change.
- Large JSON documents (`buildinfo`, `bininfo`) are `jsonb`; XML is never
  stored, it is rendered by compat-edge.
- Every mutating RPC writes an `idempotency` row `(key, result_digest,
  expires_at)` in the same transaction.

### 2.2 Source schema (source-svc)

```sql
CREATE TABLE project (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name          text NOT NULL UNIQUE,
  kind          text NOT NULL DEFAULT 'standard',
  meta_mrev     int  NOT NULL,              -- current meta revision number
  meta          jsonb NOT NULL,             -- parsed BSXML::proj (canonical JSON form)
  config_rev    int,                        -- _project rev holding current _config
  config_text   text,                       -- current prjconf
  remote_url    text, remote_project text,  -- interconnect
  scmsync       text,
  deleted_at    timestamptz,
  shard_id      int NOT NULL                -- scheduler shard (see shard_map)
);
CREATE INDEX ON project (name text_pattern_ops);   -- prefix search (starts-with)

CREATE TABLE package (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  project_id  bigint NOT NULL REFERENCES project(id),
  name        text NOT NULL,
  meta_mrev   int NOT NULL,
  meta        jsonb NOT NULL,
  head_rev    int NOT NULL DEFAULT 0,
  scmsync     text,
  deleted_at  timestamptz,
  UNIQUE (project_id, name)
);

CREATE TABLE blob (
  sha256   bytea PRIMARY KEY,
  md5      bytea NOT NULL,
  size     bigint NOT NULL,
  UNIQUE (md5)                      -- md5 collision with different sha256 is a hard error
);

CREATE TABLE tree (
  srcmd5     bytea PRIMARY KEY,     -- md5 of canonical text (incl. pseudo entries)
  entries    jsonb NOT NULL,        -- [{name, md5}] sorted; pseudo entries included
  verifymd5  bytea,                 -- content srcmd5 of real entries (NULL if same as srcmd5)
  kind       text NOT NULL          -- 'plain' | 'link-expanded' | 'service' | 'upload'
);
CREATE TABLE tree_blob (tree_srcmd5 bytea REFERENCES tree, blob_sha256 bytea REFERENCES blob, name text,
                        PRIMARY KEY (tree_srcmd5, name));

-- package-scoped witness that a tree is legitimately visible in a project (replaces trees/<proj>/<pkg>/ dir)
CREATE TABLE package_tree (
  package_id bigint REFERENCES package(id), srcmd5 bytea REFERENCES tree,
  PRIMARY KEY (package_id, srcmd5)
);

CREATE TABLE revision (          -- .rev
  package_id bigint NOT NULL REFERENCES package(id),
  rev        int NOT NULL,
  vrev       text NOT NULL,
  srcmd5     bytea NOT NULL REFERENCES tree,
  version    text,
  time       bigint NOT NULL,
  "user"     text, comment text, requestid text,
  PRIMARY KEY (package_id, rev)
);
CREATE TABLE meta_revision (     -- .mrev
  owner_kind text NOT NULL,      -- 'project' | 'package'
  owner_id   bigint NOT NULL,
  mrev       int NOT NULL,
  srcmd5     bytea NOT NULL REFERENCES tree,   -- tree of {_meta,_pubkey,_signkey,_sslcert,_attribute,_frozenlinks}
  time bigint NOT NULL, "user" text, comment text, requestid text,
  PRIMARY KEY (owner_kind, owner_id, mrev)
);
CREATE TABLE revision_deleted (LIKE revision INCLUDING ALL, generation int NOT NULL);   -- .rev.del
CREATE TABLE meta_revision_deleted (LIKE meta_revision INCLUDING ALL, generation int NOT NULL);

CREATE TABLE upload_tree (package_id bigint PRIMARY KEY REFERENCES package(id), srcmd5 bytea REFERENCES tree);

CREATE TABLE link_expansion (     -- cache of handlelinks results, keyed by lineage
  package_id bigint NOT NULL, xsrcmd5 bytea NOT NULL,
  lsrcmd5 bytea NOT NULL, target_srcmd5 bytea NOT NULL,
  error text,                    -- NULL on success; else the _linkerror text
  PRIMARY KEY (package_id, xsrcmd5)
);

CREATE TABLE service_run (
  package_id  bigint NOT NULL, servicemark bytea NOT NULL,
  state       text NOT NULL,     -- 'pending' | 'running' | 'succeeded' | 'failed'
  lsrcmd5     bytea, lxsrcmd5 bytea,
  result_srcmd5 bytea REFERENCES tree,
  error       text, run_id text, attempts int DEFAULT 0, lease_until timestamptz,
  PRIMARY KEY (package_id, servicemark)
);

CREATE TABLE linkinfo (           -- reverse index "who links to X"
  source_project text, source_package text, project text, package text, rev text,
  PRIMARY KEY (source_project, source_package)
);
CREATE INDEX ON linkinfo (project, package);

CREATE TABLE scmsync_index (project text, package text, repo text, branch text, tracking text,
                            PRIMARY KEY (project, package));
CREATE INDEX ON scmsync_index (repo, branch);

CREATE TABLE multibuild_cache (package_id bigint PRIMARY KEY, md5 bytea, flavors jsonb);

CREATE TABLE signkey (
  project_id bigint PRIMARY KEY REFERENCES project(id),
  pubkey text, privkey_ref text,   -- privkey_ref: KMS/HSM/signd handle or sealed secret id; never plaintext
  sslcert text, expires_at timestamptz, algo text
);

CREATE TABLE lastevents (         -- interconnect / long-poll stream
  number bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  time bigint, type text, project text, package text, repository text, arch text
);
CREATE TABLE lastnotifications (  -- API feed
  number bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  time bigint, type text NOT NULL, data jsonb NOT NULL, payload_sha256 bytea
);
CREATE TABLE notification_cursor (consumer text PRIMARY KEY, number bigint NOT NULL);

CREATE TABLE remote_tree_cache (project_id bigint, package text, srcmd5 bytea, fetched_at timestamptz,
                                PRIMARY KEY (project_id, package, srcmd5));
CREATE TABLE registry_owner (regrepo text PRIMARY KEY, prp text NOT NULL);
CREATE TABLE instance_config (key text PRIMARY KEY, value jsonb);   -- configuration.xml, issuetrackers.xml, repoid, quota
```

Revision numbering: `rev` is assigned with `SELECT head_rev FROM package
WHERE id=$1 FOR UPDATE`, incremented, and inserted in the same transaction.
`vrev` is computed in the same transaction by reading the last revision with
the same `version` (index on `(package_id, version, rev DESC)`).

### 2.3 Build schema (scheduler, artifact-svc, dispatcher)

```sql
CREATE TABLE shard_map (
  shard_id int PRIMARY KEY, name text, storage_zone text, weight int DEFAULT 100
);
CREATE TABLE shard_override (      -- prp/prpa granularity overrides
  project text, repository text, arch text, shard_id int REFERENCES shard_map,
  PRIMARY KEY (project, repository, arch)
);
CREATE TABLE shard_lease (shard_id int PRIMARY KEY, owner text NOT NULL, expires_at timestamptz NOT NULL, event_cursor bigint);

CREATE TABLE prpa (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  project text NOT NULL, repository text NOT NULL, arch text NOT NULL,
  shard_id int NOT NULL,
  repo_state text NOT NULL DEFAULT 'scheduling', repo_details text,
  dirty boolean NOT NULL DEFAULT true,
  buildid text, oldbuildid text, repostateid text,
  full_version bigint NOT NULL DEFAULT 0,     -- bumps whenever :full changes
  bininfo_version bigint NOT NULL DEFAULT 0,
  repodone_md5 bytea,
  last_check_started timestamptz, last_check_duration interval, next_med timestamptz, next_high timestamptz,
  UNIQUE (project, repository, arch)
) PARTITION BY HASH (shard_id);

CREATE TABLE package_status (
  prpa_id bigint REFERENCES prpa(id), package text,
  code text NOT NULL, error text,
  overlay_code text, overlay_details text,      -- finished/dispatching/signing + dispatch details
  shard_id int NOT NULL,
  PRIMARY KEY (prpa_id, package)
) PARTITION BY HASH (shard_id);

CREATE TABLE build_meta (          -- :meta/<pkg> and .meta.success pointers
  prpa_id bigint, package text, meta_sha256 bytea NOT NULL, success_meta_sha256 bytea,
  verifymd5 bytea, metamd5 bytea, lastcheck bytea,   -- lastcheck = the 128-byte decision cache
  shard_id int NOT NULL, PRIMARY KEY (prpa_id, package)
) PARTITION BY HASH (shard_id);

CREATE TABLE package_result (      -- files of the last build per package (the <gdst>/<pkg>/ dir)
  prpa_id bigint, package text, filename text, bin_sha256 bytea NOT NULL, size bigint,
  kind text,   -- 'binary' | 'meta' | 'log' | 'statistics' | 'provenance' | 'buildenv' | 'import' | 'other'
  shard_id int NOT NULL, PRIMARY KEY (prpa_id, package, filename)
) PARTITION BY HASH (shard_id);

CREATE TABLE bininfo (             -- per-package parsed binary index (gbininfo)
  prpa_id bigint, package text, info jsonb NOT NULL, outdated boolean DEFAULT false,
  shard_id int NOT NULL, PRIMARY KEY (prpa_id, package)
) PARTITION BY HASH (shard_id);

CREATE TABLE full_pool (           -- :full membership, one row per binary name
  prpa_id bigint, binname text, bin_sha256 bytea NOT NULL, meta_sha256 bytea,
  owner_package text NOT NULL,     -- or '_volatile'
  hdrmd5 bytea, shard_id int NOT NULL,
  PRIMARY KEY (prpa_id, binname)
) PARTITION BY HASH (shard_id);
CREATE TABLE full_useforbuild (prpa_id bigint PRIMARY KEY, packages jsonb);
CREATE TABLE full_index (prpa_id bigint, version bigint, solv_key text, members_key text, PRIMARY KEY (prpa_id, version));

CREATE TABLE repo_pool (           -- :repo staging for publisher
  prpa_id bigint, filename text, bin_sha256 bytea NOT NULL, origin_package text,
  shard_id int NOT NULL, PRIMARY KEY (prpa_id, filename)
) PARTITION BY HASH (shard_id);
CREATE TABLE repo_info (prpa_id bigint PRIMARY KEY, binaryorigins jsonb, conflicts jsonb, version bigint);

CREATE TABLE relsync (prpa_id bigint, package text, versrel_bcnt text, PRIMARY KEY (prpa_id, package));
CREATE TABLE relsync_max (prp text, tag_versrel text, max_bcnt int, PRIMARY KEY (prp, tag_versrel));
CREATE TABLE depends (prpa_id bigint PRIMARY KEY, pkgdeps jsonb, subpacks jsonb, pkg2src jsonb, cycles jsonb, sccs jsonb);
CREATE TABLE genbuildreqs (prpa_id bigint, package text, md5 bytea, deps jsonb, verifymd5 bytea, PRIMARY KEY (prpa_id, package));
CREATE TABLE preinstallimage (prpa_id bigint, package text, info jsonb, PRIMARY KEY (prpa_id, package));

CREATE TABLE jobhistory (
  prpa_id bigint, seq bigint, package text, rev text, srcmd5 bytea, versrel text, bcnt int,
  readytime bigint, starttime bigint, endtime bigint, code text, uri text, workerid text, hostarch text, reason text,
  PRIMARY KEY (prpa_id, seq)
);

CREATE TABLE job (
  id uuid PRIMARY KEY, name text NOT NULL UNIQUE,
  prpa_id bigint NOT NULL, package text NOT NULL, srcmd5 bytea, verifymd5 bytea, versrel text, bcnt int,
  buildinfo jsonb NOT NULL, constraints jsonb, priority int NOT NULL DEFAULT 0,
  state text NOT NULL,   -- scheduled|dispatching|building|finished|integrated|killed|badhost
  worker_id text, attempt int DEFAULT 0, lease_until timestamptz,
  readytime timestamptz NOT NULL, starttime timestamptz, endtime timestamptz,
  result_key text, result_code text,   -- succeeded|failed|unchanged|badhost|genbuildreqs
  dispatch_details text, shard_id int NOT NULL
);
CREATE INDEX ON job (state, prpa_id);
CREATE INDEX ON job (worker_id) WHERE state IN ('dispatching','building');

CREATE TABLE worker (
  id text PRIMARY KEY, hostarch text NOT NULL, archs text[] NOT NULL, sandbox text, class text, zone text,
  hardware jsonb, hostlabel text, state text NOT NULL, job_id uuid, last_seen timestamptz NOT NULL,
  cache_filter bytea, version text
);

CREATE TABLE publish_job (
  id uuid PRIMARY KEY, prp text NOT NULL, state text NOT NULL,   -- requested|running|published|failed
  requested_at timestamptz, started_at timestamptz, finished_at timestamptz, attempt int, error text,
  repoinfo_versions jsonb,      -- prpa_id -> repo_info.version consumed
  published_version text        -- published/<prp>/<version>
);
CREATE TABLE published_repo (prp text PRIMARY KEY, version text NOT NULL, repotypes text[], state jsonb, published_at timestamptz);
CREATE TABLE published_binary (prp text, name text, path text, package text, arch text, version text, release text,
                               PRIMARY KEY (prp, path));
CREATE INDEX ON published_binary (name);
CREATE TABLE published_pattern (prp text, path text, package text, name text, summary text, description text, type text, json jsonb,
                                PRIMARY KEY (prp, path));

CREATE TABLE dod_repo (id bigint PRIMARY KEY, prpa_id bigint, url text, repotype text, archfilter text, master jsonb, pubkey text,
                       cookie text, last_update timestamptz, version bigint);
```

### 2.4 Transactions that must be atomic

| Operation | Rows touched in one transaction |
|---|---|
| source commit | `blob`, `tree`, `tree_blob`, `package_tree`, `revision`, `package.head_rev`, `upload_tree` (delete), `linkinfo`, `lastnotifications`, `idempotency` |
| meta commit | `tree`, `meta_revision`, `project.meta`/`package.meta`, `signkey`, `lastnotifications` |
| job integration | `package_result`, `build_meta`, `bininfo`, `full_pool`, `prpa.full_version`, `job.state`, `jobhistory`, `relsync`, `package_status.overlay_*` |
| check completion | `package_status` (all rows of the prpa), `prpa.repo_state`, `depends`, `build_meta.lastcheck`, new `job` rows, killed `job` rows |
| publish | `published_repo`, `published_binary`, `published_pattern`, `publish_job`, `prpa.repo_state='published'` |
| shard move | `shard_lease` only (state rows carry `shard_id` and are re-tagged by the operator in batches while no lease is held) |

### 2.5 Sizing guidance

At the `00` scale targets: `revision` ~10 M rows, `package_status` ~5 M
(packages × repositories × archs where built), `full_pool` ~50 M,
`package_result` ~100 M, `job` ~1 M live + archived to `jobhistory`. All are
within a single PostgreSQL cluster with partitioning; hot paths are keyed
lookups. `full_pool` and `package_result` are the largest and are only ever
accessed by `prpa_id`.

## 3. Caches

| Cache | Where | Keyed by | Invalidation |
|---|---|---|---|
| projpack working set | scheduler memory | project | `obs.source.*` events + lease acquisition reload |
| solv pools | scheduler memory (LRU, bounded by bytes) | `(prpa, full_version)` | version bump |
| remote repo indexes | scheduler memory + `cache/remote` objects | `(remote prpa, cookie)` | remote `lastevents` |
| prjconf parse | scheduler memory | sha256 of concatenated config | content |
| binaries | worker local disk + node cache-proxy | sha256 | LRU by size |
| source diffs | `diffcache/` objects + row | cacheid | TTL |
| XML renderings | compat-edge memory (short TTL) | route + ETag of rows | 5 s or events |

## 4. Consistency and ordering

- Per package: linearizable through the `package` row lock.
- Per prpa: the owning scheduler serialises all state changes; integration
  of job results and checks never interleave within a prpa.
- Cross prpa: published pool versions are monotonic; a dependent reads
  `(full_version, solv_key)` and never a partially written pool.
- Events (`05`) carry the version they describe; consumers compare with the
  row version and ignore stale events.

## 5. Importer mapping (summary; details in `09`)

| Legacy path | Target |
|---|---|
| `sources/<pkg>/<md5>-<name>` | `blob` + object (sha256 computed on import; md5 verified) |
| `*-MD5SUMS` (sources/ or trees/) | `tree` (+ `package_tree` witness from the `trees/<proj>/<pkg>` location) |
| `.rev`, `.mrev`, `.del`, `_deleted/` | `revision`, `meta_revision`, `*_deleted` |
| `projects/*.xml`, `*.conf`, `.pkg/*.xml` | `project.meta`, `project.config_text`, `package.meta` (cross-checked against latest mrev) |
| `build/<prp>/<arch>/...` | `prpa`, `package_status`, `build_meta` + `meta/` objects, `package_result` + `bins/`, `bininfo`, `full_pool`, `repo_pool`, `relsync*`, `jobhistory`, `depends`, `genbuildreqs`, `preinstallimage` |
| `jobs/<arch>/*` | `job` rows in state `scheduled` (in-flight jobs are re-dispatched) |
| `repos/` (published tree) | `published/<prp>/<version>` + `published_repo` |
| `db/sqlite/*` | rebuilt from the tables above (not imported) |
| `events/lastevents`, `lastnotifications` | sequence continued from the last number so the Rails cursor keeps working |

## 6. Open questions

- Whether to keep md5 pointer objects or serve md5 lookups purely from
  PostgreSQL (`blob.md5` unique index). The schema supports both; the pointer
  objects exist so the public blob store can be used without a database by
  mirrors. Default: PostgreSQL only; pointer objects are optional.
- Retention defaults for logs and old published versions are instance policy;
  the values above are starting points.
