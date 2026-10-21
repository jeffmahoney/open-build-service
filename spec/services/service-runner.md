# Service: service-runner

Replaces: `bs_service`, `bs_servicedispatch`, `call-service-in-container`,
`run-service-containerized`, the forked service children of `bs_srcserver`,
and the sandboxed helper invocations of `source-svc` (`bs_productconvert`,
`patch`/`diff3`/`bs_mergechanges` for link application).

Reference: `01-legacy-system-analysis.md` § F.C.

## 1. Responsibilities

1. Consume `obs.service.run` (and `obs.service.obsscm`, `obs.service.tool`
   for product conversion and link-apply helpers).
2. For a service run: fetch the inputs (expanded sources via presigned
   bundle, previous `_service:*` outputs into `.old/`, project services
   merged list as `_serviceproject`), create an isolated sandbox pod/container
   with the obs-service image, and execute each service in `_service`
   order then `_serviceproject` order with the exact invocation contract:
   `<name> --<param> <value> … --outdir <dir>`, declared-parameter check
   against `<name>.service`, env (`OBS_SERVICE_PROJECT/PACKAGE/APIURL`,
   `OBS_NAME`, `OBS_SERVICE_DAEMON=1`, proxies), timeout
   (`service_timeout`), stdout+stderr capture, output renaming to
   `_service:<name>:<file>` (or `noprefix` for obs-scm), skip rules for
   modes `localonly|disabled|manual|buildtime`, chaining visibility of
   earlier outputs, `_service_error` on failure, exit code 3 = transient.
3. Network policy per service name as in `call-service-in-container`
   (allow-list for download/scm services, SCM cache volume keyed by
   `sha256(url)`, `obs_scm_bridge` with bridge config and CA certs,
   everything else no network).
4. Upload outputs as blobs; call `Source.CommitServiceResult` (or
   `CommitObsScm`) with the file list and md5s; on transient error schedule
   a retry (10 min) with attempt accounting; per-project concurrency limit.
5. `ListServices`: the installed `*.service` definitions (from the service
   image), served to compat-edge for `GET /service`.
6. Receive `/configuration` fan-out (obsname, api_url) through config.

## 2. Sandbox

Kubernetes Job or a warm pool of sandbox pods with `podman`/`crun` in
rootless mode and `--userns keep-id`, or gVisor; the image is the
operator-maintained `obs-source-service` image. Volumes: `src/`, `out/`,
`home/`, `scm-cache/<hash>` (persistent per node), read-only `scripts/`.
No access to object storage credentials from inside the sandbox; the
runner streams inputs/outputs across the boundary.

## 3. Scaling

KEDA on `obs.service.run` depth; `servicedispatch_maxchild` and
`limitinprogress` become consumer concurrency and a per-project semaphore.

## 4. Tasks

`work-breakdown/epics.md` E13.
