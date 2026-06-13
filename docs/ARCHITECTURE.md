# Architecture

## Overview

Six independently deployable Spring Boot services sit behind a reactive Spring Cloud Gateway. Services never share a database; they communicate two ways:

- **Synchronously** via OpenFeign HTTP clients wrapped in Resilience4j circuit breakers (e.g. ride-service looking up a user or driver).
- **Asynchronously** via RabbitMQ events that drive a **saga** for multi-service state changes (e.g. completing a ride and charging payment).

## Request lifecycle (a ride)

1. Client authenticates against **user-service** (via the gateway) and receives a JWT.
2. The gateway validates the JWT on every subsequent request and routes to the target service.
3. **ride-service** creates a ride, resolving rider/driver details through Feign calls (circuit-breaker protected).
4. On completion, ride-service emits `ride.completed` to RabbitMQ.
5. **payment-service** consumes the event, processes payment, and emits saga feedback.
6. ride-service and user-service consume the feedback to finalize ride state and update the rider's `totalSpent` ledger.
7. A cancellation (`ride.cancelled`) drives the compensating path so balances and ride state stay consistent.

## Distributed transactions — the saga

Rather than a 2-phase commit across services, the ride→payment workflow is a **choreography saga** over RabbitMQ:

- `ride.completed` / `ride.cancelled` are published by ride-service.
- payment-service is a saga participant (consume → act → emit feedback).
- Compensating actions on cancellation reverse the financial effects.

This keeps each service autonomous and the system available under partial failure, at the cost of eventual (rather than immediate) consistency.

## Persistence (polyglot)

| Store | Used by | Why |
|---|---|---|
| PostgreSQL 17 (one per service) | all 5 domain services | Authoritative relational state, isolated per service |
| Neo4j | ride-service | Rider↔driver interaction graph (`RODE_WITH` relationships) for graph queries |
| Redis | ride-service, location-service | Caching hot reads / analytics to cut Postgres load |

## Resilience

- **Circuit breakers (Resilience4j)** on Feign clients prevent a slow/unavailable dependency from cascading.
- **Async decoupling** — services that are down still process queued events when they recover.
- **Kubernetes probes** — liveness/readiness via Spring Boot health groups; `/actuator` and `/health` paths bypass JWT.

## Observability

- **Metrics** — each service exposes `/actuator/prometheus` via Micrometer; Prometheus scrapes them; Grafana renders a per-service dashboard.
- **Logs** — a loki4j logback appender ships structured logs to Loki, queryable in Grafana alongside metrics.
- **Tracing** — a `CorrelationIdFilter` puts a correlation ID into the MDC and propagates it across service hops, so a single request can be followed end-to-end in the logs.

## Deployment

- **Local** — `docker compose up --build` brings up all services, datastores, RabbitMQ, and the monitoring stack.
- **Kubernetes** — `k8s/` contains Namespaces, ConfigMaps, Secrets, StatefulSets (datastores), Deployments (services), Services, and the Prometheus/Grafana/Loki monitoring stack. Datastores run as StatefulSets with persistent volumes; services as stateless Deployments.
