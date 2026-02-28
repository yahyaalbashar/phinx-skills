# Clean Code & Architecture — Advanced Patterns Reference

## Hexagonal Architecture (Ports & Adapters)

```
                    ┌─────────────────────┐
   HTTP Request ──► │  HTTP Adapter       │ ──┐
                    └─────────────────────┘   │
                    ┌─────────────────────┐   │    ┌──────────────────┐
   CLI Command ──►  │  CLI Adapter        │ ──┼──► │  USE CASE (Port) │
                    └─────────────────────┘   │    │                  │
                    ┌─────────────────────┐   │    │  domain logic    │
   Queue Message ─► │  Queue Adapter      │ ──┘    │  pure functions  │
                    └─────────────────────┘        │  no I/O          │
                                                   └────────┬─────────┘
                                                            │
                    ┌─────────────────────┐                 │
                    │  SQL DB Adapter     │ ◄───────────────┤
                    └─────────────────────┘                 │
                    ┌─────────────────────┐                 │
                    │  Cache Adapter      │ ◄───────────────┤
                    └─────────────────────┘                 │
                    ┌─────────────────────┐                 │
                    │  Payment Adapter    │ ◄───────────────┘
                    └─────────────────────┘

Ports  = Interfaces defined by the domain (what it needs)
Adapters = Implementations that connect to the real world (how it's fulfilled)
```

### Key Rules
1. Domain defines ports (interfaces). It NEVER imports adapters.
2. Adapters implement ports. They translate between domain and external systems.
3. Application layer orchestrates: calls ports, coordinates use cases.
4. Infrastructure layer provides adapters. Wired at startup (composition root / entry point).

## Repository Pattern (DIP in Practice)

The most common application of Dependency Inversion in backend systems.

```
// 1. Domain defines the port (interface)
interface UserRepository
  method findById(id: UserId) -> User or null
  method save(user: User) -> void
  method findByEmail(email: Email) -> User or null

// 2. Infrastructure implements it for production
class SqlUserRepository implements UserRepository
  constructor(dbConnection)

  method findById(id: UserId) -> User or null
    row = dbConnection.query("SELECT * FROM users WHERE id = ?", id.value)
    if row == null return null
    return mapRowToUser(row)

  method save(user: User) -> void
    dbConnection.execute("INSERT INTO users ...", user.toRow())

// 3. A separate implementation for testing
class InMemoryUserRepository implements UserRepository
  store = empty map

  method findById(id: UserId) -> User or null
    return store.get(id) or null

  method save(user: User) -> void
    store.put(user.id, user)

// 4. Use case depends on the interface, not the implementation
class CreateUserUseCase
  constructor(repo: UserRepository, hasher: PasswordHasher)

  method execute(command: CreateUserCommand) -> User
    existing = repo.findByEmail(command.email)
    if existing != null
      throw Error("Email already registered")
    user = new User(
      id: generateId(),
      email: command.email,
      passwordHash: hasher.hash(command.password)
    )
    repo.save(user)
    return user
```

## Decorator Chaining

Stack cross-cutting concerns without modifying core logic:

```
// Base implementation
sqlRepo = new SqlUserRepository(db)

// Wrap with caching
cachedRepo = new CachingUserRepository(inner: sqlRepo, cache: redisCache)

// Wrap with logging
loggedRepo = new LoggingUserRepository(inner: cachedRepo, logger: logger)

// Wrap with metrics
finalRepo = new MetricsUserRepository(inner: loggedRepo, metrics: prometheus)

// Inject the decorated chain
service = new UserService(repo: finalRepo)

// Call flow: Metrics → Logging → Caching → SQL
// Each decorator implements the same Repository interface
// Each delegates to inner after adding its concern
```

## Value Objects

Wrap primitives in domain types to prevent misuse and enforce rules at creation:

```
class Email
  constructor(value: string)
    if not contains(value, "@") or not contains(afterLast(value, "@"), ".")
      throw ValidationError("Invalid email: " + value)
    this.value = lowercase(trim(value))

  method equals(other: Email) -> boolean
    return this.value == other.value

class Money
  constructor(amount: decimal, currency: string)
    if amount < 0
      throw ValidationError("Money cannot be negative")
    this.amount = amount
    this.currency = currency

  method add(other: Money) -> Money
    if this.currency != other.currency
      throw Error("Cannot add " + this.currency + " and " + other.currency)
    return new Money(this.amount + other.amount, this.currency)

  method multiply(factor: decimal) -> Money
    return new Money(this.amount * factor, this.currency)
```

Benefits: impossible to mix up `userId: string` with `email: string`, validation enforced at creation, immutable, equality by value not reference.

## Domain Events

Decouple side effects from core domain logic:

```
class Order
  events = empty list

  method place(items: list of OrderItem)
    // Core business logic only
    this.status = PLACED
    this.items = items
    this.total = sum(item.price for item in items)
    // Record event — don't execute side effects here
    events.add(new OrderPlacedEvent(orderId: this.id, total: this.total))

  method collectEvents() -> list of DomainEvent
    collected = copy(events)
    events.clear()
    return collected

// Application layer handles side effects
class PlaceOrderUseCase
  constructor(repo: OrderRepository, eventBus: EventBus)

  method execute(command: PlaceOrderCommand)
    order = new Order()
    order.place(command.items)
    repo.save(order)
    for event in order.collectEvents()
      eventBus.publish(event)
      // Separate handlers: send email, update analytics, notify warehouse
```

Why this matters: The Order entity stays pure — no email, no analytics, no queue dependencies. Side effects are handled by event handlers that can be added, removed, or modified independently.

## Specification Pattern

Encapsulate business rules as composable, reusable objects:

```
interface Specification
  method isSatisfiedBy(candidate) -> boolean

class ActiveUserSpec implements Specification
  method isSatisfiedBy(user) -> user.status == ACTIVE and user.deletedAt == null

class PremiumUserSpec implements Specification
  method isSatisfiedBy(user) -> user.plan.tier == PREMIUM

class EligibleForPromotionSpec implements Specification
  constructor(activeSpec: Specification, premiumSpec: Specification)
  method isSatisfiedBy(user) -> activeSpec.isSatisfiedBy(user) and premiumSpec.isSatisfiedBy(user)

// Usage
eligibleSpec = new EligibleForPromotionSpec(new ActiveUserSpec(), new PremiumUserSpec())
eligibleUsers = allUsers.filter(user -> eligibleSpec.isSatisfiedBy(user))
```

Use when: Complex filtering logic reused across queries, validations, and UI. Avoids duplicating business rules.

## Unit of Work Pattern

Coordinate multiple repository operations as a single transaction:

```
interface UnitOfWork
  method begin()
  method commit()
  method rollback()
  method getUserRepository() -> UserRepository
  method getOrderRepository() -> OrderRepository

class TransferFundsUseCase
  constructor(unitOfWork: UnitOfWork)

  method execute(fromId, toId, amount)
    unitOfWork.begin()
    try
      users = unitOfWork.getUserRepository()
      sender = users.findById(fromId)
      receiver = users.findById(toId)
      sender.debit(amount)
      receiver.credit(amount)
      users.save(sender)
      users.save(receiver)
      unitOfWork.commit()
    catch error
      unitOfWork.rollback()
      throw error
```

## Null Object Pattern

Eliminate null checks by providing a "do nothing" implementation:

```
interface Logger
  method info(message)
  method error(message)

class ConsoleLogger implements Logger
  method info(message)  -> print("[INFO] " + message)
  method error(message) -> print("[ERROR] " + message)

class NullLogger implements Logger
  method info(message)  -> // do nothing
  method error(message) -> // do nothing

// No more "if logger != null then logger.info(...)"
// Inject NullLogger when logging is unwanted
service = new PaymentService(logger: new NullLogger())
```

## Anti-Patterns to Recognize and Fix

| Anti-Pattern | Symptom | Fix |
|---|---|---|
| **God Object** | One class with 20+ methods spanning multiple concerns | Split by SRP into focused classes |
| **Anemic Domain Model** | Entities are data bags with only getters/setters, all logic in services | Move behavior into entities |
| **Service Locator** | `Container.resolve(UserRepo)` called throughout codebase | Use constructor injection (DIP) |
| **Primitive Obsession** | `userId: string`, `price: float` passed everywhere | Introduce Value Objects |
| **Feature Envy** | Method uses more data from another class than its own | Move the method to the class it envies |
| **Shotgun Surgery** | One change requires edits in 10+ files | Group related code into cohesive modules |
| **Speculative Generality** | Abstract base classes with only one implementation | Delete the abstraction; extract when a second use appears |
| **Leaky Abstraction** | Database errors or HTTP details propagating to domain logic | Catch and translate errors at adapter boundaries |
| **Circular Dependency** | Module A imports Module B which imports Module A | Extract shared interface into a third module |
| **Big Ball of Mud** | No discernible architecture; everything depends on everything | Identify bounded contexts; introduce layer boundaries incrementally |