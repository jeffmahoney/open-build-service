# Service: dod-svc

Replaces: `bs_dodup`, `BSSched/DoD.pm` (daemon side), `BSRepServer/DoD.pm`.

## 1. Responsibilities

1. Track download-on-demand repositories declared in project meta
   (`<download arch url repotype archfilter master pubkey>`) as `dod_repo`
   rows (replacing `dods/<proj>::<repo>::<arch>` files).
2. Refresh on schedule (1 h; 10 min after errors; immediately when the
   scheduler signals missing resources): fetch the index for the repotype
   (`rpmmd`, `susetags`, `deb`, `arch`, `apk`, `mdk`, `registry`), with
   master-mirror check, GPG verification against `pubkey`, checksum
   verification; compute the cookie; parse with `pkg/repo` (Go port of
   `Build::Repo::parse` per format) keeping the highest version per
   name/arch; write the parsed metadata object (`dod/<repo-id>/<version>/
   doddata`) and emit `obs.build.scanrepo` for the owning shard.
3. Registry DoD: resolve only the `container:<repo>:<tag>` resources listed
   as needed (manifests with digest short-cut, fat manifests per arch,
   CycloneDX attestation for `annotation.installed`).
4. Downloads: on `obs.dod.download` (from the scheduler's `dodfetch`), fetch
   the package file, verify, store as a `bins/` object, and call
   `artifact-svc.AddDodBinary(prpa, …)` which replaces the placeholder in the
   pool (hdrmd5 becomes real; consumers rebuild once).
5. `clean_obsolete_dodpackages` on metadata change.

## 2. Scaling

KEDA on refresh/download queues; per-repo serialisation by lease.

## 3. Tasks

`work-breakdown/epics.md` E6.
