# 09 — Migration and Import

Decision: no mixed-version operation on one `/srv/obs` tree. Migration is
**import, verify, cut over**.

## 1. Phases

| Phase | What | Exit criterion |
|---|---|---|
| M0 Shadow | Deploy the Go stack beside the Perl backend with its own storage. Run `obs-import` from an rsync snapshot of `/srv/obs`. Point a staging Rails instance at compat-edge. | Parity report (`services/importer.md` § 2.8) clean for all projects; golden-corpus tests green. |
| M1 Replay | Keep the import fresh by periodic incremental imports (by mtime/rev deltas) while the Perl backend remains authoritative. Run the Go scheduler in **dry-run** (decisions computed, jobs not dispatched) and diff decisions against the Perl scheduler's `:packstatus` / job creation. | Decision diff rate below an agreed threshold (default 0 for `scheduled` decisions on the corpus; explained differences documented). |
| M2 Cutover | Freeze writes (Rails maintenance mode), final incremental import including in-flight jobs (restarted), `lastnotifications` number continued, switch Rails `source_host` to compat-edge, re-point worker hosts to the Go stack (Go worker-agent images) or run legacy workers against a legacy-protocol adapter if one was built, unfreeze. | Rails healthy; notifications flowing; builds dispatching; publishes succeeding. |
| M3 Decommission | Keep the Perl tree read-only for rollback window (default 30 days). | Window elapsed. |

Rollback at M2: switch `source_host` back, replay writes made during the
window from the Go stack's `lastnotifications`/revision logs into the Perl
tree with an export tool (`obs-export`, the inverse of the importer for
sources and meta only; builds are simply re-run).

## 2. Compatibility obligations during migration

- Sequence numbers for `lastnotifications`/`lastevents` continue, so the
  Rails cursor and interconnect peers never see `sync=lost` at cutover.
- `_repoid` is preserved so remote schedulers/peers do not treat the
  instance as a new source store.
- Job names, `disturl`, `publishid`, container tags, `prp_ext` published
  paths and download URLs are unchanged.
- Signing keys: wrapped `_signkey` blobs continue to work through the
  `signd` adapter; HSM/KMS migration is a separate, later project.

## 3. Interconnect

Peers running the Perl backend keep polling `/public/lastevents` and
fetching `view=solvstate`/`getbinaries` from our compat-edge; our
`source-svc`/`artifact-svc` remote clients consume the same from Perl or Go
peers. Both directions are covered by parity tests against a Perl instance
in CI.

## 4. Data retention differences

Where the new system keeps more (logs, published versions) or less (delta
obscpio storage, BSDB index files) than the Perl tree, `04` § 1.1 lifecycle
defaults apply and are configurable per instance.
