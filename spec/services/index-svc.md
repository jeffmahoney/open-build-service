# Service: index-svc

Replaces: `BSDB.pm`, `BSDBIndex.pm`, `BSSrcServer/SQLite.pm` (published
tables), the published-DB feeder (`POST /search/published cmd=updatedb`),
`BSXPath.pm`/`BSXPathKeys.pm` usage in `bs_srcserver` for
`/search/project`, `/search/package`, `/search/published/*`,
`/published?view=collection`, `bs_dbtool`.

## 1. Responsibilities

1. Maintain `published_binary`, `published_pattern`, `published_repo`
   (`repoinfo`) from publisher updates (RPC `UpdatePublished`), with
   `prp_ext` (`:`→`:/`) keys and the legacy `binarypath2name` rules.
2. Serve:
   - `SearchPublished(kind=binary|pattern|repoinfo, match, limit,
     withdownloadurl, withscmsyncurl)` where `match` is the OBS XPath dialect
     over virtual `<binary>` documents (attributes `name, version, release,
     versrel, arch, type, project, repository, package, filename, filepath,
     baseproject, path/project, path/repository, downloadurl, scmsyncurl`);
     indexed attributes `name`, `project`, `package`; unanchored queries
     refused with `413` like today; results sorted version-desc, `limited`.
   - `Collection(name[], project[])` for `/published?view=collection`.
   - `SearchPackages(match)` / `SearchProjects(match)` over project/package
     meta (the Rails hot queries `starts-with(@name)`, `@project=… and
     starts-with(@name,'_product:')`, `linkinfo/@package=… and
     linkinfo/@project=…`, `@name=…` lists) backed by PostgreSQL indexes on
     `project.name`, `package.name`, `linkinfo`; a generic XPath evaluator
     (`pkg/bsxpath`) for other osc queries.
3. Download URL derivation (`BSUrlmapper`: `publishredirect`, `prp_ext_map`,
   `repodownload`, `download_url` from configuration).

## 2. Data

`04` § 2.3 `published_*`, plus read access to `project`, `package`,
`linkinfo`.

## 3. Scaling

Stateless readers; writer path serialised per prp by publisher.

## 4. Tasks

`work-breakdown/epics.md` E14.
