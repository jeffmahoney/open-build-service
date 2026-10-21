# Service: operator (obs-operator) and deployment

Replaces: `BSConfig.pm` (as the configuration surface), `bs_admin`
topology functions, `obsstoragesetup`, `obsworker`, `dist/systemd/*`,
`obs-setup`, the appliance scripts.

## 1. Custom resources

| CRD | Purpose | Key fields |
|---|---|---|
| `ObsInstance` | one OBS backend | `obsname`, `apiUrl`, `downloadUrl`, storage (S3 endpoint/buckets/zones), PostgreSQL DSN secret, NATS, signing backend, `ipaccess`-equivalent mTLS identities, `schedulers` (arch list), repotype toolchain images, `publishredirect`/`prp_ext_map`/`repodownload`/`stageserver` config, `container_registries` secrets, `publish_containers` rules, `extraconfig`, `dispatch_adjust`, `secure_sandboxes`, `powerpkgs`/`powerhosts`, retention policies |
| `ObsShard` | a scheduler shard | `shardId`, `storageZone`, `weight`, selectors (project regex list) feeding `shard_map`/`shard_override`; status: lease owner, queue depth, lag |
| `ObsWorkerPool` | a worker class | `class`, `hostarch`, `archs` (default from cando), `sandbox`, `hostlabels`, hardware profile, zone, node selector/tolerations, `minReplicas`/`maxReplicas`, KEDA trigger (queue depth per `obs.jobs.<arch>.<class>`), obs-build image digest, cache size |
| `ObsRepotypeToolchain` | generator images | repotype → image + tool versions (`createrepo_c`, `modifyrepo_c`, `dpkg`, `apk-tools`, `cosign`, `rsync`) |

## 2. Reconciliation

- Renders Deployments/StatefulSets/Services/HPAs/KEDA ScaledObjects/
  NetworkPolicies for every service; manages mTLS identities (cert-manager
  or SPIRE); runs schema migrations as Jobs before rollouts.
- Maintains `shard_map`/`shard_override` from `ObsShard` CRs; drives
  rebalancing (lease hand-over, batch re-tagging of `shard_id` columns).
- Scales scheduler replicas and assigns weights from shard load metrics.
- Exposes `ObsInstance.status` with the information `/build/_workerstatus`
  reports about daemons (compat-edge reads it).
- Config changes that the Perl backend re-read via `configuration` events
  (`obsname`, proxy, `extraconfig`) are reconciled live.

## 3. Helm chart

`deploy/charts/obs-backend` installs the operator, CRDs, and an
`ObsInstance` with sane defaults (MinIO, single PostgreSQL, single NATS,
one shard, one x86_64 KVM pool with `minReplicas: 0`). Dev profile runs the
whole stack with `kind` + nested virt disabled (podman class only).

## 4. obs-admin CLI

Reimplements the operator-facing subcommands of `bs_admin`
(`--rescan-repository`, `--rebuild-full-tree`, `--deep-check-project`,
`--check-project`, `--check-package`, `--check-all-projects`, `--show-*`,
`--dump-*`, `--publish-repository`, `--unpublish-repository`,
`--create-patchinfo-from-updateinfo`, `--change-attribute`, event injection,
`--query-config`, `--shutdown/--restart` of services, `--check-consistency`)
against gRPC admin endpoints, plus shard and pool management.

## 5. Tasks

`work-breakdown/epics.md` E16.
