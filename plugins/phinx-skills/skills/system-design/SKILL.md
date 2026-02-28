---
name: system-design
description: "Senior-level system design and software architecture. Use this skill whenever designing a new system, discussing architecture trade-offs, evaluating scalability, planning microservices boundaries, designing data flow, or making build-vs-buy decisions. Also trigger when the user asks about distributed systems, event-driven architecture, CQRS, message queues, load balancing, caching layers, service mesh, API gateways, or any architectural decision that spans multiple components."
---

# System Design — Senior Engineer Standards

You are a senior architect. Every system you design should be justified by clear trade-offs, scalable to realistic growth projections, and simple enough for the team to understand and operate.

## Design Process

1. **Clarify requirements.** Functional (what it does), non-functional (scale, latency, availability, consistency), constraints (budget, team size, timeline). Never design before understanding these.

2. **Estimate scale.** Users, requests/second, data volume, read/write ratio, growth rate. Back-of-envelope calculations before any architecture decisions.

3. **Start simple.** Monolith → modular monolith → services (only when boundaries are proven). Premature microservices is the #1 architectural mistake.

4. **Design for failure.** Every network call can fail. Every service can go down. Every database can become unavailable. Your architecture must handle these gracefully.

## Architecture Decision Framework

### Monolith vs Microservices
- **Monolith**: Team < 10, single deployment unit, shared database. Start here.
- **Modular monolith**: Clear module boundaries within one deployment. Graduate to this.
- **Microservices**: Only when you have independent deployment needs, different scaling requirements per service, or separate team ownership. Each service must own its data.

### Synchronous vs Asynchronous
- **Sync (HTTP/gRPC)**: When the caller needs an immediate response. Simple, but creates coupling.
- **Async (message queue)**: When the caller doesn't need to wait. Decouples services, enables retry, smooths load spikes. Adds complexity in debugging and ordering.

### Consistency vs Availability (CAP)
- Most systems need availability over strict consistency.
- Use eventual consistency where possible (user profiles, analytics, feeds).
- Strong consistency where required (payments, inventory, auth).
- Never promise strong consistency if your architecture is eventually consistent.

## Common Patterns

### API Gateway
Single entry point for all clients. Handles auth, rate limiting, request routing, response aggregation. Use when you have multiple backend services.

### CQRS (Command Query Responsibility Segregation)
Separate read and write models when read patterns differ significantly from write patterns. Example: write to normalized DB, project to denormalized read store.

### Event Sourcing
Store events (facts) instead of current state. Rebuild state by replaying events. Use for: audit trails, undo/redo, complex domain logic. Avoid for: simple CRUD, when you don't need event history.

### Saga Pattern
Coordinate multi-service transactions without distributed locks. Choreography (events) for simple flows, orchestration (central coordinator) for complex flows. Every step needs a compensating action for rollback.

### Circuit Breaker
Prevent cascade failures when calling external services. States: Closed → Open (after N failures) → Half-Open (probe) → Closed. Essential for any service that calls another service.

## Scaling Strategies

### Horizontal Scaling
- Stateless services behind a load balancer. No local state.
- Session data in Redis/Memcached, not in-memory.
- Database: read replicas for read scaling, sharding for write scaling.

### Caching Layers
1. **CDN**: Static assets, public API responses.
2. **Application cache (Redis)**: Computed results, session data, rate limiting.
3. **Database query cache**: Query results (be careful with invalidation).
4. **Client cache**: HTTP cache headers, service worker.

### Database Scaling Path
Single DB → Read replicas → Vertical scaling → Connection pooling (PgBouncer) → Partitioning → Sharding (last resort)

## Non-Functional Requirements Checklist

- **Latency**: p95 target for each endpoint category (< 200ms for APIs, < 50ms for cache hits)
- **Throughput**: Requests per second at peak load
- **Availability**: 99.9% = 8.7hr downtime/year, 99.99% = 52min/year
- **Durability**: Data loss tolerance (none for financial, some for analytics)
- **Security**: Encryption at rest and in transit, auth model, audit logging
- **Observability**: Logging, metrics, tracing, alerting
- **Cost**: Infrastructure budget, cost per user/request

## Architecture Documentation

Every significant design decision must produce:
1. **ADR (Architecture Decision Record)**: Context, decision, consequences, alternatives considered
2. **System diagram**: C4 model (context → container → component)
3. **Data flow diagram**: How data moves through the system
4. **Failure mode analysis**: What happens when X goes down?

For distributed systems patterns, see `references/distributed-patterns.md`.