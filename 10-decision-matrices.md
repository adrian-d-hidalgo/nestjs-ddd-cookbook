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

**Navigation:** [Previous: Recipes](./09-recipes.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Common Pitfalls](./11-common-pitfalls.md)
