# Backend Machine Coding Handbook

## 20 Node.js + TypeScript Assignments --- APIs, Architecture Patterns, and System Design

A practical assignment catalog for learning how to design, implement,
test, and explain real backend systems.

**Primary stack:** Node.js, TypeScript, Express 5, Zod,
MongoDB/Mongoose\
**Alternative stack:** PostgreSQL + Prisma\
**Testing:** Vitest or Jest, Supertest\
**API documentation:** OpenAPI / Swagger\
**Local infrastructure:** Docker Compose when useful\
**Optional integrations:** Redis, BullMQ, S3-compatible object storage,
WebSockets, payment sandbox, email provider, and an LLM API

> These are assignment specifications, not completed implementations.
> Build each one yourself, document trade-offs, and add tests. Do not
> add every technology to every project; choose infrastructure because
> the requirements justify it.

------------------------------------------------------------------------

# How to use this handbook

For each assignment:

1.  Read the problem statement and clarify assumptions.
2.  Sketch the API contract and data model before coding.
3.  Choose the architecture pattern that best fits the problem.
4.  Implement the happy path, validation, and error handling.
5.  Add authorization, edge cases, and automated tests.
6.  Write a README with setup instructions, environment variables,
    example requests, and design decisions.
7.  Explain the request lifecycle and the main trade-offs as if this
    were a machine-coding interview.

## Suggested difficulty path

  -----------------------------------------------------------------------
  Level                   Assignments             Focus
  ----------------------- ----------------------- -----------------------
  Beginner                1--4                    CRUD, validation,
                                                  pagination,
                                                  authentication

  Intermediate            5--9                    relational data,
                                                  transactions, RBAC,
                                                  file handling, caching

  Advanced                10--15                  queues, real-time
                                                  events, webhooks,
                                                  search, audit logs,
                                                  rate limits

  System design           16--20                  multi-tenancy, workflow
                                                  engines, notifications,
                                                  analytics, AI retrieval
  -----------------------------------------------------------------------

## Baseline engineering rules

Every assignment should include, where relevant:

-   TypeScript strict mode; avoid `any` unless justified.
-   Environment-based configuration with validation at startup.
-   Request validation at the API boundary.
-   A consistent response and error format.
-   Correct HTTP status codes.
-   Centralized error handling and structured logs.
-   Authentication and authorization where the domain requires them.
-   Pagination and bounded query parameters for list endpoints.
-   Tests for success, validation failures, permission failures, and
    important edge cases.
-   Database indexes that match query patterns.
-   `.env.example` with placeholders only; never commit secrets.
-   A README with setup, scripts, endpoint examples, and architecture
    decisions.

A common response shape could be:

``` json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

A common error shape could be:

``` json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": []
  }
}
```

Keep the format consistent within each project. Do not expose stack
traces or internal database errors to clients.

------------------------------------------------------------------------

# Part I --- 20 backend assignments

## 1. Task Management API

**Difficulty:** Beginner\
**Pattern:** Layered architecture / feature-based modules\
**Goal:** Build a well-structured CRUD API rather than putting all logic
in route handlers.

### Problem statement

Build an API where registered users can create tasks, organize them by
status and priority, assign due dates, and retrieve their own tasks.

### Functional requirements

-   Create, read, update, and delete tasks.
-   Fields: title, description, status, priority, dueDate, ownerId,
    createdAt, updatedAt.
-   Filter by status, priority, and due-date range.
-   Support pagination, sorting, and text search on task titles.
-   A user must never read or modify another user's tasks.
-   Validate inputs and return consistent errors.

### API contract

  Method   Endpoint           Purpose
  -------- ------------------ ------------------------
  POST     `/api/tasks`       Create a task
  GET      `/api/tasks`       List and filter tasks
  GET      `/api/tasks/:id`   Get one task
  PATCH    `/api/tasks/:id`   Update selected fields
  DELETE   `/api/tasks/:id`   Delete a task

Example list request:
`GET /api/tasks?status=todo&page=1&limit=20&sortBy=dueDate`

### Data model

`Task`: `title`, `description`, `status`, `priority`, `dueDate`,
`ownerId`, timestamps.

### Suggested structure

``` text
src/
  config/
  modules/
    tasks/
      task.routes.ts
      task.controller.ts
      task.service.ts
      task.repository.ts
      task.model.ts
      task.schema.ts
      task.types.ts
  middlewares/
  utils/
  app.ts
  server.ts
```

### Acceptance criteria

-   Invalid status or priority returns `400`.
-   Missing task returns `404`.
-   Unauthorized access does not reveal another user's task.
-   Pagination has a maximum `limit`.
-   Unit tests cover service rules; integration tests cover routes.

### Stretch goals

Add recurring tasks, soft deletion, and a task activity timeline.

------------------------------------------------------------------------

## 2. Authentication and Session Service

**Difficulty:** Beginner → Intermediate\
**Pattern:** Security-focused layered architecture

### Problem statement

Create an authentication service that supports registration, login,
logout, token refresh, and password recovery.

### Functional requirements

-   Register with name, email, and password.
-   Hash passwords with a suitable password-hashing algorithm; never
    store plain-text passwords.
-   Login with generic failure messages that do not reveal whether an
    email exists.
-   Short-lived access tokens and a secure refresh-token strategy.
-   Logout and revoke refresh sessions.
-   Forgot-password flow using a short-lived, single-use token.
-   Validate token expiry and invalidation.
-   Rate-limit sensitive endpoints.

### API contract

  Method   Endpoint                      Purpose
  -------- ----------------------------- ------------------------
  POST     `/api/auth/register`          Register
  POST     `/api/auth/login`             Login
  POST     `/api/auth/refresh`           Rotate/refresh session
  POST     `/api/auth/logout`            Revoke session
  POST     `/api/auth/forgot-password`   Request reset
  POST     `/api/auth/reset-password`    Reset password
  GET      `/api/auth/me`                Current user

### Data model

`User`: email (unique), passwordHash, name, status, timestamps.\
`RefreshSession`: userId, tokenHash or token identifier, expiresAt,
revokedAt, device metadata.

### Architecture emphasis

Separate authentication from authorization. Put token verification in
middleware, and keep credential/session rules in a service. Never put
secrets in source code.

### Acceptance criteria

-   Duplicate registration is handled cleanly.
-   Password hashes are never returned by the API.
-   Refresh-token rotation and revocation are tested.
-   Expired and malformed tokens are rejected.
-   Password reset tokens cannot be reused.

### Stretch goals

Add Google/GitHub OAuth, email verification, and session management per
device.

------------------------------------------------------------------------

## 3. URL Shortener

**Difficulty:** Beginner → Intermediate\
**Pattern:** Read-heavy service with cache and analytics

### Problem statement

Build a service that turns long URLs into short links and redirects
visitors to the original URL.

### Functional requirements

-   Create a short link from a valid URL.
-   Allow optional custom aliases and expiration dates.
-   Redirect using a short code.
-   Track click count and basic analytics.
-   Prevent alias collisions.
-   Decide how disabled or expired links behave.
-   Add pagination to the owner's link list.

### API contract

  Method   Endpoint                     Purpose
  -------- ---------------------------- -----------------------
  POST     `/api/links`                 Create short link
  GET      `/api/links/:code`           Inspect link metadata
  GET      `/r/:code`                   Redirect
  PATCH    `/api/links/:id`             Update or disable
  GET      `/api/links/:id/analytics`   Read analytics

### Data model

`ShortLink`: code (unique), originalUrl, ownerId, expiresAt, disabledAt,
createdAt.\
`ClickEvent`: linkId, timestamp, referrer, coarse user-agent category;
avoid collecting unnecessary personal data.

### Design decisions

-   Use a unique database index for `code`; do not rely only on a
    pre-check.
-   Consider Redis for hot redirects, with a defined cache invalidation
    strategy.
-   Do not make redirect latency depend on expensive analytics
    aggregation.
-   For high traffic, enqueue click events instead of synchronously
    writing every analytic.

### Acceptance criteria

-   Invalid, expired, or disabled links return the documented result.
-   Concurrent requests cannot create duplicate aliases.
-   Analytics are eventually consistent if queued.
-   Redirect and analytics tests are separate.

### Stretch goals

Add QR codes, custom domains, and abuse detection.

------------------------------------------------------------------------

## 4. Product Catalog and Inventory API

**Difficulty:** Beginner → Intermediate\
**Pattern:** Modular monolith with domain boundaries

### Problem statement

Build the product and inventory backend for a small online store.

### Functional requirements

-   Product CRUD for authorized staff.
-   Categories, price, SKU, images, active status.
-   Public product listing with filtering and pagination.
-   Stock adjustments with a reason.
-   Prevent stock from going below zero.
-   Record stock movements for auditability.

### API contract

-   `POST /api/products`
-   `GET /api/products`
-   `GET /api/products/:id`
-   `PATCH /api/products/:id`
-   `POST /api/products/:id/stock-adjustments`
-   `GET /api/products/:id/stock-movements`

### Data model

`Product`: name, description, SKU, price, categoryId, active,
timestamps.\
`Inventory`: productId, quantity, version.\
`StockMovement`: productId, delta, reason, actorId, createdAt.

### Acceptance criteria

-   SKU is unique.
-   Stock adjustments are atomic.
-   Public users cannot mutate inventory.
-   Every adjustment produces an audit record.
-   Search indexes reflect actual filters.

### Stretch goals

Add low-stock alerts and a reservation model for checkout.

------------------------------------------------------------------------

## 5. Appointment Booking API

**Difficulty:** Intermediate\
**Pattern:** Domain service + transaction / concurrency control

### Problem statement

Build a booking system for appointments with staff availability and
fixed-duration time slots.

### Functional requirements

-   Staff define working hours and unavailable periods.
-   Customers view available slots and book one.
-   Customers can cancel within a policy-defined window.
-   Prevent double booking under concurrent requests.
-   Store booking status and cancellation reason.
-   Support timezone-aware display; store instants consistently.

### API contract

-   `POST /api/staff/:id/availability`
-   `GET /api/staff/:id/slots?from=...&to=...`
-   `POST /api/bookings`
-   `GET /api/bookings/me`
-   `POST /api/bookings/:id/cancel`

### Data model

`AvailabilityRule`, `UnavailablePeriod`, `Booking`: staffId, customerId,
startAt, endAt, status, timestamps.

### Important engineering problem

A simple "check if slot is free, then insert" can race. Design a
database-level strategy appropriate to your database: transaction plus
suitable constraints/locking, or a slot-reservation model with unique
slot keys. Explain why your strategy prevents conflicting bookings.

### Acceptance criteria

-   Concurrent booking tests demonstrate that overlapping bookings
    cannot both succeed.
-   Invalid time ranges and past bookings are rejected.
-   Cancellation follows the stated policy.
-   Timezone and daylight-saving assumptions are documented.

### Stretch goals

Add recurring availability, reminders, and waitlists.

------------------------------------------------------------------------

## 6. Expense and Budget Tracker

**Difficulty:** Intermediate\
**Pattern:** Feature modules + aggregation/read models

### Problem statement

Build a personal finance API for income, expenses, budgets, savings
goals, and monthly summaries.

### Functional requirements

-   CRUD income and expense transactions.
-   Categories and transaction date ranges.
-   Monthly budgets by category.
-   Savings goals with target amounts and deadlines.
-   Monthly totals, spending breakdowns, and savings rate.
-   Users only access their own financial records.
-   Store currency explicitly and avoid unsafe floating-point arithmetic
    for money.

### API contract

-   `POST /api/transactions`
-   `GET /api/transactions?month=10&year=2026`
-   `POST /api/budgets`
-   `GET /api/analytics/monthly?month=10&year=2026`
-   `POST /api/goals`
-   `PATCH /api/goals/:id`

### Data model

`Transaction`: ownerId, type, amountMinorUnits, currency, categoryId,
occurredAt, note.\
`Budget`: ownerId, categoryId, period, limitMinorUnits.\
`SavingsGoal`: ownerId, name, targetMinorUnits, currentMinorUnits,
deadline.

### Acceptance criteria

-   Income and expense totals are computed correctly.
-   Date boundaries use a documented timezone.
-   Cross-user access is denied.
-   Budget calculations handle missing categories and zero income.
-   Tests cover rounding and currency assumptions.

### Stretch goals

Add CSV export, recurring transactions, and an AI assistant that answers
questions using only authorized user data.

------------------------------------------------------------------------

## 7. Team Workspace with Role-Based Access Control

**Difficulty:** Intermediate\
**Pattern:** RBAC + policy-based authorization

### Problem statement

Create a collaboration API with workspaces, members, roles, and
resources.

### Functional requirements

-   Users create workspaces and invite members.
-   Roles: owner, admin, member, viewer.
-   Define permissions for reading, creating, editing, inviting, and
    deleting.
-   A user may belong to multiple workspaces with different roles.
-   Membership removal takes effect immediately.
-   Prevent privilege escalation through request-body fields.

### API contract

-   `POST /api/workspaces`
-   `POST /api/workspaces/:id/invitations`
-   `POST /api/invitations/:token/accept`
-   `GET /api/workspaces/:id/members`
-   `PATCH /api/workspaces/:id/members/:userId/role`
-   `DELETE /api/workspaces/:id/members/:userId`

### Data model

`Workspace`, `Membership` (workspaceId, userId, role), `Invitation`
(tokenHash, expiresAt, acceptedAt).

### Acceptance criteria

-   Authorization is checked on every protected resource operation.
-   A viewer cannot mutate resources.
-   The final owner cannot accidentally leave a workspace ownerless.
-   Invitation tokens expire and are single-use.
-   Tests cover a role-permission matrix.

### Stretch goals

Add resource-level sharing and policy-based rules.

------------------------------------------------------------------------

## 8. File Upload and Document Metadata Service

**Difficulty:** Intermediate\
**Pattern:** Adapter / ports-and-adapters for storage providers

### Problem statement

Create an API that accepts document metadata and uploads files to object
storage without tying business logic to one provider.

### Functional requirements

-   Request an upload URL or upload through a controlled endpoint.
-   Validate file size and allowed file types.
-   Save metadata: owner, object key, original filename, content type,
    size, status.
-   Authorize file downloads.
-   Support deletion and orphan-upload cleanup.
-   Never trust a client-provided object key as authorization.

### API contract

-   `POST /api/files/upload-intents`
-   `POST /api/files/:id/complete`
-   `GET /api/files`
-   `GET /api/files/:id/download-url`
-   `DELETE /api/files/:id`

### Architecture sketch

``` text
HTTP Controller
      |
File Service
      |
ObjectStorage Port
      |
  +---+----------------+
  |                    |
S3 Adapter       Local Test Adapter
```

### Acceptance criteria

-   Business logic can be tested with a fake storage adapter.
-   File ownership is checked before generating a download URL.
-   Upload intents expire.
-   Failed uploads do not create falsely completed records.
-   File type checks do not rely only on the filename extension.

### Stretch goals

Add malware scanning, image processing, and background thumbnail
generation.

------------------------------------------------------------------------

## 9. API Rate Limiter and Usage Meter

**Difficulty:** Intermediate\
**Pattern:** Middleware + shared distributed state

### Problem statement

Build a reusable rate-limiting component for an API with different
limits for anonymous, authenticated, and premium clients.

### Functional requirements

-   Support a documented algorithm: fixed window, sliding window, or
    token bucket.
-   Configure limits by route or API plan.
-   Return `429 Too Many Requests` and useful rate-limit headers.
-   Identify clients safely using a trusted authentication identity or
    correctly configured proxy IP.
-   Explain how limits behave across multiple server instances.
-   Avoid an unbounded in-memory map.

### API contract

Apply middleware to sample endpoints such as `GET /api/search` and
`POST /api/auth/login`. Add an optional admin endpoint for viewing usage
totals.

### Acceptance criteria

-   Unit tests cover window boundaries and retry calculations.
-   Shared storage provides consistent limits across instances if
    distributed operation is claimed.
-   Redis failure behavior is explicitly chosen and documented.
-   Proxy headers cannot be spoofed under the deployment configuration.

### Stretch goals

Add per-tenant quotas, daily usage limits, and a usage dashboard.

------------------------------------------------------------------------

## 10. Background Job Processing Service

**Difficulty:** Advanced\
**Pattern:** Queue-based asynchronous processing

### Problem statement

Build a job service for slow work such as generating reports, sending
email, or processing uploaded documents.

### Functional requirements

-   Submit a job and return a job ID.
-   Track queued, processing, completed, failed, and cancelled states
    where supported.
-   Retry transient failures with backoff.
-   Configure retry limits and failed-job handling.
-   Make handlers idempotent.
-   Expose job status and a bounded history.
-   Separate API process from worker process.

### API contract

-   `POST /api/jobs`
-   `GET /api/jobs/:id`
-   `POST /api/jobs/:id/cancel`
-   `GET /api/jobs?status=failed`

### Architecture sketch

``` text
Client -> API -> Database / Queue -> Worker -> External Service
                 ^                     |
                 +---- Job status ------+
```

### Acceptance criteria

-   A worker restart does not silently lose persisted jobs.
-   Duplicate delivery does not duplicate the business effect.
-   Retryable and permanent errors are distinguished.
-   Queue depth, failures, and processing time are observable.
-   Tests can run with a fake worker or test queue.

### Stretch goals

Use Redis + BullMQ, add scheduled jobs, concurrency controls, and a
dead-letter workflow.

------------------------------------------------------------------------

## 11. Real-Time Chat Backend

**Difficulty:** Advanced\
**Pattern:** Event-driven application with WebSockets

### Problem statement

Build a one-to-one and group chat backend with persistent messages and
real-time delivery.

### Functional requirements

-   Create conversations and manage participants.
-   Send and retrieve messages with cursor pagination.
-   Deliver messages over WebSockets.
-   Verify conversation membership before sending or reading.
-   Support message acknowledgements and read receipts.
-   Define reconnect behavior and offline message retrieval.
-   Prevent duplicate messages when clients retry sends.

### API and event contract

HTTP: - `POST /api/conversations` - `GET /api/conversations` -
`GET /api/conversations/:id/messages?cursor=...`

WebSocket events: - `message:send` - `message:new` - `message:ack` -
`message:read`

Document event payloads and error events. Never assume that a socket
connection alone grants access to every room.

### Acceptance criteria

-   Messages are persisted before being acknowledged as durable.
-   A reconnecting client can fetch missed messages.
-   Unauthorized room joins are rejected.
-   Cursor pagination is stable.
-   Duplicate client message IDs are handled idempotently.

### Stretch goals

Add typing indicators, attachments, presence, and multi-instance socket
scaling.

------------------------------------------------------------------------

## 12. Payment and Order Management

**Difficulty:** Advanced\
**Pattern:** State machine + transactional business workflow

### Problem statement

Build checkout and order management using a payment provider's sandbox.
Do not store card numbers or CVVs.

### Functional requirements

-   Create an order from products and quantities.
-   Calculate totals on the server.
-   Reserve inventory during checkout with an expiry policy.
-   Initiate a payment using a provider adapter.
-   Verify signed provider webhooks.
-   Handle duplicate and out-of-order webhook events.
-   Support payment success, failure, cancellation, and refund
    workflows.
-   Maintain an immutable order/payment event history.

### API contract

-   `POST /api/orders`
-   `GET /api/orders/:id`
-   `POST /api/orders/:id/checkout`
-   `POST /api/webhooks/payments/:provider`
-   `POST /api/orders/:id/cancel`

### State model

``` text
PENDING -> PAYMENT_PENDING -> PAID -> FULFILLED
    |             |
    v             v
CANCELLED       FAILED
                  |
                  v
                RETRY

PAID -> REFUND_PENDING -> REFUNDED
```

Specify which transitions are allowed and which actor or event can
trigger them.

### Acceptance criteria

-   Client-supplied totals are never trusted.
-   Webhook signatures are verified against the raw body as required by
    the provider.
-   Duplicate webhook deliveries do not charge or fulfill twice.
-   Inventory and order state remain consistent through failures.
-   Payment and order transitions are tested.

### Stretch goals

Add partial refunds, reconciliation jobs, and an outbox for reliable
downstream events.

------------------------------------------------------------------------

## 13. Webhook Delivery Platform

**Difficulty:** Advanced\
**Pattern:** Outbox + retries + delivery logs

### Problem statement

Build a small platform that lets users register an endpoint and receive
event notifications from your system.

### Functional requirements

-   Register and verify an HTTPS endpoint.
-   Subscribe to event types.
-   Sign outbound payloads with a shared secret.
-   Deliver asynchronously with timeouts and bounded retries.
-   Record status, response code, duration, and attempt count.
-   Support replaying a failed delivery with authorization.
-   Protect against SSRF by validating destinations and blocking
    internal/reserved addresses at registration and connection time.

### API contract

-   `POST /api/webhook-endpoints`
-   `GET /api/webhook-endpoints`
-   `POST /api/webhook-endpoints/:id/test`
-   `GET /api/deliveries/:id`
-   `POST /api/deliveries/:id/replay`

### Acceptance criteria

-   Delivery is at-least-once, and the documentation tells consumers to
    handle duplicates.
-   Retries use exponential backoff with a cap.
-   Secrets are not returned after creation.
-   Delivery attempts are auditable.
-   Network destination validation is not a one-time DNS check only.

### Stretch goals

Add endpoint health scores, per-tenant concurrency limits, and a
delivery replay console.

------------------------------------------------------------------------

## 14. Search API

**Difficulty:** Advanced\
**Pattern:** Search adapter + indexing pipeline

### Problem statement

Build a search service for products, articles, or documents with
filtering and relevance ranking.

### Functional requirements

-   Full-text search with query validation.
-   Filters, sorting, pagination, and highlighting if supported.
-   Search only documents the user is allowed to see.
-   Update the index when source records change.
-   Handle index lag and deleted records.
-   Choose a database-native search capability or a dedicated engine
    based on requirements.

### API contract

`GET /api/search?q=typescript&category=engineering&page=1&limit=20`

### Architecture sketch

``` text
Write API -> Primary Database -> Indexing Event/Worker -> Search Index
                                                        ^
Client -> Search API -----------------------------------|
```

### Acceptance criteria

-   Empty and malformed queries are handled.
-   Authorization filters are applied before returning results.
-   Indexing failures can be retried or reconciled.
-   Ranking and pagination behavior are documented.
-   Tests cover stale and deleted index entries.

### Stretch goals

Add autocomplete, synonyms, typo tolerance, and relevance evaluation.

------------------------------------------------------------------------

## 15. Audit Log and Activity Timeline

**Difficulty:** Advanced\
**Pattern:** Append-only records + domain events

### Problem statement

Create a service that records meaningful actions in a business
application and exposes a searchable timeline.

### Functional requirements

-   Record actor, action, resource type, resource ID, timestamp, and
    correlation ID.
-   Record safe before/after changes when justified.
-   Never log passwords, access tokens, payment secrets, or unnecessary
    sensitive data.
-   Filter by actor, resource, action, and time range.
-   Restrict who can read audit records.
-   Avoid allowing normal application flows to silently rewrite history.

### API contract

-   `GET /api/audit-events?resourceType=invoice&resourceId=...`
-   `GET /api/audit-events?actorId=...&from=...&to=...`

For the exercise, define which events are emitted by application
services and how failures between a business write and audit write are
handled.

### Acceptance criteria

-   Audit entries cannot be edited through public APIs.
-   Sensitive fields are redacted.
-   Event ordering and timestamps have documented semantics.
-   Retention and access policies are documented.
-   Critical business changes cannot silently skip required audit
    records.

### Stretch goals

Use an outbox pattern and add exportable compliance reports.

------------------------------------------------------------------------

## 16. Multi-Tenant SaaS API

**Difficulty:** System design\
**Pattern:** Tenant-aware modular monolith

### Problem statement

Build a SaaS API where organizations share the application
infrastructure but must not access each other's data.

### Functional requirements

-   Create organizations and invite members.
-   Every tenant-owned resource belongs to an organization.
-   Support roles within each organization.
-   Resolve tenant context from a validated membership or trusted token
    claim.
-   Apply tenant scoping consistently to reads, updates, deletes, jobs,
    caches, and logs.
-   Add tenant-level usage limits.
-   Provide a documented data export or deletion process.

### API contract

-   `POST /api/organizations`
-   `POST /api/organizations/:id/members`
-   `GET /api/organizations/:id/projects`
-   `POST /api/organizations/:id/projects`
-   `GET /api/organizations/:id/usage`

### Data model

`Organization`, `Membership`, and tenant-owned resources with
`organizationId`. Decide whether the project needs shared collections or
a separate database per tenant, and justify the decision.

### Acceptance criteria

-   Tenant identity is not trusted merely because a client sends an
    `organizationId`.
-   Cross-tenant access tests cover every resource family.
-   Background jobs preserve tenant context.
-   Cache keys include tenant identity where needed.
-   Indexes reflect tenant-scoped query patterns.

### Stretch goals

Add tenant provisioning, per-tenant feature flags, and usage-based
billing.

------------------------------------------------------------------------

## 17. Approval Workflow Engine

**Difficulty:** System design\
**Pattern:** State machine + configurable workflow rules

### Problem statement

Build a generic approval engine for expenses, leave requests, purchase
orders, or document approvals.

### Functional requirements

-   Define workflow templates with ordered steps.
-   Submit a request into a workflow.
-   Assign approvers based on role or rule.
-   Approve, reject, request changes, and resubmit.
-   Keep a complete decision history.
-   Prevent actions from unauthorized users.
-   Version templates so in-progress requests have predictable behavior
    when definitions change.

### API contract

-   `POST /api/workflow-templates`
-   `POST /api/requests`
-   `GET /api/requests/:id`
-   `POST /api/requests/:id/approve`
-   `POST /api/requests/:id/reject`
-   `POST /api/requests/:id/request-changes`

### State model

``` text
DRAFT -> SUBMITTED -> IN_REVIEW -> APPROVED
                         |            |
                         v            v
                      CHANGES       COMPLETED
                         |
                         v
                     RESUBMITTED

IN_REVIEW -> REJECTED
```

Define whether parallel approvals, delegation, and escalation are in
scope.

### Acceptance criteria

-   Invalid state transitions are rejected.
-   Decisions are recorded with actor and timestamp.
-   Repeated commands are idempotent where appropriate.
-   Workflow template versions are preserved.
-   Concurrent approvals cannot produce inconsistent final state.

### Stretch goals

Add parallel approval branches, timeouts, reminders, and a visual
workflow definition format.

------------------------------------------------------------------------

## 18. Notification Center

**Difficulty:** System design\
**Pattern:** Event-driven notification pipeline

### Problem statement

Create a reusable notification service for in-app, email, and optionally
push notifications.

### Functional requirements

-   Accept notification commands or domain events.
-   Support user preferences and channel preferences.
-   Store in-app notifications with read/unread state.
-   Send email asynchronously.
-   Deduplicate notifications using an idempotency key.
-   Retry temporary provider failures.
-   Track delivery status separately from read status.
-   Respect unsubscribe and consent rules for non-essential messages.

### API contract

-   `GET /api/notifications`
-   `PATCH /api/notifications/:id/read`
-   `POST /api/notifications/read-all`
-   `GET /api/notification-preferences`
-   `PATCH /api/notification-preferences`

### Architecture sketch

``` text
Domain Event -> Notification Service -> Preference Check
                                      -> In-App Store
                                      -> Queue -> Email Adapter
                                                -> Push Adapter
```

### Acceptance criteria

-   Provider failure does not block the originating business
    transaction.
-   Duplicate events do not create duplicate logical notifications.
-   Delivery and read states are distinct.
-   User preferences are enforced.
-   Templates are versioned and variables are validated.

### Stretch goals

Add scheduled digests, quiet hours, and notification priority rules.

------------------------------------------------------------------------

## 19. Analytics and Reporting API

**Difficulty:** System design\
**Pattern:** Read models / CQRS-inspired separation

### Problem statement

Build an analytics API that serves dashboards without making every
request scan the entire operational database.

### Functional requirements

-   Ingest or derive business events.
-   Aggregate metrics by day, week, month, category, and tenant.
-   Support date ranges and timezone rules.
-   Make the freshness of metrics visible.
-   Export a report asynchronously when it is expensive.
-   Prevent analytics queries from leaking data across tenants.
-   Explain whether data is real-time, near-real-time, or batch-updated.

### API contract

-   `GET /api/analytics/overview?from=...&to=...`
-   `GET /api/analytics/trends?metric=revenue&interval=day`
-   `POST /api/reports`
-   `GET /api/reports/:id`

### Architecture sketch

``` text
Operational Writes -> Events/Outbox -> Aggregation Worker
                                          |
                                          v
                                    Read Models
                                          |
Client -> Analytics API ------------------+
```

### Acceptance criteria

-   Metric definitions are documented.
-   Aggregations are idempotent or safely rebuildable.
-   Large reports use bounded resources and job status.
-   Data freshness is returned where useful.
-   Tests verify aggregation correctness against known fixtures.

### Stretch goals

Add materialized views, incremental recomputation, and CSV exports.

------------------------------------------------------------------------

## 20. AI Knowledge Base and Document Search API

**Difficulty:** System design\
**Pattern:** Pipeline architecture + ports/adapters + asynchronous jobs

### Problem statement

Build a backend where users upload documents, the system processes them
into searchable chunks, and users can ask questions grounded in
documents they are authorized to access.

### Functional requirements

-   Create a knowledge base and upload documents.
-   Process files asynchronously: extract text, normalize it, split it
    into chunks, and generate embeddings.
-   Store document metadata, chunk metadata, and vectors in a suitable
    vector store.
-   Search relevant chunks using semantic or hybrid retrieval.
-   Generate answers using a pluggable LLM provider.
-   Return source citations or chunk references with the answer.
-   Enforce document and tenant authorization during retrieval, not just
    at upload time.
-   Track ingestion status, provider errors, and token/cost usage.
-   Support reprocessing and deletion of document chunks and vectors.

### API contract

-   `POST /api/knowledge-bases`
-   `POST /api/knowledge-bases/:id/documents`
-   `GET /api/documents/:id/status`
-   `POST /api/knowledge-bases/:id/search`
-   `POST /api/knowledge-bases/:id/ask`
-   `DELETE /api/documents/:id`

### Architecture sketch

``` text
Upload API -> Object Storage -> Queue -> Ingestion Worker
                                        |
                              Extract -> Chunk -> Embed
                                        |
                                        v
                               Vector / Search Store

Question -> Auth + Tenant Filter -> Retrieval -> LLM Adapter
                                            -> Answer + Sources
```

### Acceptance criteria

-   Upload requests do not wait for full ingestion.
-   Failed ingestion can be retried safely.
-   Retrieval filters enforce the caller's permissions.
-   Answers include source references and can say when evidence is
    insufficient.
-   Prompts treat retrieved document text as untrusted data, not as
    system instructions.
-   Deletion removes or invalidates indexed chunks as specified.
-   LLM timeouts, rate limits, and token budgets are handled.

### Stretch goals

Add hybrid search, reranking, conversation history with scoped access,
evaluation datasets, and per-tenant usage limits.

------------------------------------------------------------------------

# Part II --- Backend architecture patterns: how projects should look

There is no single folder structure that is best for every backend. Use
the simplest architecture that preserves clear responsibilities and
makes the requirements testable.

## Pattern A --- Simple MVC

**Best for:** small CRUD APIs, prototypes, and assignments 1--4 when
scope is limited.

``` text
src/
  models/
  controllers/
  routes/
  middlewares/
  app.ts
  server.ts
```

Request flow:

``` text
Client -> Route -> Controller -> Model -> Database
```

**Advantages:** easy to understand, fast to build.\
**Risks:** controllers can become too large; validation, business rules,
and database operations may become tangled.

**Rule:** keep controllers thin even in MVC. Move non-trivial business
rules into a service.

------------------------------------------------------------------------

## Pattern B --- Layered architecture

**Best for:** most medium-sized APIs, such as tasks, authentication, and
budgets.

``` text
src/
  config/
  routes/
  controllers/
  services/
  repositories/
  models/
  schemas/
  middlewares/
  utils/
  app.ts
  server.ts
```

Request flow:

``` text
HTTP Request
    |
   Route
    |
Controller  <- HTTP parsing and response mapping
    |
  Service   <- business rules and orchestration
    |
Repository  <- data access abstraction
    |
 Model/ORM
    |
 Database
```

Responsibilities:

-   **Route:** method, path, middleware composition.
-   **Controller:** parse validated request data, call service, map
    result to HTTP.
-   **Schema/validator:** validate params, query strings, and bodies.
-   **Service:** enforce business rules and coordinate work.
-   **Repository:** database queries and persistence details.
-   **Model/schema:** database representation and constraints.
-   **Middleware:** authentication, authorization context, request IDs,
    rate limits.
-   **Config:** validated environment variables and external client
    setup.
-   **Utils:** genuinely reusable helpers; avoid using this as a dumping
    ground.

**Dependency direction:** routes depend on controllers; controllers
depend on services; services depend on repository contracts or
implementations. Repositories should not import controllers.

------------------------------------------------------------------------

## Pattern C --- Feature-first modular monolith

**Best for:** growing applications with multiple business domains, such
as FinMate, workspaces, orders, and notifications.

``` text
src/
  config/
  shared/
    errors/
    http/
    logging/
    database/
  modules/
    auth/
      auth.routes.ts
      auth.controller.ts
      auth.service.ts
      auth.repository.ts
      auth.schema.ts
      auth.model.ts
      auth.types.ts
    transactions/
      transaction.routes.ts
      transaction.controller.ts
      transaction.service.ts
      transaction.repository.ts
      transaction.schema.ts
      transaction.model.ts
    budgets/
      budget.routes.ts
      budget.controller.ts
      budget.service.ts
      budget.repository.ts
      budget.schema.ts
      budget.model.ts
  app.ts
  server.ts
```

**Why use it:** files for one feature stay together, while each feature
can keep clear internal layers. It is often easier to navigate than
global folders containing dozens of files.

**Boundary rule:** one module should not directly manipulate another
module's database model. Expose a service or explicit interface for
cross-module behavior.

**Avoid:** creating a microservice for every module before scale or team
boundaries require it.

------------------------------------------------------------------------

## Pattern D --- Clean / hexagonal architecture

**Best for:** complex business rules, provider integrations, and systems
that need easy testing independent of infrastructure.

``` text
src/
  domain/
    entities/
    value-objects/
    policies/
  application/
    use-cases/
    ports/
    dto/
  infrastructure/
    database/
    repositories/
    providers/
    queues/
  interfaces/
    http/
      routes/
      controllers/
      schemas/
  config/
  app.ts
  server.ts
```

Conceptual flow:

``` text
HTTP Adapter -> Application Use Case -> Domain Rules
                       |
                    Port/Interface
                       ^
                       |
              Infrastructure Adapter
               (DB, Queue, Provider)
```

-   **Domain:** business concepts and rules, independent of Express or
    Mongoose.
-   **Application/use case:** orchestrates a business operation.
-   **Port:** interface describing a required capability.
-   **Adapter:** implementation for a database, queue, storage provider,
    or external API.
-   **HTTP interface:** translates HTTP requests and responses.

**Advantages:** strong testability and replaceable infrastructure.\
**Trade-off:** more files and abstractions. Use it when the complexity
earns the overhead.

------------------------------------------------------------------------

## Pattern E --- Event-driven / queue-based architecture

**Best for:** jobs, notifications, analytics, webhooks, and document
ingestion.

``` text
src/
  modules/
    jobs/
    notifications/
    webhooks/
  workers/
    email.worker.ts
    report.worker.ts
  events/
    event.types.ts
    event.publisher.ts
  queues/
    queue.client.ts
  infrastructure/
    database/
    providers/
  app.ts
  server.ts
```

Flow:

``` text
Request -> API -> Persist State -> Publish/Queue Work
                                      |
                                      v
                                    Worker
                                      |
                               External Provider
                                      |
                                      v
                                Update Job State
```

**Important:** a queue gives asynchronous processing, not automatic
exactly-once execution. Design for retries, idempotency, observability,
and failure recovery. For important database writes plus event
publication, study the transactional outbox pattern.

------------------------------------------------------------------------

## Pattern F --- CQRS-inspired read/write separation

**Best for:** analytics-heavy dashboards and reporting.

``` text
Command API -> Business Service -> Primary Write Model
                                      |
                                 Event / Outbox
                                      |
                              Projection Worker
                                      |
                                      v
                                 Read Model
                                      ^
                                      |
Query API ----------------------------+
```

CQRS separates the model used to change state from the model used to
query it. It does **not** require microservices or separate databases.
Start with separate query functions or read models; introduce
infrastructure only when needed.

**Trade-offs:** read models may be eventually consistent, and
projections need replay/rebuild strategies.

------------------------------------------------------------------------

## Pattern G --- State machine / workflow architecture

**Best for:** orders, payments, approvals, bookings, and other
lifecycle-heavy systems.

``` text
Command -> Load Current State -> Validate Transition
                                      |
                                      v
                               Persist New State
                                      |
                                      v
                               Record Domain Event
```

Define:

-   Valid states.
-   Valid transitions.
-   Who may trigger each transition.
-   Preconditions and side effects.
-   Duplicate-command behavior.
-   Recovery after partial failure.

Do not implement lifecycle logic as scattered `if` statements across
controllers. Centralize transition rules in a domain policy or service
and test the transition matrix.

------------------------------------------------------------------------

## Pattern H --- Multi-tenant architecture

**Best for:** B2B SaaS.

``` text
Request
  |
Authenticate User
  |
Resolve Trusted Tenant Membership
  |
Authorize Action
  |
Tenant-Scoped Service
  |
Tenant-Scoped Repository
  |
Database
```

Every query, cache key, queue message, background task, and export must
preserve tenant scope. A tenant ID supplied by a client is not proof of
membership.

------------------------------------------------------------------------

# Part III --- Recommended reusable TypeScript project skeleton

Use this as a starting point for most medium-sized assignments. Adjust
it rather than copying it blindly.

``` text
backend-project/
  src/
    config/
      env.ts
      database.ts
      logger.ts
    modules/
      example/
        example.routes.ts
        example.controller.ts
        example.service.ts
        example.repository.ts
        example.model.ts
        example.schema.ts
        example.types.ts
    middlewares/
      authenticate.ts
      authorize.ts
      validate-request.ts
      error-handler.ts
      not-found.ts
      request-id.ts
    shared/
      errors/
        app-error.ts
      http/
        response.ts
      types/
    app.ts
    server.ts
  tests/
    unit/
    integration/
  docs/
    openapi.yaml
  .env.example
  .gitignore
  package.json
  tsconfig.json
  README.md
```

## Request lifecycle

``` mermaid
flowchart TD
    A[Client Request] --> B[Request ID / Logging]
    B --> C[Route]
    C --> D[Authentication]
    D --> E[Authorization]
    E --> F[Request Validation]
    F --> G[Controller]
    G --> H[Service / Use Case]
    H --> I[Repository]
    I --> J[(Database)]
    J --> I
    I --> H
    H --> G
    G --> K[HTTP Response]
    C -. errors .-> L[Central Error Handler]
    D -. errors .-> L
    F -. errors .-> L
    H -. errors .-> L
```

The exact middleware order depends on the endpoint. Public routes do not
need authentication; validation may run before or after some middleware
depending on the threat model and cost.

## Example module responsibilities

For an `orders` feature:

``` text
orders/
  order.routes.ts       # Defines endpoints and middleware
  order.controller.ts  # Maps HTTP input/output
  order.service.ts      # Checkout and lifecycle rules
  order.repository.ts   # Database operations
  order.model.ts        # Persistence schema/model
  order.schema.ts       # Zod request schemas
  order.types.ts        # Feature-specific types
```

Avoid creating empty files for every layer when a tiny feature does not
need them. Add abstractions when they clarify responsibilities or
isolate change.

------------------------------------------------------------------------

# Part IV --- Database design exercises

For every project, write down:

1.  **Entities:** what records exist?
2.  **Relationships:** one-to-one, one-to-many, or many-to-many?
3.  **Constraints:** which values must be unique or valid?
4.  **Indexes:** which query patterns need efficient access?
5.  **Consistency:** which operations must be atomic?
6.  **Deletion:** hard delete, soft delete, or retention policy?
7.  **Pagination:** offset or cursor, and why?
8.  **Ownership:** how is user or tenant isolation enforced?
9.  **Migrations:** how will the schema change safely?
10. **Failure behavior:** what happens if an external provider or worker
    fails?

## MongoDB + Mongoose practice

Use schema validation, unique indexes, appropriate compound indexes, and
transactions where the deployment and operation require them. Remember
that a Mongoose validation check alone is not a concurrency-safe
substitute for a database unique constraint.

## PostgreSQL + Prisma practice

Use foreign keys, unique constraints, indexes, transactions, and
explicit migration files. Prefer database constraints for invariants
that must hold under concurrency. Learn transaction isolation and how to
handle serialization or uniqueness conflicts.

## Example index discussion

For a task query that commonly filters by owner and status and sorts by
due date, evaluate a compound index aligned with that query pattern,
such as `(ownerId, status, dueDate)`. Confirm with query plans and
actual workload; indexes add write and storage costs.

------------------------------------------------------------------------

# Part V --- Testing checklist

For each project, aim for three levels of testing.

## Unit tests

Test pure business rules and services with dependencies mocked or
replaced by fakes.

Examples: - Cannot cancel a completed order. - A viewer cannot change
workspace settings. - A budget summary handles zero income. - An invalid
workflow transition is rejected.

## Integration tests

Test the real application layer with a test database or isolated test
infrastructure.

Examples: - A duplicate email violates the unique constraint. - A
booking cannot be double-booked. - An upload record is not marked
complete before confirmation. - A webhook replay creates a new delivery
attempt but does not alter the original log.

## API / end-to-end tests

Use Supertest or a similar HTTP client to test the complete request
lifecycle.

For protected routes, test: - No token. - Invalid token. - Valid token
without permission. - Valid token with permission. - Resource not
found. - Resource owned by another user or tenant.

For asynchronous projects, test retry, duplicate delivery, worker
restart, and dead-letter or failed-job handling.

------------------------------------------------------------------------

# Part VI --- Machine-coding interview rubric

Score each category from 0 to 5. A score of 3 means the core requirement
works; 5 means the implementation is robust and well explained.

  Category                 What to evaluate
  ------------------------ -------------------------------------------------------
  Functional correctness   Core requirements and edge cases
  API design               Naming, HTTP semantics, pagination, consistent errors
  Architecture             Clear boundaries and dependency direction
  Data modeling            Constraints, indexes, relationships, migrations
  Security                 Authentication, authorization, validation, secrets
  Reliability              Transactions, idempotency, retries, concurrency
  Testing                  Meaningful unit and integration tests
  Maintainability          Naming, types, small modules, readable code
  Observability            Useful logs, request IDs, failure visibility
  Documentation            Setup, API examples, architecture and trade-offs

**Suggested target:** first make the core flow correct. Then add
security and tests. Only then optimize or add infrastructure.

------------------------------------------------------------------------

# Part VII --- README template for every completed assignment

Copy this structure into the README of each project and fill it in.

``` markdown
# Project Name

## Overview
What problem does this backend solve, and who uses it?

## Requirements
- Functional requirements
- Non-functional requirements
- Explicit assumptions and out-of-scope items

## Tech Stack
- Runtime and language
- Framework
- Database and ORM/ODM
- Validation
- Testing
- External integrations

## Architecture
Include a Mermaid diagram and explain why this pattern fits.

## Data Model
List entities, relationships, constraints, and important indexes.

## API Reference
Document method, path, auth requirement, request example,
success response, and expected errors.

## Environment Variables
Document every required variable using placeholder values only.

## Setup
Explain installation, database setup, migrations, and startup.

## Scripts
Document development, build, test, lint, and migration commands.

## Security
Explain authentication, authorization, validation, and secret handling.

## Failure Handling
Explain transactions, retries, idempotency, and external-provider failure.

## Testing
Describe unit, integration, and API tests.

## Trade-offs
Record alternatives considered and why the selected design was chosen.

## Future Improvements
List the next three improvements in priority order.
```

------------------------------------------------------------------------

# Part VIII --- A practical build order

If you want a sequence that develops your backend skills progressively,
use this order:

1.  **Task Management API** --- learn module boundaries and validation.
2.  **Authentication Service** --- learn sessions, security, and
    middleware.
3.  **URL Shortener** --- learn indexes, caching, and read-heavy
    traffic.
4.  **Inventory API** --- learn invariants and atomic updates.
5.  **Appointment Booking** --- solve concurrency and time handling.
6.  **Expense Tracker** --- practice aggregation and date boundaries.
7.  **RBAC Workspace** --- learn permission matrices.
8.  **File Service** --- isolate storage providers.
9.  **Rate Limiter** --- learn shared state and middleware.
10. **Job Processing** --- learn retries and idempotency.
11. **Chat Backend** --- learn persistent real-time messaging.
12. **Payments and Orders** --- learn state machines and webhooks.
13. **Webhook Platform** --- learn outbound delivery and SSRF
    protection.
14. **Search API** --- learn indexing and authorization-aware retrieval.
15. **Audit Log** --- learn append-only records and reliable event
    publication.
16. **Multi-Tenant SaaS** --- learn tenant isolation.
17. **Workflow Engine** --- learn configurable state transitions.
18. **Notification Center** --- learn event-driven delivery.
19. **Analytics API** --- learn read models and projections.
20. **AI Knowledge Base** --- learn ingestion pipelines, retrieval, and
    provider adapters.

## Definition of done

A project is complete when another developer can clone it, configure it
from `.env.example`, run it, understand the architecture, exercise the
API, and run the tests without asking you how it works.

The goal is not to use the most sophisticated architecture. The goal is
to explain why your architecture, data model, consistency guarantees,
and failure handling fit the problem.
