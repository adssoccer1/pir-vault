# PIR-2024-01-22: Redis Pipeline Batch Size Cap

- **Incident ID:** INC-2024-01-22-003
- **Severity:** SEV2
- **Date:** 2024-01-22
- **Duration:** 47m
- **Owner:** Caching Platform Team
- **Status:** Resolved

## Summary

A new code path in the user-preferences service began issuing Redis `MGET` calls with batch sizes up to 50,000 keys. Redis processed each request serially on a single thread, blocking all other clients on the shared cluster for ~3-4 seconds per call. Other services using the same Redis cluster saw timeout cascades and elevated p99 latency. Mitigation: capped client-side batch size at 500.

## Learnings

### Learning 1 — CODE_FIX: Unbounded Redis MGET / pipeline batch sizes

**Type:** `CODE_FIX`

**Anti-pattern:**

```java
// Hot path that grew an N+1 reduction into an unbounded batch.
List values = redisClient.mget(keys.toArray(new String[0]));
```

**Fix pattern:**

```java
// Cap batch size, chunk if necessary.
List values = new ArrayList<>(keys.size());
for (List chunk : Lists.partition(keys, 500)) {
    values.addAll(redisClient.mget(chunk.toArray(new String[0])));
}
```

**Scope filter:**

```json
{
  "languages": ["java", "python"],
  "frameworks": ["lettuce", "jedis", "redis-py"],
  "service_types": ["api", "worker"]
}
```

---

### Learning 2 — MONITORING_GAP: Redis slowlog alerting

**Type:** `MONITORING_GAP`

**Description:**

No alert existed for Redis slowlog entries. The 3-4 second blocking commands appeared in slowlog but were not surfaced to any team until customers reported timeouts. Teams owning Redis-dependent services should configure a slowlog alert at >100ms.

**Audience filter:**

```json
{
  "service_types": ["api", "worker"]
}
```
