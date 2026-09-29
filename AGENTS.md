# AGENTS.md — Shopping Core Service

> **Target Stack:** NestJS · TypeScript · TypeORM · PostgreSQL · Redis · Kafka · JWT/OAuth2
>
> This file is the single source of truth for any AI coding agent scaffolding or extending this service. Follow every section exactly.

---

## 1. Stack

| Layer | Technology | Role |
|---|---|---|
| Runtime | Node.js 20 LTS | Execution environment |
| Framework | NestJS 10 | HTTP server, DI container, module system |
| Language | TypeScript 5 (strict mode) | Primary language; all files `.ts` |
| ORM | TypeORM 0.3 | Entity definitions, migrations, repositories |
| Database | PostgreSQL 16 | Primary relational store |
| Cache | Redis 7 (via `ioredis`) | Read-model caching for catalog & wishlists |
| Auth | `@nestjs/passport` + `passport-jwt` + `passport-oauth2` | JWT guard, OAuth2 strategy |
| Validation | `class-validator` + `class-transformer` | DTO validation pipeline |
| Events | `kafkajs` (NestJS Kafka microservice transport) | Non-blocking analytics event emission |
| Config | `@nestjs/config` + `joi` | Env-var loading and schema validation |
| Logging | `winston` + `nest-winston` | Structured JSON logs |
| Testing | Jest 29 + Supertest | Unit, integration, e2e tests |
| API Docs | `@nestjs/swagger` | OpenAPI 3.1 spec auto-generation |
| Linting | ESLint 8 + Prettier 3 | Code style enforcement |
| Migrations | TypeORM CLI | Schema versioning |
| Containerisation | Docker + docker-compose | Local dev and CI |

---

## 2. Project Structure

```
shopping-core-service/
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions CI pipeline
├── docker/
│   ├── Dockerfile                  # Production multi-stage image
│   └── Dockerfile.dev              # Dev image with hot-reload
├── docker-compose.yml              # Local dev: app + postgres + redis + kafka
├── docker-compose.test.yml         # CI: isolated test containers
├── .env.example                    # All required env vars with placeholders
├── .eslintrc.js                    # ESLint config (extends @nestjs)
├── .prettierrc                     # Prettier config
├── jest.config.ts                  # Root Jest config (unit + integration)
├── jest.e2e.config.ts              # E2E Jest config
├── tsconfig.json                   # Base TS config (strict: true)
├── tsconfig.build.json             # Build TS config (excludes tests)
├── package.json
├── tasks.md                        # Agent work-tracking file (agent creates this)
└── src/
    ├── main.ts                     # Bootstrap: app factory, Swagger, global pipes
    ├── app.module.ts               # Root module; imports all feature modules
    ├── app.controller.ts           # Health-check endpoint GET /health
    ├── app.controller.spec.ts      # Unit test for health controller
    │
    ├── config/
    │   ├── app.config.ts           # App-level config factory (port, env)
    │   ├── database.config.ts      # TypeORM DataSource config factory
    │   ├── redis.config.ts         # Redis connection config factory
    │   ├── kafka.config.ts         # Kafka producer config factory
    │   └── config.validation.ts    # Joi schema for all env vars
    │
    ├── database/
    │   ├── data-source.ts          # TypeORM DataSource for CLI migrations
    │   └── migrations/             # All TypeORM migration files (auto-named)
    │       └── .gitkeep
    │
    ├── common/
    │   ├── decorators/
    │   │   ├── current-user.decorator.ts   # @CurrentUser() param decorator
    │   │   └── public.decorator.ts         # @Public() route decorator
    │   ├── filters/
    │   │   └── http-exception.filter.ts    # Global exception → RFC 7807 response
    │   ├── guards/
    │   │   ├── jwt-auth.guard.ts           # Global JWT guard (skips @Public)
    │   │   └── roles.guard.ts              # Role-based access guard
    │   ├── interceptors/
    │   │   ├── logging.interceptor.ts      # Request/response structured logging
    │   │   └── cache.interceptor.ts        # Redis cache-aside read interceptor
    │   ├── pipes/
    │   │   └── validation.pipe.ts          # Global class-validator pipe config
    │   ├── pagination/
    │   │   ├── pagination.dto.ts           # PageOptionsDto (page, limit, sort)
    │   │   └── pagination.response.ts      # PageDto<T> generic wrapper
    │   └── types/
    │       └── authenticated-request.ts    # Express Request + user payload type
    │
    ├── users/
    │   ├── users.module.ts
    │   ├── users.controller.ts         # POST /users/register
    │   ├── users.controller.spec.ts
    │   ├── users.service.ts            # Registration logic, password hashing
    │   ├── users.service.spec.ts
    │   ├── users.repository.ts         # TypeORM custom repository wrapper
    │   ├── entities/
    │   │   └── user.entity.ts          # @Entity User: id, email, mobile, passwordHash, role, timestamps
    │   └── dto/
    │       ├── register-user.dto.ts    # email, mobile, password — class-validator rules
    │       └── user-response.dto.ts    # Outbound shape (no passwordHash)
    │
    ├── auth/
    │   ├── auth.module.ts
    │   ├── auth.controller.ts          # POST /auth/login, POST /auth/refresh
    │   ├── auth.controller.spec.ts
    │   ├── auth.service.ts             # Validate credentials, issue JWT
    │   ├── auth.service.spec.ts
    │   ├── strategies/
    │   │   ├── jwt.strategy.ts         # passport-jwt strategy
    │   │   └── oauth2.strategy.ts      # passport-oauth2 strategy (optional provider)
    │   └── dto/
    │       ├── login.dto.ts
    │       └── token-response.dto.ts
    │
    ├── catalog/
    │   ├── catalog.module.ts
    │   ├── categories/
    │   │   ├── categories.controller.ts    # GET /categories, GET /categories/:id/products
    │   │   ├── categories.controller.spec.ts
    │   │   ├── categories.service.ts
    │   │   ├── categories.service.spec.ts
    │   │   ├── categories.repository.ts
    │   │   └── entities/
    │   │       └── category.entity.ts      # id, name, slug, parentCategory, products
    │   ├── products/
    │   │   ├── products.controller.ts      # GET /products, GET /products/:id, GET /products/search
    │   │   ├── products.controller.spec.ts
    │   │   ├── products.service.ts         # Browse, search (FTS), view-event emit
    │   │   ├── products.service.spec.ts
    │   │   ├── products.repository.ts      # FTS query builder; swap-point for search adapter
    │   │   └── entities/
    │   │       └── product.entity.ts       # id, name, slug, description, price, stock, category, tsvector column
    │   └── dto/
    │       ├── product-response.dto.ts
    │       ├── product-search.dto.ts       # query, categoryId, page, limit
    │       └── category-response.dto.ts
    │
    ├── wishlist/
    │   ├── wishlist.module.ts
    │   ├── wishlist.controller.ts      # GET /wishlist, POST /wishlist/items, DELETE /wishlist/items/:productId
    │   ├── wishlist.controller.spec.ts
    │   ├── wishlist.service.ts         # Add/remove logic, consolidated view, cache invalidation
    │   ├── wishlist.service.spec.ts
    │   ├── wishlist.repository.ts
    │   └── entities/
    │       └── wishlist-item.entity.ts  # id, user (FK), product (FK), addedAt
    │
    ├── analytics/
    │   ├── analytics.module.ts
    │   ├── analytics.service.ts        # fire-and-forget Kafka producer methods
    │   ├── analytics.service.spec.ts
    │   └── events/
    │       ├── base.event.ts           # eventId, timestamp, version, source
    │       ├── user-registered.event.ts
    │       ├── user-logged-in.event.ts
    │       ├── product-viewed.event.ts
    │       ├── product-searched.event.ts
    │       ├── wishlist-item-added.event.ts
    │       └── wishlist-item-removed.event.ts
    │
    └── cache/
        ├── cache.module.ts             # Global Redis module (ioredis)
        └── cache.service.ts            # get/set/del/invalidatePattern helpers
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder.

### Step 1 — Read Specifications
- Read all story-level spec documents provided in the task context before writing any code.
- Extract: entities, relationships, business rules, validation constraints, event names, cache TTLs, and API contracts.
- Do not invent requirements not present in the specs.

### Step 2 — Create `tasks.md`
Create `tasks.md` in the project root. Structure it as:

```markdown
# Tasks

## Backlog
- [ ] <task>

## In Progress
- [ ] <task>

## Done
- [x] <task>
```

Break work into atomic tasks (one module/feature per task). Update `tasks.md` as each task moves state. Commit `tasks.md` changes with every substantive commit.

### Step 3 — Scaffold Infrastructure First
Order of creation:
1. `package.json` — exact dependency versions pinned
2. `tsconfig.json` / `tsconfig.build.json`
3. `.eslintrc.js` / `.prettierrc`
4. `jest.config.ts` / `jest.e2e.config.ts`
5. `.env.example` — every env var the app reads
6. `src/config/` — all config factories + Joi validation schema
7. `docker-compose.yml` — postgres, redis, kafka (zookeeper), app
8. `docker/Dockerfile` (multi-stage)
9. `src/database/data-source.ts` + initial migration scaffold

### Step 4 — Implement Feature Modules
Implement in dependency order:
1. `common/` — shared pipes, guards, filters, interceptors, decorators
2. `cache/` — Redis module and service
3. `analytics/` — Kafka producer module and service
4. `users/` — entity → repository → service → controller → DTOs
5. `auth/` — strategies → service → controller
6. `catalog/categories/` — entity → repository → service → controller
7. `catalog/products/` — entity → repository → service → controller (include FTS)
8. `wishlist/` — entity → repository → service → controller
9. Wire all modules into `app.module.ts`
10. `main.ts` — global pipes, filters, Swagger setup, Kafka microservice hybrid

For each module, create the entity and run `npm run migration:generate -- src/database/migrations/<Name>` before writing service logic.

### Step 5 — Write Tests Alongside Implementation
- Write the `.spec.ts` unit test file **before or immediately after** each service/controller file.
- Write e2e tests in `test/` after all modules are complete.
- Run `npm run test:cov` — must show ≥ 90% line/branch/function coverage before proceeding.

### Step 6 — Validate
Run all checks and fix failures before marking tasks done:

```bash
npm run lint          # ESLint — zero warnings allowed
npm run format:check  # Prettier — no diffs
npm run build         # tsc compilation — zero errors
npm run test:cov      # Jest unit + integration — ≥ 90% coverage
npm run test:e2e      # Supertest e2e — all passing
docker compose up --build -d && sleep 10 && npm run test:e2e
```

### Step 7 — Final Checklist
- [ ] All `tasks.md` items marked `Done`
- [ ] No `TODO`, `FIXME`, or `console.log` in production code
- [ ] `.env.example` documents every variable used in `src/config/`
- [ ] Swagger UI reachable at `GET /api/docs`
- [ ] `GET /health` returns `200 { status: "ok" }`
- [ ] All migrations applied cleanly on a fresh DB
- [ ] Docker image builds without errors

---

## 4. Coding Conventions

### General TypeScript
- `strict: true` in `tsconfig.json` — no exceptions.
- No `any`. Use `unknown` and narrow, or define explicit types.
- Prefer `readonly` on entity fields and DTO properties where mutation is not needed.
- All async functions must return typed `Promise<T>` — no implicit `Promise<any>`.
- Use `const` by default; `let` only when reassignment is required.

### NestJS Architecture
- **One module per domain** (`users`, `auth`, `catalog`, `wishlist`, `analytics`, `cache`).
- **No cross-module direct service injection** — expose only what is needed via the module's `exports` array.
- Use NestJS **custom providers** (`useFactory`) for all external clients (TypeORM, ioredis, KafkaJS).
- Decorate every controller method with `@ApiOperation`, `@ApiResponse`, and `@ApiBearerAuth` (Swagger).
- Use `@Public()` decorator on routes that bypass the global JWT guard.

### Naming Conventions
| Artifact | Convention | Example |
|---|---|---|
| Files | `kebab-case.type.ts` | `wishlist.service.ts` |
| Classes | `PascalCase` | `WishlistService` |
| Interfaces | `PascalCase` prefixed `I` | `IWishlistRepository` |
| DTOs | `PascalCase` + `Dto` suffix | `RegisterUserDto` |
| Entities | `PascalCase` + `Entity` suffix | `UserEntity` |
| Enums | `PascalCase` + `Enum` suffix, `UPPER_SNAKE` values | `UserRoleEnum.ADMIN` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_WISHLIST_ITEMS` |
| Environment vars | `UPPER_SNAKE_CASE` | `DATABASE_URL` |
| Database tables | `snake_case` plural | `wishlist_items` |
| Database columns | `snake_case` | `password_hash` |

### Entities (TypeORM)
- Use `@Entity({ name: 'table_name' })` with explicit table names.
- Every entity extends a `BaseEntity` abstract class with `id` (UUID v4, `@PrimaryGeneratedColumn('uuid')`), `createdAt`, `updatedAt` (`@CreateDateColumn`, `@UpdateDateColumn`).
- Use `@Index` on all foreign keys and frequently queried columns.
- Define `@Column` with explicit `type`, `nullable`, and `default` — never rely on TypeORM inference.
- No business logic inside entity classes.

### DTOs
- Every inbound DTO uses `class-validator` decorators; every property is explicitly typed.
- Every outbound