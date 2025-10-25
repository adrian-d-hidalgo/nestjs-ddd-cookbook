# 15. Implementation Checklist - Module Compliance Verification

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

Exhaustive checklist to verify that a module implementation follows Hexagonal Architecture and DDD principles at 100%.

---

## How to Use This Checklist

**For New Modules:**
- Use this as a **pre-implementation guide** to ensure all patterns are correctly applied
- Check each item as you implement the feature

**For Existing Modules:**
- Use this for **code reviews** to identify missing patterns or violations
- Refactor violations incrementally using the provided solutions

**Severity Levels:**
- 🔴 **CRITICAL** - Must fix immediately (breaks architecture principles)
- 🟡 **WARNING** - Should fix soon (technical debt)
- 🟢 **BEST PRACTICE** - Recommended for production-grade code

---

## 1. Domain Layer Compliance

### 1.1 Value Objects

#### Structure & Location
- [ ] 🔴 Value Objects located in `domain/value-objects/`
- [ ] 🔴 NO `@Injectable()` decorator on Value Objects
- [ ] 🔴 NO imports from `@nestjs/*`, `typeorm`, `prisma`, or any infrastructure library
- [ ] 🔴 Value Objects are **immutable** (all properties `readonly`)

#### Implementation
- [ ] 🔴 Validation logic in constructor (fails fast with `DomainException`)
- [ ] 🔴 Value Objects extend base class (`ValueObject<T>`) for consistency
- [ ] 🔴 `equals()` method implemented for value comparison
- [ ] 🟢 Static factory method `create()` for instantiation
- [ ] 🟡 Getter methods return values, never `undefined` (fail in constructor instead)

**Example:**
```typescript
// ✅ GOOD
export class Email extends ValueObject<string> {
  private constructor(private readonly value: string) {
    super(value);
    if (!this.isValid(value)) {
      throw new InvalidEmailError(value); // DomainException
    }
  }

  static create(value: string): Email {
    return new Email(value);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  private isValid(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }
}

// ❌ BAD
export class Email {
  value?: string; // ❌ Not readonly, nullable

  constructor(value?: string) { // ❌ Allows undefined
    this.value = value;
  }
  // ❌ No validation, no equals()
}
```

---

### 1.2 Entities

#### Structure & Location
- [ ] 🔴 Entities located in `domain/entities/`
- [ ] 🔴 NO `@Injectable()` decorator on Entities
- [ ] 🔴 NO infrastructure imports (`typeorm`, `prisma`, `@nestjs/typeorm`)
- [ ] 🔴 NO ORM decorators (`@Entity`, `@Column`, `@PrimaryKey`)

#### Implementation
- [ ] 🔴 Entities extend `Entity<IdType>` base class
- [ ] 🔴 Private constructor (use static factory methods)
- [ ] 🔴 Properties are **private** with getter methods
- [ ] 🔴 Business rules enforced in methods (NOT in Use Cases)
- [ ] 🟢 Entities expose behavior methods, not just getters/setters
- [ ] 🟡 NO business logic leaking into Application or Presentation layers

**Example:**
```typescript
// ✅ GOOD - Rich Domain Entity
export class User extends Entity<UserId> {
  private email: Email;
  private status: UserStatus;

  private constructor(id: UserId, email: Email, status: UserStatus) {
    super(id);
    this.email = email;
    this.status = status;
  }

  static create(id: string, email: string): User {
    return new User(
      UserId.create(id),
      Email.create(email),
      UserStatus.ACTIVE,
    );
  }

  changeEmail(newEmail: Email): void {
    if (this.status === UserStatus.SUSPENDED) {
      throw new CannotChangeEmailForSuspendedUserError();
    }
    this.email = newEmail;
  }

  getEmail(): Email {
    return this.email;
  }
}

// ❌ BAD - Anemic Entity
export class User {
  id: string;
  email: string; // ❌ Public, primitive types
  status: string;

  setEmail(email: string) { // ❌ No validation
    this.email = email;
  }
}
```

---

### 1.3 Aggregates

#### Structure & Location
- [ ] 🔴 Aggregates located in `domain/aggregates/` or `domain/entities/`
- [ ] 🔴 Aggregates extend `AggregateRoot<IdType>` base class
- [ ] 🔴 Aggregate enforces consistency boundaries (all changes via root)
- [ ] 🔴 Child entities accessed ONLY through aggregate root

#### Domain Events
- [ ] 🔴 Aggregates emit Domain Events for state changes
- [ ] 🔴 Events added via `this.record(event)` or `this.addDomainEvent(event)`
- [ ] 🔴 Events are immutable (all properties `readonly`)
- [ ] 🟡 Events named in past tense (`UserEmailChangedEvent`, not `ChangeUserEmailEvent`)
- [ ] 🟢 Events contain only essential data (IDs, changed values, timestamp)

**Example:**
```typescript
// ✅ GOOD - Aggregate with Events
export class Order extends AggregateRoot<OrderId> {
  private items: OrderItem[] = [];
  private status: OrderStatus;

  addItem(product: Product, quantity: number): void {
    if (this.status !== OrderStatus.DRAFT) {
      throw new CannotModifyPlacedOrderError();
    }

    const item = OrderItem.create(product, quantity);
    this.items.push(item);

    this.record(new ItemAddedToOrderEvent(this.id, item.id, quantity));
  }

  place(): void {
    if (this.items.length === 0) {
      throw new CannotPlaceEmptyOrderError();
    }

    this.status = OrderStatus.PLACED;
    this.record(new OrderPlacedEvent(this.id, this.getTotalAmount()));
  }

  private getTotalAmount(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.getSubtotal()),
      Money.zero(),
    );
  }
}

// ❌ BAD - Direct child manipulation
export class Order extends AggregateRoot<OrderId> {
  items: OrderItem[] = []; // ❌ Public

  // ❌ No events emitted
  // ❌ No business rules
}
```

**Critical Check - Overuse of Aggregates:**
- [ ] 🟡 Are you using Aggregates **only when needed**?
- [ ] 🟡 Simple entities without children should be `Entity`, not `AggregateRoot`
- [ ] 🟡 Aggregates used when consistency boundaries or events are required

**Example - When NOT to use Aggregate:**
```typescript
// ❌ BAD - Aggregate without need
export class Product extends AggregateRoot<ProductId> {
  private name: ProductName;
  private price: Money;
  // ❌ No children, no events, no consistency boundary
}

// ✅ GOOD - Simple Entity
export class Product extends Entity<ProductId> {
  private name: ProductName;
  private price: Money;
}
```

---

### 1.4 Domain Services

#### When to Use
- [ ] 🟡 Domain Service used ONLY when logic doesn't fit in a single Entity/Aggregate
- [ ] 🟡 Domain Service represents **business policy** or **cross-entity calculation**
- [ ] 🟡 NO orchestration, transactions, or external calls in Domain Services

#### Implementation
- [ ] 🔴 Domain Services located in `domain/services/`
- [ ] 🔴 Domain Services are `@Injectable()`
- [ ] 🔴 Domain Services depend ONLY on domain objects (Entities, VOs, Repository Interfaces)
- [ ] 🔴 NO direct database access or external API calls
- [ ] 🔴 Domain Services do NOT emit events (Aggregates emit events)

**Example:**
```typescript
// ✅ GOOD - Pure Domain Service
@Injectable()
export class DiscountPolicyService {
  calculateDiscount(customer: Customer, cart: Cart): Money {
    // Pure business logic
    const volumeDiscount = this.calculateVolumeDiscount(cart);
    const loyaltyBonus = customer.getLoyaltyPoints().toMoney();
    return volumeDiscount.add(loyaltyBonus);
  }

  private calculateVolumeDiscount(cart: Cart): Money {
    // Business rules
  }
}

// ❌ BAD - This is an Application Service, NOT Domain Service
@Injectable()
export class OrderService {
  constructor(
    private prisma: PrismaService, // ❌ Infrastructure
    private emailSender: EmailService, // ❌ External
  ) {}

  async createOrder(data) {
    return this.prisma.$transaction(async () => { // ❌ Transaction
      const order = await this.prisma.order.create(...); // ❌ Persistence
      await this.emailSender.send(...); // ❌ External call
    });
  }
}
```

---

### 1.5 Domain Exceptions

#### Implementation
- [ ] 🔴 All exceptions in domain extend `DomainException`
- [ ] 🔴 Domain exceptions located in `domain/exceptions/`
- [ ] 🔴 NEVER use generic `Error` or NestJS exceptions in domain
- [ ] 🟢 Exception has `code` property for programmatic handling
- [ ] 🟢 Exception message is human-readable

**Example:**
```typescript
// ✅ GOOD
export class UserAlreadyExistsError extends DomainException {
  readonly code = 'USER_ALREADY_EXISTS';
  constructor(public readonly email: string) {
    super(`User with email ${email} already exists`);
  }
}

// ❌ BAD
throw new Error('User exists'); // ❌ Generic Error
throw new BadRequestException('User exists'); // ❌ NestJS exception
```

---

## 2. Application Layer Compliance

### 2.1 Use Cases

#### Structure & Location
- [ ] 🔴 Use Cases located in `application/use-cases/[action-name]/`
- [ ] 🔴 Each Use Case has dedicated folder with `.use-case.ts`, `.input.ts`, `.output.ts`, `.spec.ts`
- [ ] 🔴 Use Cases are `@Injectable()`

#### Responsibilities
- [ ] 🔴 Use Case orchestrates domain objects (NO business logic in Use Case)
- [ ] 🔴 Use Case depends on Ports (interfaces), NOT concrete adapters
- [ ] 🔴 Use Case uses `@Inject(TOKEN)` for port injection
- [ ] 🔴 NO infrastructure imports (`prisma`, `typeorm`, `sendgrid`, etc.)

#### Events
- [ ] 🔴 Use Case publishes domain events from aggregates
- [ ] 🔴 Events published via EventBus or UnitOfWork pattern
- [ ] 🟢 Use Case publishes events AFTER successful persistence

**Example:**
```typescript
// ✅ GOOD
@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(FOR_STORING_USERS) private userStorage: ForStoringUsers,
    @Inject(FOR_PUBLISHING_EVENTS) private eventBus: ForPublishingEvents,
  ) {}

  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    // 1. Check existence
    const existing = await this.userStorage.findByEmail(
      Email.create(input.email),
    );
    if (existing) {
      throw new UserAlreadyExistsError(input.email);
    }

    // 2. Create domain object (business logic in entity)
    const user = User.create(input.id, input.email);

    // 3. Persist
    await this.userStorage.save(user);

    // 4. Publish events
    await this.eventBus.publishAll(user.getUncommittedEvents());
    user.clearEvents();

    // 5. Return output
    return {
      id: user.getId().value,
      email: user.getEmail().value,
      status: 'active',
    };
  }
}

// ❌ BAD
@Injectable()
export class CreateUserUseCase {
  constructor(private prisma: PrismaService) {} // ❌ Concrete dependency

  async execute(input: CreateUserInput) {
    // ❌ Business logic in Use Case
    if (!input.email.includes('@')) {
      throw new Error('Invalid email');
    }

    // ❌ Direct persistence, no events
    return this.prisma.user.create({ data: input });
  }
}
```

---

### 2.2 Input/Output Contracts

#### Input (Command/Query)
- [ ] 🔴 Input is a **class** (not interface or type)
- [ ] 🔴 Input validates data in constructor or factory method
- [ ] 🔴 Input properties are `readonly`
- [ ] 🟢 Input located in `application/use-cases/[action]/[action].input.ts`

#### Output
- [ ] 🔴 Output is a **type** (plain object, not class)
- [ ] 🔴 Output contains ONLY primitives and plain objects (NO domain classes)
- [ ] 🟢 Output located in `application/use-cases/[action]/[action].output.ts`

**Example:**
```typescript
// ✅ GOOD
export class CreateUserInput {
  readonly id: string;
  readonly email: string;

  constructor(data: { id: string; email: string }) {
    if (!data.id || !data.email) {
      throw new ValidationException('Missing required fields');
    }
    this.id = data.id;
    this.email = data.email;
  }
}

export type CreateUserOutput = {
  id: string;
  email: string;
  status: string;
};

// ❌ BAD
export interface CreateUserInput { // ❌ Interface, no validation
  id: string;
  email: string;
}

export class CreateUserOutput { // ❌ Class, should be type
  constructor(public user: User) {} // ❌ Domain object exposed
}
```

---

### 2.3 Ports (Interfaces)

#### Output Ports (Driven Adapters)
- [ ] 🔴 Output ports located in `application/ports/output/`
- [ ] 🔴 Ports are **interfaces** (NOT classes)
- [ ] 🔴 Ports use domain types (Entities, VOs), NOT primitives or ORM types
- [ ] 🔴 Port has injection token constant (`FOR_STORING_USERS`)
- [ ] 🟢 Port method names follow naming convention: `ForDoingAction`

**Example:**
```typescript
// ✅ GOOD
// application/ports/output/for-storing-users.port.ts
export const FOR_STORING_USERS = Symbol('FOR_STORING_USERS');

export interface ForStoringUsers {
  save(user: User): Promise<void>;
  findById(id: UserId): Promise<User | null>;
  findByEmail(email: Email): Promise<User | null>;
}

// ❌ BAD
export interface UserRepository { // ❌ No token
  save(user: PrismaUser): Promise<void>; // ❌ ORM type
  findById(id: string): Promise<PrismaUser>; // ❌ Primitive + ORM
}
```

#### Input Ports (Optional)
- [ ] 🟢 Input ports located in `application/ports/input/` (if using port-adapter pattern for controllers)
- [ ] 🟢 Input port interface matches Use Case public API

---

## 3. Infrastructure Layer Compliance

### 3.1 Repositories

#### Structure & Location
- [ ] 🔴 Repository implementations in `infrastructure/persistence/[technology]/`
- [ ] 🔴 Repository implements port interface from `application/ports/output/`
- [ ] 🔴 Repository is `@Injectable()`
- [ ] 🔴 Repository registered with DI using port token

#### Implementation
- [ ] 🔴 Repository uses **Mappers** to convert between ORM and Domain objects
- [ ] 🔴 Repository NEVER returns ORM entities (always maps to domain)
- [ ] 🔴 Repository NEVER accepts ORM entities as parameters (always domain objects)
- [ ] 🟢 Repository handles database errors and converts to domain exceptions

**Example:**
```typescript
// ✅ GOOD
@Injectable()
export class TypeOrmUserRepository implements ForStoringUsers {
  constructor(
    @InjectRepository(UserOrmEntity)
    private readonly ormRepo: Repository<UserOrmEntity>,
  ) {}

  async save(user: User): Promise<void> {
    const ormEntity = UserMapper.toOrm(user); // ✅ Map domain → ORM
    await this.ormRepo.save(ormEntity);
  }

  async findById(id: UserId): Promise<User | null> {
    const ormEntity = await this.ormRepo.findOne({
      where: { id: id.value }
    });
    return ormEntity ? UserMapper.toDomain(ormEntity) : null; // ✅ Map ORM → domain
  }
}

// ❌ BAD
@Injectable()
export class UserRepository {
  async save(user: UserOrmEntity): Promise<void> { // ❌ ORM type
    await this.ormRepo.save(user);
  }

  async findById(id: string): Promise<UserOrmEntity> { // ❌ Returns ORM
    return this.ormRepo.findOne({ where: { id } });
  }
}
```

---

### 3.2 ORM Entities & Mappers

#### ORM Entities
- [ ] 🔴 ORM entities in `infrastructure/persistence/[technology]/[entity]-orm.entity.ts`
- [ ] 🔴 ORM entities named `[Name]OrmEntity` (not just `[Name]Entity`)
- [ ] 🔴 ORM entities are **separate** from domain entities
- [ ] 🔴 ORM decorators (`@Entity`, `@Column`) ONLY on ORM entities

#### Mappers
- [ ] 🔴 Mappers in `infrastructure/persistence/[technology]/[entity].mapper.ts`
- [ ] 🔴 Mapper has `toDomain()` method (ORM → Domain)
- [ ] 🔴 Mapper has `toOrm()` method (Domain → ORM)
- [ ] 🟢 Mapper handles nested objects (VOs, child entities)

**Example:**
```typescript
// ✅ GOOD
// infrastructure/persistence/typeorm/user-orm.entity.ts
@Entity('users')
export class UserOrmEntity {
  @PrimaryColumn()
  id: string;

  @Column()
  email: string;

  @Column()
  status: string;
}

// infrastructure/persistence/typeorm/user.mapper.ts
export class UserMapper {
  static toDomain(orm: UserOrmEntity): User {
    return User.reconstitute(
      UserId.create(orm.id),
      Email.create(orm.email),
      UserStatus[orm.status],
    );
  }

  static toOrm(domain: User): UserOrmEntity {
    const orm = new UserOrmEntity();
    orm.id = domain.getId().value;
    orm.email = domain.getEmail().value;
    orm.status = domain.getStatus();
    return orm;
  }
}

// ❌ BAD - Domain entity with ORM decorators
@Entity('users') // ❌ ORM decorator in domain
export class User {
  @PrimaryColumn() // ❌
  id: string;

  @Column() // ❌
  email: string;
}
```

---

### 3.3 Event Handlers

#### Structure & Location
- [ ] 🔴 Event handlers in `infrastructure/events/handlers/`
- [ ] 🔴 Event handlers are `@Injectable()`
- [ ] 🔴 Event handlers use `@OnEvent()` decorator or NestJS EventEmitter pattern

#### Implementation
- [ ] 🔴 Event handlers call external services (email, notifications, etc.)
- [ ] 🔴 Event handlers do NOT modify aggregates directly
- [ ] 🟢 Event handlers are **idempotent** (safe to retry)
- [ ] 🟢 Event handlers use Inbox pattern for exactly-once processing (production)

**Example:**
```typescript
// ✅ GOOD
@Injectable()
export class SendWelcomeEmailHandler {
  constructor(
    @Inject(FOR_SENDING_EMAILS) private emailSender: ForSendingEmails,
  ) {}

  @OnEvent('user.created')
  async handle(event: UserCreatedEvent): Promise<void> {
    await this.emailSender.sendWelcome(event.email);
  }
}

// ❌ BAD
@Injectable()
export class UserCreatedHandler {
  constructor(
    private userRepo: UserRepository, // ❌ Modifying aggregate
    private sendGrid: SendGridClient, // ❌ Concrete dependency
  ) {}

  async handle(event: UserCreatedEvent) {
    const user = await this.userRepo.findById(event.userId);
    user.status = 'emailed'; // ❌ Direct modification
    await this.sendGrid.send(...); // ❌ No abstraction
  }
}
```

---

### 3.4 External Service Adapters

#### Structure & Location
- [ ] 🔴 Adapters in `infrastructure/external/[service-name]/`
- [ ] 🔴 Adapters implement port interfaces from `application/ports/output/`
- [ ] 🔴 Adapters are `@Injectable()`

#### Implementation
- [ ] 🔴 Adapter translates between domain types and external API types
- [ ] 🟢 Adapter handles external API errors and converts to domain exceptions
- [ ] 🟢 Adapter has retry logic for transient failures

**Example:**
```typescript
// ✅ GOOD
@Injectable()
export class SendGridEmailAdapter implements ForSendingEmails {
  constructor(private sendGridClient: SendGridClient) {}

  async sendWelcome(email: Email): Promise<void> {
    try {
      await this.sendGridClient.send({
        to: email.value, // ✅ Extract primitive
        templateId: 'welcome-template',
      });
    } catch (error) {
      throw new EmailDeliveryFailedError(email.value); // ✅ Domain exception
    }
  }
}

// ❌ BAD
@Injectable()
export class EmailService {
  async send(email: string) { // ❌ Primitive, not domain type
    await this.sendGridClient.send({ to: email });
    // ❌ No error handling
  }
}
```

---

## 4. Presentation Layer Compliance

### 4.1 Controllers

#### Structure & Location
- [ ] 🔴 Controllers in `presentation/controllers/`
- [ ] 🔴 Controllers are `@Controller()`
- [ ] 🔴 Controllers depend ONLY on Use Cases (via `@Inject`)

#### Responsibilities
- [ ] 🔴 Controllers translate HTTP DTOs to Use Case Inputs
- [ ] 🔴 Controllers translate Use Case Outputs to HTTP DTOs
- [ ] 🔴 Controllers handle HTTP-specific concerns (status codes, headers)
- [ ] 🔴 Controllers do NOT contain business logic
- [ ] 🟢 Controllers use Guards for authentication/authorization

**Example:**
```typescript
// ✅ GOOD
@Controller('users')
export class UserController {
  constructor(
    @Inject(CreateUserUseCase) private createUser: CreateUserUseCase,
  ) {}

  @Post()
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    const input = new CreateUserInput({
      id: randomUUID(),
      email: dto.email,
    });

    const output = await this.createUser.execute(input);

    return {
      id: output.id,
      email: output.email,
      status: output.status,
    };
  }
}

// ❌ BAD
@Controller('users')
export class UserController {
  constructor(private prisma: PrismaService) {} // ❌ Direct DB

  @Post()
  async create(@Body() dto: CreateUserDto) {
    // ❌ Business logic in controller
    if (!dto.email.includes('@')) {
      throw new Error('Invalid email');
    }
    return this.prisma.user.create({ data: dto });
  }
}
```

---

### 4.2 DTOs (Data Transfer Objects)

#### Implementation
- [ ] 🔴 DTOs in `presentation/dtos/`
- [ ] 🔴 DTOs are classes (for validation decorators)
- [ ] 🔴 DTOs use `class-validator` decorators (`@IsString`, `@IsEmail`)
- [ ] 🔴 DTOs contain ONLY primitives and plain objects
- [ ] 🔴 DTOs are separate from domain objects and Use Case inputs

**Example:**
```typescript
// ✅ GOOD
export class CreateUserDto {
  @IsEmail()
  readonly email: string;

  @IsString()
  @MinLength(8)
  readonly password: string;
}

// ❌ BAD
export class CreateUserDto {
  email: string; // ❌ No validation
  user: User; // ❌ Domain object
}
```

---

### 4.3 Exception Filters

#### Implementation
- [ ] 🔴 Global exception filter registered in `main.ts`
- [ ] 🔴 Filter catches `DomainException` and maps to appropriate HTTP status
- [ ] 🟢 Filter logs errors with correlation IDs

**Example:**
```typescript
// ✅ GOOD
@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    const statusCode = this.mapToHttpStatus(exception);

    response.status(statusCode).json({
      statusCode,
      message: exception.message,
      code: exception.code,
      timestamp: new Date().toISOString(),
    });
  }

  private mapToHttpStatus(exception: DomainException): number {
    if (exception instanceof ValidationException) return 400;
    if (exception instanceof NotFoundException) return 404;
    if (exception instanceof ConflictException) return 409;
    return 500;
  }
}
```

---

## 5. Module Configuration Compliance

### 5.1 NestJS Module

#### Structure
- [ ] 🔴 Module file at module root: `[module-name].module.ts`
- [ ] 🔴 Module imports infrastructure modules (TypeORM, Event)
- [ ] 🔴 Module registers ports with `useClass` binding

#### Dependency Injection
- [ ] 🔴 Repositories bound to port tokens
- [ ] 🔴 External adapters bound to port tokens
- [ ] 🔴 Use Cases registered as providers
- [ ] 🔴 Controllers registered

**Example:**
```typescript
// ✅ GOOD
@Module({
  imports: [
    TypeOrmModule.forFeature([UserOrmEntity]),
    EventEmitterModule,
  ],
  providers: [
    // Use Cases
    CreateUserUseCase,
    GetUserUseCase,

    // Repositories
    {
      provide: FOR_STORING_USERS,
      useClass: TypeOrmUserRepository,
    },

    // External Services
    {
      provide: FOR_SENDING_EMAILS,
      useClass: SendGridEmailAdapter,
    },

    // Domain Services
    UserPolicyService,
  ],
  controllers: [UserController],
  exports: [FOR_STORING_USERS], // Export for other modules
})
export class UserModule {}

// ❌ BAD
@Module({
  providers: [UserService], // ❌ Generic service, no ports
  controllers: [UserController],
})
export class UserModule {}
```

---

## 6. Testing Compliance

### 6.1 Domain Layer Tests

- [ ] 🔴 Every Value Object has unit tests
- [ ] 🔴 Every Entity has unit tests
- [ ] 🔴 Every Aggregate has unit tests for invariants and events
- [ ] 🔴 Domain tests use NO mocks (pure TypeScript)
- [ ] 🟢 Domain tests cover edge cases and error conditions

**Example:**
```typescript
// ✅ GOOD
describe('User Entity', () => {
  it('should emit UserEmailChangedEvent when email changes', () => {
    const user = User.create('123', 'old@example.com');
    user.changeEmail(Email.create('new@example.com'));

    const events = user.getUncommittedEvents();
    expect(events).toHaveLength(1);
    expect(events[0]).toBeInstanceOf(UserEmailChangedEvent);
  });

  it('should throw when changing email for suspended user', () => {
    const user = User.create('123', 'user@example.com');
    user.suspend();

    expect(() => {
      user.changeEmail(Email.create('new@example.com'));
    }).toThrow(CannotChangeEmailForSuspendedUserError);
  });
});
```

---

### 6.2 Application Layer Tests

- [ ] 🔴 Every Use Case has unit tests
- [ ] 🔴 Use Case tests mock ports (NOT concrete adapters)
- [ ] 🔴 Use Case tests verify event publishing
- [ ] 🟢 Use Case tests cover error scenarios

**Example:**
```typescript
// ✅ GOOD
describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;
  let userStorage: jest.Mocked<ForStoringUsers>;
  let eventBus: jest.Mocked<ForPublishingEvents>;

  beforeEach(() => {
    userStorage = {
      save: jest.fn(),
      findByEmail: jest.fn(),
    } as any;

    eventBus = {
      publishAll: jest.fn(),
    } as any;

    useCase = new CreateUserUseCase(userStorage, eventBus);
  });

  it('should create user and publish event', async () => {
    userStorage.findByEmail.mockResolvedValue(null);

    await useCase.execute(new CreateUserInput({ id: '123', email: 'user@example.com' }));

    expect(userStorage.save).toHaveBeenCalledWith(
      expect.objectContaining({ id: expect.any(UserId) }),
    );
    expect(eventBus.publishAll).toHaveBeenCalledWith(
      expect.arrayContaining([expect.any(UserCreatedEvent)]),
    );
  });
});
```

---

### 6.3 Integration Tests

- [ ] 🟢 Repository integration tests use real database (Testcontainers)
- [ ] 🟢 External adapter tests use mock servers or sandbox APIs
- [ ] 🟢 Integration tests verify mappers work correctly

---

### 6.4 E2E Tests

- [ ] 🟢 E2E tests cover main user flows
- [ ] 🟢 E2E tests use test database
- [ ] 🟢 E2E tests verify events are published

---

## 7. Advanced Patterns Compliance (Production-Grade)

### 7.1 Outbox Pattern

- [ ] 🟢 Outbox table exists for transactional event publishing
- [ ] 🟢 Events stored in outbox atomically with aggregate persistence
- [ ] 🟢 Background worker publishes events from outbox
- [ ] 🟢 Published events marked as processed

---

### 7.2 Inbox Pattern

- [ ] 🟢 Inbox table exists for idempotent event handling
- [ ] 🟢 Event handlers check inbox before processing
- [ ] 🟢 Duplicate events are skipped

---

### 7.3 Sagas

- [ ] 🟢 Saga orchestrates multi-step transactions
- [ ] 🟢 Saga has compensation logic for failures
- [ ] 🟢 Saga state persisted between steps

---

## 8. Folder Structure Compliance

### 8.1 Complete Module Structure

- [ ] 🔴 Module follows this structure:

```
src/modules/[module-name]/
├── domain/
│   ├── aggregates/
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   ├── exceptions/
│   ├── services/
│   └── ports/              # Repository interfaces (optional location)
├── application/
│   ├── use-cases/
│   │   └── [action-name]/
│   │       ├── [action].use-case.ts
│   │       ├── [action].input.ts
│   │       ├── [action].output.ts
│   │       └── [action].use-case.spec.ts
│   └── ports/
│       ├── input/          # Use case interfaces (optional)
│       └── output/         # Repository/external service interfaces
├── infrastructure/
│   ├── persistence/
│   │   └── [orm-name]/
│   │       ├── [entity]-orm.entity.ts
│   │       ├── [entity].mapper.ts
│   │       └── [entity].repository.ts
│   ├── events/
│   │   └── handlers/
│   └── external/
│       └── [service-name]/
└── presentation/
    ├── controllers/
    ├── dtos/
    └── mappers/
```

---

## 9. Common Anti-Patterns Checklist

### 9.1 Detect Violations

Run through this list to catch common mistakes:

- [ ] 🔴 NO `import { PrismaService } from '@nestjs/prisma'` in Use Cases
- [ ] 🔴 NO `@Entity()` or `@Column()` decorators in `domain/` folder
- [ ] 🔴 NO business logic in Controllers
- [ ] 🔴 NO `throw new Error()` in domain layer (use `DomainException`)
- [ ] 🔴 NO public setters in Entities (`user.status = 'active'` is forbidden)
- [ ] 🔴 NO ORM entities returned from repositories (must use Mappers)
- [ ] 🔴 NO Use Cases calling other Use Cases directly (use Domain Services or Events)
- [ ] 🟡 NO aggregates without events (if it's just data, use Entity)
- [ ] 🟡 NO Value Objects with `@Injectable()` decorator

---

## 10. Compliance Score

**How to calculate:**

1. Count total items applicable to your module
2. Count items checked ✅
3. Calculate: `(Checked / Total) × 100 = Compliance %`

**Target Scores:**
- **New modules:** 95-100% (enforce all 🔴 CRITICAL items)
- **Legacy refactoring:** Start with 70%, improve incrementally
- **Production-ready:** 100% on 🔴 items, 80%+ on 🟡 and 🟢 items

---

## 11. Quick Audit Commands

Run these to detect violations automatically:

```bash
# Find ORM decorators in domain layer
grep -r "@Entity\|@Column\|@PrimaryColumn" src/modules/*/domain/

# Find infrastructure imports in domain/application
grep -r "from '@nestjs/typeorm'\|from 'typeorm'\|from '@prisma" src/modules/*/domain/ src/modules/*/application/

# Find generic Error throws in domain
grep -r "throw new Error" src/modules/*/domain/

# Find public properties in entities (simple heuristic)
grep -r "^\s*public\s" src/modules/*/domain/entities/

# Find Use Cases without @Inject decorators
grep -L "@Inject" src/modules/*/application/use-cases/**/*.use-case.ts
```

---

## 12. Migration Plan for Non-Compliant Modules

If your module fails compliance:

### Phase 1: Critical Fixes (Week 1)
1. Extract ORM decorators from domain entities → Create ORM entities
2. Create Mappers for ORM ↔ Domain conversion
3. Replace concrete dependencies with ports in Use Cases
4. Move business logic from Use Cases to Entities

### Phase 2: Event Compliance (Week 2)
5. Add domain events to Aggregates
6. Implement event publishing in Use Cases
7. Create event handlers in infrastructure

### Phase 3: Testing & Quality (Week 3)
8. Write domain unit tests
9. Write Use Case tests with mocked ports
10. Add integration tests for repositories

### Phase 4: Production Patterns (Week 4+)
11. Implement Outbox pattern
12. Implement Inbox pattern
13. Add Sagas for multi-step processes

---

## Summary

This checklist ensures **100% compliance** with Hexagonal Architecture and DDD principles in NestJS.

**Key Takeaways:**
- Domain layer must be **pure TypeScript** (no framework dependencies)
- Use Cases orchestrate, Entities enforce business rules
- Aggregates emit events for state changes (not on every entity)
- Repositories use Mappers to keep ORM out of domain
- All external dependencies injected via Ports

**Next Steps:**
- Print this checklist and review during code reviews
- Use severity levels to prioritize fixes
- Aim for 95%+ compliance on new modules

---

**Navigation:**
- [← Previous: Advanced Patterns](14-advanced-patterns.md)
- [Table of Contents](README.md#table-of-contents)
