# Service: log-svc

Replaces: live `_log` streaming (`bs_repserver` fetching `<worker uri>/logfile`),
`:logfiles.success/fail`, `<pkg>/logfile`, `.logfile.badhost/`.

## 1. Responsibilities

1. Receive log chunks from worker-agents over a gRPC client stream keyed by
   job id and lease; keep the live tail in memory and spill to object
   storage (`logs/<prpa>/<package>/<job-id>.log`) as it grows.
2. Serve `Read(job|prpa+package, start, end, last, view=entry, nostream)`
   and `Follow` (server stream) with the legacy semantics the Rails live-log
   view and `osc remotebuildlog` rely on: byte offsets, `view=entry` size
   query, `last`/`lastsucceeded` selection, the "link/service error first"
   check (compat-edge consults source-svc before streaming).
3. On job completion, promote the log to the package's `logfile` and the
   `last success`/`last failure` pointers (`package_result` kind `log`);
   retention per `04` § 1.1; `badhost` logs retained for 24 h for the
   dispatcher's diagnostics.
4. Enforce `buildlog_maxsize`/`buildlog_maxidle` cooperatively with the
   worker (the worker enforces; the service records the truncation marker).

## 2. Scaling

HPA; a job's stream is pinned to one replica by consistent hashing on job
id; readers are redirected to the owner (or read the spilled object when
the job is finished).

## 3. Tasks

`work-breakdown/epics.md` E7.
