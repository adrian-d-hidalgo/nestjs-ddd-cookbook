# NestJS DDD Cookbook

> Practical guide for implementing Hexagonal Architecture and Domain-Driven Design in NestJS projects

**Purpose:** Provide clear, actionable patterns for building maintainable NestJS applications with hexagonal architecture and DDD principles.

**Key Benefits:**

- Switch infrastructure (Prisma → TypeORM) without touching domain
- Test business logic without databases
- Microservices-ready from day one
- Portable domain between frameworks

---

## 📚 Documentation Structure

This cookbook is divided into focused sections for easy navigation:

### Getting Started
- **[01. Getting Started](./01-getting-started.md)** - When to use this architecture, trade-offs, and decision criteria

### Architecture & Structure
- **[02. Project Structure](./02-project-structure.md)** - DDD folder organization and module anatomy

### Layer-by-Layer Implementation
- **[03. Domain Layer](./03-domain-layer.md)** - Entities, Value Objects, Events, Domain Services
- **[04. Application Layer](./04-application-layer.md)** - Use Cases, Ports (Input/Output)
- **[05. Infrastructure Layer](./05-infrastructure-layer.md)** - Adapters, Repositories, NestJS Modules
- **[06. Presentation Layer](./06-presentation-layer.md)** - Controllers, DTOs, Guards, Filters

### Shared Components
- **[07. Shared Kernel](./07-shared-kernel.md)** - Reusable components across all modules

### Testing
- **[08. Testing Strategy](./08-testing.md)** - Complete testing guide (Unit, Integration, E2E)

### Practical Guides
- **[09. Implementation Recipes](./09-recipes.md)** - Step-by-step recipes for common scenarios
- **[10. Decision Matrices](./10-decision-matrices.md)** - When to use each pattern

### Advanced Topics
- **[13. Domain Services & Use Cases](./13-domain-services-and-use-cases.md)** - Orchestration patterns and responsibilities
- **[14. Advanced Patterns](./14-advanced-patterns.md)** - Outbox, Inbox, Sagas for production-grade event handling

### Troubleshooting & Quality
- **[11. Common Pitfalls](./11-common-pitfalls.md)** - Mistakes to avoid with solutions
- **[12. Execution Flows](./12-execution-flows.md)** - Diagrams showing request flows (HTTP, Events, Outbox, Sagas)
- **[15. Implementation Checklist](./15-implementation-checklist.md)** - Complete compliance verification for modules

---

## 🚀 Quick Start

**New to this architecture?**
1. Start with [Getting Started](./01-getting-started.md) to understand when to use it
2. Read [Project Structure](./02-project-structure.md) to understand the organization
3. Follow [Implementation Recipes](./09-recipes.md) for step-by-step guides

**Implementing a feature?**
1. Check [Implementation Recipes](./09-recipes.md) for your scenario
2. Use [Decision Matrices](./10-decision-matrices.md) for architectural choices
3. Refer to layer-specific guides (03-06) for details

**Troubleshooting?**
1. Check [Common Pitfalls](./11-common-pitfalls.md) for known issues
2. Review [Execution Flows](./12-execution-flows.md) to understand the flow
3. Consult [Testing Strategy](./08-testing.md) for testing patterns

**Reviewing code quality?**
1. Use [Implementation Checklist](./15-implementation-checklist.md) to verify 100% compliance
2. Run automated audit commands to detect violations
3. Follow the migration plan to fix non-compliant modules

---

## 📖 Reference

**Conceptual Foundations:**
- [Domain-Driven Design (Conceptual Book)](https://github.com/adrian-d-hidalgo/domain-driven-design) - Technology-agnostic DDD theory, patterns, and strategic design

---

## 💡 Key Concepts

**Hexagonal Architecture** (Ports & Adapters):
- Business logic isolated from infrastructure
- Ports define contracts, adapters implement them
- Easy to swap implementations (Prisma → TypeORM)

**Domain-Driven Design (DDD)**:
- Rich domain model with behavior
- Bounded contexts separate concerns
- Ubiquitous language shared with business

**Strategic Classification**:
- **Core Domain** (60-70% effort) - Business differentiators
- **Supporting Subdomains** (20-30% effort) - Necessary but standard
- **Generic Subdomains** (5-10% effort) - Buy/SaaS solutions

---

**Ready to start?** → [Getting Started](./01-getting-started.md)
