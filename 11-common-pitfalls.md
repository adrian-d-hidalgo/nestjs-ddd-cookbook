# 11. Common Pitfalls - Mistakes to Avoid

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

Frequent errors when implementing Hexagonal Architecture in NestJS and how to avoid them.

## 1. Anemic Domain Model

**❌ Error**: Models that only have getters/setters without behavior.

```typescript
// ❌ BAD - Anemic Model
export class User {
  id: string;
  email: string;
  status: string;

  getEmail() {
    return this.email;
  }
  setEmail(email: string) {
    this.email = email;
  }
}

// Logic in Use Case (WRONG!)
@Injectable()
export class UpdateUserUseCase {
  async execute(userId: string, newEmail: string) {
    const user = await this.repo.find(userId);
    if (!this.isValidEmail(newEmail)) throw new Error();
    user.setEmail(newEmail); // ❌ Logic outside domain
    await this.repo.save(user);
  }
}
```

**✅ Correct**: Rich Domain Model with behavior.

```typescript
// ✅ GOOD - Rich Domain Model
export class User extends AggregateRoot {
  private constructor(
    private id: UserId,
    private email: Email,
    private status: UserStatus,
  ) {}

  changeEmail(newEmail: Email): void {
    if (!newEmail.isValid()) {
      throw new InvalidEmailError();
    }

    const oldEmail = this.email;
    this.email = newEmail;

    this.record(new UserEmailChangedEvent(this.id, oldEmail, newEmail));
  }
}

// Use Case only orchestrates
@Injectable()
export class UpdateUserUseCase {
  async execute(userId: string, newEmail: string) {
    const user = await this.repo.find(userId);
    user.changeEmail(new Email(newEmail)); // ✅ Logic in domain
    await this.repo.save(user); // ✅ Repository publishes events
  }
}
```

---

## 2. Technology Leakage in Domain/Application

**❌ Error**: Use Cases mention concrete technologies.

```typescript
// ❌ BAD
@Injectable()
export class CreateUserUseCase {
  constructor(
    private prismaService: PrismaService, // ❌ Prisma leak
    private sendGridClient: SendGridClient, // ❌ SendGrid leak
  ) {}

  async execute(data: CreateUserInput) {
    await this.prismaService.user.create({ data }); // ❌
    await this.sendGridClient.send({ template: 'welcome' }); // ❌
  }
}
```

**✅ Correct**: Use Ports (interfaces).

```typescript
// ✅ GOOD
@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(FOR_STORING_USERS) private userStorage: ForStoringUsers,
    @Inject(FOR_SENDING_EMAILS) private emailSender: ForSendingEmails,
  ) {}

  async execute(data: CreateUserInput) {
    const user = User.create(data);
    await this.userStorage.save(user);
    await this.emailSender.sendWelcome(user.getEmail());
  }
}
```

---

## 3. Using Wrong Exceptions in Domain/Application

**Rule**: ALWAYS use `DomainException` (never generic `Error` or NestJS exceptions) in domain/application layers.

**3 Levels where domain exceptions are thrown**:

### Level 1 - Value Objects (Constructor validation)

```typescript
// domain/value-objects/email.vo.ts
export class Email extends ValueObject<string> {
  constructor(value: string) {
    super(value);
    if (!this.isValid(value)) {
      throw new InvalidEmailError(value); // ✅ DomainException
    }
  }

  private isValid(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }
}

// domain/exceptions/invalid-email.error.ts
export class InvalidEmailError extends DomainException {
  readonly code = 'INVALID_EMAIL';
  constructor(public readonly value: string) {
    super(`Invalid email: ${value}`);
  }
}
```

### Level 2 - Entities (Business rules)

```typescript
// domain/entities/user.entity.ts
export class User extends AggregateRoot {
  changeEmail(newEmail: Email): void {
    if (this.status === 'deleted') {
      throw new UserAlreadyDeletedException(this.id); // ✅ DomainException
    }
    this.email = newEmail;
    this.record(new UserEmailChangedEvent(this.id, newEmail));
  }
}

// domain/exceptions/user-already-deleted.error.ts
export class UserAlreadyDeletedException extends DomainException {
  readonly code = 'USER_ALREADY_DELETED';
  constructor(public readonly userId: string) {
    super(`User ${userId} is deleted`);
  }
}
```

### Level 3 - Use Cases (Orchestration errors)

```typescript
// application/use-cases/update-user/update-user.use-case.ts
@Injectable()
export class UpdateUserUseCase {
  async execute(input: UpdateUserInput) {
    const user = await this.userRepository.findById(input.userId);
    if (!user) {
      throw new UserNotFoundError(input.userId); // ✅ DomainException
    }
    user.changeEmail(new Email(input.email));
    await this.userRepository.save(user);
  }
}

// domain/exceptions/user-not-found.error.ts
export class UserNotFoundError extends DomainException {
  readonly code = 'USER_NOT_FOUND';
  constructor(public readonly userId: string) {
    super(`User ${userId} not found`);
  }
}
```

**Key points**:

- ❌ NEVER `throw new Error('message')` in domain/application
- ❌ NEVER use NestJS exceptions (`BadRequestException`, etc.) in domain/application
- ✅ ALWAYS extend `DomainException` from shared-kernel
- ✅ ALWAYS include `readonly code` property for error identification
- ✅ Use ExceptionFilter in presentation to convert to HTTP responses

---

## 4. NestJS Exceptions in Domain/Application

**❌ Error**: Using `@nestjs/common` exceptions outside Presentation.

```typescript
// ❌ BAD
import { BadRequestException } from '@nestjs/common';

@Injectable()
export class CreateUserUseCase {
  async execute(data: CreateUserInput) {
    if (await this.repo.exists(data.email)) {
      throw new BadRequestException('User already exists'); // ❌ NestJS in Application
    }
  }
}
```

**✅ Correct**: DomainException + ExceptionFilter.

```typescript
// ✅ domain/exceptions/user.exceptions.ts
export class UserAlreadyExistsError extends DomainException {
  readonly code = 'USER_ALREADY_EXISTS';
  constructor(public readonly email: string) {
    super(`User with email ${email} already exists`);
  }
}

// ✅ application/use-cases/create-user.use-case.ts
@Injectable()
export class CreateUserUseCase {
  async execute(data: CreateUserInput) {
    if (await this.repo.exists(data.email)) {
      throw new UserAlreadyExistsError(data.email); // ✅ Domain Exception
    }
  }
}

// ✅ presentation/http/filters/domain-exception.filter.ts
@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    const statusCode = this.mapExceptionToStatus(exception);

    response.status(statusCode).json({
      error: {
        code: exception.code,
        message: exception.message,
      },
    });
  }

  private mapExceptionToStatus(exception: DomainException): number {
    const mapping: Record<string, number> = {
      USER_NOT_FOUND: HttpStatus.NOT_FOUND,
      USER_ALREADY_EXISTS: HttpStatus.CONFLICT,
      INVALID_EMAIL: HttpStatus.BAD_REQUEST,
    };

    return mapping[exception.code] || HttpStatus.INTERNAL_SERVER_ERROR;
  }
}
```

---

## 5. Poorly Named Ports

**❌ Error**: Generic names or names that imply technology.

```typescript
// ❌ BAD
export const USER_REPOSITORY = Symbol('USER_REPOSITORY');
export interface IUserRepository {} // ❌ "I" prefix
export interface UserPersistence {} // ❌ Implies technology
export interface DatabasePort {} // ❌ Names technology
```

**✅ Correct**: Cockburn convention `For[Doing][Something]`.

```typescript
// ✅ GOOD
export const FOR_STORING_USERS = Symbol('FOR_STORING_USERS');
export interface ForStoringUsers {}

export const FOR_SENDING_EMAILS = Symbol('FOR_SENDING_EMAILS');
export interface ForSendingEmails {}
```

---

## 6. Over-engineering Simple CRUD

**❌ OVERKILL** for CRUD without business logic:

```
src/supporting/todo/
├── domain/
│   ├── entities/todo.entity.ts
│   ├── value-objects/todo-title.vo.ts
│   └── value-objects/todo-description.vo.ts
├── application/
│   ├── use-cases/create-todo.use-case.ts
│   ├── ports/for-storing-todos.port.ts
├── infrastructure/
│   └── adapters/prisma-todo.repository.ts
└── presentation/
    └── controllers/todo.controller.ts
```

**✅ For simple CRUD**, use standard NestJS:

```
src/supporting/todo/
├── todo.service.ts
├── todo.controller.ts
├── dto/create-todo.dto.ts
└── todo.module.ts
```

**Rule**: Use hexagonal **only** when module meets **≥3 criteria** from Getting Started checklist.

---

## 7. Mixing Business Logic in Controllers

**❌ Error**: Business rules in controllers.

```typescript
// ❌ BAD
@Controller('users')
export class UserController {
  @Post()
  async create(@Body() dto: CreateUserDto) {
    if (!dto.email.includes('@')) {
      throw new BadRequestException('Invalid email'); // ❌ Validation in controller
    }

    if (dto.age < 18) {
      throw new BadRequestException('Must be adult'); // ❌ Business rule in controller
    }

    // ...
  }
}
```

**✅ Correct**: Move logic to domain.

```typescript
// ✅ Controller only orchestrates
@Controller('users')
export class UserController {
  @Post()
  async create(@Body() dto: CreateUserDto) {
    const input = { email: dto.email, age: dto.age };
    const output = await this.createUser.execute(input);
    return new UserResponse(output);
  }
}

// ✅ Validation in Value Object
export class Email {
  constructor(value: string) {
    if (!value.includes('@')) {
      throw new InvalidEmailError(value);
    }
  }
}

// ✅ Business rule in Entity
export class User {
  static create(email: Email, age: Age): User {
    if (age.getValue() < 18) {
      throw new UserMustBeAdultError(age.getValue());
    }
    // ...
  }
}
```

---

## 8. Forgetting to Extract and Publish Events

**❌ Error**: Not publishing events from repository.

```typescript
// ❌ BAD - Events never published
async save(user: User): Promise<void> {
  await this.prisma.user.upsert({
    where: { id: user.getId() },
    create: { id: user.getId(), email: user.getEmail().toString() },
    update: { email: user.getEmail().toString() },
  });
  // ❌ Events accumulate but never published
}
```

**✅ Correct**: Extract and publish events.

```typescript
// ✅ GOOD - Events extracted and published
async save(user: User): Promise<void> {
  // 1. Persist
  await this.prisma.user.upsert({
    where: { id: user.getId() },
    create: { id: user.getId(), email: user.getEmail().toString() },
    update: { email: user.getEmail().toString() },
  });

  // 2. Extract events atomically
  const events = user.pullEvents();

  // 3. Publish AFTER successful persistence
  if (events.length > 0) {
    await this.eventBus.publish(events);
  }
}
```

---

**Navigation:** [Previous: Decision Matrices](./10-decision-matrices.md) | [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Execution Flows](./12-execution-flows.md)
