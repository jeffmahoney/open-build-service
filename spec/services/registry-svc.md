# Service: registry-svc

Replaces: `BSRegistryServer.pm`, `BSRepServer/Registry.pm`,
`BSPublisher/Registry.pm` (local registry writer), `BSSrcServer/Registry.pm`
(ownership map), `bs_regpush` (as the pusher library used by publisher),
`BSRepServer/Containertar.pm`, `BSContar.pm`, `BSPublisher/Blobstore.pm`.

Reference analysis: `01-legacy-system-analysis.md` § P.3.
Contract: `proto/registry.proto`. Storage: `04-storage.md` § 1.1
(`registry/`), `registry_owner`.

## 1. Responsibilities

1. **OCI distribution API** (`/v2/`) over object storage for:
   - published containers and charts (what `BSPublisher::Registry` wrote
     into `$bsdir/registry/`), with the OBS extras: `manifestinfos/<digest>`,
     `info.json`, atomic signatures extension (`signatures/<digest>`),
     OCI referrers, notary TUF files (`_trust/tuf/*`), `_catalog`,
     paginated `tags/list`, Range on blobs, manifest-list to single-manifest
     fallback for old clients;
   - build-time pulls by workers of base containers from `:full` pools
     (`BSRepServer::Registry` role; served from `artifact-svc` pool data).
2. **Ownership**: a registry repository name is owned by exactly one `prp`
   (`registry_owner`); pushes from another prp are refused with the legacy
   message.
3. **Normalised container storage**: layers are stored once as blobs
   (`tar_blobids`, `tar_blobcompression`), reconstruction of a tar on
   demand (`construct_container_tar`), containerinfo validation
   (`readcontainerinfo` rules).
4. **Pusher library** (`pkg/regpush`) used by publisher for external
   registries: Bearer/Basic auth, blob/manifest upload, layer
   normalisation, manifest lists, cosign tags and bundles, tag listing with
   `If-None-Match`, deletion (`DELETE` or mangled-manifest fallback), exit
   code classes for 4xx/429.
5. TUF timestamp re-signing every 14 days with the per-repo timestamp key
   (`extend_timestamp`).
6. GC of unreferenced manifests/blobs/referrers.

## 2. Endpoints

```
GET|HEAD /v2/                                   version check
GET|HEAD /v2/<repo>/manifests/<ref>             tag | digest | sha256-<d>.sig | .att
PUT      /v2/<repo>/manifests/<ref>             internal only (publisher), mTLS
GET|HEAD /v2/<repo>/blobs/<digest>              Range, Cache-Control: 31 days, presigned redirect allowed
POST/PUT /v2/<repo>/blobs/uploads/...           internal only
GET      /v2/<repo>/tags/list?n&last
GET      /v2/<repo>/referrers/<digest>[?artifactType]
GET      /v2/<repo>/signatures/<digest>         atomic sigs extension (if enabled)
GET      /v2/<repo>/manifestinfos/<digest>
GET      /v2/<repo>/info.json
GET      /v2/_catalog
GET      /v2/<gunprefix>/<repo>/_trust/tuf/{root,targets,snapshot,timestamp}.json[.<sha256>], {snapshot,timestamp}.key
```

Legacy `/registry/...` paths are served by compat-edge by rewriting to
`/v2/`; the source server's `registry_forward` behaviour (owner lookup →
partition) collapses into a direct lookup of `registry_owner`.

## 3. Data

| Legacy file | New |
|---|---|
| `registry/:repos` | `registry_owner` |
| `<regrepo>/:info` | `registry_repo(regrepo, project, repository, gun, tags jsonb)` |
| `:blobs/`, `:manifests/`, `:tags/` | `registry/blobs/<digest>`, `registry/manifests/<repo>/<digest>`, `registry_tag(regrepo, tag, digest, type)` |
| `:manifestinfos/` | `registry_manifestinfo(regrepo, digest, json)` |
| `:referrers/`, `:referrerinfo` | `registry_referrer(regrepo, subject, digest, artifact_type, manifest)` |
| `:cosign`, `:sigs`, `:tuf` | `registry_cosign`, `registry_sigs`, `registry_tuf` jsonb rows (timestamp private key as a `sign-svc` key ref) |

## 4. Scaling and locality

Stateless; blobs served via presigned redirects to object storage/CDN when
the client supports redirects (default) or proxied otherwise. Manifest and
tag metadata cached per replica with event invalidation.

## 5. Tasks

`work-breakdown/epics.md` E10.
