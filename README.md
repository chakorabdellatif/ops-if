# incident-lens

> An AI-powered incident intelligence system that maintains live connections to your CI/CD and monitoring stack, detects cascading failures as they form, maps what's affected, and delivers a structured remediation report — before anyone opens a dashboard.

---

## The problem

When something breaks in production, the first 10–15 minutes are wasted just answering three questions:

- **What broke?** Dig through pipeline logs, error trackers, deployment history.
- **What else is affected?** Trace service dependencies manually, usually from memory.
- **What do we do?** Find the right runbook, ping the right person, hope the context is still fresh.

This happens every time. The tooling exists — Grafana, PagerDuty, GitHub Actions — but nothing connects it. Each system fires its own alert in isolation, and a human has to assemble the full picture under pressure.

**incident-lens** holds that picture continuously, so by the time a human looks, the work is already done.

---

## How it works

```
GitHub Actions / GitLab CI
Grafana Alertmanager
Prometheus
        │
        │  live connections (SSE / webhook streams)
        ▼
┌─────────────────────────────────────┐
│  Ingestion API  (FastAPI)           │
│  HMAC validation · idempotency      │
│  rate limiting (Redis)              │
└────────────┬────────────────────────┘
             │
             ▼
         Apache Kafka
    ┌──────────────────────┐
    │  events.raw          │  ← all incoming signals
    │  events.alerts       │  ← agent decisions
    │  events.dlq          │  ← failed / poison
    └──────┬───────────────┘
           │
     Worker pool (async consumers)
           │
           ▼
    ┌───────────────────────────┐
    │      AI Agent             │
    │  1. Correlate signals     │
    │  2. Classify failure      │
    │  3. Map blast radius      │
    │  4. Write remediation     │
    │  5. Route by severity     │
    │  6. Detect patterns       │
    └───────┬───────────────────┘
            │
     ┌──────┴────────┐
     ▼               ▼
 PostgreSQL        Redis
 event store       locks · cache
 dependency graph  rate limiter
 audit log         dedup window
            │
            ▼
     Alerting layer
  Slack · PagerDuty · Webhook
            │
            ▼
  Prometheus · Grafana · Loki
```

The key difference from a standard webhook processor: the system maintains **persistent connections** to upstream sources. It doesn't wait for a webhook to arrive — it watches Grafana alertmanager, Prometheus rules, and CI streams continuously, so it can correlate a slow deploy + rising error rate + a recent config push into a single coherent incident, not three separate pings.

---

## What the AI agent does

For every failure signal — or correlated group of signals — the agent:

1. **Correlates** — groups related signals within a rolling time window before acting
2. **Classifies** — infra / test / build / security / config
3. **Maps blast radius** — which services, environments, and teams are affected, using the dependency graph stored in PostgreSQL
4. **Generates remediation** — concrete steps tailored to the failure type, not generic advice
5. **Assigns severity** — P1–P4, routed to the right channel
6. **Detects patterns** — same pipeline fails 3× in a rolling window → escalates as systemic, not a fluke

---

## Supported integrations

| Source | Connection type | Events |
|---|---|---|
| GitHub Actions | Webhook + SSE stream | push, workflow_run, deployment, release, security_advisory |
| GitLab CI | Webhook | pipeline, merge_request, deployment |
| Grafana Alertmanager | Live webhook receiver | firing, resolved |
| Prometheus | Alerting rules → webhook | threshold breaches |

Alerts dispatch to: Slack (rich blocks), PagerDuty (P1/P2 only), REST webhook.

---

## Tech stack

| Layer | Technology | Role |
|---|---|---|
| API | FastAPI | Ingestion, validation, rate limiting |
| Event bus | Apache Kafka | Raw events, alert decisions, DLQ |
| Workers | Python (asyncio) | Concurrent consumer pool |
| AI agent | LLM via API | Correlation, classification, blast radius, remediation |
| Database | PostgreSQL | Event store, dependency graph, outbox, audit log |
| Cache / locks | Redis | Distributed locks, sliding window rate limiter, dedup |
| Metrics | Prometheus | Scrapes API, workers, Kafka, Postgres, Redis, containers |
| Dashboards | Grafana | Pipeline health, agent latency, alert audit |
| Logs | Loki | Structured log aggregation across all services |
| Containers | cAdvisor + node-exporter | Per-container and host-level metrics |
| Runtime | Docker Compose | Single-command local environment |

---

## Distributed systems concepts in practice

| Concept | Where it shows up |
|---|---|
| Idempotency | GitHub retries the same delivery ID — safe to reprocess |
| Exactly-once processing | Transactional outbox — no message loss on worker crash |
| Retries + backoff | Agent calls fail → exponential backoff with jitter |
| Dead-letter queue | Unparseable payloads and repeated failures land in DLQ |
| Distributed locks | Workers race to claim events → Redis `SET NX EX` |
| Optimistic concurrency | `SELECT FOR UPDATE SKIP LOCKED` for job claiming in Postgres |
| Rate limiting | Redis sliding window per source |
| Consumer groups | Horizontal scaling without duplicate processing |
| Event replay | Reprocess N hours from Kafka after an agent bug fix |
| Graceful shutdown | Workers drain in-flight jobs before SIGTERM completes |
| Fault tolerance | Degrades gracefully when the AI agent is unavailable |
| Signal correlation | Stateful window aggregation across related failure events |

---

## Observability

### Prometheus metrics

| Metric | What it tells you |
|---|---|
| `events_ingested_total` | Signal volume by source and type |
| `events_processing_duration_seconds` | p50/p95/p99 worker latency |
| `agent_analysis_duration_seconds` | AI agent response time per classification |
| `kafka_consumer_lag` | Queue backlog per topic |
| `retry_total` by reason | Where failures concentrate |
| `alerts_dispatched_total` by severity | P1–P4 breakdown over time |
| `dlq_depth` | Poison message accumulation |
| `blast_radius_score` | Distribution of incident impact scores |

### Pre-built Grafana dashboards

- **Incident feed** — live view of correlated signals, blast radius, and remediation output
- **Pipeline health** — failure rate by repo, branch, and job type
- **Agent performance** — classification latency, pattern detections, severity distribution
- **Platform health** — queue depth, worker saturation, retry storms, DLQ growth
- **Alert audit** — every alert dispatched: severity, blast radius, time-to-alert

---

## Running locally

```bash
git clone https://github.com/your-username/incident-lens
cd incident-lens

cp .env.example .env
# Set: AI provider API key · Slack webhook URL · GitHub webhook secret

docker compose up -d
```

| Service | URL |
|---|---|
| Ingestion API | http://localhost:8000 |
| API docs | http://localhost:8000/docs |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Loki | http://localhost:3100 |
| Kafka UI | http://localhost:8080 |

### Simulate a cascading failure

```bash
# Fire a failing deploy + a Grafana alert at the same time
./scripts/simulate/cascading_failure.sh

# Watch the agent correlate both signals into one incident
docker compose logs -f agent-worker

# See the structured incident report in Grafana → Incident Feed
```

---

## Chaos exercises

Break each layer deliberately. Each has a matching runbook in `docs/runbooks/`.

```bash
# Kill a worker mid-processing — watch graceful drain
docker compose stop worker-2

# Flood the DLQ with malformed payloads
./scripts/chaos/flood_dlq.sh

# Throttle the AI agent — observe fallback degradation
./scripts/chaos/throttle_agent.sh

# Partition Kafka from workers — watch consumer lag spike
./scripts/chaos/kafka_partition.sh

# OOM a container — catch it in cAdvisor before you notice
./scripts/chaos/memory_pressure.sh worker-1
```

---

## Build stages

| Stage | What you build | Key concepts |
|---|---|---|
| 1 | API + PostgreSQL | Idempotency, HMAC validation, transactional outbox |
| 2 | Kafka + workers | Consumer groups, DLQ, retry + backoff |
| 3 | Live connections | SSE streams, Grafana alertmanager integration, signal correlation |
| 4 | AI agent | Classification, blast radius, remediation, pattern detection |
| 5 | Alerting | Slack/PagerDuty dispatch, severity routing, deduplication |
| 6 | Observability | Prometheus, Grafana, Loki, cAdvisor, node-exporter |
| 7 | Chaos engineering | Break each layer, observe, write runbooks |

---
