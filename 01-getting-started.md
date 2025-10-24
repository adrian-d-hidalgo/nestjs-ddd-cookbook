# 1. When to Use This Architecture

> Part of the [Hexagonal & DDD in NestJS Implementation Guide](../NESTJS_DDD_COOKBOOK.md)

## ✅ Ideal Cases for Hexagonal

- Backend API with **multiple integrations** (authentication providers, payment gateways, cloud storage, email services, etc.)
- **Microservices** with complex business logic
- **Multi-tenant SaaS** platforms with complex pricing or permission rules
- **Financial** or **billing** systems where business rules change frequently
- **E-commerce** with dynamic pricing, inventory, or promotion rules
- Applications with **multiple domain modules** (>3 business modules)

## ❌ Better to Avoid Hexagonal For

- **Simple CRUD admin** with Prisma without business logic
- **MVP prototypes** (<2 weeks of development)
- **Proxy APIs** without logic (just forwarding)
- Projects with **<3 developers** without DDD experience
- **Technical modules** (health, logger, metrics) that don't represent domain

## Checklist: Do I Need Hexagonal for This Module?

Apply Hexagonal **only if** the module meets **≥3** of these criteria:

- [ ] Has **>5 entities** or domain aggregates
- [ ] Includes **>10 distinct use cases**
- [ ] **Business rules change** frequently
- [ ] Needs to **test logic** without external dependencies
- [ ] Integrates with **≥2 external systems** (DB + API + Queue, etc.)
- [ ] May need to **change technology** in the future (Prisma → TypeORM, REST → gRPC)
- [ ] Multiple developers will work on this module for **>6 months**

**If it meets <3 criteria**: use standard NestJS structure (`service + controller + dto`).

## Trade-offs in NestJS

| Advantage                                       | Cost                                 |
| ----------------------------------------------- | ------------------------------------ |
| Switch Prisma → TypeORM without touching domain | +30-40% additional files             |
| Unit tests without real DB                      | Initial learning curve (1-2 sprints) |
| Microservices-ready from day 1                  | More initial boilerplate             |
| Portable domain between frameworks              | Requires discipline in layers        |

---

**Navigation:** [Up](../NESTJS_DDD_COOKBOOK.md) | [Next: Project Structure](./02-project-structure.md)
