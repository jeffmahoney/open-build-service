# Service: compat-edge

Replaces: the HTTP front of `bs_srcserver` (route table, `BSDispatch`,
`BSServer`/`BSStdServer`, the AJAX long-poll helper), the proxy role of the
source server towards `bs_repserver`, `bs_service`, `bs_getbinariesproxy`,
and the `/published`, `/registry` HTTP fronts of `bs_repserver`.

Specification of routes: `06-compat-edge-api.md`, `06a`.

## 1. Structure

- `cmd/compat-edge`: HTTP server (Go `net/http` + h2c), route table generated
  from a Go DSL that mirrors `BSDispatch` patterns (`internal/edge/routes`).
- `pkg/bsxml`: XML::Structured-compatible marshaller/unmarshaller with all
  103 schema definitions as Go values; generated from `BSXML.pm` by a
  one-time converter and then checked in and maintained by hand.
- `pkg/bsverify`: every `verify_*` rule.
- `internal/edge/handlers/<namespace>`: thin handlers that validate,
  call gRPC, render.
- `internal/edge/longpoll`: LISTEN/NOTIFY and event-fed waiters for the
  blocking routes.
- `internal/edge/stream`: cpio streaming (`view=cpio`, `getbinaries`,
  `getsources`) using `pkg/obscpio`.
- `internal/edge/published-gw`: published tree gateway (pointer resolution,
  redirects, index listings, `directory.yast`, ymp on the fly).

## 2. Listeners

| Listener | Class | Routes |
|---|---|---|
| `:5352` (internal ingress, mTLS from Rails) | `rw` | all `/source`, `/build`, `/published`, `/search`, streams, config |
| `:5353` (worker/osc-build/interconnect) | `worker` | § 3.5 of `06` |
| `:82` / public ingress | `public` | `/published/**` static tree, `/registry/**` (`/v2/`) |

## 3. Behavioural notes

- Per-request limits mirror `BSServer`: body size limits per route, request
  timeout defaults (1000 s GET, long for POST), slow-request logging with
  `X-Frontend-Start`.
- `/serverstatus` and `/ajaxstatus` report connection slots and in-flight
  long-polls for operators (`obs-admin` uses them).
- Form POSTs: `application/x-www-form-urlencoded` bodies are merged into the
  query before dispatch, as `BSServer` does.
- Error text catalogue in `internal/edge/errors`: every legacy message that
  Rails or osc matches on is a constant with a parity test.

## 4. Scaling

Stateless; HPA on RPS and p99. Long-poll connections are cheap goroutines;
the limit is configurable. Sticky sessions are not required.

## 5. Tasks

`work-breakdown/epics.md` E1, E11.
