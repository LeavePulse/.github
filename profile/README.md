# LeavePulse

**Next-generation monitoring and community platform for Minecraft servers.**

LeavePulse is a distributed microservices system that gives Minecraft server operators
production-grade observability, real-time event streaming, whitelist and verification
workflows, billing, a customisable community layer, and a cross-platform game launcher —
all behind a single typed platform API.

---

## Architecture at a glance

```
                               ┌─────────────────────────┐
 Players / operators ────▶     │  proxy-vps (edge)       │
                               │  Cloudflare Tunnel      │
                               │  nginx + WireGuard      │
                               └──────────┬──────────────┘
                                          │ private network (10.200.0.0/24)
                ┌─────────────────────────┼─────────────────────────┐
                ▼                         ▼                         ▼
      ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
      │  app-vps-*       │      │  data-vps        │      │  collector-vps   │
      │  Python services │      │  PostgreSQL      │      │  ingest path for │
      │  Nuxt 4 frontend │      │  ClickHouse      │      │  plugins, pings  │
      │  NATS JetStream  │      │  Redis, MinIO    │      │  & gateway       │
      └──────────────────┘      └──────────────────┘      └──────────────────┘
```

Operator access lands on `proxy-vps` via WireGuard; east-west traffic between app and
stateful nodes stays on private addresses. Only the edge and the collector are
publicly reachable.

**Request flow.** The browser, launcher, and third-party clients talk to a single
public surface — `platform-api`, a typed BFF (backend-for-frontend). It fans out to
the domain services over an internal **gRPC mesh**; services in turn exchange domain
events asynchronously over **NATS JetStream**. In-game agents and plugins connect
through dedicated WebSocket gateways rather than the public HTTP API.

---

## Stack

**Backend** — Python 3.14, [Litestar](https://litestar.dev) 2, SQLAlchemy 2 async,
asyncpg, gRPC, msgspec, uv, Ruff.

**Data** — PostgreSQL (primary OLTP, one database per service), ClickHouse
(time-series server telemetry and metrics), Redis (cache + coordination), MinIO
(object storage).

**Messaging** — NATS JetStream with domain event streams
(`{domain}.{entity}.{action}`) and durable consumers with queue groups, wrapped in a
standard envelope (`{event_id, event_type, occurred_at, data}`).

**Observability** — Prometheus, OpenTelemetry traces, Grafana / Loki / Tempo,
structured JSON logs.

**Performance core** — Rust via [PyO3](https://pyo3.rs) and
[maturin](https://www.maturin.rs) for hot-path algorithms (MOTD normalisation,
segment state resolution, log1p scoring, sitemap text normalisation, whitelist import
conflict resolution). Shipped as the optional Rust crate inside `service-toolkit`.

**Frontend** — Nuxt 4, Vue 3 Composition API, TypeScript, Tailwind CSS 4, Pinia for
auth, entity-repository caches with TTL for everything else. The frontend consumes the
generated TypeScript SDK; `bun run typecheck` gates merges.

**Launcher** — Cross-platform Minecraft launcher built on a Rust core with a
[Tauri](https://tauri.app) 2 desktop UI (Vue 3 + TypeScript) and an Android runtime.
Handles instances, modloaders, and shareable/syncable modpack builds.

**Developer SDK** — A multi-language SDK (TypeScript, Python, Rust) generated from the
platform's OpenAPI and realtime contracts, with a resource-object API, typed error
hierarchy, and realtime support.

**Minecraft integration** — Java agent targeting Bukkit/Paper, BungeeCord/Waterfall,
Velocity, Fabric, and NeoForge.

**Infrastructure** — Cloudflare Tunnel, nginx edge, WireGuard operator network, and a
custom deploy toolkit with inventory-driven firewall, WireGuard, and ingress apply
steps, plus a control-plane (agent + service) for host orchestration.

---

## Services

The platform is composed of ~20 services — Python microservices, two Nuxt frontends,
a Rust server poller, and the Minecraft agent. Public synchronous traffic goes through
`platform-api`; services talk to each other over gRPC and NATS.

| Service | Responsibility |
| --- | --- |
| `platform-api` | Typed public BFF over the internal gRPC mesh — the single public HTTP API. |
| `auth-service` | Identity, sessions, JWT/JWKS, OAuth, RBAC. |
| `server-service` | Server & project catalog, ownership, team roles, sitemap text. |
| `monitoring-service` | Server state, availability segments and scoring, ClickHouse metrics. |
| `whitelist-service` | Whitelist forms, applications, entries, and imports (Rust-backed). |
| `verification-service` | Account / player linkage and verification flows. |
| `community-service` | Community engagement: comments, reactions, votes, moderation. |
| `billing-service` | Products, checkout, orders, and subscriptions. |
| `bot-service` | Bot runtime, task dispatch, and Discord guild sync. |
| `discord-bot-adapter` | Executes bot-service tasks against Discord. |
| `gateway` | Public WebSocket router dispatching client and agent traffic. |
| `realtime-gateway` | Client-facing realtime subscriptions (live stats, status, projects). |
| `agent-gateway` | Ingress for in-game agents and plugin traffic; event-to-command bridge. |
| `server-poller` | High-performance Rust poller (Java SLP + Bedrock RakNet). |
| `server-discovery-service` | Discovers external Minecraft servers from third-party catalogues. |
| `translation-service` | Internal text translation backend. |
| `launcher-service` | Launcher modpack builds, manifests, and share/sync. |
| `update-service` | Signed update manifests and rollout for the Minecraft agent. |
| `control-service` / `control-agent` | Control-plane for host orchestration and automation. |
| `frontend` / `control-panel` | Nuxt 4 web UIs (public site + operator panel). |
| `launcher` | Cross-platform launcher (Tauri desktop + Android). |
| `minecraft-agent` | Java agent (Bukkit/Paper, BungeeCord, Velocity, Fabric, NeoForge). |

---

## Shared libraries (public)

The most reusable infrastructure pieces are open source and live in this
organisation:

- **[`service-toolkit`](https://github.com/LeavePulse/service-toolkit)** —
  shared Python toolkit: `create_service_app()` factory, JWT middleware,
  `BaseEventBus`, SQLAlchemy and gRPC helpers, NATS and Redis clients, Prometheus
  metric helpers with `ThrottledGaugeRefresh`, and the `lp-ci` quality gate. Ships an
  optional **Rust crate** (PyO3 + maturin) for hot-path work — MOTD parsing, segment
  state resolution, batched scoring, sitemap text normalisation, and whitelist import
  candidate resolution — used by the monitoring, server, and whitelist services.
  Absorbs ~100–120 lines of boilerplate per service main module.

- **[`leavepulse-sdk-ts`](https://github.com/LeavePulse/leavepulse-sdk-ts)** ·
  **[`leavepulse-sdk-python`](https://github.com/LeavePulse/leavepulse-sdk-python)** ·
  **[`leavepulse-sdk-rust`](https://github.com/LeavePulse/leavepulse-sdk-rust)** —
  the generated client SDKs (TypeScript, Python, Rust) for the public platform API:
  resource-object API, typed RFC 7807 error hierarchy, conditional-GET caching, and
  realtime support. Generated from the platform's OpenAPI and realtime contracts.

- **[`env-settings`](https://github.com/THEROER/env-settings)** — typed settings
  loader built on `msgspec.Struct`: prefix-aware `BaseSettings.load()`, env / `.env` /
  YAML / TOML sources, and `*_FILE` secret fallback. Used by every Python service.

- **[`awesome-errors`](https://github.com/THEROER/awesome-errors)** — standardised
  error handling for Python services: typed exception hierarchy, RFC 7807 Problem
  Details rendering, converters, and Litestar / FastAPI integration.

---

## Engineering principles

- **Reuse first.** Before adding a helper, DTO, or utility, look for an existing
  pattern in the service and in the shared toolkit. If the same logic lands in
  two places, it gets extracted in the same change.
- **Transport vs. domain.** API contracts and transport-facing payloads are
  `msgspec.Struct`. Internal domain state is frozen slotted dataclasses.
  Database rows are SQLAlchemy 2 `Mapped` types. The lines do not blur.
- **One public surface.** External clients go through `platform-api`; domain services
  are reached only over gRPC and NATS. The SDK is generated from the public contract,
  so client and server cannot silently drift.
- **App bootstrap through `service-toolkit`.** Most services use
  `create_service_app()` instead of wiring Litestar, middleware, DB, and event
  bus by hand. A few services (bot, agent-gateway, realtime-gateway, auth-service)
  keep a custom factory for good reason.
- **Events are first-class.** Cross-service communication defaults to NATS
  JetStream. Subjects follow `{domain}.{entity}.{action}`; payloads are wrapped
  in a standard envelope.
- **Observability is not an afterthought.** Every service publishes Prometheus
  metrics and OpenTelemetry traces; label cardinality is guarded by shared helpers.
- **Quality is a CI gate.** The `lp-ci` gate (Ruff, MyPy, arch-lint, secret scanning,
  tests) on the backend and `bun run typecheck` on the frontend must pass before a
  change ships.

---

## Status

LeavePulse is under active development. The majority of application code is
private while the platform stabilises; the shared infrastructure libraries
above are public and MIT-licensed where applicable, so you can see how the
platform is built in practice.

Questions or interest in collaborating? Open an issue on any of the public
repositories.
