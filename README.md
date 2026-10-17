# Backend Architecture with TypeScript

A hands-on collection of backend projects and machine-coding assignments built with TypeScript, Node.js, and modern backend technologies.

Explore how real backend applications are structured, how architecture patterns solve different problems, and how to design APIs that are secure, testable, and maintainable — beyond the traditional MERN stack.

## Tech Stack

<p align="left">
  <a href="https://www.typescriptlang.org/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/typescript/typescript-original.svg" width="42" height="42" alt="TypeScript" /></a>
  <a href="https://nodejs.org/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nodejs/nodejs-original.svg" width="42" height="42" alt="Node.js" /></a>
  <a href="https://expressjs.com/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/express/express-original.svg" width="42" height="42" alt="Express" /></a>
  <a href="https://www.prisma.io/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/prisma/prisma-original.svg" width="42" height="42" alt="Prisma" /></a>
  <a href="https://www.postgresql.org/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/postgresql/postgresql-original.svg" width="42" height="42" alt="PostgreSQL" /></a>
  <a href="https://www.mongodb.com/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/mongodb/mongodb-original.svg" width="42" height="42" alt="MongoDB" /></a>
  <a href="https://redis.io/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/redis/redis-original.svg" width="42" height="42" alt="Redis" /></a>
  <a href="https://www.docker.com/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/docker/docker-original.svg" width="42" height="42" alt="Docker" /></a>
  <a href="https://git-scm.com/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/git/git-original.svg" width="42" height="42" alt="Git" /></a>
  <a href="https://github.com/"><img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/github/github-original.svg" width="42" height="42" alt="GitHub" /></a>
</p>

Also explored where appropriate: Neon, AWS S3, Zod, BullMQ, Socket.IO, Vitest, Supertest, OpenAPI, payment-provider sandboxes, and vector search / LLM APIs.

> Technology choices vary by assignment. Not every project uses every tool.

## What's Inside

| #  | Project                  | What you'll learn                                                      |
| -- | ------------------------ | ---------------------------------------------------------------------- |
| 01 | Task Management API      | TypeScript, validation, CRUD, pagination, layered architecture         |
| 02 | Authentication & Sessions | Password hashing, access/refresh tokens, session revocation           |
| 03 | URL Shortener            | Unique identifiers, indexes, caching, analytics                        |
| 04 | Product & Inventory API  | Data integrity, atomic stock updates, audit history                    |
| 05 | Appointment Booking      | Timezones, transactions, concurrency, double-booking prevention        |
| 06 | Expense & Budget Tracker | Relational modeling, aggregation, money handling                       |
| 07 | Workspace & RBAC         | Roles, permissions, memberships, authorization boundaries              |
| 08 | File Storage Service     | AWS S3, upload flows, storage adapters, access control                 |
| 09 | API Rate Limiter         | Middleware, rate-limit algorithms, Redis-backed shared state           |
| 10 | Background Job Processing | Queues, workers, retries, idempotency                                 |
| 11 | Real-Time Chat           | WebSockets, message persistence, acknowledgements                      |
| 12 | Payments & Orders        | State machines, payment webhooks, transactions, refunds                |
| 13 | Webhook Delivery Platform | Signature verification, retries, delivery logs, SSRF protection       |
| 14 | Search API               | Search indexes, filtering, ranking, access-aware results               |
| 15 | Audit Log                | Append-only activity records, redaction, event capture                 |
| 16 | Multi-Tenant SaaS        | Tenant isolation, membership, quotas                                   |
| 17 | Approval Workflow Engine | State transitions, workflow rules, decision history                    |
| 18 | Notification Center      | Event-driven delivery, preferences, deduplication                      |
| 19 | Analytics & Reporting    | Aggregations, read models, asynchronous report generation              |
| 20 | AI Knowledge Base        | Document ingestion, embeddings, retrieval, LLM integration             |

Each assignment is intended to have its own folder with its requirements, API contract, data model, implementation, tests, and project-specific README as it is developed.

## Architecture Patterns

Different problems need different designs. This repository explores patterns such as:

- **MVC and Layered Architecture** — organize HTTP handlers, business logic, and data access.
- **Feature-Based Modular Architecture** — keep related code together as an application grows.
- **Clean Architecture / Hexagonal Architecture** — isolate business rules from frameworks and infrastructure.
- **Repository Pattern** — separate persistence operations from business logic when the abstraction is useful.
- **Adapter Pattern** — integrate replaceable providers such as S3, email, payments, or LLM APIs.
- **State Machines** — define valid lifecycle transitions for bookings, orders, and approvals.
- **Event-Driven Architecture** — handle asynchronous work, notifications, and webhooks.
- **CQRS-Inspired Read Models** — separate reporting and query needs from transactional writes.
- **Multi-Tenant Architecture** — enforce data boundaries between organizations.

The aim is not to apply every pattern everywhere. Choose the simplest design that meets the requirements, then document the trade-offs.

## Suggested Project Structure

A typical TypeScript assignment may use a structure like this:

```text
src/
├── config/             # Environment and infrastructure setup
├── modules/
│   └── tasks/
│       ├── task.routes.ts
│       ├── task.controller.ts
│       ├── task.service.ts
│       ├── task.repository.ts
│       ├── task.schema.ts
│       ├── task.model.ts
│       └── task.types.ts
├── middlewares/        # Authentication, validation, error handling
├── shared/             # Shared errors, logging, and utilities
├── app.ts
└── server.ts
tests/
docs/
.env.example
package.json
tsconfig.json
README.md
```

This is a starting point, not a mandatory template. A small API may need fewer layers; a complex domain may benefit from use cases, domain entities, ports, and infrastructure adapters.

## Machine-Coding Guide

Machine coding is about turning a problem statement into a working, well-designed application within a limited time. Start with the requirements and correctness before adding infrastructure.

### 1. Understand the problem

Clarify the core user flows, assumptions, constraints, and out-of-scope features. Identify edge cases before writing endpoints.

### 2. Design the API

Define HTTP methods, routes, request schemas, response shapes, status codes, pagination, and error behavior. Use Zod to validate untrusted request data at runtime.

### 3. Model the data

Identify entities, relationships, unique constraints, indexes, and operations that need transactions. Choose PostgreSQL when relational constraints and transactions fit the domain; choose MongoDB when a document model fits the data and access patterns. Neon can provide managed PostgreSQL, while Prisma can provide typed database access and migrations.

### 4. Choose the architecture

Keep controllers focused on HTTP. Put business rules in services or use cases. Isolate data access when it clarifies the design. Use adapters for external systems such as S3 or payment providers.

### 5. Implement and test the core flow

Make the main path work first. Then test invalid input, missing resources, unauthorized access, conflicting updates, and failures from external services. For asynchronous systems, test retries and duplicate delivery.

### 6. Explain your trade-offs

Document why you chose a database, architecture, index, transaction boundary, or queue. A good solution is not the one with the most layers; it is the one whose design matches the problem and whose behavior is easy to explain.

## Working with the Stack

- **TypeScript + Node.js:** use strict typing and validate external input at runtime.
- **Express:** organize routes and middleware around clear responsibilities.
- **PostgreSQL + Prisma + Neon:** practice relational modeling, constraints, migrations, and transactions.
- **MongoDB + Mongoose:** practice document modeling, indexes, validation, and atomic updates.
- **AWS S3:** store files outside the application server and enforce authorization around upload/download flows.
- **Redis + BullMQ:** introduce shared state, caching, and background processing only when a project needs them.
- **Vitest + Supertest:** test business rules and HTTP behavior.
- **Docker:** make local dependencies and development environments more reproducible.

Use `.env.example` to document required configuration, and never commit real credentials or tokens.

## Repository Philosophy

Build beyond CRUD. Learn how to handle concurrency, security, data integrity, failure recovery, external integrations, and maintainability. Use each assignment to practice a different engineering concept, and keep the implementation and documentation understandable to another developer.

**Start with:** `01-task-management`, then move to authentication, the URL shortener, inventory, and appointment booking before tackling queues, payments, multi-tenancy, and AI document retrieval.
