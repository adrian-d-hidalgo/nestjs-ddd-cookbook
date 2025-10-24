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

## Navigation

[← Previous: Common Pitfalls](11-common-pitfalls.md) | [Index](README.md)
