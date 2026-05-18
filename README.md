# ops-if

---

## The problem

When something breaks in production, the first 10–15 minutes are wasted just answering three questions:

- **What broke?** Dig through pipeline logs, error trackers, deployment history.
- **What else is affected?** Trace service dependencies manually, usually from memory.
- **What do we do?** Find the right runbook, ping the right person, hope the context is still fresh.

This happens every time. The tooling exists — Grafana, PagerDuty, GitHub Actions — but nothing connects it. Each system fires its own alert in isolation, and a human has to assemble the full picture under pressure.

**ops-if** holds that picture continuously, so by the time a human looks, the work is already done.

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
