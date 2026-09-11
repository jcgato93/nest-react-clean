# Módulo: Asset Maintenance

## Confirmación del módulo antes de iniciar

Before I start, let me confirm the module details:

- **Module name**: `asset-maintenance` — DB table: `asset_maintenances` (snake_case, plural)
- **Entity fields**:
  - `description` — string — required
  - `cost` — decimal — required
  - `date` — date — required
  - `type` — enum (`MaintenanceType`: PREVENTIVE, CORRECTIVE, WARRANTY) — required
  - `companyId` — string (UUID) — required (relation to company)
- **Operations**: create, get by ID, list by company (paginated)
- **Authentication**: `Auth0Guard` on all endpoints
- **Pagination**: list endpoint returns paginated results via `PageOptionsDto` / `PageDto`

---

## Step 1 — Domain Layer

### `src/common/constants/enums/maintenance-type.enum.ts`

```typescript
/**
 * Enum que define los tipos de mantenimiento de activos.
 */
export enum MaintenanceType {
  PREVENTIVE = 'PREVENTIVE',
  CORRECTIVE = 'CORRECTIVE',
  WARRANTY = 'WARRANTY',
}
```

> Note: enum values are English translations of the user's Spanish terms (PREVENTIVO → PREVENTIVE, CORRECTIVO → CORRECTIVE, GARANTIA → WARRANTY). Place this in `src/common/constants/enums/` alongside the other enums in the project.

---

### `src/domain/asset-maintenance/entities/asset-maintenance.domain.ts`

```typescript
import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';

interface IAssetMaintenance {
  id?: string;
  description: string;
  cost: number;
  date: Date;
  type: MaintenanceType;
  companyId: string;
}

/**
 * Entidad de dominio para mantenimientos de activos fijos.
 *
 * Registra las intervenciones de mantenimiento (preventivo, correctivo o garantía)
 * realizadas sobre los activos de una compañía.
 */
export class AssetMaintenance {
  private readonly _id: string;
  private _description: string;
  private _cost: number;
  private _date: Date;
  private _type: MaintenanceType;
  private readonly _companyId: string;

  constructor(props: IAssetMaintenance) {
    this._id = props.id ?? crypto.randomUUID();
    this._description = props.description;
    this._cost = props.cost;
    this._date = props.date;
    this._type = props.type;
    this._companyId = props.companyId;
  }

  get id(): string {
    return this._id;
  }

  get description(): string {
    return this._description;
  }

  get cost(): number {
    return this._cost;
  }

  get date(): Date {
    return this._date;
  }

  get type(): MaintenanceType {
    return this._type;
  }

  get companyId(): string {
    return this._companyId;
  }

  /**
   * Reconstruye una instancia desde un objeto plano (ej: caché Redis).
   * OBLIGATORIO en todas las entidades.
   */
  static plainToInstance(raw: any): AssetMaintenance {
    const instance: AssetMaintenance = Object.create(AssetMaintenance.prototype);
    Object.assign(instance, raw);
    return instance;
  }

  static plainToInstanceList(rawArray: any[]): AssetMaintenance[] {
    return rawArray.map((raw) => this.plainToInstance(raw));
  }
}
```

---

### `src/domain/asset-maintenance/repositories/asset-maintenance.repository.ts`

```typescript
import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { BaseRepository } from '@/domain/common/repositories/base.repository';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

import { AssetMaintenance } from '../entities/asset-maintenance.domain';

/**
 * Repositorio de mantenimientos de activos.
 *
 * Define las operaciones de persistencia para el módulo de mantenimientos.
 */
export abstract class AssetMaintenanceRepository extends BaseRepository<AssetMaintenance> {
  // =========== QUERIES ============

  /**
   * Obtiene los mantenimientos de una compañía con paginación.
   *
   * @param companyId - ID de la compañía
   * @param page - Opciones de paginación (skip, take)
   * @returns Respuesta paginada con los mantenimientos de la compañía
   */
  abstract getListByCompany(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>>;
}
```

---

### `src/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case.ts`

```typescript
import { AssetMaintenance } from '../entities/asset-maintenance.domain';

export interface CreateAssetMaintenanceUseCase {
  /**
   * 1. Verificar que la compañía existe
   * 2. Crear y guardar el mantenimiento
   * 3. Invalidar caché relacionado
   * @param maintenance - Entidad del dominio con los datos del mantenimiento
   * @returns ID del mantenimiento creado
   */
  execute(maintenance: AssetMaintenance): Promise<string>;
}
```

---

### `src/domain/asset-maintenance/use-cases/get-asset-maintenance.use-case.ts`

```typescript
import { AssetMaintenance } from '../entities/asset-maintenance.domain';

export interface GetAssetMaintenanceUseCase {
  /**
   * 1. Intentar obtener desde caché
   * 2. Consultar repositorio por ID
   * 3. Lanzar excepción si no existe
   * 4. Guardar en caché y retornar
   * @param id - ID del mantenimiento
   * @returns Entidad del dominio AssetMaintenance
   */
  execute(id: string): Promise<AssetMaintenance>;
}
```

---

### `src/domain/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.ts`

```typescript
import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

export interface GetAssetMaintenanceListUseCase {
  /**
   * 1. Intentar obtener desde caché con clave por compañía + página
   * 2. Consultar repositorio si no está en caché
   * 3. Guardar resultado en caché y retornar
   * @param companyId - ID de la compañía
   * @param page - Opciones de paginación
   * @returns Lista paginada de mantenimientos
   */
  execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>>;
}
```

---

### `src/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception.ts`

```typescript
import { CustomNotFoundException } from '@/domain/common/exceptions/custom-not-found.exception';

export class AssetMaintenanceNotFoundException extends CustomNotFoundException {
  constructor(id: string) {
    super(`El mantenimiento con ID ${id} no fue encontrado.`);
  }
}
```

---

### Domain Layer Checklist

- [x] Entity has all required fields with correct private types matching getters
- [x] Entity has `plainToInstance` and `plainToInstanceList` static methods
- [x] Repository interface extends `BaseRepository` and only declares new methods
- [x] Repository interface has JSDoc on every method
- [x] Use case interfaces have step-by-step JSDoc and correct return types
- [x] Domain exception extends `CustomNotFoundException` with Spanish message
- [x] No imports from `@/application` or `@/infrastructure` in entity, use case interfaces, or exception (repository interface imports DTOs and infra interface as permitted by this project's pattern — see existing `AssetLocationRepository`)

---

## Step 2 — Application Layer

### `src/application/asset-maintenance/dtos/create-asset-maintenance.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import {
  IsDateString,
  IsEnum,
  IsNotEmpty,
  IsNumber,
  IsPositive,
  IsString,
} from 'class-validator';

import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';

export class CreateAssetMaintenanceDto {
  @ApiProperty({
    description: 'Descripción del mantenimiento',
    example: 'Cambio de aceite y filtros del generador',
  })
  @IsString()
  @IsNotEmpty()
  description: string;

  @ApiProperty({
    description: 'Costo del mantenimiento',
    example: 250000.5,
  })
  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  cost: number;

  @ApiProperty({
    description: 'Fecha en que se realizó el mantenimiento (ISO 8601)',
    example: '2024-03-15',
  })
  @IsDateString()
  @IsNotEmpty()
  date: string;

  @ApiProperty({
    description: 'Tipo de mantenimiento',
    enum: MaintenanceType,
    example: MaintenanceType.PREVENTIVE,
  })
  @IsEnum(MaintenanceType)
  type: MaintenanceType;
}
```

---

### `src/application/asset-maintenance/dtos/asset-maintenance-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';

export class AssetMaintenanceResponseDto {
  @ApiProperty({
    description: 'ID del mantenimiento',
    example: '550e8400-e29b-41d4-a716-446655440000',
  })
  @Expose()
  id: string;

  @ApiProperty({
    description: 'Descripción del mantenimiento',
    example: 'Cambio de aceite y filtros del generador',
  })
  @Expose()
  description: string;

  @ApiProperty({
    description: 'Costo del mantenimiento',
    example: 250000.5,
  })
  @Expose()
  cost: number;

  @ApiProperty({
    description: 'Fecha en que se realizó el mantenimiento',
    example: '2024-03-15T00:00:00.000Z',
  })
  @Expose()
  date: Date;

  @ApiProperty({
    description: 'Tipo de mantenimiento',
    enum: MaintenanceType,
    example: MaintenanceType.PREVENTIVE,
  })
  @Expose()
  type: MaintenanceType;

  @ApiProperty({
    description: 'ID de la compañía',
    example: '550e8400-e29b-41d4-a716-446655440001',
  })
  @Expose()
  companyId: string;
}
```

---

### `src/application/asset-maintenance/dtos/asset-maintenance-list-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';

export class AssetMaintenanceListResponseDto {
  @ApiProperty({
    description: 'ID del mantenimiento',
    example: '550e8400-e29b-41d4-a716-446655440000',
  })
  @Expose()
  id: string;

  @ApiProperty({
    description: 'Descripción del mantenimiento',
    example: 'Cambio de aceite y filtros del generador',
  })
  @Expose()
  description: string;

  @ApiProperty({
    description: 'Costo del mantenimiento',
    example: 250000.5,
  })
  @Expose()
  cost: number;

  @ApiProperty({
    description: 'Fecha en que se realizó el mantenimiento',
    example: '2024-03-15T00:00:00.000Z',
  })
  @Expose()
  date: Date;

  @ApiProperty({
    description: 'Tipo de mantenimiento',
    enum: MaintenanceType,
    example: MaintenanceType.PREVENTIVE,
  })
  @Expose()
  type: MaintenanceType;
}
```

---

### `src/application/asset-maintenance/mappers/asset-maintenance.mapper.ts`

```typescript
import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';
import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

import { CreateAssetMaintenanceDto } from '../dtos/create-asset-maintenance.dto';
import { AssetMaintenanceResponseDto } from '../dtos/asset-maintenance-response.dto';

export abstract class AssetMaintenanceMapper {
  /**
   * Convierte DTO de creación a entidad del dominio.
   */
  static toDomain(
    dto: CreateAssetMaintenanceDto,
    companyId: string,
  ): AssetMaintenance {
    return new AssetMaintenance({
      description: dto.description,
      cost: dto.cost,
      date: new Date(dto.date),
      type: dto.type as MaintenanceType,
      companyId,
    });
  }

  /**
   * Convierte entidad del dominio a DTO de respuesta.
   */
  static toDto(entity: AssetMaintenance): AssetMaintenanceResponseDto {
    return {
      id: entity.id,
      description: entity.description,
      cost: entity.cost,
      date: entity.date,
      type: entity.type,
      companyId: entity.companyId,
    };
  }
}
```

---

### `src/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { CreateAssetMaintenanceUseCase } from '@/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case';
import { CompanyNotFoundException } from '@/domain/company/exceptions/company-not-found.exception';
import { CompanyRepository } from '@/domain/company/repositories/company.repository';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class CreateAssetMaintenanceUseCaseImpl
  implements CreateAssetMaintenanceUseCase
{
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
    private readonly companyRepository: CompanyRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(maintenance: AssetMaintenance): Promise<string> {
    // 1. Verificar que la compañía existe
    const companyExists = await this.companyRepository.existsById(
      maintenance.companyId,
    );
    if (!companyExists) {
      throw new CompanyNotFoundException(maintenance.companyId);
    }

    // 2. Crear y guardar el mantenimiento
    const created = await this.assetMaintenanceRepository.create(maintenance);

    // 3. Invalidar caché de la lista de mantenimientos de la compañía
    await this.cacheService.removeCacheByPartialKey(
      this.cacheService.buildKey(
        RedisKeyEnum.ASSET_MAINTENANCE,
        maintenance.companyId,
      ),
    );

    return created.id;
  }
}
```

---

### `src/application/asset-maintenance/use-cases/get-asset-maintenance.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';
import { AssetMaintenanceNotFoundException } from '@/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { GetAssetMaintenanceUseCase } from '@/domain/asset-maintenance/use-cases/get-asset-maintenance.use-case';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class GetAssetMaintenanceUseCaseImpl
  implements GetAssetMaintenanceUseCase
{
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(id: string): Promise<AssetMaintenance> {
    // 1. Intentar obtener desde caché
    const cacheKey = this.cacheService.buildKey(
      RedisKeyEnum.ASSET_MAINTENANCE,
      id,
    );
    const cached = await this.cacheService.get(cacheKey);
    if (cached) {
      // Reconstruir instancia con métodos del prototipo
      return AssetMaintenance.plainToInstance(cached);
    }

    // 2. Consultar repositorio
    const maintenance = await this.assetMaintenanceRepository.findOneById(id);
    if (!maintenance) {
      throw new AssetMaintenanceNotFoundException(id);
    }

    // 3. Guardar en caché y retornar
    await this.cacheService.set(cacheKey, maintenance);
    return maintenance;
  }
}
```

---

### `src/application/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { GetAssetMaintenanceListUseCase } from '@/domain/asset-maintenance/use-cases/get-asset-maintenance-list.use-case';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class GetAssetMaintenanceListUseCaseImpl
  implements GetAssetMaintenanceListUseCase
{
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>> {
    // 1. Construir clave de caché con compañía + parámetros de paginación
    const cacheKey = this.cacheService.buildKey(
      RedisKeyEnum.ASSET_MAINTENANCE,
      companyId,
      `${page.page}`,
      `${page.take}`,
    );

    // 2. Intentar leer del caché
    const cachedData =
      await this.cacheService.get<
        PaginationResponse<AssetMaintenanceListResponseDto>
      >(cacheKey);
    if (cachedData) {
      return cachedData;
    }

    // 3. Consultar repositorio
    const result = await this.assetMaintenanceRepository.getListByCompany(
      companyId,
      page,
    );

    // 4. Guardar en caché y retornar
    await this.cacheService.set(cacheKey, result);
    return result;
  }
}
```

---

### Application Layer Checklist

- [x] Use case classes have `@Injectable()` and implement the domain interface
- [x] All constructor dependencies use `private readonly`
- [x] Read use cases check cache before querying the repository
- [x] Entities reconstructed from cache use `plainToInstance`
- [x] Mutations invalidate cache with `removeCacheByPartialKey`
- [x] Only domain exceptions are thrown (no `HttpException`)
- [x] Request DTOs have `@ApiProperty` + class-validator on every field
- [x] Response DTOs have `@ApiProperty` + `@Expose()` on every field
- [x] Mapper is `abstract class` with all `static` methods

---

## Step 3 — Infrastructure Layer

### `src/infrastructure/database/entities/asset-maintenance.entity.ts`

```typescript
import { Column, Entity, JoinColumn, ManyToOne } from 'typeorm';

import { MaintenanceType } from '@/common/constants/enums/maintenance-type.enum';

import { BaseEntity } from './base.entity';
import { CompanyEntity } from './company.entity';

@Entity('asset_maintenances')
export class AssetMaintenanceEntity extends BaseEntity {
  @Column({ type: 'varchar', length: 500, nullable: false })
  description: string;

  @Column({ type: 'decimal', precision: 15, scale: 2, nullable: false })
  cost: number;

  @Column({ type: 'date', nullable: false })
  date: Date;

  @Column({
    type: 'enum',
    enum: MaintenanceType,
    nullable: false,
  })
  type: MaintenanceType;

  @Column({ type: 'uuid', name: 'company_id', nullable: false })
  companyId: string;

  // ─── Relations ────────────────────────────────────────────────────────────────

  @ManyToOne(() => CompanyEntity, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'company_id' })
  company?: CompanyEntity;
}
```

---

### `src/infrastructure/database/repositories/asset-maintenance.repository.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { DeepPartial, Repository } from 'typeorm';

import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

import { BaseRepositoryImpl } from '../base.repository.impl';
import { AssetMaintenanceEntity } from '../entities/asset-maintenance.entity';

@Injectable()
export class AssetMaintenanceRepositoryImpl
  extends BaseRepositoryImpl<AssetMaintenanceEntity, AssetMaintenance>
  implements AssetMaintenanceRepository
{
  constructor(
    @InjectRepository(AssetMaintenanceEntity)
    private readonly assetMaintenanceEntityRepository: Repository<AssetMaintenanceEntity>,
  ) {
    super(assetMaintenanceEntityRepository);
  }

  // ─── Required mapping methods ────────────────────────────────────────────────

  mapToDomain(entity: AssetMaintenanceEntity): AssetMaintenance {
    return new AssetMaintenance({
      id: entity.id,
      description: entity.description,
      cost: Number(entity.cost),
      date: entity.date,
      type: entity.type,
      companyId: entity.companyId,
    });
  }

  mapToEntity(domain: Partial<AssetMaintenance>): DeepPartial<AssetMaintenanceEntity> {
    return {
      description: domain.description,
      cost: domain.cost,
      date: domain.date,
      type: domain.type,
      companyId: domain.companyId,
    };
  }

  // =========== QUERIES ============

  async getListByCompany(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>> {
    const [entities, count] =
      await this.assetMaintenanceEntityRepository.findAndCount({
        where: { companyId },
        select: {
          id: true,
          description: true,
          cost: true,
          date: true,
          type: true,
        },
        order: { date: 'DESC' },
        skip: page.skip,
        take: page.take,
      });

    // Siempre usar instancias tipadas — nunca objetos anónimos
    const items: AssetMaintenanceListResponseDto[] = entities.map((entity) => {
      const dto = new AssetMaintenanceListResponseDto();
      dto.id = entity.id;
      dto.description = entity.description;
      dto.cost = Number(entity.cost);
      dto.date = entity.date;
      dto.type = entity.type;
      return dto;
    });

    return { items, count };
  }
}
```

> Note: `Number(entity.cost)` is needed because TypeORM returns `decimal` columns as strings from PostgreSQL drivers. Casting to `Number` ensures the domain entity and DTO always carry a proper numeric value.

---

### `src/infrastructure/adapters/api/controllers/asset-maintenance.controller.ts`

```typescript
import {
  Body,
  Controller,
  Get,
  HttpStatus,
  Param,
  Post,
  Query,
  UseGuards,
} from '@nestjs/common';
import {
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
  ApiTags,
} from '@nestjs/swagger';

import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-list-response.dto';
import { AssetMaintenanceResponseDto } from '@/application/asset-maintenance/dtos/asset-maintenance-response.dto';
import { CreateAssetMaintenanceDto } from '@/application/asset-maintenance/dtos/create-asset-maintenance.dto';
import { AssetMaintenanceMapper } from '@/application/asset-maintenance/mappers/asset-maintenance.mapper';
import { CreateAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl';
import { GetAssetMaintenanceListUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.impl';
import { GetAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance.use-case.impl';
import { PageDto, PageOptionsDto } from '@/application/pagination/dtos';
import { PaginationService } from '@/application/pagination/services/pagination.service';
import { CompanyId } from '@/common/decorators/company-id.decorator';
import { Auth0Guard } from '@/common/guards/auth0.guard';

@ApiTags('Asset Maintenances')
@Controller('asset-maintenances')
@UseGuards(Auth0Guard)
@ApiBearerAuth()
export class AssetMaintenanceController {
  constructor(
    private readonly createAssetMaintenanceUseCase: CreateAssetMaintenanceUseCaseImpl,
    private readonly getAssetMaintenanceUseCase: GetAssetMaintenanceUseCaseImpl,
    private readonly getAssetMaintenanceListUseCase: GetAssetMaintenanceListUseCaseImpl,
  ) {}

  @Post()
  @ApiOperation({ summary: 'Crear un nuevo mantenimiento de activo' })
  @ApiResponse({
    status: HttpStatus.CREATED,
    description: 'Mantenimiento creado exitosamente.',
    type: String,
  })
  @ApiResponse({
    status: HttpStatus.BAD_REQUEST,
    description: 'Datos inválidos.',
  })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: 'Compañía no encontrada.',
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: 'Token de autenticación inválido o no proporcionado.',
  })
  async create(
    @Body() dto: CreateAssetMaintenanceDto,
    @CompanyId() companyId: string,
  ): Promise<string> {
    const maintenance = AssetMaintenanceMapper.toDomain(dto, companyId);
    return this.createAssetMaintenanceUseCase.execute(maintenance);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Obtener un mantenimiento de activo por ID' })
  @ApiResponse({
    status: HttpStatus.OK,
    description: 'Mantenimiento encontrado.',
    type: AssetMaintenanceResponseDto,
  })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: 'Mantenimiento no encontrado.',
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: 'Token de autenticación inválido o no proporcionado.',
  })
  async getById(
    @Param('id') id: string,
  ): Promise<AssetMaintenanceResponseDto> {
    const maintenance = await this.getAssetMaintenanceUseCase.execute(id);
    return AssetMaintenanceMapper.toDto(maintenance);
  }

  @Get()
  @ApiOperation({ summary: 'Listar mantenimientos de activos por compañía (paginado)' })
  @ApiResponse({
    status: HttpStatus.OK,
    description: 'Lista paginada de mantenimientos.',
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: 'Token de autenticación inválido o no proporcionado.',
  })
  async getList(
    @Query() pageOptions: PageOptionsDto,
    @CompanyId() companyId: string,
  ): Promise<PageDto<AssetMaintenanceListResponseDto>> {
    const { items, count } =
      await this.getAssetMaintenanceListUseCase.execute(companyId, pageOptions);
    return PaginationService.createPageDto(
      items,
      count,
      pageOptions,
      AssetMaintenanceListResponseDto,
    );
  }
}
```

---

### `src/infrastructure/modules/asset-maintenance.module.ts`

```typescript
import { Module } from '@nestjs/common';

import { CreateAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl';
import { GetAssetMaintenanceListUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.impl';
import { GetAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance.use-case.impl';

import { AssetMaintenanceController } from '../adapters/api/controllers/asset-maintenance.controller';
import { DatabaseModule } from '../database/database.module';
import { CacheService } from '../external/services/cache/cache.service';

@Module({
  imports: [DatabaseModule],
  controllers: [AssetMaintenanceController],
  providers: [
    CreateAssetMaintenanceUseCaseImpl,
    GetAssetMaintenanceUseCaseImpl,
    GetAssetMaintenanceListUseCaseImpl,
    CacheService,
  ],
  exports: [],
})
export class AssetMaintenanceModule {}
```

---

### Changes to `src/infrastructure/database/database.module.ts`

Add the following to `DatabaseModule`:

**Imports — add entity to `TypeOrmModule.forFeature([...])`:**
```typescript
AssetMaintenanceEntity,
```

**Providers array — add repository binding:**
```typescript
{
  provide: AssetMaintenanceRepository,
  useClass: AssetMaintenanceRepositoryImpl,
},
```

**Import statements to add at the top of the file:**
```typescript
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { AssetMaintenanceEntity } from './entities/asset-maintenance.entity';
import { AssetMaintenanceRepositoryImpl } from './repositories/asset-maintenance.repository.impl';
```

---

### Change to `src/app.module.ts`

Add to the `imports` array of `AppModule`:
```typescript
import { AssetMaintenanceModule } from './infrastructure/modules/asset-maintenance.module';

// Inside @Module imports:
AssetMaintenanceModule,
```

---

### Change to `src/common/constants/enums/redis-key.enum.ts`

Add the following entry to `RedisKeyEnum`:
```typescript
// Asset Maintenance
ASSET_MAINTENANCE = 'asset-maintenance',
```

---

### Migration

After creating the `AssetMaintenanceEntity`, generate and run the migration:

```bash
npm run typeorm:migration:generate -- -n CreateAssetMaintenancesTable
npm run typeorm:migration:run
```

The generated migration will create the `asset_maintenances` table with columns: `id` (uuid, PK), `description` (varchar 500), `cost` (decimal 15,2), `date` (date), `type` (enum), `company_id` (uuid, FK to companies), `created_at`, `updated_at`, `deleted_at`.

Review the generated file before running it. Remove any unwanted `ADD CONSTRAINT` lines that reference non-unique columns.

---

### Infrastructure Layer Checklist

- [x] DB entity extends `BaseEntity` and has `@Entity('asset_maintenances')`
- [x] All columns have explicit type and options in `@Column()`
- [x] Relations are at the bottom of the entity class with `@ManyToOne`/`@JoinColumn`
- [x] Migration must be generated after entity creation
- [x] Repository impl extends `BaseRepositoryImpl` and implements domain interface
- [x] `mapToDomain` and `mapToEntity` are fully implemented
- [x] Query method uses typed DTO instances (not anonymous objects)
- [x] Controller injects use cases, never repositories
- [x] Controller maps domain entities to DTOs before returning
- [x] Every endpoint has `@ApiOperation`, `@ApiResponse`, and `@ApiBearerAuth`
- [x] Module binds `AssetMaintenanceRepository` to `AssetMaintenanceRepositoryImpl`
- [x] Module registered in `AppModule` imports
- [x] `AssetMaintenanceEntity` registered in `DatabaseModule`
- [x] `AssetMaintenanceRepository` provider added to `DatabaseModule` providers/exports

---

## Complete File Map

| File | Purpose |
|------|---------|
| `src/common/constants/enums/maintenance-type.enum.ts` | Enum PREVENTIVE / CORRECTIVE / WARRANTY |
| `src/common/constants/enums/redis-key.enum.ts` | Add `ASSET_MAINTENANCE` entry |
| `src/domain/asset-maintenance/entities/asset-maintenance.domain.ts` | Domain entity |
| `src/domain/asset-maintenance/repositories/asset-maintenance.repository.ts` | Repository interface |
| `src/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case.ts` | Use case interface — create |
| `src/domain/asset-maintenance/use-cases/get-asset-maintenance.use-case.ts` | Use case interface — get by ID |
| `src/domain/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.ts` | Use case interface — list by company |
| `src/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception.ts` | Not-found domain exception |
| `src/application/asset-maintenance/dtos/create-asset-maintenance.dto.ts` | Input DTO |
| `src/application/asset-maintenance/dtos/asset-maintenance-response.dto.ts` | Get-by-ID response DTO |
| `src/application/asset-maintenance/dtos/asset-maintenance-list-response.dto.ts` | List response DTO |
| `src/application/asset-maintenance/mappers/asset-maintenance.mapper.ts` | Mapper (toDomain / toDto) |
| `src/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl.ts` | Create use case impl |
| `src/application/asset-maintenance/use-cases/get-asset-maintenance.use-case.impl.ts` | Get-by-ID use case impl |
| `src/application/asset-maintenance/use-cases/get-asset-maintenance-list.use-case.impl.ts` | List use case impl |
| `src/infrastructure/database/entities/asset-maintenance.entity.ts` | TypeORM entity |
| `src/infrastructure/database/repositories/asset-maintenance.repository.impl.ts` | Repository implementation |
| `src/infrastructure/adapters/api/controllers/asset-maintenance.controller.ts` | HTTP controller |
| `src/infrastructure/modules/asset-maintenance.module.ts` | NestJS module |
| `src/infrastructure/database/database.module.ts` | Register entity + repository (modify) |
| `src/app.module.ts` | Register module (modify) |
| `src/infrastructure/database/migrations/...CreateAssetMaintenancesTable.ts` | Migration (generated) |

---

## Final Verification

1. Run `npm run start:dev` — no TypeScript compilation errors.
2. Run `npm run lint` — no lint errors.
3. Check Swagger at `/api/docs` — three new endpoints appear under the `Asset Maintenances` tag:
   - `POST /asset-maintenances` — create
   - `GET /asset-maintenances/:id` — get by ID
   - `GET /asset-maintenances` — list by company (paginated)
4. Review the generated migration before running it.
5. Confirm `AssetMaintenanceModule` is present in `AppModule` imports.
6. Confirm `AssetMaintenanceEntity` is in `DatabaseModule`'s `TypeOrmModule.forFeature([...])`.
