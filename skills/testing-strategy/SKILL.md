---
name: testing-strategy
description: "Senior-level testing philosophy, patterns, and strategy. Use this skill whenever writing tests, designing test architecture, discussing test coverage, debugging flaky tests, setting up testing infrastructure, or choosing testing tools. Also trigger when the user mentions unit tests, integration tests, E2E tests, TDD, mocking, fixtures, test coverage, Pytest, Jest, Vitest, Playwright, Cypress, React Testing Library, or any testing framework or pattern."
---

# Testing Strategy — Senior Engineer Standards

You are a senior engineer who treats tests as first-class code. Tests aren't overhead — they're the specification of correct behavior and the safety net for change.

## Testing Philosophy

1. **Test behavior, not implementation.** Tests should pass when the behavior is correct and fail when it breaks — regardless of how the code is structured internally. Refactoring should not break tests.

2. **The testing trophy (not pyramid).**
   - Base: Static analysis (TypeScript, linters) — catches most bugs
   - Middle (largest): Integration tests — test real workflows through real code
   - Upper: Unit tests — for complex logic, algorithms, utilities
   - Top (smallest): E2E — critical user paths only

3. **Write tests that give confidence, not coverage numbers.** 80% coverage of the right code beats 100% coverage that tests getters and setters.

4. **Tests are documentation.** A well-written test suite is the most reliable documentation of how the system behaves.

## Unit Testing Rules

### What to unit test
- Pure functions with complex logic
- Algorithms and data transformations
- Edge cases and boundary conditions
- Error handling paths
- Utility functions used widely

### What NOT to unit test
- Simple getters/setters
- Framework boilerplate
- Code that just delegates to another layer
- Private methods (test through public interface)

### Structure: Arrange-Act-Assert
```
test("calculates discount for orders over $100", () => {
  // Arrange
  const order = createOrder({ subtotal: 150 });

  // Act
  const discount = calculateDiscount(order);

  // Assert
  expect(discount).toBe(15); // 10% discount
});
```

### Naming convention
- Describe WHAT behavior you're testing and WHEN it applies
- `"returns empty array when no items match filter"`
- `"throws ValidationError when email format is invalid"`
- NOT: `"test1"`, `"it works"`, `"should do the thing"`

## Integration Testing Rules

### What to integration test
- API endpoints: full request → response including middleware
- Database queries: with real database (use test containers or in-memory)
- Service interactions: with real dependencies where feasible
- Complete user workflows across multiple components

### Test database strategy
- Use Docker containers for isolated test databases
- Run migrations before test suite, roll back between tests (transactions)
- Seed with minimal, realistic fixture data
- Never share test databases across parallel test runs

### Mocking strategy
- Mock at the boundary: HTTP calls to external services, file system, time
- Use MSW (Mock Service Worker) for frontend HTTP mocking
- Prefer fakes (in-memory implementations) over mocks for repositories
- Never mock the thing you're testing

## E2E Testing Rules

### What to E2E test
- Critical happy paths: sign up, log in, core CRUD, checkout, payment
- Flows that cross system boundaries
- Accessibility flows (keyboard navigation, screen reader)

### What NOT to E2E test
- Every possible form validation error
- Edge cases better covered by unit tests
- Styling or layout (use visual regression tools)

### E2E best practices
- Playwright over Cypress for modern apps (better parallelization, multi-tab support)
- Test against staging, not production
- Use realistic but deterministic test data (seedable)
- Isolate tests: each test creates its own data, cleans up after
- Retry flaky tests once in CI, but fix the root cause promptly

## Testing Anti-Patterns

- **Flaky tests**: Tests that sometimes pass and sometimes fail. Fix immediately — they erode trust in the entire suite.
- **Test interdependence**: Test B fails if Test A doesn't run first. Every test must be independent and idempotent.
- **Excessive mocking**: If you mock everything, you're testing the mocking framework, not your code.
- **Snapshot abuse**: Snapshot tests for large objects catch everything and nothing. Use for small, stable outputs only.
- **Slow test suite**: If CI takes > 10 minutes, developers stop running tests. Optimize or parallelize.
- **Testing private methods**: Refactoring changes privates but not behavior. Test through public API.

## Test Data Management

- **Factories/builders**: Generate test data with sensible defaults, override only what matters for the test
- **Fixtures**: Static test data for complex scenarios. Keep minimal.
- **Faker**: For generating realistic test data. Pin seeds for reproducibility.
- **Database**: Each test gets a transaction that rolls back. Fast and isolated.

## CI Integration

- Tests run on every PR. No exceptions.
- Fail fast: lint → typecheck → unit → integration → E2E
- Parallel test execution where possible
- Test results as PR comment (coverage diff, failures)
- Coverage thresholds: enforce minimum (e.g., 70%) but don't chase 100%
- Flaky test quarantine: move flaky tests to a separate job, fix within 48 hours