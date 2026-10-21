# 05 — Events and Queues

Depends on: `03-architecture.md`, `04-storage.md`.

NATS JetStream carries two kinds of traffic: **events** (facts that something
changed; consumers may miss them and must reconcile from tables) and **work
queues** (units of work that must be processed exactly once per lease).

## 1. Streams

| Stream | Subjects | Retention | Purpose |
|---|---|---|---|
| `OBS_SOURCE` | `obs.source.>` | limits, 7 days | source-side change events |
| `OBS_BUILD` | `obs.build.>` | limits, 7 days | scheduler/artifact events (pool changes, repo state, job lifecycle) |
| `OBS_JOBS` | `obs.jobs.>` | work queue, ack explicit, max deliver 5 | build jobs per arch and class |
| `OBS_PUBLISH` | `obs.publish.>` | work queue | publish requests |
| `OBS_SERVICE` | `obs.service.>` | work queue | source service runs |
| `OBS_SIGN` | `obs.sign.>` | work queue | job-result signing |
| `OBS_DOD` | `obs.dod.>` | work queue | DoD refreshes and downloads |
| `OBS_NOTIFY` | `obs.notify.>` | limits, 30 days | API-facing notifications (feeds `lastnotifications`) |
| `OBS_REMOTE` | `obs.remote.>` | limits, 1 day | translated events from remote OBS instances |

All messages are protobuf (`proto/events.proto`), carry `event_id` (ULID),
`occurred_at`, `source_service`, `trace_context`, and the **row version** of the
entity they describe.

## 2. Event catalog

### 2.1 Source events (`obs.source.*`)

| Subject | Payload | Legacy equivalent |
|---|---|---|
| `obs.source.project.changed.<project>` | project, mrev, `critical` (bool: build/lock/link/repository/config change) , `lowprio` | `project` / `lowprioproject` event |
| `obs.source.project.deleted.<project>` | project | `project` event + 404 on fetch |
| `obs.source.package.changed.<project>.<package>` | project, package, rev, srcmd5, `kind` (commit \| meta \| service \| link-retarget) | `package` event |
| `obs.source.package.deleted.<project>.<package>` | | `package` event |
| `obs.source.config.changed.<project>` | project, config digest | `project` event (only if `_config` changed) |
| `obs.source.service.started/finished.<project>.<package>` | servicemark, result | `SRCSRV_SERVICE_*` |
| `obs.source.project.suspend/resume.<project>` | job id | `suspendproject` / `resumeproject` |
| `obs.source.linkinfo.changed.<project>.<package>` | | linkinfo DB update |

Subjects embed names with `.`/`>`/`*` percent-encoded; consumers filter on
prefixes. Project names containing `.` are encoded as `%2E`.

### 2.2 Build events (`obs.build.*`)

| Subject | Payload | Legacy equivalent |
|---|---|---|
| `obs.build.pool.changed.<project>.<repository>.<arch>` | prpa, full_version, bininfo_version, `kind` (full \| bininfo-only) | `repository` / `repoinfo` events, `sendrepochangeevent` |
| `obs.build.repo.state.<project>.<repository>.<arch>` | repo_state, details, buildid | `REPO_BUILD_STARTED/FINISHED`, `:schedulerstate` |
| `obs.build.package.status.<project>.<repository>.<arch>` | batch of (package, code, error) | `:packstatus`, redis fan-out |
| `obs.build.unblocked.<project>.<repository>.<arch>` | from_arch, low (bool) | `unblocked` / `lowunblocked` |
| `obs.build.relsync.<project>.<repository>` | | `relsync` |
| `obs.build.import.<project>.<repository>.<target-arch>` | job id with exported files | `import.<job>` |
| `obs.build.job.created/dispatched/started/finished/killed.<arch>` | job id, name, prpa, package, worker, result | `:status` files, `finished:<job>` |
| `obs.build.job.dispatchdetails.<arch>` | job id, details | `dispatchdetails` |
| `obs.build.scanrepo.<project>.<repository>.<arch>` | | `scanrepo` (DoD) |
| `obs.build.wipe.<project>` | packages, allarch | `wipe` / `wipeallarch` |
| `obs.build.useforbuild.<project>` | package? | `useforbuild` |
| `obs.build.admin.<kind>` | rebuild/recheck/admincheck with scope | `rebuild`, `recheck`, `admincheck` |

### 2.3 Publish events

| Subject | Payload |
|---|---|
| `obs.publish.requested.<project>.<repository>` (queue) | prp, repo_info versions per arch, requested_at (first request time preserved on coalesce) |
| `obs.publish.state.<project>.<repository>` | publishing \| published \| failed, publishid, details |
| `obs.publish.container.<project>.<repository>` | gun/repo:tag, digest |

### 2.4 Notification events (`obs.notify.<TYPE>`)

One message per legacy notification type with the same name and parameters:
`SRCSRV_COMMIT`, `SRCSRV_VERSION_CHANGE`, `SRCSRV_UPLOAD`,
`SRCSRV_CREATE_PROJECT`, `SRCSRV_UPDATE_PROJECT`, `SRCSRV_DELETE_PROJECT`,
`SRCSRV_UNDELETE_PROJECT`, `SRCSRV_UPDATE_PROJECT_CONFIG`,
`SRCSRV_CREATE_PACKAGE`, `SRCSRV_UPDATE_PACKAGE`, `SRCSRV_DELETE_PACKAGE`,
`SRCSRV_UNDELETE_PACKAGE`, `SRCSRV_SERVICE_SUCCESS`, `SRCSRV_SERVICE_FAIL`,
`BUILD_SUCCESS`, `BUILD_FAIL`, `BUILD_UNCHANGED`, `BUILD_START`,
`REPO_BUILD_STARTED`, `REPO_BUILD_FINISHED`, `REPO_PUBLISH_STATE`,
`REPO_PUBLISHED`, `CONTAINER_PUBLISHED`, `PACKTRACK` (payload ref),
`sourcepublish`. `notifier` appends each to `lastnotifications` with a
monotonically increasing number and republishes to RabbitMQ/AMQP exactly as
`bs_notifyforward`/the Rails consumer expect (see `services/notifier.md`).

## 3. Work queues

### 3.1 Build jobs: `obs.jobs.<arch>.<class>`

- One subject per (architecture, worker class). Class defaults to `default`;
  constraints that require a specific sandbox/hardware map to classes by the
  dispatcher (`services/dispatcher.md`).
- Message = job id + priority + locality hints + the `buildinfo` digest (the
  worker fetches the full `buildinfo` via gRPC).
- Consumers: durable pull consumers per class; `AckWait` = job heartbeat
  interval × 3; the worker extends the ack with `InProgress` on every
  heartbeat; redelivery on worker death.
- Priority: JetStream has no priority; the dispatcher maintains **three
  subjects per class** (`.hi`, `.med`, `.lo`) and workers pull `hi` first with
  short timeouts. Within a subject, order is readytime. The dispatcher may
  re-enqueue to change priority (old message is purged by id).
- Kill: `obs.jobs.control.<arch>` carries `kill <job id>`; the worker agent
  subscribes and aborts.

### 3.2 Publish: `obs.publish.requested.<project>.<repository>`

Coalesced per prp: the publisher uses a per-prp lease row (`publish_job`),
so duplicate requests while running are folded into one follow-up run.

### 3.3 Service runs: `obs.service.run`

Payload: project, package, servicemark, srcmd5, link srcmd5, project
services digest, old srcmd5. Per-project concurrency limit enforced by the
service-runner via a PostgreSQL counter (`limitinprogress` semantics).

### 3.4 Signing: `obs.sign.job`

Payload: job id, arch. Flavors (`iso`, `rsasign`) map to separate subjects
so heavy ISO signing cannot starve rpm signing.

### 3.5 DoD: `obs.dod.refresh.<repo-id>`, `obs.dod.download.<repo-id>`

## 4. Delivery semantics

| Kind | Guarantee | Consumer obligation |
|---|---|---|
| events | at-least-once, ordered per subject | idempotent; compare row version; reconcile from tables on gap (`sync=lost` analogue: consumers track the last ULID and on restart list tables changed since `updated_at`) |
| queues | at-least-once with lease; exactly-once effect via idempotency row | ack only after the effect is committed |

## 5. Legacy bridges

| Legacy | New |
|---|---|
| `events/<arch>/*` files + `.ping` | subscriptions on `obs.source.*`, `obs.build.*` filtered by shard |
| `events/publish/`, `events/signer/`, `events/servicedispatch/`, `events/deltastore/`, `events/sourcepublish/`, `events/redis/` | queues above |
| `lastevents` (interconnect, DoD, public) | `notifier` materialises `lastevents` rows from `obs.source.*` and `obs.build.pool.*`/`repo.state.*` events with the same filters and numbering; compat-edge serves `/lastevents` long-poll from the table with LISTEN/NOTIFY wake-ups |
| `lastnotifications` | `notifier` → table → compat-edge |
| `undelivered/` retry files | JetStream persistence |
| redis fan-out (`BSRedisnotify`) | optional `notifier` sink that writes the same hash keys to Redis so a Rails instance configured for redis keeps working |
| remote `lastevents` polling (`RemoteWatcher`) | `remote-watch` module in scheduler publishes translated events on `obs.remote.*` |
