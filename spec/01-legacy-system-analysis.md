# 01 — Legacy System Analysis (index)

The ground truth for behavioural parity. Five engineering reports, written
from the Perl sources at commit `db93ae665b` (obs-build submodule
`1fe25db5f6`), with `file:line` citations for every claim. They total about
9 200 lines and are kept verbatim under `analysis/`. This document is the
index and tells an implementer where to look for each behaviour.

| Section id used in this spec | Report | Covers |
|---|---|---|
| **S** | [analysis/S-source-server.md](analysis/S-source-server.md) | `bs_srcserver` and its modules: on-disk model, revisions, links, services, scmsync, remote projects, partitioning, every route, events/notifications, access control, concurrency, 17 invariants, redesign implications |
| **B** (scheduler) | [analysis/B-scheduler.md](analysis/B-scheduler.md) | `bs_sched`/`BSSched`: process and event model, Lookat priority queues, the full check pipeline and 21-rule status decision, meta/rebuild algorithm, cycles, `:full`/`:repo`, job creation and the buildinfo XML, per-type handlers, result integration, repositories/DoD/remote, cross-arch coupling, scaling limits, record model for a persistent store, 11-point preservation checklist |
| **B** (build path) | [analysis/B-build-path.md](analysis/B-build-path.md) | `bs_repserver`, `bs_dispatch`, `bs_warden`, `bs_worker`, `bs_getbinariesproxy`: layouts, routes by consumer, `putjob`, `_buildinfo`, preinstall images, dispatcher algorithm and constraints oracle, worker lifecycle, cache, exact `build` command line, exit codes, result collection, protocol timeline and recovery scenarios, prjconf directive reference, recipe and VM backend catalogs, what a worker needs |
| **P** | [analysis/P-publish-sign.md](analysis/P-publish-sign.md) | `bs_publish`, `bs_signer`, containers/registry, published DB: event loop and retry classes, publish walkthrough, path classification table, per-repotype generators with exact tool invocations, flags, mirroring, source publishing, notifications, signing table and key model, `sign` CLI and signd protocol, cosign/TUF/Rekor/DSSE formats, local registry layout, redesign and tooling verdicts |
| **F** | [analysis/F-frontend-contract-and-infra.md](analysis/F-frontend-contract-and-infra.md) | A: Rails↔backend contract (connection layer, passthrough controllers, every `Backend::Api` method and caller, route usage table, notification stream and event types, data-authority table, polling jobs, trust/failure semantics). B: public API/WebUI feature inventory with DB/backend placement. C: source service server and plugin contract. D: infrastructure modules (`BSServer`, `BSStdServer`, `BSDispatch`, `BSRPC`, `BSHTTP`, events/watcher/handoff, `BSUtil`, file DBs, auth helpers), the `XML::Structured` DSL and all 103 `BSXML` schemas, every `BSVerify` rule, the full `$BSConfig::` variable table, `bs_admin` and the other admin tools, packaging and systemd units |

Route tables extracted mechanically from the dispatch tables:
[06a-legacy-route-catalog.md](06a-legacy-route-catalog.md).

## How to use these reports

1. When a document in this spec says "as B § 3.8", open the report and the
   cited Perl lines. Reproduce the behaviour, not the code.
2. Each report ends with "implications for a distributed redesign". Those
   sections informed `03`/`04`; where this spec deviates from a report's
   suggestion, the spec wins.
3. The reports describe the state of the Perl code at one commit. If the
   upstream backend changes during implementation, update the report and the
   parity corpus together.

## Cross-cutting facts every implementer must know

- The Rails application talks only to the source server endpoint; it never
  authenticates; `user=` is a label (F § A.1, A.9).
- Backend → Rails is pull-only through `lastnotifications`; Rails rejects
  unknown payload keys (F § A.6).
- All ids and hashes in `02` § 2.5 are reproduced bit-exactly (S § 8).
- The scheduler's rebuild decision is the `.meta` comparison (B § 3.8-3.10).
- The dispatcher has no cache-locality signal today; the design adds one
  (B build-path § 8.3).
- Publishing has no atomic whole-repository swap today; the design adds one
  (P § 5.2, § 6.1).
- obs-build is driven purely by a command line and four files
  (B build-path § 7.1, § 8.4).
