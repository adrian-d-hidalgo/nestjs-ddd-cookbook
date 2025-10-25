# 7. Shared Kernel - Common Reusable Components

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

Located in `src/shared-kernel/`, contains only **immutable** and **reusable** elements between modules (Bounded Contexts).

## What Belongs in Shared Kernel

> **Critical Rule:** `shared-kernel/` contains **ONLY pure domain** elements with **zero external dependencies**. Technical capabilities go to `common/`.

### Structure

```
shared-kernel/
├── domain/              # Pure domain elements
│   ├── value-objects/   # Universal VOs (Email, UUID, Money)
│   ├── events/          # DomainEvent interface, AggregateRoot
│   └── exceptions/      # DomainException, InvalidArgumentException
└── utils/               # ONLY pure functions (no external deps)
    ├── invariants/      # ensure(), guard(), assert()
    └── functional/      # Pure helpers for domain logic
```

### Content Policy: Allowed vs Forbidden

| Category | ✅ Allowed in `shared-kernel/` | ❌ Forbidden (goes to `common/`) |
|----------|-------------------------------|----------------------------------|
| **Value Objects** | Universal VOs: `Email`, `UUID`, `Money`, `PhoneNumber`, `Currency` | VOs specific to single BC |
| **Domain Events** | `DomainEvent` interface, `AggregateRoot` base class | Concrete event implementations (go to BC's `domain/events/`) |
| **Exceptions** | `DomainException`, `InvalidArgumentException` | Framework exceptions (HTTP, validation) |
| **Utilities** | Pure functions: `ensure()`, `guard()`, `assert()` | Functions with I/O, network, filesystem, time |
| **Calculations** | Pure math/business calculations (deterministic) | Calculations requiring external data/APIs |
| **Parsers** | Pure parsers/formatters for VOs (RFC, NIF) | Parsers requiring I/O or network calls |
| **Functional** | Immutable helpers, pure transformations | Stateful helpers, side effects |
| **Dependencies** | Zero external dependencies | NestJS, Express, Prisma, Axios, any framework |
| **Side Effects** | None (referentially transparent) | I/O, database, HTTP, filesystem, time, random |
| **Ports** | None | Technical ports like `UnitOfWork`, `EventPublisher`, `Clock` → `common/application/ports/` |
| **Adapters** | None | All adapters → `common/infrastructure/` |
| **Presentation** | None | Filters, interceptors, pipes, guards → `common/presentation/` |
| **Entities** | Only abstract base (`AggregateRoot`) | Concrete entities → BC's `domain/entities/` |
| **Domain Services** | Universal & stable domain services (rare!) | BC-specific domain services → BC's `domain/services/` |
| **Use Cases** | None | All use cases → BC's `application/use-cases/` |

---

### Detailed Rules

**✅ ALLOWED:**
- **Universal Value Objects**: `Email`, `UUID`, `Money`, `Currency`, `PhoneNumber` (validated, immutable)
- **Domain Events base**: `DomainEvent` interface, `AggregateRoot` base class
- **Domain Exceptions**: `DomainException`, `InvalidArgumentException`
- **Pure invariant utilities**: `ensure()`, `guard()`, `assert()` (no I/O, no side effects)
- **Pure business calculations**: Mathematical or domain-specific pure functions
- **Pure parsers/formatters**: For VOs semantic validation (e.g., RFC/NIF parsing) - **NO I/O**
- **Pure functional helpers**: Used by domain logic (immutable, no side effects)

**❌ FORBIDDEN:**
- **Framework dependencies** (NestJS, Express, decorators like `@Injectable`)
- **Any I/O operations** (filesystem, network, database)
- **System time/random** (use `Clock` port from `common/application` instead)
- **Technical ports/contracts** → Use `common/application/ports/` (e.g., `UnitOfWork`, `EventPublisher`, `Clock`)
- **Implementations/Adapters** → Use `common/infrastructure/` (e.g., `PrismaUnitOfWork`, `KafkaEventBus`)
- **Presentation layer** → Use `common/presentation/` (e.g., filters, interceptors, pipes)
- **Business logic specific** to a single BC (should live in that BC's `domain/`)
- **Complete Entities/Aggregates** (only abstract bases like `AggregateRoot`)
- **Use Cases** or **Application Services** (belong in BC's `application/` layer)

### Domain Services in Shared Kernel

**Only include Domain Services in `shared-kernel/` when they meet ALL criteria:**

1. ✅ **Universality**: Rule applies equally across ≥2 Bounded Contexts
2. ✅ **Purity**: No repositories, buses, system time, or I/O
3. ✅ **Stability**: Changes very little; semantics are transversal
4. ✅ **Domain**: Expresses business rules, not orchestration/authorization

**Valid examples:**
- `CurrencyRoundingPolicy` (if fiscal/accounting policy is universal)
- `GeoDistance` (if distance is part of business semantics)
- `TaxIdValidator` (RFC/NIF) with pure validation logic

**Invalid examples:**
- `DiscountPolicy` specific to Marketing BC
- `PricingStrategy` varying by product/market
- Services accessing repos/infrastructure

> **Rule of thumb:** If in doubt, keep it in the BC's `domain/services/`. Only promote to `shared-kernel/` after proving universality + purity + stability.

## Common Value Objects

### Email Value Object

```typescript
// shared-kernel/domain/value-objects/email.vo.ts
export class Email {
  private readonly value: string;

  constructor(value: string) {
    if (!/.+@.+\..+/.test(value)) {
      throw new InvalidEmailError();
    }
    this.value = value.toLowerCase();
  }

  toString(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}
```

### Money Value Object

```typescript
// shared-kernel/domain/value-objects/money.vo.ts
export class Money {
  private constructor(
    private readonly amount: number,
    private readonly currency: Currency,
  ) {}

  static fromCents(cents: number, currency: Currency = 'USD'): Money {
    return new Money(cents, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new CurrencyMismatchError();
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiplyBy(factor: number): Money {
    return new Money(Math.round(this.amount * factor), this.currency);
  }
}
```

## Domain Events

### Domain Event Interface

```typescript
// shared-kernel/domain/event/domain-event.interface.ts
export interface DomainEvent {
  readonly occurredAt: Date;
  readonly eventName: string;
}

export abstract class BaseDomainEvent implements DomainEvent {
  readonly occurredAt = new Date();
  abstract readonly eventName: string;
}
```

### Event Naming Convention

Use consistent naming pattern for discoverability and maintainability:

**Pattern**: `{context}.{entity}.{past-tense-action}`

```typescript
// ✅ GOOD - Follows convention
export class UserCreatedEvent implements DomainEvent {
  static readonly EVENT_NAME = 'user.domain.created';
  readonly eventName = UserCreatedEvent.EVENT_NAME;
  readonly occurredAt = new Date();

  constructor(
    public readonly userId: string,
    public readonly email: string,
  ) {}
}

export class PermissionAddedToRoleEvent implements DomainEvent {
  static readonly EVENT_NAME = 'authz.role.permission-added';
  readonly eventName = PermissionAddedToRoleEvent.EVENT_NAME;
  readonly occurredAt = new Date();

  constructor(
    public readonly roleId: string,
    public readonly resource: string,
    public readonly action: string,
  ) {}
}
```

**Naming rules**:

- Use **lowercase** with **dots** as separator
- Format: `{context}.{aggregate}.{past-tense-action}`
- Use **past tense** (created, updated, deleted, added, removed)
- Use **static readonly EVENT_NAME** for single source of truth
- Include all relevant data as public readonly properties

## Aggregate Root Base Class

```typescript
// shared-kernel/domain/event/aggregate-root.ts
import { DomainEvent } from './domain-event.interface';

export abstract class AggregateRoot {
  private domainEvents: Array<DomainEvent>;

  constructor() {
    this.domainEvents = [];
  }

  pullEvents(): Array<DomainEvent> {
    const domainEvents = this.domainEvents.slice();
    this.domainEvents = [];
    return domainEvents;
  }

  protected record(event: DomainEvent): void {
    this.domainEvents.push(event);
  }
}
```

**Key principles:**

- **Entities accumulate events internally**: Use `record()` to add events to internal collection
- **Entity methods return void**: Business methods modify state and accumulate events, but don't return them
- **Repository extracts and publishes events**: After successful save, use `pullEvents()` to extract (and automatically clear) events
- **`pullEvents()` is atomic**: Extracts events AND clears the queue in one operation
- **Transactional consistency**: Ensures atomic operation between state persistence and event publishing

**When to use factory methods:**

- **`create()`**: For NEW entities - records creation event
- **`reconstitute()`**: For EXISTING entities from DB - does NOT record events

```typescript
// ✅ Creating new entity (records events)
static create(id: string, email: Email): User {
  const user = new User(id, email, 'active');
  user.record(new UserCreatedEvent(id, email.toString())); // Records event
  return user;
}

// ✅ Reconstituting from database (no events)
static reconstitute(id: string, email: string, status: string): User {
  return new User(id, new Email(email), status); // NO events recorded
}
```

## Event Bus Abstraction

The Event Bus is a **critical infrastructure component** in DDD that enables **decoupled communication** between bounded contexts through domain events.

### Event Bus Port (Interface)

```typescript
// shared-kernel/infrastructure/event-bus/interfaces/event-bus-port.interface.ts
import { DomainEvent } from '../../domain/event/domain-event.interface';

export type EventHandler<T extends DomainEvent = DomainEvent> = (
  event: T,
) => void | Promise<void>;

export interface EventBusPort {
  publish(events: DomainEvent[]): void | Promise<void>;
  publishOne(event: DomainEvent): void | Promise<void>;
  on<T extends DomainEvent>(eventName: string, handler: EventHandler<T>): void;
  once<T extends DomainEvent>(
    eventName: string,
    handler: EventHandler<T>,
  ): void;
  removeListener<T extends DomainEvent>(
    eventName: string,
    handler: EventHandler<T>,
  ): void;
  removeAllListeners(eventName?: string): void;
}

export const EVENT_BUS = Symbol('EVENT_BUS');
```

### Usage in Repositories

```typescript
// infrastructure/adapters/persistence/prisma-user.repository.ts
import { Injectable, Inject } from '@nestjs/common';
import { PrismaService } from '@shared-kernel/infrastructure/adapters/persistence/prisma.service';
import {
  EventBusPort,
  EVENT_BUS,
} from '@shared-kernel/infrastructure/event-bus';
import { User } from '../../../domain/entities/user.entity';

@Injectable()
export class PrismaUserRepository {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(EVENT_BUS) private readonly eventBus: EventBusPort,
  ) {}

  async save(user: User): Promise<void> {
    // 1. Persist entity to database
    await this.prisma.user.upsert({
      where: { id: user.getId() },
      create: {
        id: user.getId(),
        email: user.getEmail().toString(),
      },
      update: {
        email: user.getEmail().toString(),
      },
    });

    // 2. Extract events atomically (extracts and clears)
    const events = user.pullEvents();

    // 3. Publish events AFTER successful persistence
    if (events.length > 0) {
      await this.eventBus.publish(events);
    }
  }
}
```

### Event Handlers (Subscribers)

```typescript
// supporting/notification/application/subscribers/user-created.subscriber.ts
import { Injectable, Inject, OnModuleInit } from '@nestjs/common';
import {
  EventBusPort,
  EVENT_BUS,
} from '@shared-kernel/infrastructure/event-bus';
import { UserCreatedEvent } from '@supporting/user/domain/events/user-created.event';
import { SendWelcomeEmailUseCase } from '../use-cases/send-welcome-email.use-case';

@Injectable()
export class UserCreatedSubscriber implements OnModuleInit {
  constructor(
    @Inject(EVENT_BUS) private readonly eventBus: EventBusPort,
    private readonly sendWelcomeEmail: SendWelcomeEmailUseCase,
  ) {}

  onModuleInit() {
    // Subscribe to event
    this.eventBus.on<UserCreatedEvent>(
      UserCreatedEvent.EVENT_NAME,
      this.handleUserCreated.bind(this),
    );
  }

  private async handleUserCreated(event: UserCreatedEvent): Promise<void> {
    console.log(`[UserCreatedSubscriber] Processing event: ${event.userId}`);

    await this.sendWelcomeEmail.execute({
      userId: event.userId,
      email: event.email,
    });
  }
}
```

## Best Practices

- **Keep it small**: Only elements **really shared** between ≥2 BCs
- **Independent versioning**: Own changelog, SemVer
- **Controlled evolution**: Changes must be **backward compatible**
- **Golden rule**: If only 1 BC uses it, it does **NOT** go in Shared Kernel

## Suggested Structure

```
src/shared-kernel/
├── domain/
│   ├── event/
│   │   ├── aggregate-root.ts
│   │   ├── domain-event.interface.ts
│   │   └── index.ts
│   ├── exceptions/
│   │   ├── domain.exception.ts
│   │   ├── invalid-argument.exception.ts
│   │   └── index.ts
│   ├── value-objects/
│   │   ├── email.vo.ts
│   │   ├── uuid-v7.vo.ts
│   │   ├── value-object.ts
│   │   └── index.ts
│   └── index.ts
├── application/
│   ├── ports/
│   │   └── index.ts
│   └── index.ts
├── infrastructure/
│   ├── adapters/
│   │   ├── persistence/
│   │   │   ├── prisma.service.ts
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── event-bus/
│   │   ├── interfaces/
│   │   │   └── event-bus-port.interface.ts
│   │   ├── drivers/
│   │   │   ├── in-memory-event-bus.driver.ts
│   │   │   └── index.ts
│   │   ├── event-bus.module.ts
│   │   └── index.ts
│   └── index.ts
├── presentation/
│   ├── http/
│   │   ├── dtos/
│   │   ├── filters/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── pipes/
│   │   ├── validators/
│   │   └── index.ts
│   └── index.ts
└── index.ts
```

## TypeScript Path Aliases

Configure path aliases in `tsconfig.json` for cleaner imports:

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@core/*": ["src/core/*"],
      "@supporting/*": ["src/supporting/*"],
      "@generic/*": ["src/generic/*"],
      "@shared-kernel/*": ["src/shared-kernel/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

**Usage:**

```typescript
// ✅ GOOD - Using aliases
import { Email } from '@shared-kernel/domain/value-objects/email.vo';
import { AggregateRoot } from '@shared-kernel/domain/event/aggregate-root';
import { DomainEvent } from '@shared-kernel/domain/event/domain-event.interface';
import { User } from '@supporting/user/domain/entities/user.entity';

// ❌ BAD - Relative paths
import { Email } from '../../../../shared-kernel/domain/value-objects/email.vo';
import { User } from '../../domain/entities/user.entity';
```

---

**Navigation:** [Previous: Presentation Layer](./06-presentation-layer.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Testing](./08-testing.md)
