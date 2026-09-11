---
name: application-layer
description: >
  Use when implementing application layer components in this NestJS project: use case implementations (*.use-case.impl.ts), DTOs (request/response), or Mappers. Domain interfaces must already exist before working here. Use this skill for requests like "implement use case X", "create DTO for Y", "add mapper for Z", "add cache to use case", "create request/response DTO", "implement the application logic". Always use when code needs to go in src/modules/{module}/application/.
---

## Application Layer Overview

The **Application Layer** orchestrates business flows — it coordinates domain entities, repositories, and external services without containing business logic itself. Business rules belong in domain entities; HTTP concerns belong in the infrastructure layer.

Components:
- **Use Case Implementations**: `@Injectable()` classes implementing domain interfaces
- **DTOs**: Input validation and output shape for the API
- **Mappers**: Bidirectional conversion between DTOs and domain entities

**File placement**: All application layer files for a module go in `src/modules/{module}/application/`.

**Rule**: Application layer imports from the module's `domain/` and from `src/domain/common/` freely. It may import from `src/infrastructure/` only for injected services (e.g., `CacheService`). DTOs may be imported by the infrastructure layer.

---

## 1. Implementing a Use Case

**File**: `src/modules/{module}/application/use-cases/{action}.use-case.impl.ts`

**Step-by-step**:
1. Create `@Injectable()` class implementing the domain interface
2. Inject all required repositories and services via constructor (use `readonly`)
3. Implement `execute()` following the JSDoc steps defined in the domain interface
4. Throw domain exceptions (never HTTP exceptions) for error cases
5. Add cache logic when the operation is a read (get/find/list)
6. Invalidate related cache keys after mutations (create/update/delete)
7. Use `Entity.plainToInstance()` when reconstructing entities from cache

```typescript
// src/modules/{module}/application/use-cases/get-product.use-case.impl.ts
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { Product } from '../../domain/entities/product.domain';
import { ProductNotFoundException } from '../../domain/exceptions/product-not-found.exception';
import { ProductRepository } from '../../domain/repositories/product.repository';
import { GetProductUseCase } from '../../domain/use-cases/get-product.use-case';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class GetProductUseCaseImpl implements GetProductUseCase {
  constructor(
    private readonly productRepository: ProductRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(id: string): Promise<Product> {
    // 1. Intentar obtener desde caché
    const cacheKey = this.cacheService.buildKey(RedisKeyEnum.PRODUCT, id);
    const cached = await this.cacheService.get(cacheKey);
    if (cached) {
      return Product.plainToInstance(cached); // reconstruct — cached data has no methods
    }

    // 2. Consultar repositorio
    const product = await this.productRepository.findById(id);
    if (!product) {
      throw new ProductNotFoundException(id);
    }

    // 3. Guardar en caché y retornar
    await this.cacheService.set(cacheKey, product);
    return product;
  }
}
```

**Cache patterns**:

```typescript
// Mutation — invalidate after change
await this.productRepository.save(product);
await this.cacheService.removeCacheByPartialKey(RedisKeyEnum.PRODUCT);

// List with pagination — cache with page parameters
const cacheKey = this.cacheService.buildKey(
  RedisKeyEnum.PRODUCT,
  companyId,
  `${pageOptions.page}`,
  `${pageOptions.take}`,
);
const cached = await this.cacheService.get(cacheKey);
if (cached) {
  const data = cached as PaginationResponse<any>;
  return {
    items: Product.plainToInstanceList(data.items), // reconstruct list
    count: data.count,
  };
}
```

**Query use cases** (returning DTOs directly from repository):

```typescript
// For complex queries with joins, the repository returns DTOs directly
@Injectable()
export class GetProductListUseCaseImpl implements GetProductListUseCase {
  constructor(private readonly productRepository: ProductRepository) {}

  async execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<ProductListResponseDto>> {
    return this.productRepository.getListWithCategory(companyId, page);
  }
}
```

**Rules**:
- All constructor dependencies use `private readonly`
- Throw domain exceptions, never `HttpException` or NestJS HTTP exceptions
- Read use cases: check cache first, then repository, then store result
- Mutation use cases: save to repository, then invalidate cache
- `plainToInstance` / `plainToInstanceList` are required when reading from cache

See [use-cases.md](./use-cases.md) for full documentation with examples.

---

## 2. Creating a DTO

**Files**:
- `src/modules/{module}/application/dtos/create-{entity}.dto.ts` — Input for POST
- `src/modules/{module}/application/dtos/update-{entity}.dto.ts` — Input for PATCH/PUT
- `src/modules/{module}/application/dtos/{entity}-response.dto.ts` — Output shape

**Step-by-step (Request DTO)**:
1. Add `@ApiProperty` (required) or `@ApiPropertyOptional` on every field
2. Add class-validator decorators to validate the field
3. Optional fields use `@IsOptional()` plus `?` in TypeScript

```typescript
// src/modules/{module}/application/dtos/create-product.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import {
  IsEnum,
  IsNotEmpty,
  IsNumber,
  IsOptional,
  IsString,
  Min,
} from 'class-validator';
import { ProductStatusEnum } from '@/common/constants/enums/product-status.enum';

export class CreateProductDto {
  @ApiProperty({ description: 'Nombre del producto', example: 'Laptop HP' })
  @IsString()
  @IsNotEmpty()
  name: string;

  @ApiProperty({ description: 'Código SKU único', example: 'HP-001' })
  @IsString()
  @IsNotEmpty()
  sku: string;

  @ApiProperty({ description: 'Precio sin impuestos', example: 1500.00 })
  @IsNumber()
  @Min(0)
  price: number;

  @ApiProperty({ enum: ProductStatusEnum, example: ProductStatusEnum.ACTIVE })
  @IsEnum(ProductStatusEnum)
  status: ProductStatusEnum;

  @ApiPropertyOptional({ description: 'Descripción del producto' })
  @IsOptional()
  @IsString()
  description?: string;
}
```

**Step-by-step (Response DTO)**:
1. Add `@ApiProperty` on every field
2. Add `@Expose()` on fields that should be serialized (from `class-transformer`)
3. Fields without `@Expose()` will NOT appear in the response

```typescript
// src/modules/{module}/application/dtos/product-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

export class ProductResponseDto {
  @ApiProperty()
  @Expose()
  id: string;

  @ApiProperty()
  @Expose()
  name: string;

  @ApiProperty()
  @Expose()
  sku: string;

  @ApiProperty()
  @Expose()
  price: number;

  @ApiProperty()
  @Expose()
  active: boolean;
}
```

See [dtos.md](./dtos.md) for full documentation.

---

## 3. Creating a Mapper

**File**: `src/modules/{module}/application/mappers/{entity}.mapper.ts`

**Step-by-step**:
1. Create an `abstract class` (never instantiated)
2. Add `static toDomain(dto)` — converts input DTO to domain entity
3. Add `static toDto(entity)` — converts domain entity to response DTO
4. Add `static toDtoList(entities[])` if lists are needed
5. Build value objects in `toDomain`; extract `.value` from them in `toDto`

```typescript
// src/modules/{module}/application/mappers/product.mapper.ts
import { Product } from '../../domain/entities/product.domain';
import { CreateProductDto } from '../dtos/create-product.dto';
import { ProductResponseDto } from '../dtos/product-response.dto';

export abstract class ProductMapper {
  /**
   * Convierte DTO de creación a entidad del dominio.
   */
  static toDomain(dto: CreateProductDto, id?: string): Product {
    return new Product({
      id,
      name: dto.name,
      sku: dto.sku,
      price: dto.price,
      status: dto.status,
      description: dto.description,
    });
  }

  /**
   * Convierte entidad del dominio a DTO de respuesta.
   */
  static toDto(entity: Product): ProductResponseDto {
    return {
      id: entity.id,
      name: entity.name,
      sku: entity.sku,
      price: entity.price,
      active: entity.active,
    };
  }

  static toDtoList(entities: Product[]): ProductResponseDto[] {
    return entities.map((e) => ProductMapper.toDto(e));
  }
}
```

**Rules**:
- `abstract class` — never use `new ProductMapper()`
- All methods are `static`
- No business logic — only structural transformation
- Extract value object primitives in `toDto`: `entity.email.value`, not `entity.email`
- Build value objects in `toDomain`: `new Email(dto.email)`, not `dto.email`

See [mappers.md](./mappers.md) for full documentation.

---

## Checklist Before Moving to Infrastructure Layer

- [ ] Use case class has `@Injectable()` and implements the domain interface
- [ ] All constructor dependencies use `private readonly`
- [ ] Read use cases check cache before querying the repository
- [ ] Entities reconstructed from cache use `plainToInstance`/`plainToInstanceList`
- [ ] Mutations invalidate cache with `removeCacheByPartialKey`
- [ ] Only domain exceptions are thrown (no `HttpException`)
- [ ] Request DTOs have `@ApiProperty` + class-validator on every field
- [ ] Response DTOs have `@ApiProperty` + `@Expose()` on every field
- [ ] Mapper is `abstract class` with all `static` methods
- [ ] Mapper builds value objects in `toDomain`, extracts `.value` in `toDto`
