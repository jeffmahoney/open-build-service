# E10 — registry-svc

Milestone M3. Inputs: `services/registry-svc.md`, analysis P § 3.5,
B build-path § 2.8, `proto/registry.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E10-T01 | OCI distribution read API over object storage: manifests (tag/digest/.sig/.att), blobs with Range and redirects, tags/list pagination, catalog, Accept negotiation and manifest-list fallback, error format | `cmd/registry-svc`, `internal/registry/http` | `docker pull`/`skopeo`/`oras` conformance tests | E03 | M |
| E10-T02 | Internal push (`PushContainers`): blobs, manifests, lists, tags, cosign sigs/atts and referrers, bundle mode, `:info`, atomic signatures extension, TUF update and timestamp extension, GC, `CONTAINER_PUBLISHED` | `internal/registry/push` | layout equivalence with Perl local registry on corpus | T01, E09-T03 | L |
| E10-T03 | Ownership (`OwnRepo`/`DisownRepo`), manifestinfos, info.json, `_trust/tuf` routes, referrers API | `internal/registry/extras` | parity | T02 | S |
| E10-T04 | Build-time pulls of base containers from pools (`:full` obsbinlnk → manifest/blobs) | `internal/registry/pools` | kiwi/docker builds pull base images e2e | T01, E06-T04 | M |
