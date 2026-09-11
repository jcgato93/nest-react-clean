---
name: nest-clean
description: Use when implementing any feature, module, or endpoint in this NestJS + Clean Architecture codebase — whether creating from scratch or modifying existing code. Trigger for phrases like "create module", "add endpoint", "add field", "nueva funcionalidad", "agregar funcionalidad", "crear módulo", "agregar un campo a", "modificar la entidad", "agregar endpoint", or any request to add or change business behavior. Also use as the architecture reference for naming, layer decisions, and file placement. Use before creating any file in src/.
---

## Architecture at a Glance

**Domain → Application → Infrastructure** (dependencies flow inward only, never outward).

### Shared elements (root level)

| Layer | Location | Contains |
|-------|----------|----------|
| Shared Domain | `src/domain/*` | Common value objects, base exceptions, shared interfaces |
| Shared Infrastructure | `src/infrastructure/*` | `PrismaService`/`PrismaModule`, base entity, base repository impl, external services (Redis, Auth0, etc.) |

### Per-module elements (inside `src/modules/`)

Each new module lives entirely inside `src/modules/{module}/`:

```
src/modules/{module}/
  domain/
    entities/
    repositories/
    use-cases/
    exceptions/
    value-objects/       (if needed)
  application/
    dtos/
    mappers/
    use-cases/
  infrastructure/
    repositories/
    controllers/
  {module}.module.ts
```

> **Models** live in the single `prisma/schema.prisma` file, not per-module TS classes. There's nothing to register per module — `PrismaModule` is `@Global()`, so a repository just injects `PrismaService` directly.

---

## Sub-Skill Reference

`nest-clean` is the orchestrator — it never implements a layer itself. For each layer, invoke the matching sub-skill below **before** writing any code in that layer:

| Task | Skill to invoke |
|------|------------------|
| Entity, value object, repository/use-case interface, exception | `domain-layer` |
| Use case implementation, DTOs, mapper | `application-layer` |
| Prisma model, repository impl, controller, module, migration | `infrastructure-layer` |
| Domain events, AggregateRoot, event listeners/handlers | `domain-events` |
| New/modified environment variable, `envs.ts`, external service config | `environment-config` |
| Code review / PR | `code-review-instructions` |

These are separate, model-invoked skills distributed from the same `jcgato93/nest-react-clean` repo — they are not files inside this skill's own directory, and their install location varies by agent/setup, so never hardcode a path to them (e.g. `.github/skills/...` or `.claude/skills/...`).

**Before starting a layer, confirm its skill is actually available** (check the current list of loaded/available skills). If the skill this step needs is missing:

1. Stop before writing any code for that layer.
2. Tell the user, e.g.:
   > The `domain-layer` skill isn't installed, so I can't safely follow its rules for entities/value-objects/exceptions. Install it with:
   > `npx skills@latest add jcgato93/nest-react-clean --skill domain-layer` (or `--all` to install every skill in the set).
3. Wait for the user to install it (or explicitly say to proceed without it) before continuing.

---

## Scenario Detection

Identify which scenario applies and confirm with the user before writing any code.

---

### Scenario A — New Module from Scratch

**Confirm with the user using this template:**

```
Before I start, let me confirm the module details:

- **Module name**: `{name}` → DB table: `{names}` (snake_case, plural)
- **Entity fields**: {field — type — required/optional}
- **Operations**: {e.g., create, get by id, list by company (paginated), update, delete}
- **Authentication**: Auth0Guard on all endpoints — let me know if any should be public
- **Pagination**: list endpoint returns paginated results

Does this look correct? I'll start with the domain layer once confirmed.
```

Then follow Steps 1 → 2 → 3 in order. Never start the next step until the current one is complete and verified.

**Step 1 — Domain Layer** *(invoke the `domain-layer` skill first — see [Sub-Skill Reference](#sub-skill-reference) if it's not available)*
- `src/modules/{module}/domain/entities/{entity}.domain.ts`
- `src/modules/{module}/domain/repositories/{module}.repository.ts`
- `src/modules/{module}/domain/use-cases/{action}.use-case.ts` (one per operation)
- `src/modules/{module}/domain/exceptions/*.exception.ts`
- `src/modules/{module}/domain/value-objects/*.value-object.ts` (if needed)

**Step 2 — Application Layer** *(invoke the `application-layer` skill first — see [Sub-Skill Reference](#sub-skill-reference) if it's not available)*
*(start only after Step 1 is complete)*
- `src/modules/{module}/application/dtos/create-{entity}.dto.ts`
- `src/modules/{module}/application/dtos/{entity}-response.dto.ts`
- `src/modules/{module}/application/mappers/{entity}.mapper.ts`
- `src/modules/{module}/application/use-cases/{action}.use-case.impl.ts`

**Step 3 — Infrastructure Layer** *(invoke the `infrastructure-layer` skill first — see [Sub-Skill Reference](#sub-skill-reference) if it's not available)*
*(start only after Step 2 is complete)*
- Add the `model` block to `prisma/schema.prisma` + generate migration (`pnpm run prisma:migrate:dev --name ...`)
- `src/modules/{module}/infrastructure/repositories/{entity}.repository.impl.ts`
- `src/modules/{module}/infrastructure/controllers/{module}.controller.ts`
- `src/modules/{module}/{module}.module.ts` + register in `AppModule`

---

### Scenario B — Modify Existing Functionality

**Before proposing any change, read these files first:**

| Modification type | Read first |
|-------------------|------------|
| Add field to entity | Domain entity + mapper + response DTO + DB entity |
| Add new operation/endpoint | Domain use case interface + controller + module |
| Change business rule | Domain entity + value object (if applicable) |
| Add relation between entities | Domain entity + repository interface + DB entity |
| Change response shape | Response DTO + mapper + controller |
| Add cache to a use case | Use case impl + `src/common/constants/enums/redis-key.enum.ts` |

**Then apply in order:**

1. **Read first** — read every affected file before proposing any change
2. **Identify scope** — use the impact table to catch all layers affected; a change to one layer almost always ripples to adjacent ones
3. **Apply Domain → Application → Infrastructure** — never skip inward layers
4. **Verify consistency** — after each layer, check that mappers, DTOs, and module registration are still consistent
5. **Generate migration** if the Prisma schema changed

#### Impact Table

| Requested change | Affected layers | Migration |
|-----------------|-----------------|-----------|
| Add field to entity | Domain → Application → Infrastructure (Prisma model + mappers + DTO) | Yes |
| Add new operation/endpoint | Domain (use case interface) → Application (impl) → Infrastructure (controller + module) | No |
| Change validation or business rule | Domain (entity / value object / exception) + Application (if rule is validated there) | No |
| Add relation between entities | Domain (entity + repository interface) → Infrastructure (Prisma model + repository impl + migration) | Yes |
| Change endpoint response shape | Application (response DTO + mapper) → Infrastructure (controller if return type changes) | No |
| Add cache to a use case | Application only (use case impl + RedisKeyEnum if new) | No |
| Rename field in DB | Infrastructure (Prisma model + mappers) | Yes |

For layer-specific implementation details, invoke the corresponding sub-skill (see [Sub-Skill Reference](#sub-skill-reference)).

---

### Scenario C — Hybrid (field change + new operation in the same request)

When a request touches both scenarios (e.g., "add a `serialNumber` field AND add a `transfer` endpoint"):

1. Handle the **field change** first — complete all layers including migration
2. Then implement the **new operation** on top of the updated schema

Never interleave the two changes — complete one before starting the other.

---

## Worked Example — "módulo de mantenimientos" → `asset-maintenance`

User prompt in Spanish → all code in English. A condensed trace of Scenario A for create, get by id, and list by company.

**Step 1 — Domain**
```
src/modules/asset-maintenance/domain/entities/asset-maintenance.domain.ts        → AssetMaintenance entity (id, description, cost, date, type, companyId)
src/modules/asset-maintenance/domain/repositories/asset-maintenance.repository.ts → AssetMaintenanceRepository (adds getListByCompany)
src/modules/asset-maintenance/domain/use-cases/create-asset-maintenance.use-case.ts
src/modules/asset-maintenance/domain/use-cases/get-asset-maintenance.use-case.ts
src/modules/asset-maintenance/domain/use-cases/get-asset-maintenance-list.use-case.ts
src/modules/asset-maintenance/domain/exceptions/asset-maintenance-not-found.exception.ts
  → super('Maintenance record with ID ${id} not found.')  ← message in English
src/common/constants/enums/maintenance-type.enum.ts              → MaintenanceType { PREVENTIVE, CORRECTIVE, WARRANTY }
```

**Step 2 — Application**
```
src/modules/asset-maintenance/application/dtos/create-asset-maintenance.dto.ts
src/modules/asset-maintenance/application/dtos/asset-maintenance-response.dto.ts
src/modules/asset-maintenance/application/mappers/asset-maintenance.mapper.ts
src/modules/asset-maintenance/application/use-cases/create-asset-maintenance.use-case.impl.ts
src/modules/asset-maintenance/application/use-cases/get-asset-maintenance.use-case.impl.ts
src/modules/asset-maintenance/application/use-cases/get-asset-maintenance-list.use-case.impl.ts
```

**Step 3 — Infrastructure**
```
prisma/schema.prisma                                                          → model AssetMaintenance { ... @@map("asset_maintenances") }
src/modules/asset-maintenance/infrastructure/repositories/asset-maintenance.repository.impl.ts
src/modules/asset-maintenance/infrastructure/controllers/asset-maintenance.controller.ts
src/modules/asset-maintenance/asset-maintenance.module.ts                    → registered in AppModule
migration: CreateAssetMaintenancesTable
```

---

## Naming Conventions

All code identifiers and file names are **always in English** — even when the user describes the feature in Spanish. Translate the concept to English before naming files and classes.

| Element | Convention | Example |
|---------|-----------|---------|
| Files | `kebab-case`, English | `asset-maintenance-not-found.exception.ts` |
| Classes | `PascalCase` + suffix, English | `CreateAssetMaintenanceUseCaseImpl` |
| Variables/functions | `camelCase`, verbs, English | `findByCompanyId()` |
| DB tables | `snake_case`, plural, English | `asset_maintenances`, `company_assets` |
| Enums | name `PascalCase`, values `UPPERCASE`, English | `MaintenanceType.PREVENTIVE` |
| Comments | Spanish | `// Verificar si el mantenimiento existe` |
| Exception messages | English (updated 2026-08-08 — product/API is English-only) | `'Maintenance record with ID ${id} not found.'` |

**Translation examples** — user says Spanish, code uses English:

| User says | Module name in code |
|-----------|-------------------|
| "módulo de mantenimientos" | `asset-maintenance` / `AssetMaintenance` |
| "crear módulo de depreciaciones" | `depreciation` / `Depreciation` |
| "agregar conciliación fiscal" | `fiscal-conciliation` / `FiscalConciliation` |

---

## Critical Rules

**Layer boundaries — never import from outer layers into inner ones:**
```ts
// ❌ WRONG — domain importing from infrastructure
import { AssetMaintenanceEntity } from '@/infrastructure/database/entities/asset-maintenance.entity';

// ✅ CORRECT — domain is framework-agnostic
import { Money } from '@/domain/common/value-objects/money.value-object';
```

**Controllers inject use cases — never repositories directly:**
```ts
// ❌ WRONG
constructor(private readonly maintenanceRepo: AssetMaintenanceRepository) {}

// ✅ CORRECT
constructor(private readonly createMaintenance: CreateAssetMaintenanceUseCaseImpl) {}
```

**Throw domain exceptions only — never `HttpException` in use cases:**
```ts
// ❌ WRONG — HTTP concern in the application layer
throw new NotFoundException('Mantenimiento no encontrado');

// ✅ CORRECT — domain exception, HTTP mapping handled by a filter
throw new AssetMaintenanceNotFoundException(id);
```

**Other rules:**
- Prisma `select`/`include` always use object notation: `include: { company: true }` — there is no array form
- Never use `@nestjs/config` — all config lives in `src/infrastructure/config/envs.ts` (invoke the `environment-config` skill before touching it)
- Never skip migrations when the Prisma schema changes
- All error messages in English (product/API is English-only)

---

## Developer Commands

```bash
# Migrations
pnpm run prisma:migrate:dev --name DescribeChange   # generate + apply + regenerate client
pnpm run prisma:migrate:deploy                       # apply pending migrations (CI/CD)
pnpm run prisma:generate                             # regenerate client only
pnpm run prisma:studio                                # inspect data

# Development
npm run start:dev
npm run lint
npm run format
```

---

## Key Files

| Purpose | Path |
|---------|------|
| Prisma schema (all models) | `prisma/schema.prisma` |
| `PrismaService` / `PrismaModule` | `src/infrastructure/prisma/` |
| Base repository impl | `src/infrastructure/database/base.repository.impl.ts` |
| Base entity constraint (id, createdAt, updatedAt) | `src/infrastructure/database/base.entity.ts` |
| Repository + transaction usage guide | `src/infrastructure/database/repositories.md` |
| Redis key enum | `src/common/constants/enums/redis-key.enum.ts` |
| Cache service | `src/infrastructure/external/services/cache/cache.service.ts` |
| Auth0 guard | `src/common/guards/auth0.guard.ts` |
| Decorators | `src/common/decorators/{user-id,company-id,public}.decorator.ts` |
| Pagination | `src/application/pagination/dtos/page-options.dto.ts` |
| Environment config | `src/infrastructure/config/envs.ts` |

---

## Final Verification

After completing all layers:

1. `npm run start:dev` — no TypeScript compilation errors
2. `npm run lint` — no lint errors
3. Swagger at `/api/docs` — new endpoints appear with correct request/response shapes
4. Review the generated SQL in `prisma/migrations/<timestamp>_.../migration.sql` before applying it — fix anything Prisma inferred wrong (e.g. a destructive `DROP`+`ADD` where a `RENAME` was intended)
5. Confirm the new model is in `prisma/schema.prisma` and the client was regenerated (`pnpm run prisma:migrate:dev` does this automatically)
6. Confirm the new `{module}.module.ts` is imported in `AppModule`
