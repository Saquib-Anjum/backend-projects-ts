# Backend Architecture with Node.js and TypeScript

## Detailed Learning Notes, Diagrams, Layers, Patterns, and Practical Guidance

These notes explain how to design Node.js backends that are
understandable, testable, secure, and maintainable. The examples focus
on TypeScript with Express, but most principles also apply to Fastify,
NestJS, and other Node.js frameworks.

------------------------------------------------------------------------

# 1. What is backend architecture?

Backend architecture is the way an application organizes
responsibilities, dependencies, data access, business rules, and
communication with external systems.

A backend is more than routes and database queries. It typically needs
to:

-   Accept and validate client requests.
-   Authenticate users and authorize actions.
-   Apply business rules.
-   Read and write data safely.
-   Communicate with external services.
-   Handle errors and partial failures.
-   Log meaningful events and expose useful metrics.
-   Remain testable as requirements change.

Good architecture makes it clear where a change belongs. For example,
changing a payment provider should not require rewriting unrelated order
rules.

## Architecture versus technology stack

These are different concepts.

-   **Technology stack:** Node.js, TypeScript, Express, Prisma,
    PostgreSQL, Redis, AWS S3.
-   **Architecture:** layered architecture, modular monolith, Clean
    Architecture, event-driven architecture.
-   **Design pattern:** Repository, Adapter, Strategy, Factory, State
    Machine.
-   **Infrastructure:** cloud services, containers, databases, queues,
    deployment configuration.

A project can use TypeScript and PostgreSQL with either a simple layered
architecture or Clean Architecture. Technology alone does not determine
the architecture.

------------------------------------------------------------------------

# 2. High-level backend request lifecycle

A typical web application sends an HTTP request to the backend. The
backend validates it, applies rules, accesses data, and returns a
response.

``` mermaid
flowchart TD
    A[Browser / Mobile / Frontend] --> B[HTTP Request]
    B --> C[Node.js Server]
    C --> D[Express Middleware]
    D --> E[Route Matching]
    E --> F[Controller]
    F --> G[Service / Use Case]
    G --> H[Repository / Data Access]
    H --> I[(Database)]
    I --> H
    H --> G
    G --> F
    F --> J[HTTP Response]
    J --> A
```

The diagram is a conceptual flow. Middleware may include request IDs,
logging, authentication, authorization, validation, and rate limiting.
The exact order depends on the endpoint and security requirements.

## Example: create a task

A frontend sends:

``` http
POST /api/tasks
Content-Type: application/json
Authorization: Bearer <access-token>
```

Body:

``` json
{
  "title": "Learn backend architecture",
  "priority": "high"
}
```

A well-organized backend does roughly this:

1.  Express receives the request.
2.  Middleware attaches a request ID and logs relevant metadata.
3.  Authentication verifies the access token.
4.  Validation checks that the request body matches the expected schema.
5.  The controller extracts validated data and calls the task service.
6.  The service checks business rules and determines the owner from the
    authenticated identity.
7.  The repository persists the task.
8.  The controller maps the result to an HTTP response.
9.  Centralized error handling converts unexpected failures into safe
    error responses.

The frontend should not decide who owns the task by submitting an
arbitrary `ownerId`. Ownership should come from a trusted authentication
context.

------------------------------------------------------------------------

# 3. Node.js and TypeScript: what they do

## Node.js

Node.js is a JavaScript runtime built on the V8 engine. It is commonly
used for network services, APIs, command-line tools, and background
workers.

Its event-driven architecture handles many I/O operations efficiently. A
typical Node.js application runs JavaScript on an event loop and
delegates or coordinates many I/O operations without creating a new
JavaScript thread for each request.

Important practical consequences:

-   Avoid blocking the event loop with long synchronous work.
-   Use asynchronous APIs for database and network I/O.
-   Handle promise rejections and errors intentionally.
-   CPU-heavy work may need worker threads or a separate worker process.
-   An `async` function does not automatically make CPU-heavy code
    non-blocking.

## TypeScript

TypeScript adds static type checking and developer tooling on top of
JavaScript. Type annotations are removed during compilation; they do not
validate incoming HTTP data at runtime.

For example:

``` ts
type CreateTaskInput = {
  title: string;
  priority: "low" | "medium" | "high";
};
```

This helps code that is checked by TypeScript. It does not prevent a
client from sending an invalid JSON body. Use a runtime validator such
as Zod at system boundaries.

## Why strict mode?

Strict TypeScript catches many errors before runtime. Useful practices
include:

-   Enable `strict` in `tsconfig.json`.
-   Avoid `any`; use `unknown` for values that have not been validated.
-   Prefer explicit domain types over unstructured objects.
-   Use discriminated unions for stateful workflows.
-   Keep database types separate from public API response types when
    their purposes differ.
-   Validate environment variables before starting the server.

------------------------------------------------------------------------

# 4. Understand the layers of a backend

A common structure is:

``` mermaid
flowchart TB
    Client[Client]
    Route[Route]
    Middleware[Middleware]
    Controller[Controller]
    Validator[Schema / DTO Validation]
    Service[Service / Use Case]
    Repository[Repository]
    Model[ORM / ODM Model]
    Database[(Database)]
    Adapter[External Service Adapter]

    Client --> Route
    Route --> Middleware
    Middleware --> Validator
    Validator --> Controller
    Controller --> Service
    Service --> Repository
    Repository --> Model
    Model --> Database
    Service --> Adapter
```

This is one possible arrangement, not a universal rule. Some
applications validate in middleware before the controller; some
frameworks integrate validation into route definitions. The important
part is the separation of responsibilities.

## 4.1 Routes

**Responsibility:** map HTTP methods and paths to middleware and
controller functions.

Example:

``` ts
router.post("/", authenticate, validate(createTaskSchema), taskController.create);
router.get("/:id", authenticate, taskController.getById);
```

Routes should not contain complex business logic or direct database
operations.

## 4.2 Middleware

**Responsibility:** perform reusable work around request handling.

Examples:

-   Authentication.
-   Authorization context.
-   Request ID and logging.
-   Rate limiting.
-   CORS configuration.
-   Request validation.
-   Error handling.

Middleware should have a focused purpose. Avoid putting all business
rules into a giant middleware function.

Authentication answers, "Who is making this request?" Authorization
answers, "Is this identity allowed to perform this action on this
resource?"

## 4.3 Controller

**Responsibility:** translate HTTP input into an application call and
map the result into an HTTP response.

A controller commonly:

-   Reads validated body, params, and query data.
-   Uses the authenticated principal.
-   Calls a service or use case.
-   Chooses a success status code.
-   Returns the response DTO.

A controller should not contain large database queries, complicated
business calculations, or provider-specific SDK logic.

## 4.4 Schema / DTO

These are related but different concepts.

-   **Validation schema:** defines and checks acceptable external input,
    often at runtime using Zod.
-   **DTO (Data Transfer Object):** defines the shape of data crossing
    an application boundary, such as an API request or response.
-   **Domain entity:** represents a business concept and its rules.
-   **Database model:** represents how data is stored by an ORM or ODM.

These types may look similar, but they do not have to be identical. A
database record may contain internal fields that must never appear in an
API response.

## 4.5 Service / Use case

**Responsibility:** implement or orchestrate application business rules.

Examples:

-   Ensure a task belongs to the current user.
-   Calculate an order total from trusted product prices.
-   Reserve inventory before checkout.
-   Enforce booking cancellation policy.
-   Coordinate a database transaction.
-   Request a signed URL from a storage adapter.

Services should not depend on Express request and response objects when
that dependency can be avoided. This makes business logic easier to test
without running an HTTP server.

## 4.6 Repository

**Responsibility:** isolate persistence operations when that abstraction
is useful.

Example operations:

``` ts
interface TaskRepository {
  findById(id: string): Promise<Task | null>;
  findByOwner(ownerId: string, page: number): Promise<Task[]>;
  create(input: NewTask): Promise<Task>;
}
```

A repository can hide Prisma, Mongoose, or raw SQL details from the
service.

Important: not every project needs a repository abstraction. Prisma
already offers a data-access API. Add a repository when it clarifies
boundaries, encapsulates meaningful query logic, or helps testing---not
merely to add another file.

## 4.7 Model / ORM / ODM

A model describes persistence structure and often supports queries.

-   Prisma defines a schema and generates a typed client.
-   Mongoose defines MongoDB document schemas, validation, and model
    APIs.
-   SQL databases enforce constraints such as unique keys, foreign keys,
    and checks.

Application business rules should not rely exclusively on validation in
the HTTP layer. Important invariants often need database constraints or
transactional enforcement as well.

## 4.8 Configuration

Configuration covers environment variables and infrastructure clients.

Examples:

-   Database connection.
-   JWT signing keys.
-   Redis URL.
-   S3 bucket and region.
-   Email or payment provider credentials.
-   Port and logging level.

Validate required environment variables at startup. Never commit
production secrets. Keep `.env.example` as a list of variable names and
safe placeholder values.

## 4.9 Shared utilities and common code

Shared code can include application errors, logging, date helpers,
response mapping, and reusable types.

Do not use `utils/` as a dumping ground for unrelated logic. If a helper
belongs to a specific feature, keep it near that feature.

------------------------------------------------------------------------

# 5. MVC architecture

MVC means Model--View--Controller.

In a JSON API, the "View" is often represented by serialized JSON
responses rather than server-rendered HTML.

``` mermaid
flowchart TD
    Request[HTTP Request] --> Controller
    Controller --> Model
    Model --> DB[(Database)]
    DB --> Model
    Model --> Controller
    Controller --> Response[JSON Response]
```

## Advantages

-   Simple to learn.
-   Quick to implement for small APIs.
-   Clear entry point for each endpoint.

## Limitations

As the application grows, controllers can become too large and mix HTTP
concerns with business rules, database access, and external
integrations.

## When to use it

Use simple MVC for small CRUD services and prototypes. You can still
introduce services and validation while keeping the overall structure
lightweight.

------------------------------------------------------------------------

# 6. Layered architecture

Layered architecture separates responsibilities into layers, usually
HTTP, application/business logic, and data access.

``` mermaid
flowchart TD
    subgraph Presentation
      Routes --> Controllers
      Controllers --> Validation
    end
    subgraph Application
      Services[Services / Use Cases]
      Rules[Business Rules]
      Services --> Rules
    end
    subgraph Data
      Repositories[Repositories]
      ORM[Prisma / Mongoose]
      DB[(Database)]
      Repositories --> ORM --> DB
    end
    Controllers --> Services
    Services --> Repositories
```

## Dependency principle

A simple implementation may allow the service to import a repository
implementation directly. In a more strictly layered or clean design, the
service depends on a repository interface, and infrastructure supplies
the implementation.

The right choice depends on complexity and the benefit of the
abstraction.

## Benefits

-   Clear responsibilities.
-   Easier unit testing of business rules.
-   More manageable code than route handlers containing everything.
-   Suitable for many medium-sized APIs.

## Common mistake

Creating a `service.ts` and `repository.ts` for every tiny operation
without improving clarity. Layers should represent useful boundaries,
not ceremony.

------------------------------------------------------------------------

# 7. Feature-based modular architecture

Instead of grouping every controller together and every service together
globally, organize code by business feature.

``` text
src/
├── modules/
│   ├── auth/
│   │   ├── auth.routes.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.repository.ts
│   │   ├── auth.schema.ts
│   │   └── auth.types.ts
│   ├── tasks/
│   │   ├── task.routes.ts
│   │   ├── task.controller.ts
│   │   ├── task.service.ts
│   │   ├── task.repository.ts
│   │   ├── task.model.ts
│   │   └── task.schema.ts
│   └── users/
│       └── ...
├── config/
├── middlewares/
├── shared/
├── app.ts
└── server.ts
```

## Why this works

A task-related change is usually found in the `tasks` module. The module
owns its routes, service, validation, and persistence logic.

## Module boundaries

One module should not casually manipulate another module's database
model. Instead, expose a clear service or application interface for
cross-feature operations.

For example, an orders module should request inventory reservation
through an inventory service or use case rather than silently changing
inventory records itself.

This is often a good default for a growing application: a **modular
monolith**.

A modular monolith is one deployable application with well-defined
internal modules. It is not a microservice architecture.

------------------------------------------------------------------------

# 8. Clean Architecture and Hexagonal Architecture

Clean Architecture and Hexagonal Architecture aim to isolate business
rules from frameworks and infrastructure. Their exact diagrams differ,
but both encourage clear dependency boundaries.

``` mermaid
flowchart TD
    HTTP[Express Controller / HTTP Adapter] --> UseCase[Application Use Case]
    UseCase --> Domain[Domain Rules / Entities]
    UseCase --> Port[Repository or Provider Interface]
    RepoAdapter[Prisma / Mongoose Adapter] -. implements .-> Port
    ProviderAdapter[S3 / Payment / Email Adapter] -. implements .-> Port
    RepoAdapter --> DB[(Database)]
    ProviderAdapter --> External[External Service]
```

The important idea is that the application defines the capability it
needs, while infrastructure implements that capability.

## Common folders

``` text
src/
├── domain/
│   ├── entities/
│   ├── value-objects/
│   └── policies/
├── application/
│   ├── use-cases/
│   ├── ports/
│   └── dto/
├── infrastructure/
│   ├── database/
│   ├── repositories/
│   └── providers/
├── interfaces/
│   └── http/
├── config/
├── app.ts
└── server.ts
```

## Terms

-   **Domain:** business concepts and rules.
-   **Use case:** coordinates one application action, such as
    `CreateOrder`.
-   **Port:** interface for a required capability.
-   **Adapter:** implementation of a port, such as a Prisma repository
    or S3 client.
-   **Composition root:** the place where implementations are created
    and wired together.

## When it is useful

-   Complex business rules.
-   Multiple external providers.
-   High testability requirements.
-   Infrastructure that may change.
-   Business logic that should not depend on Express or an ORM.

## Trade-off

It introduces more abstractions and files. Do not use the full structure
for a trivial CRUD API unless the exercise is specifically about
learning it.

------------------------------------------------------------------------

# 9. Domain-Driven Design (DDD) fundamentals

DDD is a way to model software around the business domain. It is not
simply a folder structure.

Key concepts:

-   **Entity:** has identity over time, such as an order.
-   **Value object:** defined by its values, such as a money amount with
    currency.
-   **Aggregate:** a consistency boundary for related domain objects.
-   **Aggregate root:** the entry point through which the aggregate's
    rules are enforced.
-   **Repository:** retrieves and persists domain aggregates.
-   **Domain service:** business logic that does not naturally belong to
    one entity.
-   **Bounded context:** an area where a model has a specific meaning.

Example: `Order` and `Payment` may be separate domain concepts with
different lifecycle rules. Do not force them into one giant model simply
because they are related.

DDD becomes valuable when the business domain is complex enough to
justify the modeling effort.

------------------------------------------------------------------------

# 10. Common design patterns in backend systems

## Repository Pattern

Hides persistence details behind a domain- or application-oriented
interface.

Useful when database operations are complex or when isolating
infrastructure meaningfully improves tests. Avoid building a generic
repository framework that merely duplicates the ORM API.

## Adapter Pattern

Converts one interface into another. It is useful for S3 storage, email
providers, payment gateways, and LLM providers.

``` text
Application Service
       |
Storage Interface
       |
  +----+----------------+
  |                     |
S3 Adapter        Local Test Adapter
```

The service depends on the storage interface rather than directly
constructing an AWS SDK client everywhere.

## Strategy Pattern

Encapsulates interchangeable algorithms behind a common interface. For
example, different shipping-price or discount calculation strategies can
be selected based on the order context.

## Factory Pattern

Centralizes object creation when constructing an object requires
non-trivial decisions or setup. Do not introduce a factory just to
instantiate a simple object.

## Dependency Injection

Dependencies are supplied to a class or function instead of being
created internally.

``` ts
class CreateTaskService {
  constructor(private readonly tasks: TaskRepository) {}

  async execute(input: NewTask): Promise<Task> {
    return this.tasks.create(input);
  }
}
```

This makes it easier to substitute a fake repository in tests.
Dependency injection does not require a framework.

## State Machine Pattern

Represents valid states and transitions explicitly. It is useful for
orders, bookings, approvals, and payment flows.

``` mermaid
stateDiagram-v2
    [*] --> Pending
    Pending --> Confirmed
    Pending --> Cancelled
    Confirmed --> Completed
    Confirmed --> Cancelled
    Completed --> [*]
    Cancelled --> [*]
```

Only allow transitions permitted by the domain. A state machine helps
prevent scattered and inconsistent status checks.

------------------------------------------------------------------------

# 11. CQRS and read/write separation

CQRS stands for Command Query Responsibility Segregation.

-   A **command** changes state.
-   A **query** reads state.

A CQRS-inspired design may use separate application services or read
models without introducing separate databases or microservices.

``` mermaid
flowchart TD
    Client --> CommandAPI[Command API]
    Client --> QueryAPI[Query API]
    CommandAPI --> WriteService[Write Service]
    WriteService --> WriteDB[(Operational Database)]
    WriteDB --> Event[Outbox / Event]
    Event --> Projection[Projection Worker]
    Projection --> ReadModel[(Read Model)]
    QueryAPI --> ReadModel
```

## When useful

-   Reporting queries are expensive.
-   Read patterns differ significantly from write patterns.
-   Aggregated dashboards need precomputed results.
-   Read models have different shapes from transactional records.

## Trade-offs

Read models can lag behind writes. You need projection rebuild
strategies, event handling, and clear freshness guarantees.

Do not implement CQRS just because an application has GET and POST
endpoints; most APIs do.

------------------------------------------------------------------------

# 12. Event-driven architecture and background workers

An event-driven system reacts to facts that have occurred, such as
`OrderPlaced` or `DocumentUploaded`.

``` mermaid
flowchart TD
    Client --> API
    API --> DB[(Database)]
    API --> Queue[Job Queue]
    Queue --> Worker[Background Worker]
    Worker --> Email[Email / External API]
    Worker --> DB
    Worker --> Logs[Logs and Metrics]
```

Use queues for slow or retryable work such as report generation, email
delivery, document ingestion, or webhook delivery.

Important concepts:

-   **Idempotency:** repeated processing does not duplicate the business
    effect.
-   **Retry policy:** transient failures may be retried with backoff.
-   **Dead-letter handling:** repeated failures are recorded for
    inspection or controlled replay.
-   **Outbox pattern:** persist a business change and an event record in
    the same database transaction; a worker publishes pending events.
-   **Observability:** track queue depth, attempts, failures, and
    processing time.

A queue does not automatically provide exactly-once business execution.
Consumers must be designed for duplicate delivery and recovery.

------------------------------------------------------------------------

# 13. Database architecture: PostgreSQL, Neon, Prisma, MongoDB

Choose a database based on the domain and query patterns rather than
popularity.

## PostgreSQL

A relational database that supports relationships, constraints,
transactions, indexes, and SQL queries. It is a strong choice for
orders, payments, bookings, memberships, and many business applications.

## Neon

A managed PostgreSQL platform. Neon provides hosted PostgreSQL
infrastructure; it is not a separate database model from PostgreSQL.
Your application still uses PostgreSQL concepts, SQL, constraints, and
transactions.

## Prisma

A TypeScript-friendly ORM that provides a schema, generated client, and
migration workflow for supported databases.

Prisma is a data-access tool, not a database. With PostgreSQL, the
architecture is typically:

``` text
Service -> Repository (optional) -> Prisma Client -> PostgreSQL / Neon
```

## MongoDB

A document database. MongoDB can be a good fit for document-shaped data
and access patterns that benefit from flexible documents. It still needs
schema design, indexes, validation, and careful transaction boundaries.

Mongoose is an ODM commonly used with MongoDB.

## S3

Amazon S3 is object storage, not a relational or document database.
Store file bytes in S3 and metadata such as owner, object key, content
type, and upload status in your database.

``` mermaid
flowchart TD
    Client --> API
    API --> Auth[Authenticate and Authorize]
    Auth --> Metadata[(PostgreSQL or MongoDB)]
    Auth --> S3[S3 Object Storage]
    API --> SignedURL[Create Time-Limited Upload / Download URL]
    SignedURL --> Client
    Client --> S3
```

A presigned URL grants time-limited access to a specific object
operation. The backend should verify permissions before issuing it.

------------------------------------------------------------------------

# 14. Data integrity and transactions

A transaction groups operations so that the database enforces the
intended atomicity and consistency rules.

Example: creating an order and reserving inventory may require a
transaction, conditional updates, locking, or a reservation model
depending on the database and workflow.

Important principles:

-   Use database constraints for invariants such as uniqueness.
-   Avoid a read-then-write sequence when concurrent requests can
    violate the invariant.
-   Keep transactions short.
-   Do not assume an external API call can be rolled back by a database
    transaction.
-   Use idempotency keys and reconciliation for workflows involving
    external providers.
-   Design concurrency behavior explicitly.

Example: checking that a username is available and then inserting it is
not enough by itself. Two requests can pass the check simultaneously. A
unique database constraint is the final guard.

------------------------------------------------------------------------

# 15. Security architecture

Security must be considered at every layer.

## Authentication

Establishes identity, for example by verifying a session or access
token.

## Authorization

Checks whether the authenticated identity can perform the requested
action on the specific resource.

## Input validation

Treat query strings, path parameters, request bodies, uploaded files,
and external provider responses as untrusted data.

## Secrets

Keep signing keys, database credentials, AWS credentials, and provider
tokens out of source control. Prefer a suitable secrets-management
mechanism in deployed environments.

## Additional concerns

-   Rate limiting on sensitive endpoints.
-   Password hashing using a suitable password-hashing algorithm.
-   Correct token expiry and refresh/revocation strategy.
-   Least-privilege database and cloud permissions.
-   Secure CORS configuration.
-   Safe error responses.
-   Avoid logging tokens, passwords, or sensitive payloads.
-   Protect outbound URL-fetching features against SSRF.
-   Apply tenant/resource authorization to background jobs and search
    results too.

Security is not a single middleware file. It is a set of controls across
request handling, business logic, data access, and infrastructure.

------------------------------------------------------------------------

# 16. Error handling and observability

A backend should distinguish expected application errors from unexpected
infrastructure or programming failures.

Examples of expected errors:

-   Validation failed.
-   Resource not found.
-   Permission denied.
-   Conflict with existing state.
-   Rate limit exceeded.

A consistent error response might be:

``` json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Task not found"
  }
}
```

The client should not receive stack traces, database internals, secrets,
or raw provider errors.

## Observability

-   **Logs:** explain individual events and failures.
-   **Metrics:** show rates, latencies, error counts, queue depth, and
    resource usage.
-   **Traces:** follow a request across multiple services or operations.
-   **Request/correlation IDs:** connect related logs for one workflow.

Do not log sensitive data just because it may help debugging.

------------------------------------------------------------------------

# 17. Testing architecture

A good backend design supports multiple testing levels.

``` mermaid
flowchart TD
    Unit[Unit Tests: Domain / Services]
    Integration[Integration Tests: Database / Adapters]
    API[API Tests: HTTP + Middleware]
    E2E[End-to-End Tests: Full Workflow]
    Unit --> Integration --> API --> E2E
```

The diagram indicates increasing scope, not a mandatory execution order.

-   **Unit tests:** validate isolated rules, often with fake
    dependencies.
-   **Integration tests:** verify database constraints, queries, and
    adapter behavior.
-   **API tests:** verify HTTP status codes, validation, authentication,
    and response shape.
-   **End-to-end tests:** verify important workflows across the system.

For Node.js + TypeScript, Vitest or Jest can run tests; Supertest can
exercise Express endpoints. Use a test database or isolated test
infrastructure for database-dependent tests.

Important cases include invalid inputs, permissions, concurrent writes,
duplicate delivery, transaction failures, and worker retries.

------------------------------------------------------------------------

# 18. Recommended TypeScript backend structure

This is a practical starting point for a medium-sized modular monolith.

``` text
src/
├── config/
│   ├── env.ts
│   ├── database.ts
│   └── logger.ts
├── modules/
│   ├── auth/
│   ├── users/
│   └── tasks/
│       ├── task.routes.ts
│       ├── task.controller.ts
│       ├── task.service.ts
│       ├── task.repository.ts
│       ├── task.schema.ts
│       ├── task.model.ts
│       └── task.types.ts
├── middlewares/
│   ├── authenticate.ts
│   ├── authorize.ts
│   ├── validate-request.ts
│   └── error-handler.ts
├── shared/
│   ├── errors/
│   ├── logging/
│   └── http/
├── app.ts
└── server.ts
tests/
docs/
.env.example
package.json
tsconfig.json
```

## `app.ts` versus `server.ts`

A useful separation is:

-   `app.ts` configures Express: middleware, routes, and error handling.
    Export the app for tests.
-   `server.ts` loads validated configuration, connects to
    infrastructure, and starts listening.

This makes API tests easier because they can import the app without
opening a real listening port.

## Avoid overengineering

A small assignment might need only routes, controllers, services,
schemas, and a database client. Add repositories, domain entities,
ports, queues, and separate workers when the requirements justify them.

------------------------------------------------------------------------

# 19. How to choose the right architecture

  -----------------------------------------------------------------------
  Problem                             Practical starting point
  ----------------------------------- -----------------------------------
  Small CRUD API                      Simple layered architecture

  Several business features           Feature-based modular monolith

  Complex business rules              Domain modeling and use cases

  Multiple external providers         Adapter interfaces

  Order and payment lifecycles        State machine + transactions +
                                      idempotency

  Slow background tasks               Queue + worker

  Analytics-heavy dashboard           Read models / CQRS-inspired
                                      separation

  Organization-based SaaS             Tenant-aware modules and
                                      authorization

  Large independently deployed        Consider services only when
  domains                             operational and team needs justify
                                      them
  -----------------------------------------------------------------------

Microservices are not the natural next step for every growing
application. A modular monolith often provides strong boundaries with
much less deployment and operational complexity.

------------------------------------------------------------------------

# 20. A practical learning sequence

1.  **Task Management API:** routes, controllers, services, schemas, and
    repositories.
2.  **Authentication:** password security, token lifecycle, and
    middleware.
3.  **URL Shortener:** indexes, uniqueness, caching, and analytics.
4.  **Inventory API:** atomic updates and data integrity.
5.  **Appointment Booking:** concurrency, transactions, and timezones.
6.  **Workspace with RBAC:** authorization and module boundaries.
7.  **File Storage:** S3 adapter, metadata, and signed URLs.
8.  **Background Jobs:** queues, retries, and idempotency.
9.  **Payments and Orders:** state machines, webhooks, and
    reconciliation.
10. **Analytics / AI Knowledge Base:** read models, pipelines,
    retrieval, and provider adapters.

For each project, write down the requirements, API contract, data model,
architecture diagram, edge cases, tests, and trade-offs before expanding
the implementation.

------------------------------------------------------------------------

# Final principles

1.  Architecture is about responsibility and dependency boundaries, not
    just folders.
2.  Keep controllers focused on HTTP and business rules in services or
    use cases.
3.  Validate external input at runtime; TypeScript types alone are not
    runtime validation.
4.  Enforce important data invariants with database constraints and
    transactions where appropriate.
5.  Choose PostgreSQL, MongoDB, S3, queues, and caches according to the
    problem.
6.  Design security, observability, and failure handling as part of the
    system.
7.  Add abstractions when they improve clarity, testability, or
    replaceability.
8.  Prefer a simple architecture that fits the requirements over a
    complex architecture that only looks impressive.

**The ultimate goal:** be able to explain how a request flows through
the system, where each business rule lives, how data remains correct
under failure or concurrency, and why the architecture fits the problem.
