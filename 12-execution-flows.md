# Execution Flow Diagrams

> Visual representations of complete request flows through hexagonal architecture layers

This section provides comprehensive sequence diagrams showing how different components interact during typical operations in a hexagonal architecture with NestJS.

---

## 10.1 HTTP Request Complete Flow

This diagram shows the complete journey of an HTTP request through all hexagonal layers with NestJS tools:

```mermaid
sequenceDiagram
    participant Client
    participant Guard as @UseGuards()
    participant Controller
    participant DTO as class-validator
    participant UseCase
    participant Entity
    participant Repository
    participant Prisma

    Client->>Guard: POST /users
    Guard->>Guard: Validate JWT token
    Guard-->>Controller: Token valid

    Controller->>DTO: new CreateUserDto(body)
    DTO->>DTO: @IsEmail() validation
    DTO-->>Controller: DTO validated

    Controller->>UseCase: execute(CreateUserInput)
    UseCase->>UseCase: Constructor validates input

    UseCase->>Entity: User.create(email, name)
    Entity->>Entity: Validate business rules
    Entity-->>UseCase: User entity

    UseCase->>Repository: save(user)
    Repository->>Repository: toDatabase(entity)
    Repository->>Prisma: prisma.user.create(data)
    Prisma-->>Repository: PrismaUser
    Repository->>Repository: toDomain(prismaUser)
    Repository-->>UseCase: User entity

    UseCase-->>Controller: CreateUserOutput (type)
    Controller->>Controller: new CreateUserResponse(output)
    Controller-->>Client: HttpResponse<UserData>
```

**Key NestJS tools in action**:

- `@UseGuards()`: Authentication/authorization before controller
- `class-validator`: DTO validation (`@IsEmail()`, `@IsString()`)
- Dependency injection: `@Inject(USER_REPOSITORY)`
- Response transformation: `HttpResponse<T>` wrapper

---

## 10.2 Use Case with Domain Events Flow

This diagram shows how domain events are recorded during use case execution and published after successful persistence:

```mermaid
sequenceDiagram
    participant Controller
    participant UseCase
    participant Entity
    participant Repository
    participant EventBus
    participant Handler1 as SendWelcomeEmail
    participant Handler2 as CreateProfile

    Controller->>UseCase: execute(input)

    UseCase->>Entity: User.create(email, name)
    Entity->>Entity: Record UserCreated event
    Note over Entity: events: [UserCreated]
    Entity-->>UseCase: User entity

    UseCase->>Repository: save(user)
    Repository->>Repository: Extract events from entity
    Note over Repository: events = user.pullEvents()
    Repository->>Repository: Save entity to database
    Repository-->>UseCase: Saved user

    Repository->>EventBus: publishAll(events)

    par Parallel event handling
        EventBus->>Handler1: UserCreated event
        Handler1->>Handler1: Send welcome email
        Handler1-->>EventBus: Done
    and
        EventBus->>Handler2: UserCreated event
        Handler2->>Handler2: Create user profile
        Handler2-->>EventBus: Done
    end

    UseCase-->>Controller: Output
```

**Critical flow details**:

1. **Event Recording**: Entity records events internally during business operations
2. **Event Extraction**: Repository calls `entity.pullEvents()` before saving
3. **Transactional Safety**: Events published AFTER successful database save
4. **Parallel Handlers**: Multiple handlers process the same event independently

**NestJS implementation**:

```typescript
// Repository extracts and publishes events
async save(user: User): Promise<User> {
  const events = user.pullEvents(); // Extract before save
  const saved = await this.prisma.user.create(...);
  await this.eventBus.publishAll(events); // Publish after save
  return this.toDomain(saved);
}
```

---

## 10.3 Repository Entity Mapping Flow

This diagram shows the bidirectional transformation between Domain Entities and Prisma models:

```mermaid
sequenceDiagram
    participant UseCase
    participant IUserRepository as IUserRepository<br/>(Port)
    participant UserRepository as UserRepository<br/>(Adapter)
    participant Entity as User Entity<br/>(Domain)
    participant Prisma as Prisma Client
    participant DB as PostgreSQL

    Note over UseCase,DB: SAVE FLOW (Domain → Database)

    UseCase->>IUserRepository: save(user: User)
    IUserRepository->>UserRepository: save(user: User)

    UserRepository->>UserRepository: toDatabase(user)
    Note over UserRepository: Transform:<br/>User Entity → PrismaUserCreateInput
    Note over UserRepository: user.id.value → string<br/>user.email.value → string<br/>user.status → string

    UserRepository->>Prisma: create(data)
    Prisma->>DB: INSERT INTO users
    DB-->>Prisma: Row inserted
    Prisma-->>UserRepository: PrismaUser (raw data)

    UserRepository->>UserRepository: toDomain(prismaUser)
    Note over UserRepository: Transform:<br/>PrismaUser → User Entity
    Note over UserRepository: string → UserId VO<br/>string → Email VO<br/>string → UserStatus enum

    UserRepository->>Entity: User.reconstitute(...)
    Entity->>Entity: Validate invariants
    Entity-->>UserRepository: User entity

    UserRepository-->>IUserRepository: User entity
    IUserRepository-->>UseCase: User entity

    Note over UseCase,DB: FIND FLOW (Database → Domain)

    UseCase->>IUserRepository: findById(id: UserId)
    IUserRepository->>UserRepository: findById(id: UserId)

    UserRepository->>Prisma: findUnique({id: id.value})
    Prisma->>DB: SELECT * FROM users WHERE id = ?
    DB-->>Prisma: Row data
    Prisma-->>UserRepository: PrismaUser | null

    alt User found
        UserRepository->>UserRepository: toDomain(prismaUser)
        UserRepository->>Entity: User.reconstitute(...)
        Entity-->>UserRepository: User entity
        UserRepository-->>IUserRepository: User entity
    else User not found
        UserRepository-->>IUserRepository: null
    end

    IUserRepository-->>UseCase: User | null
```

**Transformation responsibilities**:

**`toDatabase(entity: User)`** - Domain → Persistence:

```typescript
private toDatabase(user: User): Prisma.UserCreateInput {
  return {
    id: user.id.value,              // UserId VO → string
    email: user.email.value,        // Email VO → string
    name: user.name,                // string → string
    status: user.status,            // UserStatus enum → string
    createdAt: user.createdAt,      // Date → Date
  };
}
```

**`toDomain(prismaUser: PrismaUser)`** - Persistence → Domain:

```typescript
private toDomain(data: PrismaUser): User {
  return User.reconstitute({
    id: new UserId(data.id),             // string → UserId VO
    email: new Email(data.email),        // string → Email VO
    name: data.name,                     // string → string
    status: data.status as UserStatus,   // string → enum
    createdAt: data.createdAt,           // Date → Date
  });
}
```

**Key principles**:

1. **Port (Interface)**: Works only with domain types (`User`, `UserId`)
2. **Adapter (Repository)**: Handles all transformations and Prisma interaction
3. **Entity Reconstitution**: `User.reconstitute()` bypasses constructor validation for existing data
4. **Value Object Unwrapping**: Always use `.value` to extract primitive from VO
5. **Null Safety**: Handle `null` results from Prisma appropriately

---

## 10.4 Outbox Pattern Flow

This diagram shows the complete flow of the Outbox pattern for reliable event publishing:

```mermaid
sequenceDiagram
    participant UC as Use Case
    participant AGG as Aggregate
    participant TX as Transaction
    participant User as users Table
    participant Outbox as outbox Table
    participant Worker as Publisher Worker
    participant Kafka as Event Bus
    participant Sub as Event Subscriber

    Note over UC,Sub: WRITE PATH (Transactional)

    UC->>AGG: Load/Create aggregate
    UC->>AGG: Execute business logic
    AGG->>AGG: record(DomainEvent)
    Note over AGG: events: [UserCreated]

    UC->>TX: BEGIN TRANSACTION
    UC->>AGG: pullEvents()
    AGG-->>UC: [DomainEvent]

    UC->>User: INSERT user data
    UC->>Outbox: INSERT events
    Note over Outbox: {<br/>  eventName: "user.domain.created",<br/>  payload: {...},<br/>  published: false<br/>}

    UC->>TX: COMMIT
    Note over TX,Outbox: ✅ Atomic: State + Events

    Note over UC,Sub: PUBLISH PATH (Asynchronous)

    loop Every 100ms
        Worker->>Outbox: SELECT WHERE published=false
        Outbox-->>Worker: [unpublished events]

        alt Event publish succeeds
            Worker->>Kafka: publish(event)
            Kafka-->>Worker: ACK
            Worker->>Outbox: UPDATE published=true
            Kafka->>Sub: Deliver event
            Sub->>Sub: Handle event
        else Event publish fails
            Worker->>Outbox: UPDATE attempts+1,<br/>nextAttemptAt (backoff)
            Note over Worker: Retry with<br/>exponential backoff
        end
    end
```

**Key guarantees:**
1. **Atomicity**: User state and events committed together (same TX)
2. **Durability**: Events persisted before publishing
3. **Reliability**: Worker retries failed publishes with backoff
4. **Ordering**: Events published in occurrence order (ORDER BY occurredAt)

---

## 10.5 Saga Orchestration Flow (with Compensation)

This diagram shows a Saga handling a multi-bounded-context transaction with failure compensation:

```mermaid
sequenceDiagram
    participant Client
    participant OrderBC as Order BC
    participant Saga as CreateOrderSaga
    participant PaymentBC as Payment BC
    participant InventoryBC as Inventory BC
    participant EmailBC as Email BC

    Note over Client,EmailBC: HAPPY PATH

    Client->>OrderBC: POST /orders
    OrderBC->>OrderBC: Create Order (pending)
    OrderBC->>Saga: Start saga

    Saga->>PaymentBC: ChargePayment
    PaymentBC-->>Saga: ✅ paymentId: "pay_123"
    Note over Saga: Push RefundPayment<br/>to compensation stack

    Saga->>InventoryBC: ReserveInventory
    InventoryBC-->>Saga: ✅ reservationId: "res_456"
    Note over Saga: Push ReleaseInventory<br/>to compensation stack

    Saga->>EmailBC: SendConfirmation
    EmailBC-->>Saga: ✅ Email sent

    Saga->>OrderBC: Mark order as completed
    OrderBC-->>Client: 200 OK

    Note over Client,EmailBC: FAILURE PATH (Compensation)

    Client->>OrderBC: POST /orders (different order)
    OrderBC->>OrderBC: Create Order (pending)
    OrderBC->>Saga: Start saga

    Saga->>PaymentBC: ChargePayment
    PaymentBC-->>Saga: ✅ paymentId: "pay_789"
    Note over Saga: Push RefundPayment<br/>to compensation stack

    Saga->>InventoryBC: ReserveInventory
    InventoryBC-->>Saga: ❌ Out of stock
    Note over Saga: Saga failed,<br/>start compensation

    Saga->>Saga: Pop compensation stack
    Saga->>PaymentBC: RefundPayment("pay_789")
    PaymentBC-->>Saga: ✅ Refunded

    Saga->>OrderBC: Mark order as failed
    OrderBC-->>Client: 409 Conflict (Out of stock)
```

**Key principles:**
1. **Compensation Stack**: Each successful step pushes compensation to stack (LIFO)
2. **Forward Recovery**: Try to complete; if fail, compensate in reverse order
3. **Idempotency**: Compensations must be idempotent (can be called multiple times)
4. **Eventual Consistency**: Order may temporarily be in "pending" state

**Saga State Transitions:**
```
PENDING → IN_PROGRESS → COMPLETED (happy path)
                      → COMPENSATING → FAILED (error path)
```

---

## 10.6 UnitOfWork Pattern Flow

This diagram shows how UnitOfWork coordinates multiple repository operations in a single transaction:

```mermaid
sequenceDiagram
    participant UC as Use Case
    participant UoW as UnitOfWork
    participant UserRepo as UserRepository
    participant OrderRepo as OrderRepository
    participant Prisma as Prisma Client
    participant DB as PostgreSQL

    UC->>UoW: begin()
    UoW->>Prisma: prisma.$transaction()
    Prisma-->>UoW: Transactional client (tx)

    UC->>UserRepo: save(user)
    UserRepo->>UoW: getTransaction()
    UoW-->>UserRepo: tx
    UserRepo->>Prisma: tx.user.update(...)
    Prisma->>DB: UPDATE users...
    DB-->>Prisma: Updated

    UC->>OrderRepo: save(order)
    OrderRepo->>UoW: getTransaction()
    UoW-->>OrderRepo: tx
    OrderRepo->>Prisma: tx.order.create(...)
    Prisma->>DB: INSERT INTO orders...
    DB-->>Prisma: Created

    alt Commit succeeds
        UC->>UoW: commit()
        UoW->>Prisma: COMMIT
        Prisma->>DB: COMMIT TRANSACTION
        DB-->>Prisma: ✅ Committed
        UoW-->>UC: Success
    else Commit fails
        UC->>UoW: rollback()
        UoW->>Prisma: ROLLBACK
        Prisma->>DB: ROLLBACK TRANSACTION
        DB-->>Prisma: ✅ Rolled back
        UoW-->>UC: Error
    end
```

**Key benefits:**
1. **Single Transaction**: Multiple repositories share the same DB transaction
2. **Consistency**: All-or-nothing commit across multiple aggregates
3. **Performance**: Batch operations in single roundtrip

**Code example:**
```typescript
async execute(input: CreateOrderInput) {
  await this.unitOfWork.begin();

  try {
    const user = await this.userRepo.findById(input.userId);
    user.incrementOrderCount();
    await this.userRepo.save(user);

    const order = Order.create(input);
    await this.orderRepo.save(order);

    await this.unitOfWork.commit();
  } catch (error) {
    await this.unitOfWork.rollback();
    throw error;
  }
}
```

---

## 10.7 Cross-BC Communication Flow (Event-Driven)

This diagram shows how bounded contexts communicate asynchronously via domain events:

```mermaid
sequenceDiagram
    participant Client
    participant UserBC as User BC<br/>(Supporting)
    participant Outbox as outbox Table
    participant EventBus as Event Bus
    participant NotifBC as Notification BC<br/>(Supporting)
    participant BillingBC as Billing BC<br/>(Supporting)

    Client->>UserBC: POST /users (create user)

    UserBC->>UserBC: User.create()
    Note over UserBC: Record UserCreatedEvent

    UserBC->>Outbox: Save user + events (TX)
    Note over Outbox: ✅ Atomic commit

    UserBC-->>Client: 201 Created

    Note over Outbox,BillingBC: Asynchronous Event Processing

    Outbox->>EventBus: Publish UserCreatedEvent

    par Parallel Event Handling
        EventBus->>NotifBC: UserCreatedEvent
        NotifBC->>NotifBC: Send welcome email
        NotifBC-->>EventBus: Done
    and
        EventBus->>BillingBC: UserCreatedEvent
        BillingBC->>BillingBC: Create free subscription
        BillingBC-->>EventBus: Done
    end

    Note over NotifBC,BillingBC: Both BCs react independently
```

**Key characteristics:**
1. **Decoupling**: User BC doesn't know about Notification/Billing BCs
2. **Async**: Event handling doesn't block user creation response
3. **Independence**: Each BC processes events at its own pace
4. **Scalability**: Multiple subscribers can handle same event

---

## Navigation

[← Previous: Common Pitfalls](11-common-pitfalls.md) | [Index](README.md) | [Next: Domain Services & Use Cases](13-domain-services-and-use-cases.md)
