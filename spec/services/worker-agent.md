# Service: worker-agent

Replaces: `bs_worker`, `dist/obsworker`, `bs_getbinariesproxy` (as
`cache-proxy`), the worker-side use of `BSBuild.pm`, `BSContar.pm`,
`BSTar.pm`, `BSCpio.pm`. **`obs-build` is reused unchanged.**

Reference analysis: `01-legacy-system-analysis.md` § B.4-B.7.
Contract: `proto/worker.proto`.

## 1. Responsibilities

1. Probe capabilities (`/proc`-based: hostarch, cpu flags incl. synthetic
   x86-64 levels, memory/swap, disk, processors, linux flavor/version,
   `nativeonly` on aarch64 without 32-bit EL0, sandbox type availability,
   hostlabels from config, zone from node labels) and register.
2. Pull jobs for its class; honour hints; claim with lease; heartbeat.
3. Execute the job lifecycle exactly as `bs_worker` § 4.5:
   - working dirs `.build-srcdir`, `.pkgs`, `.build.oldpackages`, optional
     tmpfs;
   - fetch sources by `srcmd5`, verify `verifymd5`; extra sources for kiwi;
     `_projectcert.crt`/`_pubkey` when the recipe asks;
   - fetch binaries **by content digest** through the node `cache-proxy`
     into the local CAS cache (LRU by bytes, `--cachesize`), with the legacy
     verification (`hdrmd5` from the rpm signature header, `leadsigmd5`,
     `md5sum`, sha256 for blobs); preinstall image as selected by the
     dispatcher; image/container/product/buildenv variants
     (`getbinaries_image`, `getbinaries_containers`, `getbinaries_product`,
     `getbinaries_buildenv`) with the same directory layouts
     (`repos/<prj>/<repo>/[<arch>/]`, `containers/<prj>/<repo>/`,
     `containers/annotation`, `repos/.createrepo_checksums`);
   - write `.build.config`, `.build.rpmlist` (all tag forms), `.build.meta`
     (`gen_meta` port, `genmetaalgo` 0/1);
   - run `build` with the exact command line of B § 4.5.4 for the sandbox
     type (kvm, qemu, xen, zvm, pvm, openstack, ec2, emulator, uml, lxc,
     docker, podman, nspawn, chroot), personality helper prefix, cross and
     sysroot handling, `--norootforbuild` policy, `--jobs` cap, ccache,
     multibuild flavor flags, kiwi parameters;
   - map exit codes `0/1/2/3/4/9` to `succeeded/failed/unchanged/badhost/
     dead/genbuildreqs`, apply `nobadhost` and the "broken config, not
     broken host" log heuristics, hostcheck hooks, empty-log rule;
   - post-process: `_statistics` XML, `_buildenv`, `.report`/`_channel`
     generation, container normalisation (obs-gzip-go or Go port, with
     `.tar.recompressed` marker), `.kiwitree` manifest for product trees,
     SLSA provenance (v0.2/v1, materials collected during fetch, subjects
     over uploaded artefacts), `_generated_buildreqs` handling;
   - package the result as a tar bundle, upload via presigned PUT with
     resumable retries, then `UploadResult`; `kiwitree` entries that the
     server can hardlink by `leadsigmd5` are not uploaded (server-side
     `receivekiwitree` reconstructs from existing binaries, fetching the
     rest from the uploaded bundle);
   - stream the log to `log-svc` (chunked, with idle/size limits enforced
     locally); serve `logtail` requests through the control stream;
   - handle `kill`, `discard`, `sysrq`, `badhost` self-report, `dead`.
4. Self-update is replaced by immutable images: the agent image pins an
   `obs-build` version; the job carries the expected image digest and the
   agent refuses jobs for a different digest (the operator rolls pools).

## 2. Deployment forms

| Form | Isolation | Notes |
|---|---|---|
| Pod on KVM node pool (default) | `build-vm-kvm` inside the pod | needs `/dev/kvm` (device plugin), `CAP_SYS_ADMIN` or `--vm-use-mkfs-copyin`, loop devices optional, `kernel-obs-build` kernel/initrd in the image, hugepages optional. Several instances per node share one `cache-proxy`. |
| KubeVirt VM per worker | outer VM, `build-vm-kvm` nested or chroot inside | for clusters without nested virt on nodes |
| Cloud VM per job (agent as init) | outer hypervisor, chroot backend | `build-vm-openstack/ec2` prior art; results uploaded directly |
| Container sandboxes (`podman`/`nspawn`/`docker`) | not secure | only for classes with `sandbox != secure` |

`cache-proxy` (DaemonSet) implements the `bs_getbinariesproxy` role over
S3: content-addressed GET with a node-local disk LRU; no project knowledge.

## 3. Cache summary

Every heartbeat carries a cuckoo filter of cached `hdrmd5`/sha256 keys and
the cache byte size, so the dispatcher can rank hints. Filter size is
bounded (default 64 KiB).

## 4. Host requirements (for the operator)

Root or the capability set documented in B § 4.7; `qemu-kvm`, `bsdtar`,
`fuser`, `mkfs.*`, `obs-gzip-go`, `deltarpm` (for `_delta` jobs),
`qemu-user` + `build-initvm-<arch>` for foreign-ISA emulation classes.

## 5. Tasks

`work-breakdown/epics.md` E7 (protocol), E15 (agent).
