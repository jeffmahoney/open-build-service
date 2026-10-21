# E16 — operator, Helm, obs-admin, observability

Milestone M4. Inputs: `services/operator.md`, `08`, `03` § 7, analysis
F § D.11 and admin tools section.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E16-T01 | CRDs (`ObsInstance`, `ObsShard`, `ObsWorkerPool`, `ObsRepotypeToolchain`) and controller skeleton (kubebuilder) | `deploy/operator` | CRDs install; reconcile loop runs | E03-T05 | M |
| E16-T02 | Rendering of all services, HPAs, KEDA ScaledObjects, NetworkPolicies, mTLS identities, migrations Jobs | `deploy/operator/render` | full stack from one `ObsInstance` in kind | T01 | L |
| E16-T03 | Shard management: `shard_map` sync, lease hand-over, rebalancing with batch re-tagging, status | `deploy/operator/shards` | rebalance test with running scheduler | T02, E04-T01 | M |
| E16-T04 | Worker pool management: node selectors, KVM device plugin, KEDA triggers, image roll with drain | `deploy/operator/pools` | scale-to-zero and back e2e | T02, E15-T08 | M |
| E16-T05 | Helm chart + dev/prod values; docs | `deploy/charts/obs-backend` | `helm install` on kind | T02 | S |
| E16-T06 | obs-admin CLI (all legacy subcommands mapped + shard/pool ops) | `cmd/obs-admin` | command tests | E04-T16, E06-T12, E07-T08, E08-T12 | M |
| E16-T07 | Dashboards, alerts, SLO recording rules, runbooks skeletons | `deploy/observability` | alerts fire in chaos tests | T02 | M |
