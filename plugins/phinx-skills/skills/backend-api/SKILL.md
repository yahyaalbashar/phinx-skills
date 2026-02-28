---
name: backend-api
description: "Senior-level backend API engineering covering REST design, authentication, authorization, database access patterns, error handling, logging, and service architecture. Use this skill whenever building, reviewing, or debugging API endpoints, middleware, services, repositories, database queries, auth flows, or any server-side code. Also trigger for discussions about API design, microservices vs monolith, request validation, rate limiting, caching strategy, or when the user mentions Go, Django, FastAPI, Express, Fiber, GORM, SQLAlchemy, Prisma, or any backend framework."
---

# Backend API — Senior Engineer Standards

You are a senior backend engineer. Every API you build should be secure, consistent, well-documented, and resilient under load.

## API Design Principles

1. **RESTful by default.** Resources as nouns (`/users`, `/projects/{id}/tasks`), HTTP verbs for actions. Use `POST` for creation, `PUT` for full replacement, `PATCH` for partial update. Reserve custom actions for genuinely non-CRUD operations: `POST /orders/{id}/cancel`.

2. **Consistent response envelope.**
```json
{
  "data": { ... },
  "meta": { "page": 1, "total": 42, "per_page": 20 },
  "error": null
}
```
Error responses:
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description",
    "details": [{ "field": "email", "issue": "Invalid format" }]
  }
}
```

3. **Versioning.** URL path versioning (`/api/v1/`) for public APIs. Header versioning for internal services if needed. Never break existing contracts without a version bump.

4. **Pagination.** Cursor-based for infinite/real-time feeds. Offset-based for admin tables. Always include `total`, `page`, `per_page`, `next_cursor` in meta.

## Request Validation

- Validate at the handler/controller layer BEFORE touching business logic.
- Use schema validation libraries (Zod, Pydantic, Go struct tags + validator).
- Return 400 with field-level errors. Never let invalid data reach the database.
- Sanitize all string inputs. Trim whitespace. Normalize emails to lowercase.
- Enforce size limits on request bodies, file uploads, and array fields.

## Authentication & Authorization

### Authentication (Who are you?)
- JWT with short-lived access tokens (15min) and long-lived refresh tokens (7-30 days).
- Store refresh tokens server-side (DB or Redis). Rotate on use.
- Hash passwords with bcrypt (cost 12+) or argon2id. NEVER SHA256/MD5.
- Support API keys for service-to-service auth. Hash stored keys.

### Authorization (What can you do?)
- RBAC as the foundation: roles → permissions → resource access.
- Enforce at middleware level for route-wide rules, at handler level for resource-specific rules.
- Always check resource ownership: `WHERE user_id = ?` in queries, not just role checks.
- Principle of least privilege: default deny, explicitly grant.

## Database Access Patterns

### Repository Pattern
Separate data access from business logic:
```
Handler → Service (business logic) → Repository (data access) → Database
```
Services never write raw SQL. Repositories never contain business rules.

### Query Safety
- Parameterized queries ALWAYS. No string interpolation in SQL.
- Use transactions for multi-step mutations. Rollback on any failure.
- Set query timeouts. A missing timeout = a potential outage.
- Use `SELECT ... FOR UPDATE` carefully; prefer optimistic locking with version columns.

### Connection Management
- Connection pooling: min 5, max 25 per service instance (tune per load).
- Health checks on connections. Auto-reconnect on failure.
- Read replicas for read-heavy queries. Write to primary only.

## Error Handling

- Distinguish between client errors (4xx) and server errors (5xx) rigorously.
- 400: Bad request / validation failure
- 401: Not authenticated
- 403: Authenticated but not authorized
- 404: Resource not found
- 409: Conflict (duplicate, state violation)
- 422: Semantically invalid (valid format, wrong meaning)
- 429: Rate limited
- 500: Unexpected server error (NEVER expose internals)
- Wrap all panics/exceptions in the outermost middleware. Log the stack trace, return a generic 500.
- Use error codes (machine-readable) alongside messages (human-readable).
- Custom error types in your language (Go: sentinel errors + `errors.Is/As`, Python: exception hierarchy).

## Logging & Observability

### Structured Logging
- JSON format. Fields: `timestamp`, `level`, `message`, `request_id`, `user_id`, `service`, `duration_ms`.
- Log at boundaries: incoming requests, outgoing calls, database queries, errors.
- NEVER log passwords, tokens, PII, or full request bodies in production.

### Request Tracing
- Generate a unique `request_id` for every request. Propagate via headers (`X-Request-ID`).
- Include `request_id` in every log line and error response.
- For microservices: propagate trace context (OpenTelemetry).

### Metrics
- Request count, latency (p50/p95/p99), error rate per endpoint.
- Database query duration and pool utilization.
- Queue depth and processing time for async operations.

## Rate Limiting & Throttling

- Rate limit by API key or user ID, not just IP (IPs are shared/spoofed).
- Use token bucket or sliding window algorithms.
- Return `429` with `Retry-After` header.
- Different limits for different endpoints: auth endpoints strict, read endpoints lenient.
- Implement graceful degradation: serve cached responses under extreme load.

## Caching Strategy

- **Never cache authenticated, user-specific responses** at CDN level.
- Cache expensive computations and rarely-changing data.
- Use cache-aside pattern: check cache → miss → fetch from DB → populate cache.
- Set explicit TTLs. Prefer short TTLs + cache invalidation over long TTLs.
- Cache keys must include all query parameters that affect the response.
- Implement cache stampede protection (locking or probabilistic early expiry).

## Middleware Stack (in order)

1. Request ID generation
2. Structured logging (start)
3. CORS (if needed)
4. Rate limiting
5. Authentication
6. Authorization
7. Request validation
8. → Handler
9. Structured logging (end, with duration)
10. Error recovery / panic handler

## Background Jobs & Async Processing

- Use a proper job queue (Redis/BullMQ, RabbitMQ, SQS) not database polling.
- Idempotent job handlers: safe to retry on failure.
- Dead letter queue for failed jobs after N retries.
- Separate job processing from API serving (different processes/containers).

For framework-specific patterns and advanced architecture, see `references/patterns.md`.