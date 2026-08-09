# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

This is a **design documentation challenge** for an MBA in AI course. The repository contains a fully functional Order Management System (OMS) in Node.js + TypeScript that serves as the foundation for creating comprehensive technical documentation.

**CRITICAL**: The codebase (`src/`, `prisma/`, `tests/`, configurations) **MUST NOT be modified**. This is a documentation-only deliverable. The code exists solely as reference context for creating design documents.

## The Challenge

Transform a meeting transcript (`TRANSCRICAO.md`) into a complete package of design documents for a new Webhook Notification System feature:
- **PRD** (Product Requirements Document) - `docs/PRD.md`
- **RFC** (Request for Comments) - `docs/RFC.md`
- **FDD** (Feature Design Document) - `docs/FDD.md`
- **ADRs** (Architecture Decision Records) - `docs/adrs/ADR-NNN-*.md`
- **TRACKER** (Traceability Matrix) - `docs/TRACKER.md`

All documentation must be traceable to either the transcript or the existing codebase. No invented requirements.

## Common Development Commands

### Running the Application
```bash
# Start dev server with hot reload
npm run dev

# Build for production
npm run build

# Run production build
npm start
```

### Database Management
```bash
# Run migrations
npm run db:migrate

# Reset database and run migrations
npm run db:reset

# Seed database with test data
npm run db:seed
```

### Testing
```bash
# Run all tests (Vitest)
npm test

# Run tests in watch mode
npm run test:watch
```

### Code Quality
```bash
# Lint TypeScript code
npm run lint

# Format code with Prettier
npm run format
```

### Docker
```bash
# Start MySQL container
docker compose up -d

# Stop containers
docker compose down
```

## Architecture Overview

### Modular Structure

The application follows a **modular, layered architecture** with clear separation of concerns:

```
src/
├── modules/           # Business domain modules
│   ├── auth/         # Authentication (JWT-based)
│   ├── users/        # User management
│   ├── customers/    # Customer management
│   ├── products/     # Product catalog with stock control
│   └── orders/       # Order lifecycle with state machine
├── shared/           # Cross-cutting concerns
│   ├── errors/       # Centralized error handling
│   ├── http/         # HTTP utilities (pagination, responses)
│   └── logger/       # Pino logger with redaction
├── middlewares/      # Express middlewares
│   ├── auth.middleware.ts      # JWT authentication + requireRole
│   ├── error.middleware.ts     # Centralized error handler
│   ├── validate.middleware.ts  # Zod schema validation
│   └── request-logger.middleware.ts
├── routes/           # Route aggregation
├── config/           # Environment and database config
├── app.ts            # Express app factory with DI
└── server.ts         # Entry point
```

### Dependency Injection Pattern

The application uses **manual dependency injection** through factory functions:
- `buildApp(deps: { prisma })` creates the Express app
- `buildControllers(prisma)` instantiates all controllers with their dependencies
- Pattern: Repository → Service → Controller
- No IoC container; dependencies passed via constructors

### Key Architectural Components

#### 1. Order State Machine (`src/modules/orders/order.status.ts`)

The order lifecycle is controlled by a **finite state machine** with explicit transition rules:

```
PENDING → PAID → PROCESSING → SHIPPED → DELIVERED
       ↘ CANCELLED ↙
```

**Critical functions:**
- `canTransition(from, to)` - Validates transitions
- `shouldDebitStock(from, to)` - Triggers on PENDING → PAID
- `shouldReplenishStock(from, to)` - Triggers on PAID/PROCESSING → CANCELLED

**Integration point for webhooks**: The `changeStatus` method in `OrderService` (src/modules/orders/order.service.ts:126) executes status transitions within a Prisma transaction and is the natural place to integrate event publishing.

#### 2. Error Handling Pattern

All errors extend `AppError` (src/shared/errors/app-error.ts) with:
- `statusCode` - HTTP status
- `errorCode` - Machine-readable string (e.g., `INVALID_STATUS_TRANSITION`)
- `details` - Optional structured context

**Standard error classes** (src/shared/errors/http-errors.ts):
- `ValidationError` (400, `VALIDATION_ERROR`)
- `UnauthorizedError` (401, `UNAUTHORIZED`)
- `ForbiddenError` (403, `FORBIDDEN`)
- `NotFoundError` (404, `NOT_FOUND`)
- `ConflictError` (409, `CONFLICT`)
- `UnprocessableEntityError` (422, `UNPROCESSABLE_ENTITY`)

**Domain-specific errors**:
- `InvalidStatusTransitionError` - Illegal state transition
- `InsufficientStockError` - Stock unavailable

**Error code pattern**: `UPPERCASE_SNAKE_CASE` with optional prefixes (e.g., `WEBHOOK_DELIVERY_FAILED`)

The `errorMiddleware` (src/middlewares/error.middleware.ts) catches all errors and formats JSON responses.

#### 3. Authentication & Authorization

**JWT-based authentication** (src/middlewares/auth.middleware.ts):
- `authenticate` middleware: Validates `Authorization: Bearer <token>`
- `requireRole(...roles)` middleware: Enforces role-based access (ADMIN, OPERATOR)
- Token payload: `{ sub: userId, email, role, iat, exp }`

**Integration point for webhooks**: Authentication will need HMAC-SHA256 signatures instead of JWT.

#### 4. Transaction Management

Prisma transactions using `prisma.$transaction(async (tx) => { ... })`:
- **Order creation** (src/modules/orders/order.service.ts:58): Creates order + items + history in one transaction
- **Status changes** (src/modules/orders/order.service.ts:131): Updates order + stock + history atomically

**Integration point for webhooks**: Outbox pattern will extend these transactions to include event persistence.

#### 5. Logging

**Pino logger** (src/shared/logger/index.ts) with:
- Automatic redaction of sensitive fields (`password`, `token`, `authorization`)
- Request ID correlation via `pino-http`
- Pretty-print in development, structured JSON in production

#### 6. Database Schema (Prisma)

Key models in `prisma/schema.prisma`:
- **User**: Authentication + audit trail (`role`, `passwordHash`)
- **Customer**: Customer data with JSON address
- **Product**: Catalog with `stockQuantity` and `active` flag
- **Order**: Order header with state machine (`status`, `orderNumber`)
- **OrderItem**: Line items with captured prices
- **OrderStatusHistory**: Complete audit trail of all status changes with `reason` and `changedBy`

**No events, queues, or webhooks** exist yet. This is intentional - the feature being documented will add them.

## Design Document Constraints

When creating documentation for the webhook feature:

1. **Traceability is mandatory**: Every requirement, decision, or constraint must be traceable to `TRANSCRICAO.md` (with timestamp) or a specific file path in the codebase.

2. **Reference real code paths**:
   - State machine: `src/modules/orders/order.status.ts`
   - Transaction logic: `src/modules/orders/order.service.ts:126` (changeStatus method)
   - Error classes: `src/shared/errors/http-errors.ts`
   - Auth patterns: `src/middlewares/auth.middleware.ts:49` (requireRole)
   - Error codes: All follow `UPPERCASE_SNAKE_CASE` pattern
   - Logger: `src/shared/logger/index.ts`

3. **Error code pattern for webhooks**: All webhook-related error codes should follow `WEBHOOK_*` prefix (e.g., `WEBHOOK_DELIVERY_FAILED`, `WEBHOOK_INVALID_SIGNATURE`).

4. **Integration points to reference**:
   - Extend `changeStatus` transaction to persist events to outbox
   - Reuse `AppError` hierarchy for webhook-specific errors
   - Reuse Pino logger for observability
   - Extend Prisma schema with new tables (but don't modify it - just document)

5. **What NOT to document**: Items explicitly rejected or deferred in the meeting transcript.

## Test Structure

Tests use **Vitest** with:
- In-memory SQLite for integration tests (tests/setup.ts)
- Supertest for HTTP endpoint testing
- Test helpers in `tests/helpers/`

Current test coverage:
- `tests/auth.test.ts` - Authentication flows
- `tests/orders.test.ts` - Order lifecycle and state machine

## Important Notes

- **Node.js 20+** required (ESM modules with `.js` extensions in imports)
- **MySQL 8.0** in production (via Docker), SQLite in tests
- **Zod** for runtime validation (schemas in `*.schemas.ts` files)
- **UUID** (v4) for all entity IDs
- All monetary values stored in **cents** (integer precision)
- Timestamps use MySQL `DATETIME` with `@default(now())` and `@updatedAt`
