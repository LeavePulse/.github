# LeavePulse

**Next-generation monitoring and community platform for Minecraft servers.**

LeavePulse is a distributed microservices system that gives Minecraft server operators
production-grade observability, real-time event streaming, whitelist and verification
workflows, billing, and a customisable community layer — all behind a single platform API.

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
      │  Nuxt 4 frontend │      │  ClickHouse      │      │  plugins &       │
      │  NATS JetStream  │      │  Redis, MinIO    │      │  gateway traffic │
      └──────────────────┘      └──────────────────┘      └──────────────────┘
```

Operator access lands on `proxy-vps` via WireGuard; east-west traffic between app and
stateful nodes stays on private addresses. Only the edge and the collector are
publicly reachable.

---

## Stack

**Backend** — Python 3.14, [Litestar](https://litestar.dev), SQLAlchemy 2 async,
asyncpg, msgspec, uv, Ruff.

**Data** — PostgreSQL (primary OLTP), ClickHouse (time-series metrics and events),
Redis (cache + coordination), MinIO (object storage).

**Messaging** — NATS JetStream with domain event streams
(`{domain}.{entity}.{action}`) and durable consumers with queue groups.

**Observability** — Prometheus, OpenTelemetry traces, structured JSON logs.

**Performance core** — Rust via [PyO3 0.28](https://pyo3.rs) and
[maturin](https://www.maturin.rs) for hot-path algorithms
(MOTD normalisation, segment state resolution, log1p scoring, sitemap text
normalisation, whitelist import conflict resolution).

**Frontend** — Nuxt 4, Vue 3 Composition API, TypeScript, Tailwind CSS, Pinia for
auth, entity-repository caches with TTL for everything else. `bun run typecheck`
gates merges.

**Minecraft integration** — Java plugin targeting Bukkit, Velocity, and Fabric.

**Infrastructure** — Cloudflare Tunnel, nginx edge, WireGuard operator network,
custom deploy toolkit with inventory-driven firewall and WireGuard apply steps.

---

## Services

The platform is composed of roughly 13 Python microservices plus the frontend and
the Minecraft agent. Services communicate over NATS for events and over the
platform API for synchronous calls.

| Service | Responsibility |
| --- | --- |
| `platform-api` | Unified public HTTP facade over the service mesh. |
| `gateway` / `realtime-gateway` | WebSocket ingress for live events and operator dashboards. |
| `agent-gateway` | Ingress for in-game agents and plugin traffic. |
| `auth-service` | Identity, sessions, JWT issuance. |
| `verification-service` | Account / player linkage and verification flows. |
| `whitelist-service` | Whitelist state, imports, and conflict resolution (Rust-backed). |
| `monitoring-service` | Server state, segments, availability scoring (Rust-backed). |
| `server-service` | Server profile, sitemap text, metadata (Rust-backed). |
| `community-service` | Community features: tickets, threads, moderation. |
| `billing-service` | Subscriptions and billing events. |
| `bot-service` | Bot runtime and dispatch. |
| `discord-bot-adapter` | Bridge between bot-service and Discord. |
| `update-service` | Platform update orchestration. |
| `frontend` | Nuxt 4 web UI. |
| `minecraft-agent` | Java plugin (Bukkit / Velocity / Fabric). |

---

## Shared libraries (public)

The most reusable infrastructure pieces are open source and live in this
organisation:

- **[`service-toolkit`](https://github.com/LeavePulse/service-toolkit)** —
  shared Python toolkit: `create_service_app()` factory, JWT middleware,
  `BaseEventBus`, SQLAlchemy config helpers, NATS and Redis clients, Prometheus
  metric helpers with `ThrottledGaugeRefresh`. Absorbs ~100–120 lines of
  boilerplate per service main module.

- **[`leavepulse-core`](https://github.com/LeavePulse/leavepulse-core)** —
  Rust crate (PyO3 0.28 + maturin) for hot-path work: MOTD parsing, segment
  interval / state resolution, batched scoring, sitemap text normalisation,
  whitelist import candidate resolution. Used from monitoring, server, and
  whitelist services.

- **[`leavepulse-public-api-contract`](https://github.com/LeavePulse/leavepulse-public-api-contract)** —
  public API contract types shared between the platform API and external
  clients.

- **[`env-settings`](https://github.com/THEROER/env-settings)** — typed
  settings loader built on top of `msgspec.Struct`, prefix-aware
  `BaseSettings.load()`, used by every Python service.

---

## Engineering principles

- **Reuse first.** Before adding a helper, DTO, or utility, look for an existing
  pattern in the service and in the shared toolkit. If the same logic lands in
  two places, it gets extracted in the same change.
- **Transport vs. domain.** API contracts and transport-facing payloads are
  `msgspec.Struct`. Internal domain state is frozen slotted dataclasses.
  Database rows are SQLAlchemy 2 `Mapped` types. The lines do not blur.
- **App bootstrap through `service-toolkit`.** Most services use
  `create_service_app()` instead of wiring Litestar, middleware, DB, and event
  bus by hand. A few services (bot, agent-gateway, realtime-gateway,
  auth-service) keep a custom factory for good reason.
- **Events are first-class.** Cross-service communication defaults to NATS
  JetStream. Subjects follow `{domain}.{entity}.{action}`; payloads are wrapped
  in a standard envelope `{event_id, event_type, occurred_at, data}`.
- **Observability is not an afterthought.** Every service publishes Prometheus
  metrics and OpenTelemetry traces; label cardinality is guarded by shared
  helpers.
- **Type checks are CI gates.** `uv run python -m pytest` on the backend and
  `bun run typecheck` on the frontend must pass before a change ships.

---

## Status

LeavePulse is under active development. The majority of application code is
private while the platform stabilises; the shared infrastructure libraries
above are public and MIT-licensed where applicable, so you can see how the
platform is built in practice.

Questions or interest in collaborating? Open an issue on any of the public
repositories.
