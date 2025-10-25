# 10. Decision Matrices - When to Use What

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

Quick reference tables to help make architectural decisions.

## 4.1 Input Ports: When to Use

| Scenario                                  | Use Input Port? | Reason                                                           |
| ----------------------------------------- | --------------- | ---------------------------------------------------------------- |
| Single consumer (HTTP only)               | ❌ No           | Controller → UseCase directly. No abstraction needed.            |
| Multiple consumers (HTTP + GraphQL + CLI) | ✅ Yes          | Controller/Resolver/CLI → Input Port → UseCase. Stable contract. |
| Module without presentation               | ❌ No           | Just export Use Cases. Other modules inject them directly.       |
| Future-proofing for multiple consumers    | ⚠️ Maybe        | Only if planned, not "just in case". YAGNI principle.            |

**Example: When NOT to use Input Port**

```typescript
// ✅ SIMPLE - Direct injection when only HTTP consumer
@Controller('users')
export class UserController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  @Post()
  async create(@Body() dto: CreateUserDto) {
    const input = { email: dto.email };
    const output = await this.createUser.execute(input);
    return new UserResponse(output);
  }
}
```

**Example: When to use Input Port**

```typescript
// ✅ WITH PORT - Multiple consumers need stable contract

// application/ports/input/for-managing-users.port.ts
export interface ForManagingUsers {
  createUser(input: CreateUserInput): Promise<CreateUserOutput>;
}

// application/use-cases/create-user/create-user.use-case.ts
@Injectable()
export class CreateUserUseCase implements ForManagingUsers {
  async createUser(input: CreateUserInput): Promise<CreateUserOutput> {
    // ...
  }
}

// Consumed by HTTP
@Controller('users')
export class UserHttpController {
  constructor(@Inject(FOR_MANAGING_USERS) private port: ForManagingUsers) {}
}

// Consumed by GraphQL
@Resolver()
export class UserGraphQLResolver {
  constructor(@Inject(FOR_MANAGING_USERS) private port: ForManagingUsers) {}
}
```

---

## 4.2 Class vs Type vs DTO

| Layer                         | Input                               | Output                                 | Reason                                             |
| ----------------------------- | ----------------------------------- | -------------------------------------- | -------------------------------------------------- |
| **Controller (Presentation)** | `class` with `class-validator`      | `class` implementing `HttpResponse<T>` | Runtime HTTP validation needed                     |
| **Use Case (Application)**    | `class` with constructor validation | `type` (plain interface/type)          | Constructor validates domain rules, output is data |
| **Domain**                    | Never DTOs                          | Entities, VOs, or types                | Pure business, no framework                        |
| **Repository Adapter**        | Entity (domain)                     | Entity or null                         | Maps between domain and DB                         |

**Example:**

```typescript
// ✅ Presentation Layer - class with decorators
export class CreateUserRequestDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;
}

export class UserResponseDto {
  data: UserData;
}

// ✅ Application Layer Input - class with constructor
export class CreateUserInput {
  readonly email: Email;

  constructor(emailStr: string) {
    this.email = new Email(emailStr); // Validates in VO constructor
  }
}

// ✅ Application Layer Output - type (plain data)
export type CreateUserOutput = {
  id: string;
  email: string;
  createdAt: Date;
};

// ✅ Domain - never DTOs
export class User extends AggregateRoot {
  // Pure domain entity
}
```

---

## 4.3 Validation Placement

| Validation Type        | Layer                  | Tool              | Example                              |
| ---------------------- | ---------------------- | ----------------- | ------------------------------------ |
| **HTTP format**        | Presentation (DTO)     | `class-validator` | `@IsEmail()`, `@IsNotEmpty()`        |
| **Value format**       | Domain (VO)            | Constructor       | `new Email(value)` throws if invalid |
| **Entity invariants**  | Domain (Entity)        | Methods           | `order.addItem()` checks max items   |
| **Cross-entity rules** | Domain (Service)       | Domain Service    | Price calculation across entities    |
| **DB constraints**     | Application (Use Case) | Use case logic    | Check email uniqueness before save   |

**Example:**

```typescript
// 1. HTTP Validation (Presentation)
export class CreateUserDto {
  @IsEmail() // ✅ Format validation
  @MaxLength(255)
  email: string;
}

// 2. Value Object Validation (Domain)
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!email.match(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)) {
      throw new InvalidEmailError(); // ✅ Domain rule
    }
    this.value = email.toLowerCase();
  }
}

// 3. Entity Invariant (Domain)
export class Order {
  private items: OrderItem[] = [];

  addItem(item: OrderItem): void {
    if (this.items.length >= MAX_ITEMS) {
      throw new MaxItemsExceededError(); // ✅ Business rule
    }
    this.items.push(item);
  }
}

// 4. DB Constraint (Use Case)
export class CreateUserUseCase {
  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) {
      throw new UserAlreadyExistsError(); // ✅ DB constraint
    }
    // ...
  }
}
```

---

## 4.4 Module Exports

| Module Type                   | What to Export         | Reason                                    |
| ----------------------------- | ---------------------- | ----------------------------------------- |
| **Public Module (with HTTP)** | Nothing                | Consumed via HTTP, not internal injection |
| **Internal Module (no HTTP)** | Use Cases              | Other modules inject use cases            |
| **Shared Domain**             | Entities, VOs          | Reference domain objects (by ID usually)  |
| **Infrastructure Module**     | Adapters (if reusable) | Share Prisma, Logger, EventBus, etc.      |

**Examples:**

```typescript
// ✅ Public Module - No exports
@Module({
  imports: [UserInfrastructureModule, UserPresentationModule],
  providers: [CreateUserUseCase, GetUserUseCase],
  exports: [], // ✅ Empty - consumed via HTTP
})
export class UserModule {}

// ✅ Internal Module - Export use cases
@Module({
  providers: [GetCatalogUseCase, ListCatalogsUseCase],
  exports: [GetCatalogUseCase, ListCatalogsUseCase], // ✅ For other modules
})
export class CatalogModule {}

// ✅ Shared Domain - Export entity for reference
@Module({
  providers: [
    /* ... */
  ],
  exports: [User], // ✅ Other modules can reference User type
})
export class UserDomainModule {}
```

---

## 4.5 When to Use Domain Services

| Scenario                        | Use Domain Service? | Alternative                 |
| ------------------------------- | ------------------- | --------------------------- |
| Logic affects single entity     | ❌ No               | Entity method               |
| Logic spans multiple entities   | ✅ Yes              | Domain Service              |
| Complex calculation (1 entity)  | ⚠️ Maybe            | Entity method or Service    |
| Need to load/persist entities   | ❌ No               | Use Case (not Domain)       |
| External API calls              | ❌ No               | Adapter in Infrastructure   |
| Database operations             | ❌ No               | Repository in Infrastructure |

**Decision Tree:**

```
Does the logic involve:
├─ Single entity only?
│  └─ Entity method (user.suspend())
├─ Multiple entities?
│  └─ Domain Service (transferBetweenAccounts())
├─ Loading/persisting data?
│  └─ Use Case (not Domain Service)
└─ External API/DB?
   └─ Adapter (not Domain Service)
```

---

## 4.6 Repository Choice Matrix

| Criteria | Prisma | TypeORM | Raw SQL (pg/mysql2) |
|----------|--------|---------|---------------------|
| **Type Safety** | ⭐⭐⭐ Excellent | ⭐⭐ Good | ⭐ Manual types |
| **DX (DevEx)** | ⭐⭐⭐ Best-in-class | ⭐⭐ Good | ⭐ Verbose |
| **Performance** | ⭐⭐ Good (N+1 watch) | ⭐⭐ Good | ⭐⭐⭐ Best control |
| **Migrations** | ⭐⭐⭐ Auto-generated | ⭐⭐⭐ TypeScript-based | ⭐ Manual |
| **Learning Curve** | ⭐⭐⭐ Easy | ⭐⭐ Moderate | ⭐ Steep |
| **Raw Query Support** | ⭐⭐⭐ `$queryRaw` | ⭐⭐⭐ `query()` | ⭐⭐⭐ Native |
| **Ecosystem** | ⭐⭐⭐ Growing fast | ⭐⭐⭐ Mature | ⭐ DIY |
| **Team Onboarding** | ⭐⭐⭐ Fast | ⭐⭐ Moderate | ⭐ Slow |

**Decision Tree:**

```
Team Skill Level?
├─ Junior/Mixed → Prisma (best DX)
├─ Senior → TypeORM (flexibility)
└─ High-performance needs → Raw SQL (full control)

Migration Strategy?
├─ Auto-generated → Prisma
├─ Code-first migrations → TypeORM
└─ Manual control → Raw SQL

Greenfield vs Legacy?
├─ Greenfield → Prisma (modern)
├─ Existing TypeORM → Keep TypeORM (avoid migration cost)
└─ Legacy DB schema → Raw SQL (complex mappings)
```

**Migration Path:**
- Prisma ↔ TypeORM: Moderate effort (repository layer only)
- Any ORM → Raw SQL: High effort (lose abstraction)
- Raw SQL → ORM: High effort (schema modeling)

---

## 4.7 Event Handling Strategy Matrix

| Scenario | Pattern | When to Use | Trade-offs |
|----------|---------|-------------|------------|
| **MVP / Single BC** | In-memory EventBus | - Prototyping<br/>- Single bounded context<br/>- Low event volume | ✅ Simple<br/>❌ Events can be lost<br/>❌ No replay |
| **Reliability Needed** | Outbox Pattern | - Events must not be lost<br/>- Transaction + events atomic<br/>- Scale-up phase | ✅ Guaranteed delivery<br/>✅ Transactional<br/>❌ Added complexity |
| **Idempotency Needed** | Inbox Pattern | - Events can be duplicated<br/>- At-least-once delivery<br/>- Multiple consumers | ✅ Idempotent handlers<br/>✅ Duplicate protection<br/>❌ Storage overhead |
| **Multi-BC Transactions** | Saga Orchestration | - Cross-BC operations<br/>- Compensations required<br/>- Complex workflows | ✅ Consistency<br/>✅ Compensating actions<br/>❌ High complexity |
| **High Volume** | Outbox + Message Broker | - >1000 events/sec<br/>- Async processing<br/>- Multiple consumers | ✅ Scalable<br/>✅ Decoupled<br/>❌ Operational overhead |

**Decision Tree:**

```
Event Volume?
├─ <100/sec, Single BC → In-memory EventBus
├─ >100/sec, Multi-BC → Outbox + Message Broker (Kafka/NATS)
└─ >1000/sec → Outbox + Kafka + Worker Pool

Can Events be Lost?
├─ No → Outbox Pattern (transactional)
└─ Yes (monitoring, analytics) → In-memory

Can Events be Duplicated?
├─ No → Inbox Pattern (idempotency)
└─ Yes (idempotent handlers) → No Inbox needed

Cross-BC Coordination?
├─ Yes → Saga Orchestration
└─ No → Simple event handlers
```

**Migration Path:**
1. **Start:** In-memory EventBus (MVP)
2. **Add reliability:** Outbox table + Publisher worker
3. **Add idempotency:** Inbox table + checks in handlers
4. **Add orchestration:** Saga orchestrators for multi-BC flows
5. **Scale:** Replace in-memory with Kafka/NATS broker

---

## 4.8 Authentication & Authorization Strategy

| Aspect | Generic (IdP Adapter) | Supporting (Internal IAM) |
|--------|----------------------|---------------------------|
| **When to Use** | - Standard OIDC/OAuth2<br/>- No custom flows<br/>- Multi-tenant SaaS<br/>- SSO required | - Complex policies (ABAC)<br/>- Custom workflows<br/>- Fine-grained permissions<br/>- Business rules in authz |
| **Examples** | Auth0, Cognito, Keycloak, Okta | Custom RBAC/ABAC system |
| **Classification** | **Generic** subdomain | **Supporting** subdomain |
| **Investment** | 5-10% (thin adapter) | 20-30% (build in-house) |
| **Control** | Low (vendor-controlled) | High (full control) |
| **Cost** | Pay per user (variable) | Fixed infra cost |
| **Maintenance** | Vendor handles | Team maintains |
| **Custom Policies** | Limited (hooks, rules) | Unlimited (code) |

**Decision Tree:**

```
Auth Requirements?
├─ Standard OIDC/OAuth2 only?
│  └─ Yes → Generic (Auth0/Cognito)
├─ Custom business rules in authz?
│  └─ Yes → Supporting (internal IAM)
├─ Need fine-grained permissions (resource-level)?
│  └─ Yes → Supporting (ABAC system)
└─ SSO + basic roles only?
   └─ Yes → Generic (Keycloak/Okta)

Team Capability?
├─ Small team, limited security expertise → Generic (IdP)
├─ Large team, security-critical → Supporting (internal)
└─ Hybrid (authn generic, authz internal) → Both!
```

**Folder Structure:**

```
# Generic - IdP Adapter
generic/
└── idp-adapter/              # Generic subdomain
    ├── domain/               # Minimal (User token VO)
    ├── application/
    │   └── use-cases/
    │       └── verify-token.use-case.ts
    └── infrastructure/
        └── adapters/
            ├── auth0-adapter.ts
            ├── cognito-adapter.ts
            └── keycloak-adapter.ts

# Supporting - Internal IAM
supporting/
└── iam/                      # Bounded context
    ├── authn/                # Subdomain: login, MFA, sessions
    ├── authz/                # Subdomain: permissions, roles, policies
    │   ├── domain/
    │   │   ├── entities/
    │   │   │   ├── role.entity.ts
    │   │   │   └── permission.entity.ts
    │   │   └── services/
    │   │       └── policy-evaluation.service.ts
    │   └── application/
    │       └── use-cases/
    │           ├── check.use-case.ts
    │           └── assign-role.use-case.ts
    └── user/                 # Subdomain: user management
```

---

## 4.9 Mailing Strategy

| Aspect | Generic (mail-driver) | Supporting (campaign-delivery) |
|--------|-----------------------|-------------------------------|
| **Purpose** | Send transactional emails | Orchestrate campaigns |
| **Classification** | **Generic** subdomain | **Supporting** subdomain |
| **Responsibility** | - Adapter to email service<br/>- Send single email<br/>- Handle API credentials | - When to send (triggers)<br/>- What to send (templates)<br/>- Whom to send (segments)<br/>- Throttling, retries<br/>- A/B testing<br/>- Bounce handling |
| **Investment** | 5-10% (thin wrapper) | 20-30% (business logic) |
| **Examples** | SendGrid adapter, SES adapter | Campaign scheduler, segmentation |

**Decision Tree:**

```
Mailing Complexity?
├─ Simple transactional emails only?
│  └─ Generic only (mail-driver)
├─ Complex campaigns, A/B testing, segmentation?
│  └─ Supporting (campaign-delivery) + Generic (mail-driver)
└─ User-defined templates, scheduling?
   └─ Supporting (campaign-delivery)

Business Logic?
├─ Just send email when told → Generic
├─ When/what/whom logic → Supporting
└─ Throttling, retries, bounce handling → Supporting
```

**Folder Structure:**

```
# Generic - Mail Driver
generic/
└── mail-driver/              # Generic subdomain
    ├── domain/               # Minimal (Email VO)
    ├── application/
    │   └── ports/
    │       └── for-sending-mail.port.ts
    └── infrastructure/
        └── adapters/
            ├── sendgrid-adapter.ts
            ├── ses-adapter.ts
            └── mailgun-adapter.ts

# Supporting - Campaign Delivery
supporting/
└── campaign-delivery/        # Supporting subdomain
    ├── domain/
    │   ├── entities/
    │   │   ├── campaign.entity.ts
    │   │   └── segment.entity.ts
    │   └── services/
    │       └── throttling-policy.service.ts
    ├── application/
    │   ├── use-cases/
    │   │   ├── schedule-campaign.use-case.ts
    │   │   └── send-batch.use-case.ts
    │   └── subscribers/
    │       └── user-created.subscriber.ts
    │           # Listens to UserCreated event
    │           # Calls mail-driver to send welcome email
    └── infrastructure/
        └── adapters/
            └── mail-driver-client.ts
                # Uses mail-driver port
```

**Usage Example:**

```typescript
// Supporting - campaign-delivery listens to events
@Injectable()
export class UserCreatedSubscriber {
  constructor(
    @Inject(FOR_SENDING_MAIL)
    private readonly mailDriver: ForSendingMail, // Generic port
    private readonly templateRepo: TemplateRepository,
  ) {}

  async handle(event: UserCreatedEvent) {
    // Supporting logic: decide what/when/whom
    const template = await this.templateRepo.find('welcome');
    const shouldSend = this.canSendTo(event.userCountry);

    if (shouldSend) {
      // Generic: just send
      await this.mailDriver.send({
        to: event.email,
        subject: template.subject,
        body: template.render({ name: event.name }),
      });
    }
  }
}
```

---

**Navigation:** [Previous: Recipes](./09-recipes.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Common Pitfalls](./11-common-pitfalls.md)
