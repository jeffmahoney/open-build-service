# Service: importer (`obs-import`)

One-shot Job that migrates an existing `/srv/obs` tree into PostgreSQL and
object storage. Specification of the cutover: `09-migration-and-import.md`.

## 1. Inputs

Read-only mount of `$bsdir` (or an rsync copy), the instance's `BSConfig.pm`
(for `nosharedtrees`, `partitioning`, `publishredirect`, `obsname`, sign
settings), the Rails database cursor (`backend_infos.lastnotification_nr`).

## 2. Phases (resumable, idempotent, parallel per project)

1. **Blobs**: walk `sources/<pkg>/<md5>-<name>`; expand delta obscpio;
   compute sha256; upload to `blobs/`; insert `blob`. Deduplicate by md5.
2. **Trees**: all `*-MD5SUMS` (sources/ and trees/) → `tree` rows with
   pseudo entries preserved; `package_tree` witness from the `trees/<proj>`
   location; `_linkerror` logs → `link_expansion.error`;
   `_serviceerror` markers → `service_run` state.
3. **Projects/packages**: `projects/*.xml|.conf`, `.pkg/*.xml`, `.rev`,
   `.mrev`, `.del`, `_deleted/`, `upload-MD5SUMS`, `:multibuild`, keys.
   Cross-check live XML against the last mrev tree; report drift.
4. **Build trees**: per prpa, `:schedulerstate`, `:packstatus(.finished)`,
   `:meta/*`, `<pkg>/*` (binaries → `bins/`, logs → `logs/`), `.bininfo`,
   `:bininfo(.merge)`, `:full` membership + `.meta`, `:full.useforbuild`,
   `:repo`/`:repoinfo`/`:repodone`, `:relsync*`, `:jobhistory`, `:depends`,
   `:genbuildreqs`, `:preinstallimages`, `_deltas`, `:logfiles.*`, SLSA
   store, `blobs/sha256`.
5. **Jobs**: `jobs/<arch>/*` → `job` rows as `scheduled` (in-flight ones
   are restarted); `jobs/load`, `dispatchprios`, `badhosts`.
6. **Published**: `repos/` → `published/<prp>/<publishid>` version objects
   and `published_repo`; publisher `:repoinfo` → state; registry dirs →
   registry tables; `db/sqlite/published` is **not** imported (rebuilt).
7. **Streams**: `lastnotifications` and `lastevents` sequences continued
   from the last numbers; pending `undelivered/` events replayed as
   `obs.source.*` events.
8. **Verification**: for a sample (or all) of packages, compare
   `GetFileList`/`srcmd5`/expanded trees and `_result` output between the
   Perl backend (still running read-only) and the Go stack; emit a parity
   report.

## 3. Performance

Parallel workers per phase; checkpoints in a `import_progress` table;
throughput target 50 k files/min per worker for blobs.
