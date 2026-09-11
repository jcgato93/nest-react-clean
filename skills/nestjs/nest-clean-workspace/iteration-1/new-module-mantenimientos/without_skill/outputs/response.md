# Modulo de Mantenimientos de Activos

## Resumen de la estructura

El modulo sigue la arquitectura Clean + NestJS que ya usa el proyecto. Hay tres capas bien delimitadas: **domain**, **application**, e **infrastructure**. Cada capa tiene una responsabilidad clara y las dependencias van siempre hacia adentro (domain no conoce a nadie, application conoce domain, infrastructure conoce a los dos).

---

## Archivos a crear, en orden

### 1. Domain layer

Es lo primero que se crea porque nada depende de ella, pero todo depende de ella.

#### `src/domain/asset-maintenance/entities/asset-maintenance.domain.ts`

La entidad de dominio es una clase con propiedades privadas y getters/setters. Contiene las validaciones de negocio en el constructor y en los setters correspondientes. Debe tener un metodo `toPlain()` para serializar.

```typescript
export enum MaintenanceType {
  PREVENTIVO = 'PREVENTIVO',
  CORRECTIVO = 'CORRECTIVO',
  GARANTIA = 'GARANTIA',
}

interface AssetMaintenanceProps {
  id?: string;
  companyId: string;
  description: string;
  cost: number;
  date: Date;
  type: MaintenanceType;
}

export class AssetMaintenance {
  private readonly _id: string;
  private readonly _companyId: string;
  private _description: string;
  private _cost: number;
  private _date: Date;
  private _type: MaintenanceType;

  constructor(props: AssetMaintenanceProps) {
    this.validateDescription(props.description);
    this.validateCost(props.cost);

    this._id = props.id || crypto.randomUUID();
    this._companyId = props.companyId;
    this._description = props.description.trim();
    this._cost = props.cost;
    this._date = props.date;
    this._type = props.type;
  }

  private validateDescription(description: string): void {
    if (!description || description.trim().length === 0) {
      throw new InvalidAssetMaintenanceDescriptionException();
    }
  }

  private validateCost(cost: number): void {
    if (cost < 0) {
      throw new InvalidAssetMaintenanceCostException();
    }
  }

  get id(): string { return this._id; }
  get companyId(): string { return this._companyId; }
  get description(): string { return this._description; }
  get cost(): number { return this._cost; }
  get date(): Date { return this._date; }
  get type(): MaintenanceType { return this._type; }

  toPlain() {
    return {
      id: this._id,
      companyId: this._companyId,
      description: this._description,
      cost: this._cost,
      date: this._date,
      type: this._type,
    };
  }
}
```

**Nota importante:** el enum `MaintenanceType` se define en este mismo archivo de entidad de dominio, no en `src/common/constants/enums/`. Los enums que son parte del dominio de negocio pertenecen en la capa de dominio. El proyecto ya tiene este patron con `PersonTypeEnum` en `src/domain/catalog/value_object/person-type.enum.ts`.

---

#### `src/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception.ts`

```typescript
import { CustomNotFoundException } from '@/domain/common/exceptions/custom-not-found.exception';

export class AssetMaintenanceNotFoundException extends CustomNotFoundException {
  constructor() {
    super('Mantenimiento de activo no encontrado');
    this.name = 'AssetMaintenanceNotFoundException';
  }
}
```

#### `src/domain/asset-maintenance/exceptions/invalid-asset-maintenance-description.exception.ts`

```typescript
import { CustomBadRequestException } from '@/domain/common/exceptions/custom-bad-request.exception';

export class InvalidAssetMaintenanceDescriptionException extends CustomBadRequestException {
  constructor() {
    super('La descripcion del mantenimiento no puede estar vacia');
    this.name = 'InvalidAssetMaintenanceDescriptionException';
  }
}
```

#### `src/domain/asset-maintenance/exceptions/invalid-asset-maintenance-cost.exception.ts`

```typescript
import { CustomBadRequestException } from '@/domain/common/exceptions/custom-bad-request.exception';

export class InvalidAssetMaintenanceCostException extends CustomBadRequestException {
  constructor() {
    super('El costo del mantenimiento no puede ser negativo');
    this.name = 'InvalidAssetMaintenanceCostException';
  }
}
```

---

#### `src/domain/asset-maintenance/repositories/asset-maintenance.repository.ts`

El repositorio de dominio es una clase abstracta que extiende `BaseRepository<AssetMaintenance>`. Aqui se declaran los metodos que la implementacion concreta debe proveer, especialmente los que requieren queries personalizadas (el listado paginado por compania).

```typescript
import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { BaseRepository } from '@/domain/common/repositories/base.repository';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

import { AssetMaintenance } from '../entities/asset-maintenance.domain';

export abstract class AssetMaintenanceRepository extends BaseRepository<AssetMaintenance> {
  abstract getByCompanyId(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>>;
}
```

**Nota:** el repositorio abstracto de dominio importa el DTO de respuesta de la capa de application para el metodo de query. Este patron ya lo usa `AssetLocationRepository`, que importa `AssetLocationListResponseDto` desde application. Es una concesion pragmatica: el repositorio de dominio define el contrato del query projection directamente en terminos del DTO que retorna, evitando una capa extra de mapeo en el use case.

---

#### `src/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case.ts`

```typescript
import { AssetMaintenance } from '../entities/asset-maintenance.domain';

export abstract class CreateAssetMaintenanceUseCase {
  abstract execute(maintenance: AssetMaintenance): Promise<AssetMaintenance>;
}
```

#### `src/domain/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.ts`

```typescript
import { AssetMaintenanceDetailResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-detail-response.dto';

export interface GetAssetMaintenanceByIdUseCase {
  execute(
    id: string,
    companyId: string,
  ): Promise<AssetMaintenanceDetailResponseDto>;
}
```

#### `src/domain/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.ts`

```typescript
import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

export interface GetAssetMaintenancesByCompanyUseCase {
  execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>>;
}
```

---

### 2. Application layer (DTOs, mapper, implementaciones de use cases)

#### `src/application/asset-maintenance/dto/create-asset-maintenance.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import {
  IsDate,
  IsEnum,
  IsNotEmpty,
  IsNumber,
  IsPositive,
  IsString,
  MaxLength,
} from 'class-validator';

import { MaintenanceType } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

export class CreateAssetMaintenanceDto {
  @ApiProperty({ description: 'Descripcion del mantenimiento', maxLength: 500 })
  @IsString()
  @IsNotEmpty()
  @MaxLength(500)
  description: string;

  @ApiProperty({ description: 'Costo del mantenimiento', example: 1500000.50 })
  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  cost: number;

  @ApiProperty({ description: 'Fecha del mantenimiento', example: '2026-03-06' })
  @Type(() => Date)
  @IsDate()
  date: Date;

  @ApiProperty({ enum: MaintenanceType, description: 'Tipo de mantenimiento' })
  @IsEnum(MaintenanceType)
  type: MaintenanceType;
}
```

**Nota:** `companyId` no viene en el body. Se extrae del JWT via el decorador `@CompanyId()` en el controlador, exactamente como lo hace `AssetLocationController`. El DTO de creacion no lo incluye.

---

#### `src/application/asset-maintenance/dto/asset-maintenance-detail-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

import { MaintenanceType } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

export class AssetMaintenanceDetailResponseDto {
  @Expose()
  @ApiProperty()
  id: string;

  @Expose()
  @ApiProperty()
  companyId: string;

  @Expose()
  @ApiProperty()
  description: string;

  @Expose()
  @ApiProperty({ type: Number })
  cost: number;

  @Expose()
  @ApiProperty({ type: Date })
  date: Date;

  @Expose()
  @ApiProperty({ enum: MaintenanceType })
  type: MaintenanceType;
}
```

#### `src/application/asset-maintenance/dto/asset-maintenance-list-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

import { MaintenanceType } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

export class AssetMaintenanceListResponseDto {
  @Expose()
  @ApiProperty()
  id: string;

  @Expose()
  @ApiProperty()
  description: string;

  @Expose()
  @ApiProperty({ type: Number })
  cost: number;

  @Expose()
  @ApiProperty({ type: Date })
  date: Date;

  @Expose()
  @ApiProperty({ enum: MaintenanceType })
  type: MaintenanceType;
}
```

---

#### `src/application/asset-maintenance/mapper/asset-maintenance.mapper.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { plainToInstance } from 'class-transformer';

import { IdResponseDto } from '@/common/dtos/id-response.dto';
import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

import { CreateAssetMaintenanceDto } from '../dto/create-asset-maintenance.dto';

@Injectable()
export class AssetMaintenanceMapper {
  static toDomain(
    dto: CreateAssetMaintenanceDto,
    companyId: string,
    id?: string,
  ): AssetMaintenance {
    return new AssetMaintenance({
      id,
      companyId,
      description: dto.description,
      cost: dto.cost,
      date: dto.date,
      type: dto.type,
    });
  }

  static toIdResponse(maintenance: AssetMaintenance): IdResponseDto {
    return plainToInstance(IdResponseDto, { id: maintenance.id });
  }
}
```

**Nota:** a diferencia de `AssetLocationMapper`, aqui `toDomain` recibe `companyId` como parametro separado porque no viene en el DTO sino del decorador del controlador.

---

#### `src/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { AssetMaintenance } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { CreateAssetMaintenanceUseCase } from '@/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case';
import { CompanyNotFoundException } from '@/domain/company/exceptions/company-not-found.exception';
import { CompanyRepository } from '@/domain/company/repositories/company.repository';

@Injectable()
export class CreateAssetMaintenanceUseCaseImpl implements CreateAssetMaintenanceUseCase {
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
    private readonly companyRepository: CompanyRepository,
  ) {}

  async execute(maintenance: AssetMaintenance): Promise<AssetMaintenance> {
    const companyExists = await this.companyRepository.existsById(
      maintenance.companyId,
    );
    if (!companyExists) {
      throw new CompanyNotFoundException(maintenance.companyId);
    }

    return this.assetMaintenanceRepository.create(maintenance);
  }
}
```

---

#### `src/application/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { plainToInstance } from 'class-transformer';

import { AssetMaintenanceDetailResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-detail-response.dto';
import { AssetMaintenanceNotFoundException } from '@/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { GetAssetMaintenanceByIdUseCase } from '@/domain/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case';

@Injectable()
export class GetAssetMaintenanceByIdUseCaseImpl implements GetAssetMaintenanceByIdUseCase {
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
  ) {}

  async execute(
    id: string,
    companyId: string,
  ): Promise<AssetMaintenanceDetailResponseDto> {
    const maintenance = await this.assetMaintenanceRepository.findOneById(id);

    if (!maintenance) {
      throw new AssetMaintenanceNotFoundException();
    }

    if (maintenance.companyId !== companyId) {
      throw new AssetMaintenanceNotFoundException();
    }

    return plainToInstance(AssetMaintenanceDetailResponseDto, maintenance.toPlain(), {
      excludeExtraneousValues: true,
    });
  }
}
```

---

#### `src/application/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-list-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { GetAssetMaintenancesByCompanyUseCase } from '@/domain/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

@Injectable()
export class GetAssetMaintenancesByCompanyUseCaseImpl implements GetAssetMaintenancesByCompanyUseCase {
  constructor(
    private readonly assetMaintenanceRepository: AssetMaintenanceRepository,
  ) {}

  async execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<AssetMaintenanceListResponseDto>> {
    return this.assetMaintenanceRepository.getByCompanyId(companyId, page);
  }
}
```

---

### 3. Infrastructure layer

#### `src/infrastructure/database/entities/asset-maintenance.entity.ts`

```typescript
import { Column, Entity, Index, JoinColumn, ManyToOne } from 'typeorm';

import { MaintenanceType } from '@/domain/asset-maintenance/entities/asset-maintenance.domain';

import { BaseEntity } from './base.entity';
import { CompanyEntity } from './company.entity';

@Entity('asset_maintenances')
@Index('idx_asset_maintenance_company', ['companyId'])
export class AssetMaintenanceEntity extends BaseEntity {
  @Column({ type: 'uuid', name: 'company_id', nullable: false })
  companyId: string;

  @Column({ type: 'varchar', length: 500, nullable: false })
  description: string;

  @Column({ type: 'decimal', precision: 18, scale: 2, nullable: false })
  cost: number;

  @Column({ type: 'date', nullable: false })
  date: Date;

  @Column({ type: 'enum', enum: MaintenanceType, nullable: false })
  type: MaintenanceType;

  // === Relations ===
  @ManyToOne(() => CompanyEntity, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'company_id' })
  company: CompanyEntity;
}
```

**Nota sobre `decimal`:** TypeORM devuelve columnas `decimal`/`numeric` como `string` por defecto en PostgreSQL. Para que el valor sea un `number` en JS se debe usar el transformer de TypeORM o convertir explicitamente en `mapToDomain`. La opcion mas limpia en este proyecto es castear con `Number()` en el mapper.

---

#### `src/infrastructure/database/repositories/asset-maintenance.repository.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { plainToInstance } from 'class-transformer';
import { Repository } from 'typeorm';

import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-list-response.dto';
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

  async getByCompanyId(
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

    const items = plainToInstance(AssetMaintenanceListResponseDto, entities, {
      excludeExtraneousValues: true,
    });

    return { items, count };
  }

  mapToDomain(entity: AssetMaintenanceEntity): AssetMaintenance {
    return new AssetMaintenance({
      id: entity.id,
      companyId: entity.companyId,
      description: entity.description,
      cost: Number(entity.cost),
      date: entity.date,
      type: entity.type,
    });
  }

  mapToEntity(domain: Partial<AssetMaintenance>): Partial<AssetMaintenanceEntity> {
    const entity = new AssetMaintenanceEntity();
    if (domain.id !== undefined) entity.id = domain.id;
    if (domain.companyId !== undefined) entity.companyId = domain.companyId;
    if (domain.description !== undefined) entity.description = domain.description;
    if (domain.cost !== undefined) entity.cost = domain.cost;
    if (domain.date !== undefined) entity.date = domain.date;
    if (domain.type !== undefined) entity.type = domain.type;
    return entity;
  }
}
```

---

#### `src/infrastructure/database/migrations/<timestamp>-asset-maintenance-structure.ts`

El timestamp se genera con `npm run typeorm migration:generate` o con la fecha actual en formato Unix ms. Ejemplo de nombre: `1772000000000-asset-maintenance-structure.ts`.

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AssetMaintenanceStructure1772000000000 implements MigrationInterface {
  name = 'AssetMaintenanceStructure1772000000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TYPE "public"."asset_maintenances_type_enum" AS ENUM('PREVENTIVO', 'CORRECTIVO', 'GARANTIA')
    `);
    await queryRunner.query(`
      CREATE TABLE "asset_maintenances" (
        "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
        "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
        "deleted_at" TIMESTAMP WITH TIME ZONE,
        "company_id" uuid NOT NULL,
        "description" character varying(500) NOT NULL,
        "cost" numeric(18,2) NOT NULL,
        "date" date NOT NULL,
        "type" "public"."asset_maintenances_type_enum" NOT NULL,
        CONSTRAINT "PK_asset_maintenances" PRIMARY KEY ("id")
      )
    `);
    await queryRunner.query(`
      CREATE INDEX "idx_asset_maintenance_company" ON "asset_maintenances" ("company_id")
    `);
    await queryRunner.query(`
      ALTER TABLE "asset_maintenances"
        ADD CONSTRAINT "FK_asset_maintenances_company_id"
        FOREIGN KEY ("company_id") REFERENCES "company"("id") ON DELETE CASCADE ON UPDATE NO ACTION
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "asset_maintenances" DROP CONSTRAINT "FK_asset_maintenances_company_id"`);
    await queryRunner.query(`DROP INDEX "idx_asset_maintenance_company"`);
    await queryRunner.query(`DROP TABLE "asset_maintenances"`);
    await queryRunner.query(`DROP TYPE "public"."asset_maintenances_type_enum"`);
  }
}
```

---

#### `src/infrastructure/adapters/api/controllers/asset-maintenance.controller.ts`

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
import { ApiBearerAuth, ApiOperation, ApiResponse, ApiTags } from '@nestjs/swagger';

import { AssetMaintenanceDetailResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-detail-response.dto';
import { AssetMaintenanceListResponseDto } from '@/application/asset-maintenance/dto/asset-maintenance-list-response.dto';
import { CreateAssetMaintenanceDto } from '@/application/asset-maintenance/dto/create-asset-maintenance.dto';
import { AssetMaintenanceMapper } from '@/application/asset-maintenance/mapper/asset-maintenance.mapper';
import { CreateAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl';
import { GetAssetMaintenanceByIdUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.impl';
import { GetAssetMaintenancesByCompanyUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.impl';
import { PageDto, PageOptionsDto } from '@/application/pagination/dtos';
import { PaginationService } from '@/application/pagination/services/pagination.service';
import { ApiPaginatedResponse } from '@/common/decorators/api-paginate-response.decorator';
import { CompanyId } from '@/common/decorators/company-id.decorator';
import { IdResponseDto } from '@/common/dtos/id-response.dto';
import { Auth0Guard } from '@/common/guards/auth0.guard';

@ApiTags('Asset Maintenances')
@Controller('asset-maintenances')
@UseGuards(Auth0Guard)
@ApiBearerAuth()
export class AssetMaintenanceController {
  constructor(
    private readonly createAssetMaintenanceUseCase: CreateAssetMaintenanceUseCaseImpl,
    private readonly getAssetMaintenanceByIdUseCase: GetAssetMaintenanceByIdUseCaseImpl,
    private readonly getAssetMaintenancesByCompanyUseCase: GetAssetMaintenancesByCompanyUseCaseImpl,
  ) {}

  @Post()
  @ApiOperation({ summary: 'Registrar un nuevo mantenimiento de activo' })
  @ApiResponse({
    status: HttpStatus.CREATED,
    description: 'Mantenimiento registrado exitosamente',
    type: IdResponseDto,
  })
  @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'Datos invalidos' })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Compania no encontrada' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token invalido o ausente' })
  async createAssetMaintenance(
    @CompanyId() companyId: string,
    @Body() dto: CreateAssetMaintenanceDto,
  ): Promise<IdResponseDto> {
    const maintenance = AssetMaintenanceMapper.toDomain(dto, companyId);
    const created = await this.createAssetMaintenanceUseCase.execute(maintenance);
    return AssetMaintenanceMapper.toIdResponse(created);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Obtener un mantenimiento por ID' })
  @ApiResponse({
    status: HttpStatus.OK,
    description: 'Detalle del mantenimiento',
    type: AssetMaintenanceDetailResponseDto,
  })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Mantenimiento no encontrado' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token invalido o ausente' })
  async getAssetMaintenanceById(
    @Param('id') id: string,
    @CompanyId() companyId: string,
  ): Promise<AssetMaintenanceDetailResponseDto> {
    return this.getAssetMaintenanceByIdUseCase.execute(id, companyId);
  }

  @Get()
  @ApiOperation({ summary: 'Listar mantenimientos de la compania con paginacion' })
  @ApiPaginatedResponse(AssetMaintenanceListResponseDto)
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token invalido o ausente' })
  async getAssetMaintenancesByCompany(
    @CompanyId() companyId: string,
    @Query() pageOptionsDto: PageOptionsDto,
  ): Promise<PageDto<AssetMaintenanceListResponseDto>> {
    const { items, count } =
      await this.getAssetMaintenancesByCompanyUseCase.execute(
        companyId,
        pageOptionsDto,
      );

    return PaginationService.createPageDto(
      items,
      count,
      pageOptionsDto,
      AssetMaintenanceListResponseDto,
    );
  }
}
```

---

#### `src/infrastructure/modules/asset-maintenance.module.ts`

```typescript
import { Module } from '@nestjs/common';

import { CreateAssetMaintenanceUseCaseImpl } from '@/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl';
import { GetAssetMaintenanceByIdUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.impl';
import { GetAssetMaintenancesByCompanyUseCaseImpl } from '@/application/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.impl';

import { AssetMaintenanceController } from '../adapters/api/controllers/asset-maintenance.controller';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [AssetMaintenanceController],
  providers: [
    CreateAssetMaintenanceUseCaseImpl,
    GetAssetMaintenanceByIdUseCaseImpl,
    GetAssetMaintenancesByCompanyUseCaseImpl,
  ],
  exports: [],
})
export class AssetMaintenanceModule {}
```

---

### 4. Registrar en DatabaseModule y AppModule

#### En `src/infrastructure/database/database.module.ts`

Agregar el entity al `TypeOrmModule.forFeature([...])`:

```typescript
import { AssetMaintenanceEntity } from './entities/asset-maintenance.entity';
// ...
TypeOrmModule.forFeature([
  // ... entities existentes
  AssetMaintenanceEntity,
])
```

Agregar el provider al array `providers`:

```typescript
import { AssetMaintenanceRepository } from '@/domain/asset-maintenance/repositories/asset-maintenance.repository';
import { AssetMaintenanceRepositoryImpl } from './repositories/asset-maintenance.repository.impl';
// ...
{
  provide: AssetMaintenanceRepository,
  useClass: AssetMaintenanceRepositoryImpl,
},
```

#### En `src/app.module.ts`

```typescript
import { AssetMaintenanceModule } from './infrastructure/modules/asset-maintenance.module';
// ...
@Module({
  imports: [
    // ... modulos existentes
    AssetMaintenanceModule,
  ],
})
export class AppModule {}
```

---

## Lista completa de archivos en orden de creacion

| # | Archivo | Descripcion |
|---|---------|-------------|
| 1 | `src/domain/asset-maintenance/entities/asset-maintenance.domain.ts` | Entidad de dominio + enum MaintenanceType |
| 2 | `src/domain/asset-maintenance/exceptions/asset-maintenance-not-found.exception.ts` | Excepcion 404 |
| 3 | `src/domain/asset-maintenance/exceptions/invalid-asset-maintenance-description.exception.ts` | Excepcion descripcion invalida |
| 4 | `src/domain/asset-maintenance/exceptions/invalid-asset-maintenance-cost.exception.ts` | Excepcion costo invalido |
| 5 | `src/domain/asset-maintenance/repositories/asset-maintenance.repository.ts` | Repositorio abstracto de dominio |
| 6 | `src/domain/asset-maintenance/use-cases/create-asset-maintenance.use-case.ts` | Contrato create (abstract class) |
| 7 | `src/domain/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.ts` | Contrato get by id (interface) |
| 8 | `src/domain/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.ts` | Contrato list (interface) |
| 9 | `src/application/asset-maintenance/dto/create-asset-maintenance.dto.ts` | DTO de creacion |
| 10 | `src/application/asset-maintenance/dto/asset-maintenance-detail-response.dto.ts` | DTO de respuesta detalle |
| 11 | `src/application/asset-maintenance/dto/asset-maintenance-list-response.dto.ts` | DTO de respuesta lista |
| 12 | `src/application/asset-maintenance/mapper/asset-maintenance.mapper.ts` | Mapper DTO <-> Domain |
| 13 | `src/application/asset-maintenance/use-cases/create-asset-maintenance.use-case.impl.ts` | Implementacion create |
| 14 | `src/application/asset-maintenance/use-cases/get-asset-maintenance-by-id.use-case.impl.ts` | Implementacion get by id |
| 15 | `src/application/asset-maintenance/use-cases/get-asset-maintenances-by-company.use-case.impl.ts` | Implementacion list |
| 16 | `src/infrastructure/database/entities/asset-maintenance.entity.ts` | Entidad TypeORM |
| 17 | `src/infrastructure/database/repositories/asset-maintenance.repository.impl.ts` | Implementacion repositorio |
| 18 | `src/infrastructure/database/migrations/<timestamp>-asset-maintenance-structure.ts` | Migracion SQL |
| 19 | `src/infrastructure/adapters/api/controllers/asset-maintenance.controller.ts` | Controlador HTTP |
| 20 | `src/infrastructure/modules/asset-maintenance.module.ts` | Modulo NestJS |
| 21 | `src/infrastructure/database/database.module.ts` | **Modificar** para agregar entity y provider |
| 22 | `src/app.module.ts` | **Modificar** para importar AssetMaintenanceModule |

---

## Notas importantes

### Decimal en PostgreSQL con TypeORM
TypeORM retorna las columnas de tipo `decimal`/`numeric` como `string` en JavaScript cuando se usa el driver de PostgreSQL (`pg`). Esto es conocido y se debe manejar explicitamente. En `mapToDomain` del repositorio siempre se usa `Number(entity.cost)` para garantizar que el valor sea numerico.

### El companyId viene del JWT, no del body
El controlador usa `@CompanyId() companyId: string` en todos los endpoints. El decorador `CompanyId` ya existe en `src/common/decorators/company-id.decorator.ts` y extrae el ID de la compania del token JWT autenticado. Nunca se expone `companyId` en el body de las requests del cliente.

### Verificacion de pertenencia a la compania en GET by ID
Cuando se obtiene un mantenimiento por ID, el use case verifica que `maintenance.companyId === companyId`. Si no coincide, lanza `AssetMaintenanceNotFoundException` (no `ForbiddenException`). Este es el patron que usa el proyecto: no se revela si el recurso existe pero pertenece a otra compania.

### Tipo de contrato en los use cases de dominio
El proyecto usa dos estilos: `abstract class` para use cases que son inyectados como dependencia con `useClass` en NestJS (como `CreateAssetLocationUseCase`), e `interface` para use cases donde la impl usa `implements` directamente. Ambos funcionan. En este modulo se usa `abstract class` para el create (que necesita inyeccion en DatabaseModule como abstract provider si se quisiera) e `interface` para los queries. En la practica, dado que los providers en el modulo apuntan directamente a la impl, la distincion es menos relevante.

### Sin cache en este modulo
El modulo no usa Redis/cache. Los otros modulos mas complejos lo usan para listas de lectura frecuente. Para mantenimientos, con la paginacion directa a base de datos es suficiente para empezar. Si el volumen de datos justifica cache en el futuro, se agrega `CacheService` como dependencia en los use cases de lectura.

### Permisos (PermissionsGuard)
El modulo no agrega `PermissionsGuard` ni `@Permissions()` en el ejemplo. Si el proyecto tiene permisos granulares para mantenimientos, se deben agregar las entradas correspondientes en `PermissionsEnum` y decorar cada endpoint con `@Permissions(PermissionsEnum.ASSET_MAINTENANCE_CREATE)` etc., siguiendo el patron de `AssetLocationController`.
