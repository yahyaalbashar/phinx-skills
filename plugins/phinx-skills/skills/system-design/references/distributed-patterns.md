# Distributed Systems Patterns Reference

## Consensus & Coordination
- **Leader election**: Use existing tools (etcd, ZooKeeper, Redis Redlock) — never build your own.
- **Distributed locking**: Prefer optimistic concurrency (version numbers) over pessimistic locks. If you must lock, set short TTLs.
- **Idempotency keys**: Every mutation API should accept an idempotency key. Store results keyed by it. Return cached result on retry.

## Message Queue Patterns

### Topic vs Queue
- **Queue** (point-to-point): One consumer processes each message. Use for job processing.
- **Topic** (pub/sub): All subscribers receive each message. Use for event broadcasting.

### Delivery guarantees
- **At-most-once**: Fire and forget. Fast but lossy. Use for metrics, logs.
- **At-least-once**: Retry until acknowledged. May duplicate. Use for most business events + idempotent consumers.
- **Exactly-once**: Extremely expensive. Usually achieved via at-least-once + idempotent processing.

### Dead Letter Queue (DLQ)
Messages that fail N times go to DLQ. Monitor DLQ size. Build a replay mechanism. Never silently drop messages.

## Data Consistency Patterns

### Outbox Pattern
Write to business table AND outbox table in one database transaction. A separate process reads the outbox and publishes events. Guarantees events are published if and only if the business write succeeds.

### Change Data Capture (CDC)
Stream database changes (Debezium, DynamoDB Streams) as events. Useful for keeping read models, caches, and search indexes in sync without modifying application code.

### Two-Phase Commit (2PC)
Coordinator asks all participants to prepare, then commit. Avoid in distributed systems — it's slow, blocks on coordinator failure, and doesn't scale. Prefer sagas instead.

## Rate Limiting Algorithms

### Token Bucket
Tokens added at fixed rate. Each request consumes a token. Allows bursts up to bucket size. Most common choice.

### Sliding Window
Track request count in a sliding time window. More precise than fixed windows. Slightly more complex to implement.

### Leaky Bucket
Requests queue and process at fixed rate. Smooths traffic but adds latency. Good for downstream services with strict rate limits.

## Load Balancing Strategies

- **Round robin**: Simple, even distribution. Default choice.
- **Least connections**: Route to server with fewest active connections. Better for varied request durations.
- **Weighted**: Assign weights based on server capacity. Useful for heterogeneous fleets.
- **Consistent hashing**: Route based on request key. Enables caching and session affinity. Use for stateful services.

## Failure Handling

### Retry Strategy
```
Attempt 1: immediate
Attempt 2: 1s + jitter
Attempt 3: 2s + jitter
Attempt 4: 4s + jitter
Attempt 5: give up, DLQ
```
Always add jitter to prevent thundering herd. Always set a max retry count.

### Bulkhead Pattern
Isolate resources per dependency. If Service A's thread pool is exhausted by calls to Service B, it shouldn't affect calls to Service C. Separate connection pools, thread pools, or circuit breakers per dependency.

### Timeout Cascades
Set timeouts shorter as you go deeper:
- API Gateway: 30s
- Service A: 10s
- Service B (called by A): 3s
- Database query: 1s

If Service B is slow, Service A fails fast instead of holding connections.