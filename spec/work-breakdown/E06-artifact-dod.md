# E06 — artifact-svc and dod-svc

Milestone M2. Inputs: `services/artifact-svc.md`, `services/dod-svc.md`,
analysis B scheduler § 3.11, § 4.5, § 5; B build-path § 1-2; `04` § 2.3.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E06-T01 | Store: `package_result`, `build_meta`, `bininfo`, `full_pool`, `full_index`, `repo_pool`, `repo_info`, `preinstallimage`, `jobhistory`, `bin_ref` GC tables | `internal/artifact/store` | migrations, tests | E03 | M |
| E06-T02 | Pool indexes: build solv from `full_pool` incrementally; `GetPoolIndex` with versions, modules, repostate; local solv cache | `internal/artifact/pool` | solv equals Perl `:full.solv` content on corpus (solvable set) | T01, E01-T10 | M |
| E06-T03 | Result bundle unpack + validation (`ValidateResult`): meta format, forbidden files, containerinfo validation and tar normalisation (`pkg/contar`), JSON limits, kiwitree reconstruction by `leadsigmd5` | `internal/artifact/ingest`, `pkg/contar` | B build-path § 2.5 cases | T01, E01-T11 | L |
| E06-T04 | `IntegrateResult` = `update_dst_full` (part 1 package dir, part 2 `:full` precedence/useforbuild/exportfilter/imports/volatile, bininfo merge, metacache, events) | `internal/artifact/integrate` | `:full` membership equals Perl after replaying corpus job results | T02, T03 | L |
| E06-T05 | `SetRepoPool`/`GetRepoPool`/unpublish edits; `repo_info` versions | `internal/artifact/repopool` | P § 1.2 outputs | T01 | S |
| E06-T06 | `GetBininfo`, `GetPackStatus`, binary list views (`names`, `binaryversions`, `binarychecksums`, `fileinfo`, `fileinfo_ext`, `cpio`, `cpioheaderchksums`, `gbininfocode`, `binaryversionscode`, `solvstate`, `cache`) | `internal/artifact/views` | parity for `_repository` and package views | T02, T04 | M |
| E06-T07 | Remote OBS binary cache (`remotecache` solv/bininfo cookies, serialisation, `getremotebinaries` chunking) | `internal/artifact/remote` | against Perl peer | T02 | M |
| E06-T08 | dod-svc: decl sync, refresh per repotype (`pkg/repo` parsers rpmmd/susetags/deb/arch/apk/mdk/registry), cookies, GPG verify, downloads, placeholder replace | `cmd/dod-svc`, `internal/dod`, `pkg/repo` | parsed metadata equals Perl `doddata` on fixture repos | T01 | L |
| E06-T09 | `_buildinfo` generator (shared `buildjob` library in repserver mode, POST variants, `fixupbuildinfo`, preinstall image selection, `add=`, `debug`) | `internal/artifact/buildinfo` | `osc build` e2e; parity | E04-T09 | M |
| E06-T10 | Binary access: presigned, cpio, `getbinaryversions`, `getpackagebinaryversionlist`, `PutBinary`/`DeleteBinary`, package files (`status`, `reason`, `history`, `_buildenv`, `_statistics`, `rpmlint.log`, `.stats`), jobhistory queries, `copybuild`, `uploadbuild`, `_availablebinaries` | `internal/artifact/api` | route parity | T04 | M |
| E06-T11 | SLSA material store and refs; YMP generation | `internal/artifact/slsa`, `internal/artifact/ymp` | P § 2.7 SLSA pinning; ymp parity | T01 | S |
| E06-T12 | `Verify` consistency check/repair; GC job | `internal/artifact/verify`, `internal/artifact/gc` | injected drift detected and repaired | T04 | M |
