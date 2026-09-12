---
name: infrastructure-layer
description: >
  Use when creating or modifying infrastructure layer components in this NestJS project: database models/entities (Prisma or TypeORM), repository implementations, API controllers, NestJS modules, or external service integrations. Domain and application layers should be defined first. Use this skill for requests like "create a controller for X", "implement repository for Y", "add a model for Z", "generate a migration", "register module", "create NestJS module", "add endpoint". Always use when code needs to go in src/infrastructure/ or src/modules/{module}/infrastructure/.
---

## Infrastructure Layer Overview

The **Infrastructure Layer** implements technical details: database persistence, HTTP routing, external services. It depends on both application and domain layers but nothing depends on it (except the NestJS bootstrap).

This project's persistence uses **either Prisma or TypeORM, never both**. Everything in this skill that touches models/entities, repositories, or migrations branches on that choice.

### Which ORM does this project use?

1. Check `.claude/nest-clean.config.json` at the project root for `{ "orm": "prisma" | "typeorm" }`.
2. If it's missing, run the same detection the `nest-clean` skill documents (check `package.json` for `@prisma/client`/`prisma` vs `typeorm`/`@nestjs/typeorm`, ask the user if ambiguous), then **write that file** so this only happens once per project.
3. Once known, read **only** the matching reference subfolder below — never load both:
   - **Prisma** → [prisma/database-entities.md](./prisma/database-entities.md), [prisma/custom-queries.md](./prisma/custom-queries.md)
   - **TypeORM** → [typeorm/database-entities.md](./typeorm/database-entities.md), [typeorm/custom-queries.md](./typeorm/custom-queries.md)

### File placement

| Component | Location |
|-----------|----------|
| Prisma schema (all models) — **if Prisma** | `prisma/schema.prisma` — single source of truth, shared by the whole app |
| TypeORM entities — **if TypeORM** | `src/modules/{module}/infrastructure/entities/{entity}.entity.ts` — one class per table |
| Migrations | Prisma: `prisma/migrations/` (via `prisma migrate dev`). TypeORM: `src/infrastructure/database/migrations/` (via `typeorm migration:generate`) |
| Prisma `PrismaService`/`PrismaModule` — **if Prisma** | `src/infrastructure/prisma/` — `@Global()`, no need to import elsewhere |
| TypeORM `DataSource` setup — **if TypeORM** | `src/infrastructure/database/database.module.ts` (`TypeOrmModule.forRootAsync`) — see [Creating the NestJS Module](#4-creating-the-nestjs-module) for how feature modules get access to it |
| Base entity / base repository impl | `src/infrastructure/database/` |
| External services (Redis, Auth0, Blob…) | `src/infrastructure/external/` |
| Environment variables / config | `src/infrastructure/config/envs.ts` — invoke the `environment-config` skill before adding or reading one |
| Repository implementations | `src/modules/{module}/infrastructure/repositories/` |
| API controllers | `src/modules/{module}/infrastructure/controllers/` |
| NestJS module file | `src/modules/{module}/{module}.module.ts` |

> **Prisma**: there is no shared `entities/` folder to register anywhere — adding a model only means editing `prisma/schema.prisma` and generating a migration. **TypeORM**: each module owns its own `entities/` folder, and the entity must be reachable by the global `entities: [...]` glob in `DatabaseService`/`DatabaseModule` (or be added there explicitly) plus wired into the module as shown below.

---

## 1. Adding a Model / Entity

### 1a. Prisma — adding a model

**File**: `prisma/schema.prisma`

**Step-by-step**:
1. Add a `model` block — table name via `@@map('table_name')` (snake_case, plural)
2. Every model declares its own `id`, `createdAt`, `updatedAt` — Prisma has no model inheritance, so these are **not** shared from a base class
3. Add scalar fields with explicit Prisma types (`String`, `Int`, `Decimal`, `Boolean`, `DateTime`, an `enum`, …)
4. Add relations (`@relation`) — put them after the scalar fields, same convention as before
5. Run `pnpm run prisma:migrate:dev --name DescribeChange` to generate + apply the migration and regenerate the client

```prisma
// prisma/schema.prisma
model Product {
  id          String   @id @default(uuid()) @db.Uuid
  name        String   @db.VarChar(200)
  sku         String   @unique @db.VarChar(100)
  price       Decimal  @db.Decimal(10, 2)
  description String?
  active      Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  companyId   String   @db.Uuid
  company     Company  @relation(fields: [companyId], references: [id])

  @@map("products")
}
```

```bash
pnpm run prisma:migrate:dev --name CreateProductTable
```

This also regenerates `@prisma/client`'s types (`prisma generate` runs automatically as part of `migrate dev`) — the `Product` type used below comes straight from that generated client, not from a hand-written entity class.

See [prisma/database-entities.md](./prisma/database-entities.md) for the full field-type reference and relation patterns.

### 1b. TypeORM — adding an entity

**File**: `src/modules/{module}/infrastructure/entities/{entity}.entity.ts`

**Step-by-step**:
1. Extend `BaseEntity` (from `src/infrastructure/database/base.entity.ts`) — provides `id` (UUID), `createdAt`, `updatedAt` automatically via `@PrimaryGeneratedColumn('uuid')` / `@CreateDateColumn()` / `@UpdateDateColumn()`
2. `@Entity('table_name')` — `snake_case`, plural
3. Add `@Column(...)` fields — TypeORM maps `camelCase` properties to `snake_case` columns by default; only pass `name` when the column name isn't that default
4. Add `nullable: true` explicitly on optional columns — default is `NOT NULL`
5. Relations (`@ManyToOne`, `@OneToMany`, `@ManyToMany`) go at the end of the class, after every `@Column`
6. Run `pnpm run typeorm:migration:generate --name=DescribeChange` then review the generated SQL before `pnpm run typeorm:migration:run`

```typescript
// src/modules/product/infrastructure/entities/product.entity.ts
import { BaseEntity } from '@/infrastructure/database/base.entity';
import { CompanyEntity } from '@/modules/company/infrastructure/entities/company.entity';

import { Column, Entity, JoinColumn, ManyToOne } from 'typeorm';

@Entity('products')
export class ProductEntity extends BaseEntity {
  @Column({ type: 'varchar', length: 200 })
  name: string;

  @Column({ type: 'varchar', length: 100, unique: true })
  sku: string;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  price: number;

  @Column({ type: 'text', nullable: true })
  description: string | null;

  @Column({ type: 'boolean', default: true })
  active: boolean;

  @Column({ type: 'uuid', name: 'company_id' })
  companyId: string;

  @ManyToOne(() => CompanyEntity)
  @JoinColumn({ name: 'company_id' })
  company?: CompanyEntity;
}
```

```bash
pnpm run typeorm:migration:generate --name=CreateProductsTable
pnpm run typeorm:migration:run
```

See [typeorm/database-entities.md](./typeorm/database-entities.md) for the full column-type reference and relationship patterns.

---

## 2. Implementing a Repository

### 2a. Prisma

**File**: `src/modules/{module}/infrastructure/repositories/{entity}.repository.impl.ts`

**Step-by-step**:
1. Extend `BaseRepositoryImpl<PrismaModel, DomainEntity>` (imported from `@/infrastructure/database/base.repository.impl`)
2. Implement the domain repository interface (e.g., `ProductRepository`)
3. Inject `PrismaService` and pass a *delegate accessor* to `super(...)` — a function `(client) => client.product` that resolves the right Prisma model delegate off either `PrismaService` or a `Prisma.TransactionClient`
4. Implement `mapToDomain(entity)` — converts the Prisma model to the domain entity
5. Implement `mapToEntity(domain)` — converts the domain entity to a Prisma `*CreateInput`/`*UncheckedCreateInput`
6. Implement any abstract methods declared in the domain repository interface
7. Queries with relations go in the `QUERIES` section

```typescript
// src/modules/{module}/infrastructure/repositories/product.repository.impl.ts
import { Injectable } from '@nestjs/common';

import { BaseRepositoryImpl } from '@/infrastructure/database/base.repository.impl';
import { PrismaService } from '@/infrastructure/prisma/prisma.service';

import { Prisma, Product as PrismaProduct } from '@prisma/client';

import { ProductListResponseDto } from '../../application/dtos/product-list-response.dto';
import { PageOptionsDto } from '@/common/pagination/dtos';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';
import { Product } from '../../domain/entities/product.domain';
import { ProductRepository } from '../../domain/repositories/product.repository';

@Injectable()
export class ProductRepositoryImpl
  extends BaseRepositoryImpl<PrismaProduct, Product>
  implements ProductRepository {

  constructor(prisma: PrismaService) {
    super(prisma, (client) => client.product);
  }

  // ─── Abstract method implementations ────────────────────────────────────────

  async findBySku(sku: string): Promise<Product | null> {
    const entity = await this.getRepo().findUnique({ where: { sku } });
    return entity ? this.mapToDomain(entity) : null;
  }

  async existsBySku(sku: string, excludeId?: string): Promise<boolean> {
    const count = await this.getRepo().count({
      where: { sku, id: excludeId ? { not: excludeId } : undefined },
    });
    return count > 0;
  }

  // ─── Required mapping methods ────────────────────────────────────────────────

  mapToDomain(entity: PrismaProduct): Product {
    return new Product({
      id: entity.id,
      name: entity.name,
      sku: entity.sku,
      price: Number(entity.price),
      description: entity.description ?? undefined,
      active: entity.active,
      companyId: entity.companyId,
    });
  }

  mapToEntity(domain: Partial<Product>): Prisma.ProductUncheckedCreateInput {
    return {
      name: domain.name!,
      sku: domain.sku!,
      price: domain.price!,
      description: domain.description ?? null,
      active: domain.active,
      companyId: domain.companyId!,
    };
  }

  // =========== QUERIES ============

  async getListWithCompany(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<ProductListResponseDto>> {
    const [entities, count] = await Promise.all([
      this.getRepo().findMany({
        where: { companyId },
        orderBy: { name: 'asc' },
        take: page.take,
        skip: page.skip,
        select: { id: true, name: true, sku: true, price: true, active: true },
        include: { company: true },
      }),
      this.getRepo().count({ where: { companyId } }),
    ]);

    // ALWAYS use typed instances — never anonymous objects
    const items: ProductListResponseDto[] = entities.map((e) => {
      const dto = new ProductListResponseDto();
      dto.id = e.id;
      dto.name = e.name;
      dto.sku = e.sku;
      dto.price = Number(e.price);
      dto.active = e.active;
      dto.companyName = e.company?.name;
      return dto;
    });

    return { items, count };
  }
}
```

**Rules**:
- Always update `mapToDomain` and `mapToEntity` when the domain entity or the Prisma model changes
- In queries: use `select` to fetch only needed fields, `include` for relations (both take an object, same shape either way)
- In queries: create typed DTO instances (`new SomeDto()`), never anonymous objects
- Prefer `*UncheckedCreateInput`/`*UncheckedUpdateInput` over the relation-nested `*CreateInput`/`*UpdateInput` when you already have the foreign key as a scalar (`companyId`) — it avoids writing `{ company: { connect: { id } } }` for the common case
- Every abstract `BaseRepository` method (`findAll`, `findOneById`, `findAllPaginated`, `create`, `update`, `delete`) already accepts an optional `tx?: Prisma.TransactionClient` — pass it through on any custom query method that must be able to run inside a transaction
- After any schema change: generate a new migration
- See [repositories.md](../database/repositories.md) for a complete worked example (including how transactions from a use case flow through the repository)

See [prisma/custom-queries.md](./prisma/custom-queries.md) for complex query patterns.

### 2b. TypeORM

**File**: `src/modules/{module}/infrastructure/repositories/{entity}.repository.impl.ts`

**Step-by-step**:
1. Extend `BaseRepositoryImpl<EntityClass, DomainEntity>` (imported from `@/infrastructure/database/base.repository.impl`)
2. Implement the domain repository interface (e.g., `ProductRepository`)
3. Inject `DataSource` and pass it to `super(EntityClass, dataSource)` — **not** `@InjectRepository`; the base class resolves the right `Repository<T>` internally via `dataSource.manager.getRepository(EntityClass)` (or `manager.getRepository(...)` when a transaction `EntityManager` is passed through)
4. Implement `mapToDomain(entity)` / `mapToEntity(domain)` the same way as the Prisma variant
5. Implement any abstract methods declared in the domain repository interface
6. Queries with relations go in the `QUERIES` section

```typescript
// src/infrastructure/database/base.repository.impl.ts (shared base — already exists, shown for reference)
import { PageOptionsDto } from '@/application/pagination/dtos';
import { RecordHasDependenciesException } from '@/domain/common/exceptions/record-has-dependencies.exception';

import { PaginationResponse } from './interfaces/pagination.response';
import {
  DataSource, DeepPartial, EntityManager, ObjectLiteral, QueryFailedError, Repository,
} from 'typeorm';

export abstract class BaseRepositoryImpl<
  T extends ObjectLiteral, // TypeORM entity
  D extends ObjectLiteral, // Domain entity
> {
  protected manager: EntityManager;

  constructor(
    protected readonly ormEntity: new () => T,
    protected readonly dataSource: DataSource,
  ) {
    this.manager = dataSource.manager;
  }

  protected getRepository(manager?: EntityManager): Repository<T> {
    return manager
      ? manager.getRepository(this.ormEntity)
      : this.manager.getRepository(this.ormEntity);
  }

  async findAll(manager?: EntityManager): Promise<D[]> {
    const entities = await this.getRepository(manager).find();
    return entities.map((entity) => this.mapToDomain(entity));
  }

  async findOneById(id: string, manager?: EntityManager): Promise<D | null> {
    const entity = await this.getRepository(manager).findOne({ where: { id } as any });
    return entity ? this.mapToDomain(entity) : null;
  }

  async findAllPaginated(
    pageOptionsDto: PageOptionsDto,
    manager?: EntityManager,
  ): Promise<PaginationResponse<D>> {
    const [entities, total] = await this.getRepository(manager).findAndCount({
      skip: pageOptionsDto.skip,
      take: pageOptionsDto.take,
    });
    return { items: entities.map((e) => this.mapToDomain(e)), count: total };
  }

  async create(data: Partial<D>, manager?: EntityManager): Promise<D> {
    const repo = this.getRepository(manager);
    const entity = repo.create(this.mapToEntity(data));
    return this.mapToDomain(await repo.save(entity));
  }

  async update(id: string, data: Partial<D>, manager?: EntityManager): Promise<boolean> {
    const result = await this.getRepository(manager).update(id, this.mapToEntity(data) as any);
    return (result.affected ?? 0) > 0;
  }

  async delete(id: string, manager?: EntityManager): Promise<boolean> {
    try {
      const result = await this.getRepository(manager).delete(id);
      return (result.affected ?? 0) > 0;
    } catch (error) {
      // 23503 is Postgres' FK-violation code — other engines use a different one
      if (error instanceof QueryFailedError && (error as any).driverError?.code === '23503') {
        throw new RecordHasDependenciesException();
      }
      throw error;
    }
  }

  abstract mapToDomain(entity: T): D;
  abstract mapToEntity(domain: Partial<D>): DeepPartial<T>;
}
```

```typescript
// src/modules/product/infrastructure/repositories/product.repository.impl.ts
import { Injectable } from '@nestjs/common';

import { BaseRepositoryImpl } from '@/infrastructure/database/base.repository.impl';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

import { DataSource, DeepPartial, EntityManager } from 'typeorm';

import { ProductListResponseDto } from '../../application/dtos/product-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { Product } from '../../domain/entities/product.domain';
import { ProductRepository } from '../../domain/repositories/product.repository';
import { ProductEntity } from '../entities/product.entity';

@Injectable()
export class ProductRepositoryImpl
  extends BaseRepositoryImpl<ProductEntity, Product>
  implements ProductRepository
{
  constructor(protected dataSource: DataSource) {
    super(ProductEntity, dataSource);
  }

  // ─── Abstract method implementations ────────────────────────────────────────

  async findBySku(sku: string, manager?: EntityManager): Promise<Product | null> {
    const entity = await this.getRepository(manager).findOne({ where: { sku } });
    return entity ? this.mapToDomain(entity) : null;
  }

  async existsBySku(sku: string, excludeId?: string): Promise<boolean> {
    return this.getRepository().exists({
      where: excludeId ? { sku, id: Not(excludeId) } : { sku },
    });
  }

  // ─── Required mapping methods ────────────────────────────────────────────────

  mapToDomain(entity: ProductEntity): Product {
    return new Product({
      id: entity.id,
      name: entity.name,
      sku: entity.sku,
      price: Number(entity.price),
      description: entity.description ?? undefined,
      active: entity.active,
      companyId: entity.companyId,
    });
  }

  mapToEntity(domain: Partial<Product>): DeepPartial<ProductEntity> {
    return {
      name: domain.name,
      sku: domain.sku,
      price: domain.price,
      description: domain.description ?? null,
      active: domain.active,
      companyId: domain.companyId,
    };
  }

  // =========== QUERIES ============

  async getListWithCompany(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<ProductListResponseDto>> {
    const [entities, count] = await this.getRepository().findAndCount({
      where: { companyId },
      order: { name: 'ASC' },
      take: page.take,
      skip: page.skip,
      select: { id: true, name: true, sku: true, price: true, active: true },
      relations: { company: true },
    });

    // ALWAYS use typed instances — never anonymous objects
    const items: ProductListResponseDto[] = entities.map((e) => {
      const dto = new ProductListResponseDto();
      dto.id = e.id;
      dto.name = e.name;
      dto.sku = e.sku;
      dto.price = Number(e.price);
      dto.active = e.active;
      dto.companyName = e.company?.name;
      return dto;
    });

    return { items, count };
  }
}
```

**Rules**:
- Always inject `DataSource` and call `super(EntityClass, dataSource)` — never `@InjectRepository`/`Repository<T>` directly in a domain-facing repository impl, so every method can transparently accept an optional `manager?: EntityManager` for transactions
- `select`/`relations` always use object notation: `relations: { company: true }` — there is no array-of-strings form in this project's style
- In queries: create typed DTO instances (`new SomeDto()`), never anonymous objects
- Every `BaseRepositoryImpl` method accepts an optional `manager?: EntityManager` — pass it through on any custom query method that must run inside a transaction
- After any entity change: generate a new migration
- Foreign-key violations on `delete` surface as Postgres error code `23503` — the base class turns that into `RecordHasDependenciesException`; other DB engines use a different code, adjust if not on Postgres

See [typeorm/custom-queries.md](./typeorm/custom-queries.md) for complex query patterns.

---

## 3. Creating an API Controller

This layer is **ORM-agnostic** — controllers only ever talk to use cases and DTOs, never to Prisma or TypeORM directly, so the same pattern applies regardless of which ORM the project uses.

**File**: `src/modules/{module}/infrastructure/controllers/{module}.controller.ts`

**Step-by-step**:
1. Decorate with `@ApiTags`, `@Controller('route')`
2. Inject use case implementations via constructor (not repositories)
3. Add `@UseGuards(Auth0Guard)` on protected methods; `@Public()` for public ones
4. Validate inputs with DTO types on `@Body()`, `@Param()`, `@Query()`
5. Map to DTO in the controller using the mapper — never return domain entities
6. Add Swagger decorators on every endpoint

```typescript
// src/modules/{module}/infrastructure/controllers/product.controller.ts
import {
  Body, Controller, Get, HttpStatus, Param, Post,
  Query, UseGuards,
} from '@nestjs/common';
import {
  ApiBearerAuth, ApiOperation, ApiResponse, ApiTags,
} from '@nestjs/swagger';

import { ProductMapper } from '../../application/mappers/product.mapper';
import { CreateProductDto } from '../../application/dtos/create-product.dto';
import { ProductResponseDto } from '../../application/dtos/product-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos/page-options.dto';
import { Auth0Guard } from '@/common/guards/auth0.guard';
import { CompanyId } from '@/common/decorators/company-id.decorator';
import { CreateProductUseCaseImpl } from '../../application/use-cases/create-product.use-case.impl';
import { GetProductUseCaseImpl } from '../../application/use-cases/get-product.use-case.impl';
import { GetProductListUseCaseImpl } from '../../application/use-cases/get-product-list.use-case.impl';
import { PaginationService } from '@/application/pagination/pagination.service';
import { PageDto } from '@/application/pagination/dtos/page.dto';

@ApiTags('Products')
@Controller('products')
@UseGuards(Auth0Guard)
@ApiBearerAuth()
export class ProductController {
  constructor(
    private readonly createProductUseCase: CreateProductUseCaseImpl,
    private readonly getProductUseCase: GetProductUseCaseImpl,
    private readonly getProductListUseCase: GetProductListUseCaseImpl,
  ) {}

  @Post()
  @ApiOperation({ summary: 'Crear un nuevo producto' })
  @ApiResponse({ status: HttpStatus.CREATED, description: 'Producto creado.', type: String })
  @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'Datos inválidos.' })
  async create(
    @Body() dto: CreateProductDto,
    @CompanyId() companyId: string,
  ): Promise<string> {
    const product = ProductMapper.toDomain(dto);
    return this.createProductUseCase.execute(companyId, product);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Obtener producto por ID' })
  @ApiResponse({ status: HttpStatus.OK, type: ProductResponseDto })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Producto no encontrado.' })
  async getById(@Param('id') id: string): Promise<ProductResponseDto> {
    const product = await this.getProductUseCase.execute(id);
    return ProductMapper.toDto(product); // map to DTO here, never return domain entity
  }

  @Get()
  @ApiOperation({ summary: 'Listar productos paginados' })
  async getList(
    @Query() pageOptions: PageOptionsDto,
    @CompanyId() companyId: string,
  ): Promise<PageDto<ProductResponseDto>> {
    const { items, count } = await this.getProductListUseCase.execute(companyId, pageOptions);
    return PaginationService.createPageDto(items, count, pageOptions, ProductResponseDto);
  }
}
```

See [api-controllers.md](./api-controllers.md) for full documentation.

---

## 4. Creating the NestJS Module

### 4a. Prisma

`PrismaModule` is `@Global()` — unlike `TypeOrmModule.forFeature([...])`, there is **nothing to register per model**. A module only wires the repository binding and its use cases:

```typescript
// src/modules/product/product.module.ts
import { Module } from '@nestjs/common';

import { ProductRepositoryImpl } from './infrastructure/repositories/product.repository.impl';
import { ProductController } from './infrastructure/controllers/product.controller';
import { CreateProductUseCaseImpl } from './application/use-cases/create-product.use-case.impl';
import { GetProductUseCaseImpl } from './application/use-cases/get-product.use-case.impl';
import { GetProductListUseCaseImpl } from './application/use-cases/get-product-list.use-case.impl';
import { ProductRepository } from './domain/repositories/product.repository';

@Module({
  controllers: [ProductController],
  providers: [
    // Bind domain interface to its implementation
    { provide: ProductRepository, useClass: ProductRepositoryImpl },
    CreateProductUseCaseImpl,
    GetProductUseCaseImpl,
    GetProductListUseCaseImpl,
  ],
  exports: [ProductRepository],
})
export class ProductModule {}
```

**Then register in `AppModule`**:
```typescript
// Add ProductModule to the imports array of AppModule
imports: [..., ProductModule],
```

> The one exception is a **standalone bootstrap that doesn't go through `AppModule`** (e.g. `src/cli.ts` for seeders) — since `PrismaModule` is only auto-available once something imports it into that context's root module, import it explicitly there. See [seeder.module.ts](../database/seeders/seeder.module.ts).

### 4b. TypeORM

The `BaseRepositoryImpl` pattern above injects `DataSource` directly (not `@InjectRepository`), so a feature module doesn't need `TypeOrmModule.forFeature([...])` — it only needs `DataSource` to be resolvable via DI. Two ways to get that, pick whichever this project already does:

- **Recommended, mirrors Prisma's ergonomics**: mark the module that calls `TypeOrmModule.forRootAsync(...)` (e.g. `DatabaseModule`) as `@Global()`, so `DataSource` is injectable everywhere without every feature module importing anything extra.
- **Explicit alternative**: keep `DatabaseModule` non-global and `imports: [DatabaseModule]` in every feature module that needs `DataSource`.

Either way, the module itself binds the repository and wires use cases exactly like the Prisma version:

```typescript
// src/modules/product/product.module.ts
import { Module } from '@nestjs/common';

import { DatabaseModule } from '@/infrastructure/database/database.module'; // omit this import if DatabaseModule is @Global()
import { ProductRepositoryImpl } from './infrastructure/repositories/product.repository.impl';
import { ProductController } from './infrastructure/controllers/product.controller';
import { CreateProductUseCaseImpl } from './application/use-cases/create-product.use-case.impl';
import { GetProductUseCaseImpl } from './application/use-cases/get-product.use-case.impl';
import { GetProductListUseCaseImpl } from './application/use-cases/get-product-list.use-case.impl';
import { ProductRepository } from './domain/repositories/product.repository';

@Module({
  imports: [DatabaseModule], // omit if DatabaseModule is @Global()
  controllers: [ProductController],
  providers: [
    // Bind domain interface to its implementation
    { provide: ProductRepository, useClass: ProductRepositoryImpl },
    CreateProductUseCaseImpl,
    GetProductUseCaseImpl,
    GetProductListUseCaseImpl,
  ],
  exports: [ProductRepository],
})
export class ProductModule {}
```

**Then register in `AppModule`**:
```typescript
// Add ProductModule to the imports array of AppModule
imports: [..., ProductModule],
```

`DatabaseModule` itself typically looks like:

```typescript
// src/infrastructure/database/database.module.ts
import { Module } from '@nestjs/common'; // add @Global() here if using the recommended approach above
import { TypeOrmModule } from '@nestjs/typeorm';

import { DatabaseService } from './database.service';

@Module({
  imports: [TypeOrmModule.forRootAsync({ useClass: DatabaseService })],
  exports: [TypeOrmModule],
})
export class DatabaseModule {}
```

```typescript
// src/infrastructure/database/database.service.ts
import { Injectable } from '@nestjs/common';
import { TypeOrmModuleOptions, TypeOrmOptionsFactory } from '@nestjs/typeorm';

import { Environment, envs } from '../config/envs'; // both exported by the environment-config skill's envs.ts
import { join } from 'path';

@Injectable()
export class DatabaseService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'postgres',
      host: envs.dbHost,
      port: envs.dbPort,
      username: envs.dbUsername,
      password: envs.dbPassword,
      database: envs.dbName,
      entities: [join(__dirname, '../../modules/**/infrastructure/entities/*.entity.{ts,js}')],
      migrations: [join(__dirname, 'migrations/*.{ts,js}')],
      synchronize: false, // never true outside a throwaway local DB
      logging: envs.environment === Environment.Development,
      ssl: envs.dbSsl ? { rejectUnauthorized: false } : false,
    };
  }
}
```

---

## 5. Generating a Migration

### Prisma

Run after any change to `prisma/schema.prisma`:

```bash
pnpm run prisma:migrate:dev --name DescribeTheChange   # dev: generates + applies + regenerates the client
pnpm run prisma:migrate:deploy                         # CI/CD: applies pending migrations, no prompts
pnpm run prisma:generate                                # regenerate the client only, no schema/migration change
pnpm run prisma:studio                                  # inspect data with Prisma Studio
```

Check the generated SQL in `prisma/migrations/<timestamp>_describe_the_change/migration.sql` before it's committed. Prisma migrations are plain SQL files — if the tool generates something unwanted (e.g. a `DROP COLUMN` it inferred from a rename), edit the file directly before running it, the same way you'd review a TypeORM migration.

### TypeORM

Run after any change to a `*.entity.ts`:

```bash
pnpm run typeorm:migration:generate --name=DescribeTheChange   # diffs entities vs DB, writes a migration file
pnpm run typeorm:migration:run                                  # applies pending migrations
pnpm run typeorm:revert                                          # reverts the last applied migration
```

Check the generated file in `src/infrastructure/database/migrations/<timestamp>-DescribeTheChange.ts` before running it — TypeORM's diff can propose a destructive `DROP`+`ADD` where a `RENAME` was intended, and if a relation targets a column without a `UNIQUE` constraint it may emit an `ADD CONSTRAINT` that fails; remove that line and the relation still works for TypeORM queries, just without DB-level referential integrity.

---

## Checklist

- [ ] ORM for this project resolved via `.claude/nest-clean.config.json` (or detected + cached there if missing) — never guessed mid-task
- [ ] Model/entity added: Prisma `model` block in `prisma/schema.prisma`, or TypeORM `*.entity.ts` extending `BaseEntity`
- [ ] Table mapped with `@@map('snake_case_table')` (Prisma) or `@Entity('snake_case_table')` (TypeORM)
- [ ] All fields have explicit types and constraints (unique, default, nullability)
- [ ] Relations declared, placed after scalar fields
- [ ] Migration generated with the ORM's own command and reviewed before applying
- [ ] Repository impl in `src/modules/{module}/infrastructure/repositories/`
- [ ] Repository impl extends `BaseRepositoryImpl<PersistenceModel, DomainEntity>` — Prisma: delegate accessor `(client) => client.model` passed to `super()`; TypeORM: `super(EntityClass, dataSource)`, never `@InjectRepository`
- [ ] `mapToDomain` and `mapToEntity` are fully implemented
- [ ] Query methods use typed DTO instances (never anonymous objects); `select`/`include`/`relations` always use object notation
- [ ] Controller in `src/modules/{module}/infrastructure/controllers/`
- [ ] Controller injects use cases, never repositories
- [ ] Controller maps domain entities to DTOs before returning
- [ ] Every endpoint has `@ApiOperation`, `@ApiResponse`, and `@ApiBearerAuth` (if protected)
- [ ] Module file at `src/modules/{module}/{module}.module.ts` — Prisma: no `TypeOrmModule.forFeature`/registration needed; TypeORM: `DataSource` reachable via DI (global `DatabaseModule` or explicit `imports: [DatabaseModule]`)
- [ ] Module binds domain interface to implementation with `{ provide: XRepository, useClass: XRepositoryImpl }`
- [ ] Module registered in `AppModule` imports
