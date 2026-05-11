# PIR-2024-03-17: Thread Pool Exhaustion in Payment Service

- **Incident ID:** INC-2024-03-17-001
- **Severity:** SEV0
- **Date:** 2024-03-17
- **Duration:** 2h 9m (1h 45m hard down)
- **Owner:** Payments Platform Team
- **Status:** Resolved

## Summary

A latency spike in a downstream processor's webhook endpoints (200ms → 11s p50) caused our 50-thread `processorCallback` executor to saturate. Because Jetty request threads were calling `future.get()` on the callback executor synchronously and without a timeout, all 1024 Jetty threads accumulated in `WAITING` state inside ~4 minutes. Health checks couldn't acquire a thread, hosts were marked unhealthy, the load balancer drained the entire fleet, and the service went hard-down across all production hosts. We recovered after a force-redeploy with fresh hosts.

The root anti-pattern is a "big pipe / small pipe" mismatch between two thread pools (1024 vs 50), bridged by a synchronous blocking handoff (`future.get()` with no timeout). When the consumer pool saturates, the producer pool drains into `WAITING` state and the service stops accepting any work — including health checks.

## Learnings

This incident produced three distinct learnings that need to propagate across the org. Each requires a different remediation channel.

---

### Learning 1 — CODE_FIX: Mismatched thread pools with synchronous blocking handoff

**Type:** `CODE_FIX`

**Anti-pattern:**

A service has two thread pools: a large producer pool (Jetty / dispatcher / request-handling, typically 1024) and a smaller consumer pool used for outbound calls (typically 50–100). The producer thread submits work to the consumer pool and synchronously blocks on the result via `Future.get()` with no timeout.

```java
// Producer thread (Jetty / dispatcher) calls into consumer executor
Future future = callbackExecutor.submit(() ->
    httpClient.post(externalEndpoint, payload)
);

// ANTI-PATTERN: synchronous blocking with no timeout.
// When the consumer pool saturates, this producer thread enters WAITING.
// All producer threads can end up here, exhausting the producer pool.
Response response = future.get();
```

Configuration smell to look for:

```yaml
server:
  maxThreads: 1024              # producer pool
processorCallback:
  threadPoolSize: 50            # consumer pool — 20:1 ratio
  # no per-call timeout, or timeout >> producer pool drain time
```

**Fix pattern:**

Two acceptable remediations, in priority order:

1. **Asynchronous, non-blocking handoff** (preferred): use `CompletableFuture.thenAccept` / reactive streams / `AsyncContext` so the producer thread is released immediately. The consumer pool can still saturate, but it no longer drags the producer pool down with it.

```java
CompletableFuture
    .supplyAsync(() -> httpClient.post(externalEndpoint, payload), callbackExecutor)
    .orTimeout(5, TimeUnit.SECONDS)
    .thenAccept(response -> asyncContext.complete(buildResponse(response)))
    .exceptionally(ex -> { asyncContext.complete(errorResponse(ex)); return null; });
```

2. **If a synchronous call must remain, align the pool sizes and add a hard timeout:**

```java
// Producer pool 1024, consumer pool 1024 — no big-pipe/small-pipe gap.
Response response = future.get(5, TimeUnit.SECONDS);
```

**Scope filter:**

```json
{
  "languages": ["java"],
  "frameworks": ["dropwizard", "jetty", "spring-boot"],
  "service_types": ["api", "worker"]
}
```

---

### Learning 2 — PROCESS_AWARENESS: Force-redeploy playbook for thread-pool exhaustion

**Type:** `PROCESS_AWARENESS`

**Description:**

When all hosts are simultaneously unhealthy due to thread starvation, a normal deploy will not recover the service — the deployer reuses unhealthy hosts because the software version hasn't changed. Recovery requires a *force-redeploy* with the explicit "boot fresh hosts" flag set, and a 0-minute canary so all replacement hosts enter the load balancer immediately.

During this incident we lost roughly 25 minutes attempting normal redeploys before realizing we needed the force-redeploy path. Every team running a containerized JVM service should have this in their runbook before they need it. The exact recovery procedure:

1. Trigger a manual deploy of the current version with the `force_redeploy=true` flag (or equivalent in your deploy tooling)
2. Set the canary to 0 minutes so all hosts enter the LB immediately
3. Do not run a force-redeploy across multiple realms simultaneously — pin to the affected realm only
4. Have a parallel SSM/SSH playbook to `docker restart` containers as a faster fallback (~30s vs ~10min for full host replacement)

Teams should review their own runbooks for this scenario and confirm:
- Force-redeploy procedure is documented and the flag location is named
- Container-restart fallback exists for the case where deploys are also blocked
- Auto-host-replacement rate limits are understood (e.g., Lazarus rate-limits to 3 events per role per 10 min — this *will* slow recovery)

**Audience filter:**

```json
{
  "languages": ["java"],
  "service_types": ["api", "worker"]
}
```

---

### Learning 3 — MONITORING_GAP: Thread pool utilization alerting

**Type:** `MONITORING_GAP`

**Description:**

We had no alert for thread pool utilization. The pool went from a typical 2% utilization to 100% in 24 minutes, with 4 minutes between the 90% threshold and complete exhaustion. A monitor on producer-pool utilization at 20% (10x normal) would have given us a ~20-minute head start and likely prevented the customer-facing impact entirely.

A second blind spot: PagerDuty's "intelligent alert grouping" rolled up 222 individual host-unhealthy and e2e-test-failure alerts into a single page. The on-call engineer received one notification regardless of whether 1 host or 100 hosts were unhealthy. This masked the scale of the incident and contributed to the slow diagnosis.

**Suggested actions for affected services:**

1. **Add a thread-pool utilization monitor** that alerts at 10x your steady-state utilization. For most JVM services that means: page at 20% if your normal is 2%. Include both the main pool and any backup/queue pool in the metric.
2. **Audit PagerDuty intelligent-grouping settings** for the services you own. For health-check and e2e-tester signals, prefer un-grouped pages so the on-call engineer sees the *count* of failures, not just one consolidated alert.
3. **Add a dedicated dashboard panel** for producer/consumer thread-pool utilization on every service that has this two-pool pattern. This is the leading indicator we missed.

**Audience filter:**

```json
{
  "service_types": ["api", "worker"]
}
```

---

## Original incident timeline

- **19:26 UTC** — Producer pool utilization crosses 10% (typical: 2%)
- **19:50 UTC** — Producer pool utilization crosses 90%
- **19:51 UTC** — First 5xx responses; upstream services begin failing
- **19:54 UTC** — Producer pool fully exhausted; nginx healthchecks start failing
- **20:00 UTC** — SEV2 declared
- **20:50 UTC** — Escalated to SEV0
- **21:35 UTC** — 65% traffic recovery (partial fleet replacement)
- **22:00 UTC** — 100% recovery after full force-redeploy with fresh hosts

## References

- Internal: `docs/runbooks/thread-pool-exhaustion-recovery.md` (created post-incident)
- External: ["The Big Pipe / Small Pipe Pattern" — Marc Brooker](https://brooker.co.za)
- Internal: `dashboards/jvm-thread-pool-overview` (created post-incident)
