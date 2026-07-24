# Production Platform Architecture

**A solo-built, solo-operated microservices platform running a real business in production since 2023.**

**▶ Interactive version: [rahb3rt.github.io/platform-architecture](https://rahb3rt.github.io/platform-architecture/)**

This document describes the architecture of a production platform I designed, built, and operate end-to-end: 15 services plus embedded vehicle hardware, with CI/CD, observability, SLO tracking, and multi-environment deployment behind a self-service control plane. The application code is proprietary (it runs my company); this repo documents the engineering.

**By the numbers:** 15 services · 16,000+ jobs scheduled · 5,100+ invoices processed · 58,000+ vehicle telemetry readings · hourly per-tenant backups · 1 operator. Counts are `SELECT COUNT(*)` aggregates from the production database.

---

## System Topology

```mermaid
flowchart TB
    subgraph edge [Edge]
        NGINX[nginx reverse proxy<br/>TLS termination, routing]
    end

    subgraph apps [Application Services]
        WEB[Public website<br/>Next.js]
        APP[Operations dashboard<br/>Next.js]
        KIOSK[On-site kiosk<br/>TypeScript]
        PORTAL[Customer portal<br/>Next.js · magic-link auth]
        API[Core API<br/>Python / FastAPI]
    end

    subgraph workers [Async & Integration Services]
        SMS[SMS service<br/>Python]
        EMAIL[Email ingest & send<br/>JavaScript]
        MAIL[Transactional mail<br/>HTML templating]
        PAY[Payment reconciliation<br/>Python]
        EXTRACT[Document extraction<br/>Python]
    end

    subgraph platform [Platform Services]
        HEALTH[Health aggregator<br/>FastAPI + Docker SDK]
        MON[Monitoring & SLO dashboard<br/>Next.js + RBAC]
        MYSQL[(MySQL 8.0)]
        MINIO[(MinIO object storage)]
    end

    subgraph field [Field Hardware]
        TELEM[Vehicle telemetry<br/>ESP32 / OBD-II / GPS / LTE]
    end

    NGINX --> WEB & APP & KIOSK & PORTAL & API
    API --> MYSQL & MINIO
    SMS & EMAIL & MAIL & PAY & EXTRACT --> API
    TELEM -->|gzip NDJSON over LTE| API
    HEALTH -.->|HTTP / MySQL / TCP probes| apps & workers & platform
    MON -.->|container stats, logs, SLOs| apps & workers & platform
```

## Service Inventory

| Service | Role | Stack |
|---|---|---|
| Core API | Business logic — 60 route domains, 520 endpoints | Python, Flask, MySQL |
| Public website | Customer-facing site | Next.js |
| Operations dashboard | Internal operations app | Next.js |
| Kiosk | On-site self-service | TypeScript |
| Customer portal | Balance, invoices, next visit, service requests | Next.js 15, passwordless magic-link auth |
| SMS / Email / Mail | Customer messaging (inbound + outbound) | Python, Node |
| Payment reconciliation | Matches external payments to invoices | Python |
| Document extraction | Parses inbound documents into structured data | Python |
| Health aggregator | Fleet-wide health checks with auto-discovery | Python, FastAPI, Docker SDK |
| Monitoring dashboard | Container metrics, logs, alerting, SLO tracking | Next.js, RBAC |
| Vehicle telemetry | OBD-II + GPS collection from field vehicles | C++ / ESP32 / PlatformIO |
| nginx | Reverse proxy, TLS, routing | nginx:alpine |
| MySQL, MinIO | Persistence and object storage | Managed as containers |

## Core API Surface

The core API is a Flask monolith-by-choice: **60 route domains, 520 endpoints**, one deployable.

| Area | Capabilities |
|---|---|
| Billing | invoicing, payments, payment methods, statements, quotes, customer credit, expenses |
| Contracts | templates, e-signing, audit trail |
| CRM | customers, properties, leads, requests, sites |
| Workforce | employees, teams, timeclock, timecards, time-off, payroll |
| Operations | jobs, scheduling, calendar, route optimization, mowing, plowing plans, service plans |
| Communications | messaging, notifications, marketing campaigns, social |
| Field & assets | vehicle telemetry, assets, access devices, access events, weather |
| Platform | auth, RBAC roles, orgs, dashboards, reports, insights, webhook deliveries |

## CI/CD

- Every service builds via **GitHub Actions → multi-arch images (amd64/arm64, Buildx + QEMU) → GHCR**.
- Semantic version tags (`v*.*.*`) cut releases; PRs build without publishing.
- Deployment is one idempotent tool (`stackctl`): pulls service repos, pre-builds the Next.js apps, provisions per-environment infrastructure, regenerates edge routes, and brings the stack up under Docker or Podman — rebuilding only the services whose commit SHA actually moved.

## Multi-Environment Deployment

One deploy system serves **N fully isolated environments on the same host** — production, staging, and a dedicated stack per customer are all the same mechanism, just differently named:

```
stackctl deploy production
stackctl deploy staging
stackctl deploy acme
```

Each stack gets its own network, its own MySQL server, its own object storage, its own secrets, and its own domain routes — no data infrastructure is shared between environments. The only shared component is a single Caddy edge proxy that terminates TLS and routes by hostname; each stack attaches its own routes at deploy time. A staging environment shaped exactly like production catches configuration drift before customers do, and the same isolation makes standing up a dedicated customer instance a one-command operation instead of a re-architecture.

The system is two layers: `stackctl` deploys environments, and a control plane turns them into a product.

### stackctl — the deploy layer

```bash
stackctl new acme --domain acmelawns.com --company "Acme Lawns LLC"
stackctl deploy acme               # rebuilds only services whose repo SHA moved
stackctl deploy acme api --force   # force one service
stackctl rollback acme api         # per-stack rollback to the previous image
stackctl backup acme               # gzipped dump; newest 72 retained (~3 days hourly)
stackctl restore acme <file>       # destructive; typed confirmation required
stackctl destroy acme --yes
stackctl proxy up | reload | status
```

- **Naming is the isolation boundary.** Everything is prefixed by stack name — network `acme-net`, containers `acme-api` / `acme-app`, database `acme`, storage `acme-minio`.
- **Container DNS, not IPs.** Services address each other by name on the stack network (`http://acme-api:5002`). The previous generation substituted live container IPs into an nginx config, which forced sequential container swaps and a proxy redeploy on every change. Deploy order is now irrelevant.
- **Change detection by SHA.** A service redeploys only when its repo SHA differs from the stack's recorded deployed SHA; already-built SHAs deploy from cache.
- **Image sharing where it's safe.** Backend services build one image per git SHA and share it across every stack. Next.js frontends bake per-environment branding at build time, so those build per stack.
- **Feature flags per environment.** `ENABLE_SMS`, `ENABLE_KIOSK`, `ENABLE_WEB`, … — a disabled service is neither deployed nor routed.
- **Automatic TLS.** Caddy provisions and renews Let's Encrypt certificates for every routed domain, including each new subdomain on first deploy.
- **Idempotent schema bootstrap.** A fresh database is seeded from the API repo's checked-in schema and tracked by a marker table, so re-running provisioning is safe.

### Control plane — the tenant layer

A Node/Express service with a React SPA sits on top of `stackctl`. **It never reimplements deploy logic** — every state change shells out to `stackctl`, and status is read from the stackctl registry plus a single container-engine query. Anything done by hand on the CLI is reflected in the UI automatically, so the two can't drift.

- **Signup → workspace.** A customer claims a name and gets a full isolated stack at `<name>.<base-domain>`, on wildcard DNS with certificates issued on first deploy.
- **Admin dispatch board.** Every stack as a grid with one cell per service, live status, the tenant list, and a job queue with a streaming log drawer.
- **Provisioning is gated by default.** Auto-provision is off: an anonymous form that immediately consumes a full stack's worth of RAM and disk is a denial-of-wallet vector, so signups queue for approval unless they're gated upstream.
- **Serialized jobs.** Deploys are heavy and not concurrency-safe per stack, so the job runner executes exactly one at a time.
- **Tenant security.** Owner/member roles, expiring invitations, and a per-tenant audit log.
- **Billing and licensing.** Subscription plans with entitlements (Community through Enterprise), annual terms and add-ons, plus Ed25519-signed licenses verified and enforced at the stack level for self-hosted installs.

The earlier single-business deploy script is still in the tree, untouched, as the rollback path. Its genericized multi-environment form is published at [compose-multienv-deploy](https://github.com/rahb3rt/compose-multienv-deploy).

## Reliability Engineering

**Health aggregation.** A dedicated service auto-discovers containers via the Docker/Podman API and probes each over HTTP, MySQL, or TCP on a background loop — no manual registration, no stale check configs.

**Monitoring & SLOs.** A purpose-built dashboard tracks container metrics, aggregates logs, fires alerts, and tracks SLOs, behind database-backed RBAC.

**Backups.** The control-plane process drives hourly per-tenant backups: one gzipped dump of both of a stack's databases (business + monitoring), written with drop-and-create semantics so a restore fully reconstructs them, mode-600 because the file is the whole dataset, and pruned to the newest 72 (~3 days). Restores are explicitly destructive and require typed confirmation.

**Design-for-failure at the edge.** The vehicle telemetry firmware assumes connectivity is unreliable and data loss is unacceptable: NDJSON rows persist to SD with size/time-based file rotation, survive reboots, and upload as gzip-compressed batches over LTE with retry and backoff.

## A Failure, on the Record

**Postmortem — June 2026 — "The observer effect."** The monitoring dashboard gained a live topology view (container stats, sparklines, request traces) on top of its real-user-monitoring ingest. The monitoring database lives on the same MySQL server as production, and within hours the topology view was timing out — every timeout representing load pressure on the database that also serves the business.

The first fix made it worse: parallelizing all DB queries and trace fetches multiplied the concurrent load on an already-stressed database, and was reverted the same day. The durable fix went the opposite direction — batch the RUM inserts, reduce query limits, poll slower, fetch fewer traces, and put explicit timeouts and error containment on every query so the dashboard degrades instead of hammering.

**The lesson: monitoring is production.** The observer carries the same load budget as the observed, and parallelism is not a fix for overload — it is a multiplier on it.

```
jun 08  ff77443  Fix topology timeout: batch RUM inserts into chunks of 25
jun 08  730466e  Fix topology timeout: parallelize all DB queries and trace fetches
jun 08  d4ce6c3  Revert "Fix topology timeout: parallelize all DB queries and trace fetches"
jun 08  9036fc7  Fix topology timeout: reduce query limits safely
jun 08  b743295  Fix topology timeouts: slower polling, fewer trace fetches
jun 09  4aa70e5  Fix topology loading: reduce limits, add timeouts, catch errors
```

## Security Posture

- **Isolation by default** — every tenant stack gets its own network, database, object storage, and secrets; there is no shared state to leak across.
- **Secrets out of band** — credentials live in per-environment env files injected at deploy time, never in images or git.
- **TLS at the edge** — each stack fronts through its own edge proxy with TLS termination; certificates auto-provisioned and renewed.
- **Passwordless customers** — portal sign-in is a 15-minute magic link over SMS; the request endpoint is enumeration-proof and rate-limited, sessions are stateless JWTs revocable by one secret rotation.
- **AuthN/AuthZ** — database-backed RBAC with role-to-permission mapping, session management, and a token blacklist for immediate revocation.

## Design Decisions

**Compose over Kubernetes.** I operate multi-tenant Kubernetes at day-job scale — which is exactly why this platform doesn't use it. On a single host, Kubernetes buys autoscaling, bin-packing, and rolling deploys this system doesn't need, and the price is a control plane to patch, upgrades to sequence, and a much larger failure surface to debug alone. Compose gives a one-file topology, deterministic deploys, and disaster recovery that amounts to "restore volumes, re-run the deploy script." The revisit trigger is explicit: a second host, or a genuine need for zero-downtime rollouts.

**Build vs. buy for monitoring.** Hosted observability is priced per-container and per-GB of ingest — for a single-host platform, that bill would rival the entire infrastructure budget. The actual requirements were narrow and Docker-native: container stats, log aggregation, alerting, and SLO tracking. Building a purpose-fit dashboard kept everything on one pane, kept data on-host, and let me implement SLO tracking the way I practice it professionally — explicit targets reviewed against reality, not dashboard-watching.

**Store-and-forward telemetry.** Field vehicles are the harshest environment in the system: LTE dead zones, uploads dying mid-flight, power cut at ignition-off. The firmware treats the SD card as the source of truth — every reading lands on disk as NDJSON before anything else, files rotate by size and time, and an uploader drains them as gzip-compressed batches with retry and backoff whenever connectivity allows. For telemetry, durability beats latency: a reading that arrives ten minutes late is fine; one that never arrives is not.

**Per-environment isolation on one host.** Production, staging, and dedicated customer instances run side by side, each with its own env file, network, containers, database, and volumes — one mechanism, differently named. The cost is honest: shared-nothing per environment means a full MySQL and object store per stack, which trades RAM and disk for a blast radius that stops at one environment — the right trade while stack count is small, and the thing to revisit before it isn't.

**One source of truth for deploys.** The control plane could have talked to the container engine directly and been faster to build. Instead every state change shells out to the same `stackctl` I use by hand, and status is read back from its registry. A second implementation of "what does deployed mean" is a guarantee that the UI and the CLI will eventually disagree — usually during an incident, when the dashboard is the thing you're trusting. Shelling out costs a process spawn; disagreeing costs the outage.

---

*The platform has been in continuous production operation since 2023. I'm happy to walk through any component in depth — including code — in an interview setting.*
