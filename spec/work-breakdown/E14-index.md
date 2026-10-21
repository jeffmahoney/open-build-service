# E14 — index-svc

Milestones M1 (package/project search) / M3 (published). Inputs:
`services/index-svc.md`, analysis P § 4, S § 1.8, § 3.5,
`proto/index.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E14-T01 | Published tables + `UpdatePublished` (repoinfo upsert, binary diff from binaryorigins, patterns), `prp_ext` keys, `binarypath2name` | `cmd/index-svc`, `internal/index/published` | P § 4.2 parity on corpus updates | E03 | M |
| E14-T02 | `SearchPublished` with `pkg/bsxpath` over virtual binary docs, indexed-key planner, 413 rule, sort/limit; `Collection`; download URL derivation (`BSUrlmapper`) | `internal/index/search` | Rails and osc queries parity | T01, E01-T15 | M |
| E14-T03 | `SearchPackages`/`SearchProjects` hot paths + generic XPath over meta | `internal/index/source` | Rails `Backend::Api::Search` callers pass | E02-T01 | S |
