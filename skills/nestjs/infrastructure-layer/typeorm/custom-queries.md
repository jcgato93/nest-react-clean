# Custom Queries Pattern

## Overview

Este patrón se utiliza cuando necesitas retornar datos específicos de una consulta que difieren de la entidad de dominio estándar. Es común cuando:

- Necesitas incluir información de relaciones (joins) en el response
- Requieres campos calculados o agregados
- El DTO de respuesta tiene campos adicionales que no están en la entidad de dominio
- Quieres optimizar queries seleccionando solo campos específicos

**Principio clave**: Las queries personalizadas retornan DTOs directamente desde el repositorio, no entidades de dominio.

## Cuándo usar este patrón

✅ **Usar cuando:**

- Necesitas información de tablas relacionadas (ej: nombre de ciudad desde `cities`)
- El response incluye campos que vienen de múltiples tablas
- Requieres optimización de queries con `select` específicos
- El caso de uso es estrictamente de lectura (query)

❌ **NO usar cuando:**

- Operaciones de escritura (create, update, delete)
- La entidad de dominio ya tiene toda la información necesaria
- No hay relaciones o campos adicionales requeridos

## Paso a Paso: Implementación

### 1. Actualizar o Crear el DTO de Respuesta

**Ubicación**: `src/application/{module}/dto/{name}-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';

import { Expose } from 'class-transformer';

export class BusinessEstablishmentListResponseDto {
  @Expose()
  @ApiProperty()
  id: string;

  @Expose()
  @ApiProperty()
  name: string;

  @Expose()
  @ApiProperty()
  cityId?: string;

  // ✨ Campo adicional de la relación
  @Expose()
  @ApiProperty()
  cityName?: string;

  @Expose()
  @ApiProperty()
  active: boolean;
}
```

**Reglas:**

- Usar `@Expose()` para todos los campos que deben serializarse
- Usar `@ApiProperty()` para documentación Swagger
- Mantener tipos explícitos
- Usar `?` para campos opcionales

### 2. Agregar Query Method en Repositorio Domain

**Ubicación**: `src/domain/{module}/repositories/{module}.repository.ts`

```typescript
import { PageOptionsDto } from '@/application/pagination/dtos';
import { SomeResponseDto } from '@/application/{module}/dto/some-response.dto';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

export abstract class BusinessEstablishmentRepository extends BaseRepository<BusinessEstablishment> {
  // ... métodos existentes ...

  // =========== QUERIES ============
  /**
   * Obtiene los establecimientos de una compañía con información de la ciudad.
   * Retorna los establecimientos con el nombre de la ciudad incluido.
   * @param companyId - ID de la compañía
   * @param page - Opciones de paginación
   * @returns Lista paginada de establecimientos con información de ciudad
   */
  abstract getEstablishmentsWithCityByCompanyId(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<BusinessEstablishmentListResponseDto>>;
}
```

**Reglas:**

- Colocar queries al final del repositorio después del comentario `// =========== QUERIES ============`
- Usar nombres descriptivos que indiquen qué información adicional retorna (ej: `WithCity`, `WithDetails`)
- Documentar con JSDoc: propósito, parámetros y tipo de retorno
- Retornar `Promise<PaginationResponse<DTO>>` si usa paginación
- Retornar `Promise<DTO>` o `Promise<DTO[]>` si no usa paginación
- **IMPORTANTE**: El tipo de retorno debe ser el DTO, NO la entidad de dominio

### 3. Implementar Query en Repositorio Infrastructure

**Ubicación**: `src/infrastructure/database/repositories/{module}.repository.impl.ts`

```typescript
import { SomeResponseDto } from '@/application/{module}/dto/some-response.dto';

import { DataSource, EntityManager } from 'typeorm';

@Injectable()
export class BusinessEstablishmentRepositoryImpl
  extends BaseRepositoryImpl<BusinessEstablishmentEntity, BusinessEstablishment>
  implements BusinessEstablishmentRepository
{
  constructor(protected dataSource: DataSource) {
    super(BusinessEstablishmentEntity, dataSource);
  }

  // ... métodos existentes ...

  // =========== QUERIES ============
  async getEstablishmentsWithCityByCompanyId(
    companyId: string,
    page: PageOptionsDto,
    manager?: EntityManager,
  ): Promise<PaginationResponse<BusinessEstablishmentListResponseDto>> {
    const [establishments, count] =
      await this.getRepository(manager).findAndCount({
        where: { companyId },
        order: { position: 'ASC' },
        take: page.take,
        skip: page.skip,
        select: {
          id: true,
          name: true,
          cityCode: true,
          active: true,
          // Solo seleccionar campos necesarios
        },
        relations: {
          city: true, // Incluir relación
        },
      });

    // Mapear a DTO manualmente con tipado fuerte
    const items: BusinessEstablishmentListResponseDto[] = establishments.map(
      (establishment) => {
        const dto = new BusinessEstablishmentListResponseDto();

        dto.id = establishment.id;
        dto.name = establishment.name;
        dto.cityId = establishment.cityCode;
        dto.cityName = establishment.city?.name; // Campo de la relación
        dto.active = establishment.active;

        return dto;
      },
    );

    return {
      items,
      count,
    };
  }
}
```

**Reglas importantes:**

- Importar el DTO desde `@/application`
- Usar `findAndCount` para queries con paginación
- Usar objeto notation para `relations`: `{ city: true }` (NO usar array de strings)
- En `select`, especificar solo los campos necesarios como objeto: `{ id: true, name: true }`
- **CRÍTICO**: NO usar objetos anónimos - siempre crear instancias tipadas del DTO
  - Declarar el tipo del array: `const items: SomeDto[] = ...`
  - Crear instancia del DTO: `const dto = new SomeDto()`
  - Asignar propiedades individualmente
  - Esto permite detección de errores de tipado en tiempo de compilación
- Acceder a campos de relaciones con optional chaining: `establishment.city?.name`
- Retornar objeto con estructura `{ items, count }` para paginación

### 4. Actualizar Use Case Interface (Domain)

**Ubicación**: `src/domain/{module}/use-cases/{name}.use-case.ts`

```typescript
import { PageOptionsDto } from '@/application/pagination/dtos';
import { SomeResponseDto } from '@/application/{module}/dto/some-response.dto';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

export interface GetBusinessEstablishmentsUseCase {
  /**
   * Retorna los establecimientos comerciales de una compañía.
   * Incluye el nombre de la ciudad de cada establecimiento.
   * @param companyId - ID de la compañía
   * @param page - Opciones de paginación
   * @returns Lista paginada de establecimientos con información de ciudad
   */
  execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<SomeResponseDto>>;
}
```

**Reglas:**

- Cambiar el tipo de retorno de la entidad de dominio al DTO
- Importar el DTO desde `@/application`
- Actualizar documentación JSDoc
- Mantener la misma firma del método `execute`

### 5. Actualizar Use Case Implementation (Application)

**Ubicación**: `src/application/{module}/use-cases/{name}.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { PageOptionsDto } from '@/application/pagination/dtos';
import { SomeResponseDto } from '@/application/{module}/dto/some-response.dto';
import { SomeRepository } from '@/domain/{module}/repositories/{module}.repository';
import { GetSomeUseCase } from '@/domain/{module}/use-cases/get-some.use-case';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

@Injectable()
export class GetBusinessEstablishmentsUseCaseImpl implements GetBusinessEstablishmentsUseCase {
  constructor(
    private readonly establishmentRepository: BusinessEstablishmentRepository,
  ) {}

  async execute(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<BusinessEstablishmentListResponseDto>> {
    // Llamar directamente al método de query del repositorio
    return this.establishmentRepository.getEstablishmentsWithCityByCompanyId(
      companyId,
      page,
    );
  }
}
```

**Reglas:**

- Importar el DTO desde `@/application`
- Actualizar el tipo de retorno para que coincida con la interfaz
- Llamar directamente al método de query del repositorio
- NO transformar ni mapear, el repositorio ya retorna el DTO correcto

### 6. Actualizar Controlador (si es necesario)

**Ubicación**: `src/infrastructure/adapters/api/controllers/{module}.controller.ts`

**El controlador generalmente NO requiere cambios** si ya usa `PaginationService.createPageDto`, ya que este servicio transforma automáticamente usando el DTO especificado:

```typescript
@Get('company/:companyId')
@UseGuards(Auth0Guard)
@ApiBearerAuth()
@ApiOperation({ summary: 'Get company business establishments' })
@ApiPaginatedResponse(BusinessEstablishmentListResponseDto)
async getEstablishments(
  @Query() pageOptionsDto: PageOptionsDto,
  @Param('companyId') companyId: string,
): Promise<PageDto<BusinessEstablishmentListResponseDto>> {
  const { items, count } = await this.getEstablishmentsUseCaseImpl.execute(
    companyId,
    pageOptionsDto,
  );

  return PaginationService.createPageDto(
    items,
    count,
    pageOptionsDto,
    BusinessEstablishmentListResponseDto, // El DTO ya viene con la estructura correcta
  );
}
```

## Configuración de Relaciones en Entidades

### Agregar Relación en la Entidad

**Ubicación**: `src/infrastructure/database/entities/{module}.entity.ts`

```typescript
import { CityEntity } from './city.entity';
import { JoinColumn, ManyToOne } from 'typeorm';

@Entity('business_establishment')
export class BusinessEstablishmentEntity extends BaseEntity {
  // ... columnas existentes ...

  @Column({ type: 'varchar', length: 250, nullable: false, name: 'city_code' })
  cityCode: string;

  // Relación al final de la entidad
  @ManyToOne(() => CityEntity)
  @JoinColumn({ name: 'city_code', referencedColumnName: 'code' })
  city?: CityEntity;
}
```

**Reglas para relaciones:**

- Colocar relaciones al final de la entidad después de todas las columnas
- Usar `@ManyToOne`, `@OneToMany`, `@OneToOne` según corresponda
- Usar `@JoinColumn` para especificar la columna de la foreign key
- `referencedColumnName` debe apuntar al campo correcto de la entidad relacionada
- Hacer la propiedad opcional con `?` si la columna es nullable
- **IMPORTANTE**: Si la columna referenciada no tiene UNIQUE constraint individual en BD, la relación será solo lógica en TypeORM (sin FK en BD)

### Crear Migración

```bash
npm run typeorm:migration:generate --name=AddRelationToEntity
```

Si la migración intenta crear una FK pero la columna referenciada no tiene UNIQUE constraint:

1. Editar la migración para eliminar la línea de `ADD CONSTRAINT`
2. La relación funcionará en TypeORM para queries pero sin integridad referencial en BD

## Ejemplo Completo

Ver la implementación completa en:

- DTO: `src/application/business-establishment/dto/business-establishment-list-response.dto.ts`
- Repositorio Domain: `src/domain/business-establishment/repositories/business-establishment.repository.ts`
- Repositorio Impl: `src/infrastructure/database/repositories/business-establishment.repository.impl.ts`
- Use Case Domain: `src/domain/business-establishment/use-cases/get-business-establishments.use-case.ts`
- Use Case Impl: `src/application/business-establishment/use-cases/get-business-establishments.use-case.impl.ts`

## Tipos de Queries y su Ubicación

### Queries Simples (Dominio)

Las queries simples que retornan una entidad completa o con relaciones directas se definen en el **repositorio del dominio**:

```typescript
// domain/repositories/company.repository.ts
export abstract class CompanyRepository extends BaseRepository<Company> {
  // Comandos - retornan dominio para validación de reglas de negocio
  abstract save(company: Company): Promise<Company>;

  // Consultas simples - retornan dominio
  abstract findById(id: string): Promise<Company | null>;
  abstract findByCode(code: string): Promise<Company | null>;

  // =========== QUERIES ============
  // Consultas con relaciones - retornan DTOs directamente
  abstract findWithUsers(id: string): Promise<CompanyWithUsersDto>;
  abstract findWithTaxes(id: string): Promise<CompanyWithTaxesDto>;
  abstract getEconomyActivitiesByCompanyId(
    id: string,
  ): Promise<CompanyEconomyActivitiesDto>;
}
```

**Principios:**

- Comandos (create, update, delete) siempre retornan dominio para validar reglas de negocio
- Queries simples (findById, findByEmail) retornan dominio
- Queries con relaciones o joins retornan DTOs para optimización
- Se definen en el repositorio del dominio porque son específicas de esa entidad

### Queries Complejas (Aplicación)

Para casos más complejos que requieren información de **múltiples entidades o módulos** (ej: Dashboard), se definen a **nivel de aplicación**:

**Ubicación**: `src/application/queries/{feature}/`

#### Ejemplo: Dashboard Query

```typescript
// application/queries/dashboard/dashboard.dto.ts
export class DashboardDto {
  @Expose()
  @ApiProperty()
  userStats: UserStatsDto;

  @Expose()
  @ApiProperty()
  companyStats: CompanyStatsDto;

  @Expose()
  @ApiProperty()
  revenueStats: RevenueStatsDto;

  @Expose()
  @ApiProperty({ type: [ActivityDto] })
  recentActivities: ActivityDto[];

  @Expose()
  @ApiProperty()
  taxSummary: TaxSummaryDto;
}
```

```typescript
// application/queries/dashboard/get-dashboard.query.ts
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class GetDashboardQuery {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly companyRepository: CompanyRepository,
    private readonly revenueRepository: RevenueRepository,
    private readonly activityRepository: ActivityRepository,
  ) {}

  async execute(filters: DashboardFiltersDto): Promise<DashboardDto> {
    // Ejecutar consultas en paralelo cuando sea posible
    const [userStats, companyStats, revenueStats, activities] =
      await Promise.all([
        this.userRepository.getUserStats(filters),
        this.companyRepository.getCompanyStats(filters),
        this.revenueRepository.getRevenueStats(filters),
        this.activityRepository.getRecentActivities(filters),
      ]);

    const dto = new DashboardDto();
    dto.userStats = userStats;
    dto.companyStats = companyStats;
    dto.revenueStats = revenueStats;
    dto.recentActivities = activities;
    dto.taxSummary = this.calculateTaxSummary(companyStats);

    return dto;
  }

  private calculateTaxSummary(companyStats: CompanyStatsDto): TaxSummaryDto {
    // Lógica de agregación si es necesaria
    const summary = new TaxSummaryDto();
    // ... cálculos
    return summary;
  }
}
```

**Si se requiere un repositorio específico para queries complejas:**

**Ubicación**: `src/infrastructure/database/repositories/queries/{feature}.repository.ts`

```typescript
// infrastructure/database/repositories/queries/dashboard.repository.ts
import { Injectable } from '@nestjs/common';
import { DataSource, FindOptionsWhere } from 'typeorm';

@Injectable()
export class DashboardQueryRepository {
  constructor(protected dataSource: DataSource) {}

  /**
   * Obtiene la lista paginada de compañías asociadas a un usuario.
   * Se seleccionan solo los campos necesarios y se mapea fuertemente a DTOs.
   * @param userId - ID del usuario (opcional) para filtrar compañías
   * @param page - Opciones de paginación
   */
  async getListCompany(
    userId: string | undefined,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<CompanyListResponseDto>> {
    // Construir filtros con tipado fuerte
    const filters: FindOptionsWhere<CompanyEntity> = {};
    if (userId) {
      filters.userId = userId;
    }

    // Ejecutar consulta con paginación y selección de campos necesarios
    const [companies, count] = await this.dataSource
      .getRepository(CompanyEntity)
      .findAndCount({
        where: filters,
        order: { firstName: 'ASC' },
        take: page.take,
        skip: page.skip,
        select: {
          id: true,
          document: true,
          verificationDigit: true,
          firstName: true,
          lastName: true,
          logoUrl: true,
          active: true,
        },
      });

    // Mapear a DTOs con tipado fuerte (NO objetos anónimos)
    const items: CompanyListResponseDto[] = companies.map((company) => {
      const dto = new CompanyListResponseDto();
      dto.id = company.id;
      dto.document = company.document;
      dto.verificationDigit = company.verificationDigit;
      dto.firstName = company.firstName;
      dto.lastName = company.lastName;
      dto.logoUrl = company.logoUrl;
      dto.active = company.active;
      return dto;
    });

    return { items, count };
  }
}
```

**Estructura de carpetas para queries complejas:**

```
src/application/
  queries/
    dashboard/
      dashboard.dto.ts
      get-dashboard.query.ts
      dashboard-filters.dto.ts
    reports/
      report.dto.ts
      generate-report.query.ts
```

**Cuándo usar cada ubicación:**

| Tipo de Query | Ubicación | Ejemplo |
|---------------|-----------|---------|
| Query de una entidad con relaciones directas | `domain/{module}/repositories/` | `getEstablishmentsWithCity()` |
| Query que agrega datos de múltiples módulos | `application/queries/{feature}/` | `getDashboard()` |
| Query SQL compleja multi-tabla | `infrastructure/database/repositories/queries/` | `getAggregatedStats()` |

## Checklist de Implementación

- [ ] Actualizar o crear DTO con campos adicionales (incluir `@Expose()` y `@ApiProperty()`)
- [ ] Agregar método abstract en repositorio domain con tipo de retorno DTO
- [ ] Implementar query en repositorio infrastructure
  - [ ] Usar `select` con solo campos necesarios
  - [ ] Usar `relations` con notación de objeto
  - [ ] **Mapear con tipado fuerte**: crear instancias del DTO, NO objetos anónimos
  - [ ] Declarar tipo del array: `const items: SomeDto[] = ...`
  - [ ] Retornar `{ items, count }` para paginación
- [ ] Actualizar interfaz de use case con tipo de retorno DTO
- [ ] Actualizar implementación de use case para llamar al nuevo método
- [ ] Verificar que controlador use el DTO correcto en decoradores
- [ ] Agregar relación en entidad si es necesaria
- [ ] Generar y ejecutar migración
- [ ] Verificar que no hay errores de compilación con `get_errors`

## Errores Comunes y Soluciones

### Error: "Column does not support length property"
**Causa**: Tipo de columna incorrecto para la relación
**Solución**: Verificar que el tipo de la columna coincida con el tipo del campo referenciado

### Error: "There is no unique constraint matching given keys"
**Causa**: La columna referenciada no tiene UNIQUE constraint individual
**Solución**:
1. Eliminar la línea de `ADD CONSTRAINT` de la migración
2. La relación funcionará solo en TypeORM sin FK en BD

### Error: "Property 'execute' is not assignable"
**Causa**: Tipo de retorno del use case impl no coincide con la interfaz
**Solución**: Actualizar ambos para que retornen `Promise<PaginationResponse<DTO>>`

### Error: "Unsafe return of type any"
**Causa**: Método de repositorio retorna `Promise<any>` en lugar del tipo específico
**Solución**: Especificar el tipo completo `Promise<PaginationResponse<DTO>>`

## Notas Importantes

1. **Separación de Responsabilidades**:
   - Repositorio domain: Define el contrato (abstract method)
   - Repositorio infrastructure: Implementa la query y mapeo a DTO
   - Use case: Orquesta la llamada al repositorio
   - Controlador: Maneja HTTP, delega a use case

2. **No mezclar patrones**:
   - Comandos (create, update, delete) retornan entidades de dominio
   - Queries simples retornan entidades de dominio
   - Queries con relaciones/joins retornan DTOs directamente
   - Queries complejas multi-módulo se definen en `application/queries/`

3. **Tipado Fuerte (CRÍTICO)**:
   - **NUNCA usar objetos anónimos** en el mapeo
   - **SIEMPRE crear instancias tipadas**: `const dto = new SomeDto()`
   - **SIEMPRE declarar el tipo del array**: `const items: SomeDto[] = ...`
   - Asignar propiedades individualmente: `dto.id = entity.id`
   - Esto permite que TypeScript detecte errores si el DTO cambia
   - Ejemplo correcto:
     ```typescript
     const items: BusinessEstablishmentDto[] = entities.map(entity => {
       const dto = new BusinessEstablishmentDto();
       dto.id = entity.id;
       dto.name = entity.name;
       return dto;
     });
     ```
   - Ejemplo incorrecto (NO HACER):
     ```typescript
     const items = entities.map(entity => ({
       id: entity.id,
       name: entity.name
     })); // ❌ Objeto anónimo sin tipo
     ```

4. **Optimización**:
   - Usar `select` para traer solo campos necesarios
   - Usar paginación para grandes volúmenes de datos
   - Ejecutar queries en paralelo con `Promise.all()` cuando sea posible

5. **Type Safety**:
   - Siempre especificar tipos explícitos
   - Evitar usar `any` en cualquier parte del código
   - Aprovechar el sistema de tipos de TypeScript para detectar errores

6. **Documentación**:
   - Comentar métodos de query con JSDoc
   - Indicar qué información adicional retorna la query
   - Especificar el propósito y casos de uso
