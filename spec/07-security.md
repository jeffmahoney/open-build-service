# 07 — Security

## 1. Trust model (unchanged intent, stronger mechanism)

The Perl backend performs **no user authorization**; `user=` is a label.
The Rails application is the policy decision point. The Go backend keeps
that split but replaces IP-based trust with identities:

| Principal | Identity | Allowed |
|---|---|---|
| Rails API | mTLS client cert (SPIFFE id `spiffe://obs/api`) | class `rw` on compat-edge |
| worker-agent | per-pool client cert + per-worker registration token | `WorkerGateway`, presigned URLs it is handed, log-svc stream |
| interconnect peers | none (anonymous `/public`), outgoing requests use the `Signature` (ssh) auth scheme as today where the peer requires it | `public` class routes |
| services | mTLS, service-to-service authorization matrix enforced by gRPC interceptors (e.g. only scheduler may call `IntegrateResult`; only publisher may `UpdatePublished`) | per matrix in `proto/README.md` |
| operators | `obs-admin` with OIDC/K8s RBAC | admin endpoints |

Network policies deny everything not in the matrix. Object storage
credentials exist only in services; workers and sandboxes get presigned
URLs scoped to specific keys with short TTLs.

## 2. Isolation boundaries

- **Build VMs** (`sandbox=secure`: kvm, xen, zvm) are the boundary for
  untrusted recipes. Container sandboxes are explicitly non-secure and
  excluded by constraints unless the operator opts in per pool.
- **Source services** run in rootless containers with the network policy
  of `call-service-in-container` and no credentials.
- **Repo metadata generators** run in sandboxes with read-only inputs.
- **Link application** (`patch`, `diff3`) runs in the service-runner
  sandbox, never in `source-svc`'s process.

## 3. Keys and signing

- Private keys never leave `sign-svc`/KMS/HSM/signd. The per-project
  wrapped `_signkey` blob is stored as an opaque reference.
- Audit log for every signature; rate limits per caller.
- Cosign cookies and TUF keys as specified in `services/signer.md`.

## 4. Supply chain

- obs-build pinned by digest in worker images; the job spec carries the
  expected digest (`buildcode` successor); mismatches are refused.
- All service images built reproducibly, signed, and verified by admission
  policy.
- SLSA provenance generation and material pinning preserved
  (`services/worker-agent.md`, `artifact-svc.md`).

## 5. Data protection

- `access`/`sourceaccess` semantics enforced at the documented edges;
  `lastevents` suppression for `access`-disabled projects; `nosourceaccess`
  markers honoured by the publisher and aggregates.
- Presigned URLs for protected projects are issued only to authenticated
  callers and expire in ≤ 15 min; public published trees are served via CDN.
- Secrets (registry credentials, DB DSNs, KMS) in Kubernetes Secrets or an
  external secret store; never in CRs.

## 6. Hardening checklist for implementers

1. Validate every name with `pkg/bsverify` before touching storage.
2. Reject symlinks, directories, and `..` in every tar/cpio the system
   ingests (results, service outputs, kiwitree).
3. Size limits on JSON/XML bodies (`containerinfo`, SBOMs < 256 MiB, meta
   lines format).
4. Time-skew check on worker results (server timestamps authoritative).
5. No shelling out with user-controlled arguments outside sandboxes;
   allow-list tool arguments.
