# 2. Project Structure & Module Anatomy

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

## DDD Folder Structure

Projects using this architecture are organized by **DDD subdomain classification** (Core, Supporting, Generic):

```
src/
├── core/                              # CORE DOMAIN (60-70% investment)
│   └── {bounded-context}/             # Optional: large projects only
│       └── {subdomain}/               # Business differentiators
│           ├── domain/
│           ├── application/
│           ├── infrastructure/
│           └── presentation/
├── supporting/                        # SUPPORTING (20-30% investment)
│   └── {bounded-context}/             # Optional: large projects only
│       └── {subdomain}/               # Necessary but standard
│           ├── domain/
│           ├── application/
│           ├── infrastructure/
│           └── presentation/
├── generic/                           # GENERIC (5-10% investment)
│   └── {bounded-context}/             # Optional: large projects only
│       └── {subdomain}/               # Bought/SaaS solutions
│           ├── domain/
│           ├── application/
│           ├── infrastructure/
│           └── presentation/
├── shared-kernel/                     # Pure domain shared across ALL modules
│   ├── domain/                        # VOs, Events, Exceptions, AggregateRoot
│   ├── utils/                         # Pure functions (no external deps)
│   └── libs/                          # Only if no technical deps
├── common/                            # Transversal technical capabilities
│   ├── application/                   # Technical ports (UoW, EventPublisher, Clock)
│   ├── infrastructure/                # Adapters (Prisma, Kafka, Auth0, HTTP)
│   ├── presentation/                  # Global HTTP (filters, interceptors, pipes)
│   └── utils/                         # Technical utilities (Result, functional)
├── types/                             # TypeScript type definitions
└── main.ts                            # NestJS bootstrap
```

**Note on Bounded Contexts:**

**For small/medium projects** (1-3 developers, <10 modules), omit `{bounded-context}` level:

```
src/supporting/
├── user/              # Subdomain directly
├── authz/             # Subdomain directly
└── notification/      # Subdomain directly
```

**For large projects** (>3 developers, >10 modules), use bounded context grouping:

```
src/supporting/
└── iam/                    # Bounded Context
    ├── user/               # Subdomain
    │   ├── domain/
    │   ├── application/
    │   └── infrastructure/
    ├── authz/              # Subdomain
    │   ├── domain/
    │   ├── application/
    │   └── infrastructure/
    └── authentication/     # Subdomain
        ├── domain/
        ├── application/
        └── infrastructure/
```

## Strategic Classification

| Type           | Investment | Strategy                              | Examples                            |
| -------------- | ---------- | ------------------------------------- | ----------------------------------- |
| **Core**       | 60-70%     | Custom-built, best team, high quality | Business-critical modules           |
| **Supporting** | 20-30%     | Build in-house, good quality          | user/, notification/, file-storage/ |
| **Generic**    | 5-10%      | Buy/SaaS/OSS, minimal customization   | health/, setup/                     |

## Shared Kernel vs Common

Understanding the distinction between `shared-kernel/` and `common/` is critical for maintaining clean architecture boundaries:

| Aspect | `shared-kernel/` | `common/` |
|--------|------------------|-----------|
| **Purpose** | Pure domain shared across BCs | Transversal technical capabilities |
| **Contains** | VOs, Events, Exceptions, invariants | Ports, Adapters, Filters, Pipes |
| **Dependencies** | Zero external dependencies | NestJS, Prisma, frameworks allowed |
| **Purity** | 100% pure functions | Impure (I/O, DB, network, time) |
| **Mutability** | Immutable | Can have side effects |
| **Examples** | `Email.vo`, `DomainEvent`, `ensure()` | `UnitOfWork`, `PrismaService`, `HttpExceptionFilter` |
| **Changes with** | Business language evolution | Technology stack changes |

### Shared Kernel Rules

**Allowed (✅):**
- Value Objects universal to all BCs
- Domain Events base classes
- Domain Exceptions
- Pure utility functions (invariants, guards, assertions)
- Pure mathematical or business calculation functions
- Pure parsers/formatters for VOs (no I/O, network, time, FS)

**Not Allowed (🚫):**
- Anything with framework dependencies (NestJS, Express)
- Anything with I/O, network, system time, random global
- Technical ports/contracts → go to `common/application`
- Implementations (DB, messaging, HTTP, auth) → go to `common/infrastructure`
- Global presentation (filters, interceptors) → go to `common/presentation`

### Common Rules

**Purpose:** Concentrate transversal technical capabilities that are NOT domain.

**Structure:**
- `common/application/` - Technical contracts (interfaces/ports): `UnitOfWork`, `EventPublisher`, `Clock`, `IdGenerator`
- `common/infrastructure/` - Technical implementations: `PrismaUnitOfWork`, `KafkaEventBus`, `Auth0Adapter`
- `common/presentation/` - Global HTTP layer: filters, interceptors, pipes, validators (only if truly transversal)
- `common/utils/` - Technical utilities: `Result<T>`, functional helpers

## Module Anatomy (Hexagonal Structure)

```
src/core/[module-name]/
├── domain/                           # PURE BUSINESS LOGIC
│   ├── entities/
│   │   ├── [entity].entity.ts        # Business object with behavior
│   │   └── [entity].entity.spec.ts   # Unit tests
│   ├── value-objects/
│   │   ├── [value-object].vo.ts      # Validated immutable value
│   │   └── [value-object].vo.spec.ts # Unit tests
│   ├── events/
│   │   └── [entity]-[action].event.ts # Domain event
│   ├── services/
│   │   ├── [service].domain-service.ts      # Cross-entity logic
│   │   └── [service].domain-service.spec.ts # Unit tests
│   └── exceptions/
│       └── [module].exceptions.ts     # Domain-specific errors
├── application/                      # USE CASES & PORTS
│   ├── ports/
│   │   ├── input/                    # Optional: for multiple consumers
│   │   │   └── for-managing-[entities].port.ts
│   │   └── output/
│   │       └── for-storing-[entities].port.ts
│   └── use-cases/
│       └── [action-entity]/          # One folder per use case
│           ├── [action-entity].use-case.ts      # Service/orchestrator
│           ├── [action-entity].input.ts         # Input class
│           ├── [action-entity].output.ts        # Output type
│           └── [action-entity].use-case.spec.ts # Unit tests
├── infrastructure/                   # ADAPTERS (TECHNOLOGY)
│   ├── adapters/
│   │   ├── persistence/
│   │   │   ├── prisma-[entity].repository.ts      # Implements output port
│   │   │   └── prisma-[entity].repository.spec.ts # Integration tests
│   │   └── messaging/
│   │       ├── [service]-[entity].adapter.ts      # Implements output port
│   │       └── [service]-[entity].adapter.spec.ts # Integration tests
│   └── [module]-infrastructure.module.ts     # Providers for adapters
├── presentation/                     # HTTP/GRAPHQL/CLI
│   ├── http/
│   │   ├── controllers/
│   │   │   ├── [module].controller.ts        # HTTP endpoints
│   │   │   └── [module].controller.spec.ts   # E2E tests
│   │   └── dtos/
│   │       ├── [action-entity].request.dto.ts  # class-validator
│   │       └── [entity].response.dto.ts        # HttpResponse<T>
│   └── [module]-presentation.module.ts       # Controllers
└── [module].module.ts                # ROOT MODULE (imports all)
```

## What Goes in Each File

| File                       | Purpose                          | Contains                                   |
| -------------------------- | -------------------------------- | ------------------------------------------ |
| `*.entity.ts`              | Business object with identity    | Properties, behavior methods, invariants   |
| `*.entity.spec.ts`         | Entity unit tests                | Test business rules, invariants, events    |
| `*.vo.ts`                  | Validated immutable value        | Validation in constructor, equals() method |
| `*.vo.spec.ts`             | Value Object unit tests          | Test validation rules, behavior methods    |
| `*.event.ts`               | Something happened in domain     | Event data, occurredAt, eventName          |
| `*.domain-service.ts`      | Logic spanning multiple entities | Stateless business operations              |
| `*.domain-service.spec.ts` | Domain service unit tests        | Test cross-entity logic                    |
| `*.exceptions.ts`          | Domain errors                    | DomainException subclasses                 |
| `*.port.ts` (input)        | What the app can do              | execute() methods for use cases            |
| `*.port.ts` (output)       | What the app needs               | Interfaces for external deps               |
| `*.use-case.ts`            | Orchestrate one user action      | execute(input): Promise<output>            |
| `*.use-case.spec.ts`       | Use case unit tests              | Mock ports, test orchestration logic       |
| `*.input.ts`               | Use case input                   | Class with constructor validation          |
| `*.output.ts`              | Use case output                  | Type (plain data structure)                |
| `*.repository.ts`          | Implement persistence port       | Maps Entity ↔ Prisma                      |
| `*.repository.spec.ts`     | Repository integration tests     | Test DB operations, mapping, events        |
| `*.adapter.ts`             | Implement external port          | Maps Domain ↔ External API                |
| `*.adapter.spec.ts`        | Adapter integration tests        | Test external API integration              |
| `*.controller.ts`          | HTTP endpoints                   | Maps DTO → Input → Output → Response       |
| `*.controller.spec.ts`     | Controller E2E tests             | Test HTTP flow end-to-end                  |
| `*.dto.ts`                 | HTTP request/response            | class-validator decorators                 |
| `*.module.ts`              | Wire dependencies                | Providers, imports, exports                |

## File Organization Conventions

### 1. Use Cases: Each use case in its own folder

```
application/use-cases/
├── create-user/
│   ├── create-user.use-case.ts      # Service logic
│   ├── create-user.input.ts         # Input contract
│   ├── create-user.output.ts        # Output contract
│   └── create-user.use-case.spec.ts # Unit tests
└── update-user-email/
    ├── update-user-email.use-case.ts
    ├── update-user-email.input.ts
    ├── update-user-email.output.ts
    └── update-user-email.use-case.spec.ts
```

### 2. Entities & Value Objects: Test files alongside implementation

```
domain/entities/
├── user.entity.ts       # Entity class
└── user.entity.spec.ts  # Entity tests

domain/value-objects/
├── email.vo.ts          # Value Object class
└── email.vo.spec.ts     # VO tests
```

### 3. Repositories & Adapters: Test files alongside implementation

```
infrastructure/adapters/persistence/
├── prisma-user.repository.ts       # Repository implementation
└── prisma-user.repository.spec.ts  # Integration tests
```

## Naming Conventions

- Use case folders: `[action]-[entity]` (create-user, update-user-email)
- Use case files: `[action]-[entity].[type].ts`
- Test files: Same name as implementation + `.spec.ts`
- Integration tests: `*.spec.ts` (repositories, adapters)
- Unit tests: `*.spec.ts` (entities, VOs, use cases, domain services)
- E2E tests: `*.spec.ts` or `*.e2e-spec.ts` (controllers)

---

**Navigation:** [Previous: Getting Started](./01-getting-started.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Domain Layer](./03-domain-layer.md)
