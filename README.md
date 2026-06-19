# Senior Software Architect — Technical Exercise

Welcome. This exercise is part of the interview process for the [Senior Software Architect](https://careers.tether.io/o/senior-software-architect-100-remote-worldwide) role.

The goal is **not** to see if you can build a CRUD service, it's to see how you reason about service boundaries, contracts, failure modes, deployment topology, and operability. The written sections of this README are weighted at least as heavily as the code.

> **Time budget:** target 4-6 hours of focused work.

---

## Table of Contents

1. [The Brief](#1-the-brief)
2. [What You Will Build](#2-what-you-will-build)
    - [2.1 Upstream contracts](#21-upstream-contracts)
3. [Functional Requirements](#3-functional-requirements)
    - [3.1 Hydration & timezone projection](#31-hydration--timezone-projection)
    - [3.2 Conflict detection](#32-conflict-detection)
4. [Constraints & Ground Rules](#4-constraints--ground-rules)
5. [Deliverables](#5-deliverables)
6. [Candidate Sections to Fill In](#6-candidate-sections-to-fill-in)
    - [6.0 Development](#60-development)
    - [6.1 Architecture Overview](#61-architecture-overview)
    - [6.2 Service Boundaries & Contracts](#62-service-boundaries--contracts)
    - [6.3 Data Model & Consistency](#63-data-model--consistency)
    - [6.4 Failure Modes & Status-Code Strategy](#64-failure-modes--status-code-strategy)
    - [6.5 Validation Strategy](#65-validation-strategy)
    - [6.6 Testing Strategy](#66-testing-strategy)
    - [6.7 Observability](#67-observability)
    - [6.8 Security](#68-security)
    - [6.9 Scaling & Performance](#69-scaling--performance)
    - [6.10 Deployment](#610-deployment)
    - [6.11 Trade-offs & What You Would Do Next](#611-trade-offs--what-you-would-do-next)
    - [6.12 Other Notables](#612-other-notables)

---

## 1. The Brief

There are various internal services, two of which need to be integrated with and cannot be change:

- **`catalog`** — an HTTP service that stores resource metadata. Each resource has a numeric `id`, a `name`, a `kind` (e.g. `"room"`, `"gpu"`, `"vehicle"`), a `capacity`, and an IANA `timezone` (e.g. `"Europe/Lisbon"`).
- **`reservations`** — an HTTP service that stores reservations against those resources. Each reservation has a numeric `id`, a `resourceId`, a `holder` (opaque string), an ISO-8601 UTC `startsAt` and `endsAt`.

Both upstream services are minimal mock HTTP servers — small Node `http` servers, no framework, no auth, no retries built in. Treat them as black-boxes: they may go down, slow down, return 4xx/5xx, or occasionally return malformed payloads.

Design and implement a third service — the **`scheduling-api`** — that sits in front of these two services and exposes a clean, opinionated, well-behaved HTTP API to its consumers.

## 2. What You Will Build

```
                       ┌─────────────────────┐
   client ───HTTP──▶   │    scheduling-api   │  ◀── you build this
                       └────────┬────────────┘
                                │
                ┌───────────────┴────────────────┐
                ▼                                ▼
       ┌──────────────────┐            ┌───────────────────┐
       │     catalog      │            │   reservations    │
       └──────────────────┘            └───────────────────┘
            (given)                          (given)
```

**Given.** The two upstream mock services are provided, fully formed, as [`catalog-service.js`](./catalog-service.js) and [`reservations-service.js`](./reservations-service.js) at the repo root. They are zero-dependency Node `http` servers. You will not modify them — they represent third-party systems your `scheduling-api` integrates with. Their contracts and the deterministic failure-mode fixtures baked into them are documented in [§2.1](#21-upstream-contracts).

**Deliverable.** The `scheduling-api` service that consumes the upstreams, exposes the capabilities in [§3](#3-functional-requirements), and accepts writes that create, modify and cancel reservations.

### 2.1 Upstream contracts

Both upstreams listen on the port supplied by the `PORT` environment variable. Defaults: `catalog` 4040, `reservations` 5050.

#### `catalog`

- `GET /{id}` → `200 { id, name, kind, capacity, timezone }` | `404` if unknown | `400` for a non-numeric path | `405` for any non-GET method.

Failure-mode fixtures — use these `id`s to exercise the unhappy paths in your `scheduling-api`:

| `id`  | Behavior                                    |
| ----- | -------------------------------------------- |
| `4`   | 404 — deliberate hole in the dataset         |
| `6`   | 500 — deliberate server error                |
| `8`   | 200 with a truncated, unparseable JSON body  |
| `9`   | 200 after a ~5 s delay                       |
| `10+` | 404                                          |

#### `reservations`

- `GET /{id}` → `200 { id, resourceId, holder, startsAt, endsAt }` | `404`.
- `GET /?resourceId={id}` → `200 [...]` — list, optionally filtered by `resourceId`. **No pagination.**
- `POST /` → `201 {...}` — id assigned by the service; body is echoed back.
- `PUT /{id}` → `200 {...}` — full replace.
- `PATCH /{id}` → `200 {...}` | `404`.
- `DELETE /{id}` → `204` | `404`.

The reservations service performs **no** validation, **no** conflict detection, and **no** authorisation — it will gladly persist overlapping or nonsensical records. Those concerns live in your `scheduling-api`.

Failure-mode fixtures:

| `id`  | Behavior                                    |
| ----- | -------------------------------------------- |
| `901` | `GET` returns 500                            |
| `902` | `GET` returns 200 with truncated JSON        |
| `903` | `GET` returns 200 after a ~5 s delay         |

## 3. Functional Requirements

The `scheduling-api` MUST expose HTTP capabilities for the operations listed below. **The URI scheme, HTTP methods, status-code policy, content negotiation, pagination scheme, filter syntax and error-body shape are architectural decisions you own** — document them in [§6.2](#62-service-boundaries--contracts) and justify the contentious ones in [§6.4](#64-failure-modes--status-code-strategy). The list below is the contract of *what must be possible*, not *how it must look on the wire*.

### Reservation lifecycle

- **Read a single reservation** in hydrated form (see [§3.1](#31-hydration--timezone-projection)).
- **List reservations** in hydrated form, supporting at minimum: pagination, filter by `resourceId`, and filter by overlap with a caller-supplied time window. Further filters are at your discretion.
- **Create a reservation.** Validate the body. Reject conflicts (see [§3.2](#32-conflict-detection)). The created reservation must be addressable by a subsequent read.
- **Replace a reservation in full.** Idempotent. Subject to the same conflict rule as create.
- **Apply a partial change to a reservation** — typically a time shift or holder change. Behavior under retries must be specified.
- **Cancel a reservation.**

### Aggregation

- **Resource utilisation summary** — given a resource and a caller-supplied window (`from`, `to`), return the resource together with an aggregated view of utilisation across the reservations in that window. Choose the metrics you can defend (e.g. total reserved minutes, count of distinct holders, peak concurrency).

### Operational

- **Liveness probe** — cheap, in-process only. Does not touch upstreams. Tells the orchestrator whether to restart the process.
- **Readiness probe** — reflects the state of upstream dependencies. Tells the load balancer whether to route traffic. What counts as "ready" is your call — discuss in [§6.10](#610-deployment).

### 3.1 Hydration & timezone projection

A **hydrated reservation** is the reservation joined with its resource's identifying metadata and projected into the resource's local timezone. At minimum it must surface: the reservation's identity, its resource (or enough of the resource for a consumer to act on it), the holder, the start and end as UTC instants, the equivalent local-time projection, and the duration. An illustrative shape — **not** a required shape — is:

```json
{
  "id": 42,
  "resourceId": 1,
  "resource": { "name": "Conference Room A", "kind": "room", "capacity": 8, "timezone": "Europe/Lisbon" },
  "holder": "alice@example.com",
  "startsAt": "2026-06-01T09:00:00Z",
  "endsAt":   "2026-06-01T10:00:00Z",
  "localStartsAt": "2026-06-01T10:00:00+01:00",
  "localEndsAt":   "2026-06-01T11:00:00+01:00",
  "durationMinutes": 60
}
```


### 3.2 Conflict detection

Create and full-replace must reject reservations whose window overlaps an existing reservation for the same `resourceId`. Define the exact semantics (inclusive vs. exclusive endpoints, what "overlap" means for back-to-back windows) and document them in [§6.2](#62-service-boundaries--contracts). Surface conflicts in a way that a client can act on programmatically.

## 4. Constraints & Ground Rules

- Use any open-source libraries but be sure that the sources used are credible. 
- Do not modify the contracts of the upstream services
- Do not rely on an external database. Process state may be ephemeral for purposes of the exercise POC. However, design discussion in [§6.3](#63-data-model--consistency) must treat persistence as if it were real.
- AI-assistance is permitted but offloading any issues found as overlooked AI mistakes is indefensible. Every design decision will be discussed in the follow-up. Be ready to defend the design as submitted.

## 5. Deliverables

1. The code, runnable per [§6.0](#60-development) and deployable per [§6.10](#610-deployment).
2. This README, with every section under [§6](#6-candidate-sections-to-fill-in) filled in. Replace each `<TODO …>` block with your answer.

---

## 6. Candidate Sections to Fill In

> Replace every `<TODO …>` placeholder. Do **not** delete the headings — we use them as a rubric.

### 6.0 Development

<TODO - describe how to run the application in development mode, develoment tooling, how to test, how to build (if applicable) >

### 6.1 Architecture Overview

<TODO — describe the overall architecture, the runtime topology, the framework choice and the rationale? Speak to concurrency, backpressure & limits>

### 6.2 Service Boundaries & Contracts

<TODO — describe ownership, public contracts (HTTP shape, error shape), and the reasoning for the boundary placement. If you would draw the boundaries differently in a real production setting, say so here.>

### 6.3 Data Model & Consistency

<TODO — describe the data model, the invariants, the production persistence story, and the consistency model. Be explicit about which reads can be stale and which cannot.>

### 6.4 Failure Modes & Status-Code Strategy

<TODO — enumerate the upstream failure modes and the resulting downstream Behavior and status codes. Cover retries, idempotency, and the "partial success" question.>

### 6.5 Validation Strategy

<TODO — describe inbound and outbound validation. Name the library/approach. Decide whether you treat upstream responses as trusted or as untrusted, and justify.>

### 6.6 Testing Strategy

<TODO — describe the test strategy >

### 6.7 Observability

<TODO — describe the observability surface. Logging shape, metric names and types, tracing strategy, alerts... >

### 6.8 Security

<TODO — describe the security posture. Be explicit about what is in scope for the exercise and what would be required in production >

### 6.9 Scaling & Performance

<TODO — identify the first bottleneck, then the second. Describe the horizontal-scaling plan, caching strategy (if any), and the conditions under which you would change the architecture rather than scale the existing one.>

### 6.10 Deployment

<TODO — describe how this architecture is deployed. Cover at minimum:
- Package / image / artifact provenance.
- Topology.
- Configuration & secrets
- Release strategy 
- Rollback story
- Local-dev → staging → production parity acknowleding accepted divergence points
- Follow up concerns — e.g. log shipping, metrics scraping, on-call paging path

### 6.11 Trade-offs & What You Would Do Next

<TODO — list the trade-offs you made and what you would tackle in the next phase. Be specific (i.e. "add tests" is too general, what kind? what for?)

### 6.12 Other Notables

<TODO - anything else that is noteworthy>

### 6.13 AI Usage

<TODO - was AI used? if so describe *exactly* how AI has been used>