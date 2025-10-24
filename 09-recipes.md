# 9. Implementation Cookbook - Practical Recipes

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

Practical step-by-step guides for common implementation scenarios.

## Recipe 1: New Feature From Scratch

**Goal:** Create a new "Create User" feature with complete hexagonal structure.

**Files to create (in order):**

```
1. Domain Layer (pure business)
   └── domain/entities/user.entity.ts
   └── domain/entities/user.entity.spec.ts          # ✅ Unit tests
   └── domain/value-objects/email.vo.ts
   └── domain/value-objects/email.vo.spec.ts        # ✅ Unit tests
   └── domain/events/user-created.event.ts
   └── domain/exceptions/user.exceptions.ts

2. Application Layer (use cases & ports)
   └── application/ports/output/for-storing-users.port.ts
   └── application/use-cases/create-user/
       ├── create-user.input.ts
       ├── create-user.output.ts
       ├── create-user.use-case.ts
       └── create-user.use-case.spec.ts             # ✅ Unit tests

3. Infrastructure Layer (adapters)
   └── infrastructure/adapters/persistence/
       ├── prisma-user.repository.ts
       └── prisma-user.repository.spec.ts           # ✅ Integration tests
   └── infrastructure/user-infrastructure.module.ts

4. Presentation Layer (HTTP)
   └── presentation/http/dtos/create-user.request.dto.ts
   └── presentation/http/dtos/user.response.dto.ts
   └── presentation/http/controllers/
       ├── user.controller.ts
       └── user.controller.spec.ts                  # ✅ E2E tests
   └── presentation/user-presentation.module.ts

5. Root Module
   └── user.module.ts (wires everything together)
```

**Implementation order (with TDD):**

1. **Value Objects** (Email) - Write test first, then implementation
2. **Entity** (User) - Write test first, then implementation
3. **Output Port** (ForStoringUsers) - Define interface
4. **Use Case** (CreateUserUseCase) - Write test with mocked port, then implementation
5. **Adapter** (PrismaUserRepository) - Write integration test, then implementation
6. **Infrastructure Module** - Wire adapter
7. **DTOs** (Request, Response) - Plain classes
8. **Controller** - Write E2E test, then implementation
9. **Presentation Module** - Wire controller
10. **Root Module** - Import all modules

---

## Recipe 2: Add Endpoint to Existing Module

**Goal:** Add "Update User Email" to existing User module.

**Files to modify/create:**

```
1. Domain (if needed)
   MODIFY: domain/entities/user.entity.ts
   - Add changeEmail(newEmail: Email) method
   CREATE: domain/events/user-email-changed.event.ts

2. Application
   CREATE: application/use-cases/update-user-email/update-user-email.input.ts
   CREATE: application/use-cases/update-user-email/update-user-email.output.ts
   CREATE: application/use-cases/update-user-email/update-user-email.use-case.ts

3. Presentation
   CREATE: presentation/http/dtos/update-user-email.request.dto.ts
   MODIFY: presentation/http/controllers/user.controller.ts
   - Add PATCH /users/:id/email endpoint

4. No changes needed
   - Infrastructure (repository already supports save)
   - Modules (already wired)
```

**Steps:**

1. Add business logic to entity: `user.changeEmail(newEmail)`
2. Create use case with Input/Output
3. Create request DTO
4. Add controller method
5. Test

---

## Recipe 3: Share Code Between Modules

**Decision tree:**

```
Is the code generic/reusable across domains?
├─ YES: Is it a Value Object (Email, Money, etc)?
│  ├─ YES: → shared-kernel/domain/value-objects/
│  └─ NO: Is it infrastructure (Prisma, Logger)?
│     ├─ YES: → shared-kernel/infrastructure/
│     └─ NO: Is it HTTP (Guards, Filters, Interceptors)?
│        ├─ YES: → shared-kernel/presentation/
│        └─ NO: → shared-kernel/utils/
└─ NO: Is it domain-specific but shared?
   ├─ User entity needed in another module?
   │  └─ Export from user.module.ts
   └─ Use case needed in another module?
      └─ Export UseCase from module (rare, prefer events)
```

**Examples:**

```typescript
// ✅ GOOD - Generic Email VO in shared-kernel
// shared-kernel/domain/value-objects/email.vo.ts
export class Email {
  /* ... */
}

// Used everywhere
import { Email } from '@shared-kernel/domain/value-objects/email.vo';

// ✅ GOOD - Domain entity exported from module
// supporting/user/user.module.ts
@Module({
  exports: [User], // Allow other modules to reference User
})

// Used in other modules
import { User } from '@supporting/user/domain/entities/user.entity';

// ❌ BAD - Copying Email VO to each module
// Don't duplicate shared-kernel code
```

---

## Recipe 4: Module Without Presentation (Internal Hexagon)

**When to use:** Module consumed only by other modules, not exposed via HTTP.

**Example:** Catalog module used internally by business modules.

**Structure:**

```
src/supporting/catalog/
├── domain/
├── application/
├── infrastructure/
├── catalog.module.ts  # NO presentation/ folder
```

**Module exports:**

```typescript
// supporting/catalog/catalog.module.ts
@Module({
  imports: [CatalogInfrastructureModule],
  providers: [GetCatalogUseCase, ListCatalogsUseCase],
  exports: [
    // Export use cases for other modules
    GetCatalogUseCase,
    ListCatalogsUseCase,
  ],
})
export class CatalogModule {}
```

**Consumed by another module:**

```typescript
// core/[business-module]/[business-module].module.ts
@Module({
  imports: [
    CatalogModule, // Import internal module
  ],
})
export class BusinessModule {}

// Use case
@Injectable()
export class CreateEntityUseCase {
  constructor(
    private readonly getCatalog: GetCatalogUseCase, // Inject use case directly
  ) {}
}
```

---

## Recipe 5: Create Output Port with Symbol Token

**Goal:** Create a new repository port following Symbol token pattern.

**Step 1: Define Port Interface + Token**

```typescript
// application/ports/output/for-storing-users.port.ts
import { User } from '../../../domain/entities/user.entity';

export interface ForStoringUsers {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
}

// Symbol for dependency injection
export const FOR_STORING_USERS = Symbol('FOR_STORING_USERS');
```

**Step 2: Implement Adapter**

```typescript
// infrastructure/adapters/persistence/prisma-user.repository.ts
@Injectable()
export class PrismaUserRepository implements ForStoringUsers {
  constructor(private readonly prisma: PrismaService) {}

  async save(user: User): Promise<void> {
    await this.prisma.user.upsert({
      where: { id: user.getId() },
      create: { id: user.getId(), email: user.getEmail().toString() },
      update: { email: user.getEmail().toString() },
    });
  }

  async findById(id: string): Promise<User | null> {
    const row = await this.prisma.user.findUnique({ where: { id } });
    return row ? User.reconstitute(row.id, row.email) : null;
  }
}
```

**Step 3: Wire in Module**

```typescript
// user.module.ts
@Module({
  providers: [
    {
      provide: FOR_STORING_USERS,
      useClass: PrismaUserRepository,
    },
    CreateUserUseCase,
  ],
})
export class UserModule {}
```

**Step 4: Use in Use Case**

```typescript
@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(FOR_STORING_USERS)
    private readonly userStorage: ForStoringUsers,
  ) {}
}
```

---

**Navigation:** [Previous: Testing](./08-testing.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Decision Matrices](./10-decision-matrices.md)
