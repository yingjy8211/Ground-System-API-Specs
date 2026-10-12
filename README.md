# Ground System API Architecture — Pattern Guide

A generic, reusable guide for designing a standards-based, multi-mission spacecraft
ground system API. Based on public standards (OpenAPI, AsyncAPI, CCSDS, IETF RFCs)
and general design practice. Contains no mission-specific schemas, endpoints, or
organizational details.

---

## 1. System Decomposition

### 1.1 Three kinds of actors

Classify every party before designing any interface:

| Kind | Meaning | Interface style |
|------|---------|-----------------|
| **Internal subsystem** | Part of your ground system; you own its API | Full subsystem API (REST + async) |
| **External facility** | Separate organization/system you exchange data with | Formal interface spec, inbound/outbound split |
| **Operated asset** | Something you *operate* (spacecraft) — not a peer | Reached through your command/telemetry subsystem, never modeled as an "external interface" |

Getting this classification wrong is the most common source of bad boundaries.

### 1.2 Typical internal subsystems

- Telemetry & Command — real-time telemetry, health, command uplink
- Mission Planning — activities, timelines, command loads, contact schedules
- Data Management — file catalog, distribution, archive; **hub for all external exchanges**
- Data Analysis — trending, reports
- Simulation — procedure validation, constraint checks, flight software testing
- Status Monitoring — ground system health, anomalies, notification
- Mission Automation — rules/triggers/autonomous procedures

### 1.3 Typical external facilities

Attitude determination, flight dynamics (orbit/navigation), science operations,
conjunction assessment, ground station network.

### 1.4 Hub-and-spoke for external interfaces

Route every external exchange through one hub subsystem (Data Management):
- One place for ingest validation, provenance, archival, and distribution
- External partners integrate once, not with N subsystems
- Internal subsystems subscribe to the hub's events rather than polling partners

---

## 2. REST vs Async Decision Rule

```
Real-time (continuous stream or sub-second latency needed)?
├─ NO  → REST (OpenAPI 3.1): CRUD, queries, history, configuration
└─ YES → Async (AsyncAPI 3.0): telemetry streams, alerts, command acks,
                                live status, event notifications
```

Most subsystems need **both**: a REST spec for resources and an async spec for
state-change events about those same resources. Keep payload schemas shared
between them so an event body and a GET response are the same shape.

---

## 3. Specification Layout

### 3.1 Repository structure

```
main/
  rest-entry.yaml            # aggregates all subsystem paths
  async-entry.yaml           # aggregates all channels
  standards/                 # standards-mapping specs (e.g. MAL/COM layer)
  common/
    schemas/                 # shared types (see §4)
    parameters/              # shared query params (pagination, time ranges)
    responses/               # shared error responses
subsystems/<name>/
  <name>-api.yaml            # OpenAPI
  <name>-streams.yaml        # AsyncAPI
  paths/<resource>.yaml
  schemas/<resource>.yaml
interfaces/
  internal/                  # subsystem-to-subsystem contracts
  external/<facility>/
    <facility>-interface-api.yaml
    paths/inbound.yaml       # facility → you
    paths/outbound.yaml      # you → facility
    schemas/
file-formats/                # non-JSON exchange formats (CSV, standard messages)
governance/                  # versioning, health, capabilities docs
tools/                       # validation, bundling, ICD generation
```

### 3.2 Modularization rules

- One file per resource for paths and schemas; subsystem file is a thin index.
- Split external interfaces by **direction** (inbound/outbound) — this maps
  directly to ICD sections and ownership.
- Entry points reference subsystem files; subsystem files never reference entry points.

### 3.3 `$ref` gotchas (learned the hard way)

- JSON Pointer escaping: `/` → `~1`, `~` → `~0`.
  `$ref: '../x/api.yaml#/paths/~1activities'`
- Path templates in pointers: braces must be percent-encoded in the URI fragment:
  `/activities/{id}` → `#/paths/~1activities~1%7Bid%7D`
- Relative refs resolve from the **referencing file's** directory — entry points in a
  subfolder need `../`.
- Every schema named in a `$ref` must actually exist; run a bundler as a lint step
  to catch dangling refs early.

---

## 4. Shared Schema Library

Build these first; everything else depends on them.

| File | Contents |
|------|----------|
| temporal types | RFC 3339 timestamp, time interval (start/end), duration (ISO 8601), bi-temporal envelope (§5.4), version history entry |
| common types | UUID identifiers, status enums, priority enum, quantity-with-unit, metadata (created/modified by/at, tags), links |
| error responses | RFC 7807/9457 Problem Details (`type`, `title`, `status`, `detail`, `instance`) + domain extension fields; one response per HTTP error class |
| pagination | Offset/limit **and** cursor-based variants; response envelope with `items`, `total`, `next`/`prev` links |
| spatial types | Geodetic coordinates, 3-vectors with frame + units, quaternion attitude, Keplerian elements, state vectors with epoch and reference frame |

Rules:
- All timestamps RFC 3339 with explicit offset (prefer `Z`).
- Every physical quantity carries its unit and reference frame. Never bare numbers.
- Enums are UPPER_SNAKE_CASE and documented per value.

---

## 5. Resource Design Principles

### 5.1 Identity
Every resource has an immutable unique ID (UUID) and a canonical URI.

### 5.2 Immutability & versioning
Updates create a new version; old versions remain addressable
(`/resources/{id}/versions/{n}`). Gives a free audit trail.

### 5.3 Idempotency
- Clients send an `Idempotency-Key` header on POST.
- Server stores key → result for a retention window; repeats return the same result.
- Async consumers must tolerate at-least-once delivery (dedupe on event ID).

### 5.4 Bi-temporality
Every fact has two time axes:
- **Valid time** — when it is true in the world (e.g. ephemeris applicable span)
- **Transaction time** — when the system recorded it

Query params: `validAt`, `asOf` (transaction time). Lets you answer
"what did we believe at time T about time X?" — essential for anomaly review
and corrected predictions.

### 5.5 HATEOAS links
Responses include `_links` (self, related resources, available state transitions).
Clients discover workflow steps rather than hard-coding them.

### 5.6 Approval workflows for hazardous operations
Commands/loads move through explicit states:
`DRAFT → SUBMITTED → APPROVED → QUEUED → SENT → VERIFIED | FAILED`.
Model approval as a sub-resource (`/commands/{id}/approvals`), support
multi-approver rules, and emit an event on every transition.

---

## 6. Versioning Strategy (three tiers)

1. **API major version** in URL path: `/v1/...` — breaking changes only.
2. **Resource version** — per-resource revision history (§5.2).
3. **Representation version** — media type negotiation:
   `Accept: application/vnd.<org>.<resource>.v2+json`

---

## 7. Governance Endpoints

| Endpoint | Answers | Returns |
|----------|---------|---------|
| `GET /health` | Is it running? | Aggregate + per-subsystem + per-external-link status |
| `GET /capabilities` | What can it do? | Enabled features, available subsystems and interfaces |
| `GET /governance/versions` | What's deployed? | Module versions, spec versions, standards versions implemented |

Cheap to build, very valuable for multi-mission onboarding and ops dashboards.

---

## 8. Domain Modeling Patterns

### 8.1 Events vs Activities

| | Events | Activities |
|-|--------|------------|
| What | Things that happen (eclipse, AOS/LOS, apsides) | Things you choose to do (slew, observe, downlink) |
| Control | Predicted, not controllable | Scheduled by planners |
| Role | Constraints and triggers | Planned operations |

Events constrain when activities may be scheduled. Keep them separate resources.

### 8.2 Request → plan → execution → report

Most external exchanges follow the same lifecycle; model each stage as its own resource:
- Request (e.g. observation request, contact request, maneuver request)
- Plan/schedule (accepted, scheduled instance)
- Execution status (live, via async events)
- Report (post-execution metrics, quality, products)

### 8.3 Automation model

Decompose autonomy into composable resources:
- **Trigger** — telemetry threshold, event, time, external signal, compound (AND/OR)
- **Condition / Constraint** — time windows, state, resource, mutual exclusion
- **Action** — atomic operation
- **Procedure / Workflow** — steps with branching and parallelism
- **Rule** — condition → action mapping
- **Execution** — runtime instance with lifecycle: start, pause, resume, abort
- **Schedule** — one-time, recurring, cron, or tied to orbital events
- Plus: variables/contexts, audit log, metrics

Emit async events for every trigger activation, execution state change, and
constraint violation/bypass.

---

## 9. Standards Alignment (all public)

| Standard | Use |
|----------|-----|
| CCSDS 521.0 — Mission Operations MAL | Interaction patterns: SEND, SUBMIT, REQUEST, INVOKE, PROGRESS, PUBLISH-SUBSCRIBE. Map to HTTP (sync/202+status polling) and async channels |
| CCSDS 522.0 — Mission Operations COM | Archive, Event, Activity Tracking services; object identity model |
| CCSDS 502.0 — Orbit Data Messages (OPM/OMM/OEM) | Ephemeris and state exchange |
| CCSDS 503.0 — Tracking Data Message | Range, Doppler, angles |
| CCSDS 504.0 — Attitude Data Messages | Quaternions, attitude ephemeris |
| CCSDS 508.0 — Conjunction Data Message | Collision assessment |
| RFC 3339 | Timestamps |
| RFC 7807 / 9457 | Error bodies |
| CloudEvents | Async event envelope |

**MAL mapping sketch:**
- SEND → fire-and-forget async publish
- SUBMIT → `POST` returning 202/204
- REQUEST → `GET`/`POST` returning 200 with body
- INVOKE → `POST` 202 + async ack, then result event
- PROGRESS → `POST` 202 + stream of progress events + final result
- PUBLISH-SUBSCRIBE → AsyncAPI channel + REST subscription management resource

---

## 10. Async Design Conventions

- Channel naming: `<domain>.<subsystem>.<resource>.<event>` (e.g.
  `gs.automation.execution.completed`)
- Envelope: CloudEvents attributes (`id`, `source`, `type`, `time`, `subject`)
  plus `correlationId` linking back to the originating REST request.
- Alert events carry severity, acknowledgement requirement, and a link to the
  REST resource for detail.
- Version event types in the type string (`...completed.v1`).

---

## 11. Tooling

- Validation: `@redocly/cli lint`, Spectral rulesets, `asyncapi validate`,
  `openapi-generator-cli validate`
- Bundling: `redocly bundle` to produce a single-file spec (catches broken refs)
- Docs/ICDs: render with Redoc / AsyncAPI generator; generate ICD sections per
  external interface from the inbound/outbound split
- Codegen: openapi-generator for client SDKs as a check that specs are generatable
- CI: lint + bundle + validate on every PR

---

## 12. Workflow Conventions

- Branch: `{type}/issue-{n}-{desc}`; commit: `{type}: {desc} (Issue #{n})`
- One issue = one branch = one focused change; don't extend a branch after its PR opens
- Every subsystem PR includes: REST spec, async spec, shared schemas updates,
  validation passing, docs/changelog update

---

## 13. Suggested Rebuild Order

1. Repo skeleton, tooling, CI lint/validate
2. Shared schema library (§4) and standard error/pagination responses
3. Governance endpoints (§7)
4. One complete internal subsystem as the template (Mission Planning works well —
   exercises events, activities, bi-temporality, HATEOAS)
5. Data Management hub
6. External interfaces, one at a time, inbound/outbound split
7. Remaining internal subsystems, copying the template's structure
8. Standards mapping layer (MAL/COM)
9. ICD generation and client codegen

---

## 14. Clean-Room Notes

- Write everything fresh from this guide and public standards; do not copy files,
  schemas, or text from prior work products.
- Use your own naming, domain, and examples.
- Check your employment/contract IP terms before publishing.
