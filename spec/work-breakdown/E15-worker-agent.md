# E15 — worker-agent and cache-proxy

Milestone M2. Inputs: `services/worker-agent.md`, analysis B build-path
§ 4, § 5, § 7, § 8.4; `proto/worker.proto`, `proto/log.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E15-T01 | Agent skeleton: capability probe, register, claim loop with hints, heartbeat/control stream, state machine, graceful drain | `cmd/worker-agent`, `internal/agent` | protocol tests against dispatcher | E07-T06 | M |
| E15-T02 | CAS cache (`cachedir` layout, LRU by bytes, reservation, verification by hdrmd5/sha256/metamd5) and fetch orchestration through cache-proxy/presigned URLs; sources bundle with `verifymd5` check | `internal/agent/cache`, `internal/agent/fetch` | cache behaviour tests; corpus job inputs | T01, E01-T11 | M |
| E15-T03 | Package-build preparation: rpmlist (all tag forms), `.build.config`, `.build.meta` (gen_meta), preinstall image handling incl. metas, sysroot/cross split, `_buildenv` seed, SLSA materials | `internal/agent/prepare` | control files byte-equal to Perl worker on corpus jobs | T02, E01-T09 | L |
| E15-T04 | Image/container/product/buildenv input layouts (`repos/`, `containers/`, annotations, product trees, `.createrepo_checksums`, kiwi recipe patching, RPMHDRS_ONLY) | `internal/agent/prepare/images` | corpus image/product jobs | T03 | L |
| E15-T05 | `build` invocation per sandbox type (exact command lines), exit-code mapping, hostcheck hooks, log limits, kill/discard/sysrq, statistics parsing, `_statistics` XML, `_buildenv` | `internal/agent/run` | command lines equal Perl on fixtures; kvm e2e on a nested-virt runner | T03 | L |
| E15-T06 | Post-processing: reports/channel/annotate, container normalisation (`pkg/contar` + obs-gzip-go), `.kiwitree`, SLSA provenance, genbuildreqs, result bundle + upload with retries, `unchanged`/`badhost` payload rules | `internal/agent/result` | corpus results identical to Perl uploads (file sets, meta) | T05 | L |
| E15-T07 | cache-proxy DaemonSet (content-addressed GET, node disk LRU, metrics) | `cmd/cache-proxy` | throughput test | E03-T02 | S |
| E15-T08 | Images: worker-agent image with obs-build pinned by digest (+ qemu-user/initvm variants), KVM device plugin manifests, pool examples | `deploy/images/worker`, `deploy/charts` | builds in kind with KVM node | T05 | M |
| E15-T09 | Agent-as-VM-init profile (cloud VM per job, chroot backend) | `internal/agent/vmmode` | e2e on one cloud provider | T05 | M |
