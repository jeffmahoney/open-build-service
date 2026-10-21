# E12 — notifier

Milestone M1. Inputs: `services/notifier.md`, analysis F § A.6,
S § 4.3, `proto/notifier.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E12-T01 | Appender (leader-elected) from `obs.notify.*` to `lastnotifications`; payload objects; pruning; `LastNotifications` RPC with `sync=lost`/`limit_reached`/`block` | `cmd/notifier`, `internal/notifier` | Rails `UpdateNotificationEvents` ingests all corpus types; payload-key sets exact per type | E03 | M |
| E12-T02 | `lastevents` materialiser from source/build events with `addevent` rules | `internal/notifier/lastevents` | S § 4.2 parity | T01 | S |
| E12-T03 | Sinks: AMQP publisher (plugin parity), Redis build-status writer, webhook `notify_plugins` | `internal/notifier/sinks` | integration tests | T01 | S |
