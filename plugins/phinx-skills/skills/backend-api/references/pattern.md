# Backend API Advanced Patterns

## Go/Fiber Patterns

### Clean handler structure
```go
func (h *Handler) CreateProject(c *fiber.Ctx) error {
    var req CreateProjectRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(ErrorResponse("PARSE_ERROR", "Invalid request body"))
    }
    if err := h.validator.Struct(req); err != nil {
        return c.Status(400).JSON(ValidationErrorResponse(err))
    }

    userID := c.Locals("userID").(uuid.UUID)
    project, err := h.service.CreateProject(c.Context(), userID, req)
    if err != nil {
        return h.handleServiceError(c, err)
    }

    return c.Status(201).JSON(SuccessResponse(project))
}
```

### Middleware chain
```go
app.Use(middleware.RequestID())
app.Use(middleware.Logger())
app.Use(middleware.Recover())
app.Use(middleware.RateLimiter(100, time.Minute))

api := app.Group("/api/v1")
api.Use(middleware.Auth(jwtSecret))

projects := api.Group("/projects")
projects.Get("/", h.ListProjects)
projects.Post("/", middleware.RequireRole("admin", "manager"), h.CreateProject)
```

## Django/DRF Patterns

### Viewset with permissions
```python
class ProjectViewSet(ModelViewSet):
    serializer_class = ProjectSerializer
    permission_classes = [IsAuthenticated, IsProjectMember]
    filter_backends = [DjangoFilterBackend, OrderingFilter]
    filterset_fields = ["status", "owner"]
    ordering_fields = ["created_at", "name"]

    def get_queryset(self):
        return Project.objects.filter(
            team__members=self.request.user
        ).select_related("owner").prefetch_related("tasks")

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Custom exception handler
```python
def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if response is not None:
        response.data = {
            "data": None,
            "error": {
                "code": get_error_code(exc),
                "message": str(exc.detail) if hasattr(exc, "detail") else str(exc),
                "details": response.data if isinstance(response.data, list) else None,
            }
        }
    return response
```

## Database Migration Best Practices

1. **Always reversible.** Every migration has both `up` and `down`.
2. **One concern per migration.** Don't mix schema changes with data migrations.
3. **Non-blocking for large tables:**
   - Add columns as nullable first, backfill, then add NOT NULL constraint
   - Create indexes concurrently (`CREATE INDEX CONCURRENTLY`)
   - Rename via new column + trigger + backfill + swap (for zero-downtime)
4. **Test migrations against production-like data volumes.**

## API Versioning Strategy

### Breaking changes (require version bump)
- Removing or renaming a field
- Changing a field's type
- Changing validation rules to be more restrictive
- Changing authentication requirements

### Non-breaking changes (safe within version)
- Adding optional fields to responses
- Adding optional query parameters
- Adding new endpoints
- Relaxing validation rules

## Circuit Breaker Pattern (for external calls)

```
States: CLOSED → OPEN → HALF_OPEN → CLOSED

CLOSED: Normal operation. Track failure rate.
  → If failure rate > threshold (e.g., 50% in 30s) → OPEN

OPEN: All requests fail immediately with fallback response.
  → After timeout (e.g., 30s) → HALF_OPEN

HALF_OPEN: Allow one probe request.
  → If success → CLOSED
  → If failure → OPEN (reset timeout)
```

## Webhook Design

- Sign payloads with HMAC-SHA256. Include signature in header.
- Retry with exponential backoff (1s, 2s, 4s, 8s... up to 1hr).
- Include `event_type`, `event_id` (for idempotency), `timestamp`, `data`.
- Implement a webhook event log for debugging and replay.
- Allow recipients to verify webhook source via signature validation.