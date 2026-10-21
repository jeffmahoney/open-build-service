# Service: notifier

Replaces: `BSNotify.pm`, `bs_notifyforward`, `BSRedisnotify.pm`, `bs_redis`,
the `lastnotifications`/`notificationpayload` handling in `bs_srcserver`,
`BSSrcServer/Notify.pm` (plugin fan-out), the `lastevents` log writer.

## 1. Responsibilities

1. Subscribe to `obs.notify.*` and append to `lastnotifications` with a
   monotonically increasing `number`; store `PACKTRACK` payloads as objects
   referenced by key; prune to the legacy retention rule (keep 500 behind the
   consumer cursor once 1000 ahead) or a configured window.
2. Serve `LastNotifications(start, block)` to compat-edge with the exact
   semantics: no `start` → `next` only; `start` older than retained →
   `sync=lost`; max 1000 per answer with `limit_reached`; `block` waits on
   LISTEN/NOTIFY.
3. Materialise `lastevents` from `obs.source.*` and `obs.build.pool/repo.state`
   events (`addevent` rules: skip projects with `access` disabled; types
   `package`, `project`, `repository`, `repoinfo`, …), prune by size/age as
   today, and serve filtered reads.
4. Optional sinks: RabbitMQ/AMQP publish of the same notifications (routing
   keys as the Rails `RabbitmqBus` produces them are **not** needed: Rails
   publishes to the bus itself after ingesting; the backend sink exists for
   deployments that enabled `notify_rabbitmq.pm`), Redis build-status hash
   writes (`updateresult`/`updateoneresult`/`updatejobstatus`/`deleteresult`)
   for Rails instances configured with `redisserver`, webhook sinks
   (`notify_plugins`).
5. Emit payload-key compatibility: every notification type carries exactly
   the parameter set listed in `01` § F.A.6 (Rails rejects unknown keys).

## 2. Data

`lastnotifications`, `notification_cursor`, `lastevents` tables
(`04` § 2.2); payload objects under `notify/payload/<key>`.

## 3. Scaling

Two replicas with leader election for the appender (ordering); readers on
any replica.

## 4. Tasks

`work-breakdown/epics.md` E12.
