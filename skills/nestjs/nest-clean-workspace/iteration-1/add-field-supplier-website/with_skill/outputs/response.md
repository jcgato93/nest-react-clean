# Agregar campo `website` a la entidad Supplier

## Identificacion del escenario

**Escenario B — Modificar funcionalidad existente.**
El cambio solicitado es: agregar un campo `website` (string, opcional) a la entidad `Supplier`. El campo debe aparecer en el response DTO y en Swagger.

Segun la tabla de impacto del skill:

| Cambio solicitado | Capas afectadas | Migracion |
|---|---|---|
| Agregar campo a entidad | Domain -> Application -> Infrastructure (DB entity + mappers + DTO) | Si |

---

## Archivos leidos antes de proponer cambios

Segun el skill (Scenario B), se leen primero todos los archivos afectados:

| Archivo | Motivo |
|---|---|
| `src/domain/supplier/entities/supplier.domain.ts` | Entidad de dominio — agregar el campo aqui primero |
| `src/application/supplier/dto/supplier-detail-response.dto.ts` | Response DTO del detalle — debe exponer `website` |
| `src/application/supplier/dto/supplier-list-response.dto.ts` | Response DTO de lista — evaluar si aplica |
| `src/application/supplier/dto/create-supplier.dto.ts` | DTO de creacion — agregar campo opcional |
| `src/application/supplier/dto/update-supplier.dto.ts` | DTO de actualizacion (extiende create) — heredara el campo |
| `src/application/supplier/mapper/supplier.mapper.ts` | Mapper — debe incluir `website` en `toDomain` |
| `src/infrastructure/database/entities/supplier.entity.ts` | Entidad TypeORM — agregar columna `website` |
| `src/infrastructure/database/repositories/supplier.repository.impl.ts` | Repositorio — actualizar `mapToDomain`, `mapToEntity` y `findSupplierById` |
| `src/infrastructure/adapters/api/controllers/supplier.controller.ts` | Controlador — no requiere cambios (ya usa los DTOs correctos) |

---

## Hallazgos clave de la lectura

- `UpdateSupplierDto` extiende `CreateSupplierDto` directamente — al agregar `website` en `CreateSupplierDto`, el campo queda disponible en ambas operaciones automaticamente.
- `findSupplierById` en el repositorio construye el `SupplierDetailResponseDto` con `plainToInstance` a partir de un objeto literal — se debe agregar `website` en ese objeto.
- El controlador para GET by ID devuelve `SupplierDetailResponseDto` directamente desde el use case (que a su vez lo obtiene del repositorio) — no hay mapper intermedio para la respuesta de detalle; los cambios van solo en el repositorio y el DTO.
- `SupplierListResponseDto` no incluye `website` — es razonable no agregarlo ahi ya que es un listado resumido, pero se deja como decision explicita (ver nota al final).
- El campo `fullName` en la entidad TypeORM se calcula en el dominio; `website` es un campo simple sin logica derivada.

---

## Cambios por capa (Domain -> Application -> Infrastructure)

### Capa 1 — Domain

**Archivo:** `src/domain/supplier/entities/supplier.domain.ts`

Agregar `website?: string` en la interfaz `SupplierProps`, declarar el campo privado `_website`, asignarlo en el constructor, y agregar el getter (y setter, por consistencia con otros campos opcionales).

```typescript
// En la interfaz SupplierProps — agregar:
website?: string;

// En los campos privados de la clase — agregar:
private _website?: string;

// En el constructor — agregar:
this._website = props.website;

// Agregar getter y setter despues del getter de `initial`:
get website(): string | undefined {
  return this._website;
}

set website(value: string | undefined) {
  this._website = value;
}
```

Diff completo del constructor (solo la linea nueva, el resto no cambia):

```typescript
constructor(props: SupplierProps) {
  // ... campos existentes ...
  this._initial = props.initial;
  this._website = props.website;  // <-- NUEVO
  // ... resto del constructor ...
}
```

---

### Capa 2 — Application

#### 2a. DTO de creacion

**Archivo:** `src/application/supplier/dto/create-supplier.dto.ts`

Agregar el campo `website` como opcional al final de la clase, antes del campo `taxProfile` (o despues de `legalRepresentativeIdentity`, manteniendo agrupacion logica):

```typescript
import { IsUrl } from 'class-validator'; // agregar al import existente

@ApiProperty({
  description: 'Sitio web del proveedor',
  example: 'https://www.empresa.com',
  maxLength: 500,
  required: false,
})
@IsOptional()
@IsUrl()
@MaxLength(500)
website?: string;
```

Nota: `IsUrl` viene de `class-validator` (ya es dependencia del proyecto). Si el equipo prefiere una validacion mas permisiva (sin requerir protocolo), puede usarse `@IsString()` en su lugar. Se recomienda `@IsUrl()` para asegurar que el valor sea una URL valida.

> `UpdateSupplierDto` extiende `CreateSupplierDto` sin sobreescribir nada, por lo que hereda `website` automaticamente. No requiere cambios.

#### 2b. Response DTO de detalle

**Archivo:** `src/application/supplier/dto/supplier-detail-response.dto.ts`

Agregar el campo `website` con `@ApiProperty` y `@Expose()`. Por ser opcional se usa `@ApiPropertyOptional` para que Swagger lo muestre como no requerido:

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger'; // ApiPropertyOptional ya esta importado en este archivo

@ApiPropertyOptional({
  description: 'Sitio web del proveedor',
  example: 'https://www.empresa.com',
  required: false,
})
@Expose()
website?: string;
```

Ubicacion sugerida: despues del campo `legalRepresentativeIdentity` y antes de `firstEconomicActivityId`, manteniendo el agrupamiento de datos basicos del proveedor.

#### 2c. Mapper

**Archivo:** `src/application/supplier/mapper/supplier.mapper.ts`

Agregar `website` en el metodo `toDomain` para que el campo se pase a la entidad de dominio al crear/actualizar:

```typescript
static toDomain(dto: CreateSupplierDto, supplierId?: string): Supplier {
  return new Supplier({
    // ... campos existentes ...
    legalRepresentativeIdentity: dto.legalRepresentativeIdentity,
    website: dto.website,  // <-- NUEVO
    firstEconomicActivityId: dto.taxProfile.firstEconomicActivityId,
    // ... resto ...
  });
}
```

No se requiere un `toDto` en este mapper porque la respuesta de detalle se construye directamente en el repositorio con `plainToInstance`.

---

### Capa 3 — Infrastructure

#### 3a. Entidad TypeORM

**Archivo:** `src/infrastructure/database/entities/supplier.entity.ts`

Agregar la columna `website` como `varchar` nullable, ubicada antes del bloque de relaciones:

```typescript
@Column({
  type: 'varchar',
  length: 500,
  nullable: true,
  name: 'website',
})
website?: string;
```

#### 3b. Repositorio — `mapToDomain`

**Archivo:** `src/infrastructure/database/repositories/supplier.repository.impl.ts`

Actualizar `mapToDomain` para incluir `website`:

```typescript
mapToDomain(entity: SupplierEntity): Supplier {
  return new Supplier({
    // ... campos existentes ...
    legalRepresentativeIdentity:
      entity.legalRepresentativeIdentity ?? undefined,
    website: entity.website ?? undefined,  // <-- NUEVO
    firstEconomicActivityId: entity.firstEconomicActivityId,
    // ...
  });
}
```

#### 3c. Repositorio — `mapToEntity`

Actualizar `mapToEntity` para incluir `website`:

```typescript
mapToEntity(domain: Supplier): SupplierEntity {
  const entity = new SupplierEntity();
  // ... campos existentes ...
  entity.legalRepresentativeIdentity = domain.legalRepresentativeIdentity;
  entity.website = domain.website;  // <-- NUEVO
  return entity;
}
```

#### 3d. Repositorio — `findSupplierById`

El metodo construye el DTO con `plainToInstance` a partir de un objeto literal. Se deben hacer dos ajustes:

1. Agregar `website` en el bloque `select` de la query:

```typescript
select: {
  // ... campos existentes ...
  legalRepresentativeIdentity: true,
  website: true,  // <-- NUEVO
  firstEconomicActivityId: true,
  // ...
},
```

2. Agregar `website` en el objeto pasado a `plainToInstance`:

```typescript
return plainToInstance(SupplierDetailResponseDto, {
  // ... campos existentes ...
  legalRepresentativeIdentity: supplier.legalRepresentativeIdentity ?? null,
  website: supplier.website ?? null,  // <-- NUEVO
  firstEconomicActivityId: supplier.firstEconomicActivityId,
  // ...
});
```

#### 3e. Migracion

Despues de modificar la entidad TypeORM, generar la migracion:

```bash
npm run typeorm:migration:generate -- -n AddWebsiteToSuppliers
```

Revisar el archivo generado en `src/infrastructure/database/migrations/`. El diff esperado es un simple `ALTER TABLE suppliers ADD COLUMN website varchar(500) NULL`.

---

## Resumen de todos los archivos modificados

| Archivo | Tipo de cambio |
|---|---|
| `src/domain/supplier/entities/supplier.domain.ts` | Agregar campo `website` en interfaz, campo privado, constructor, getter y setter |
| `src/application/supplier/dto/create-supplier.dto.ts` | Agregar campo `website` opcional con `@IsOptional`, `@IsUrl`, `@MaxLength`, `@ApiProperty` |
| `src/application/supplier/dto/supplier-detail-response.dto.ts` | Agregar campo `website` con `@ApiPropertyOptional` y `@Expose()` |
| `src/application/supplier/mapper/supplier.mapper.ts` | Agregar `website: dto.website` en `toDomain` |
| `src/infrastructure/database/entities/supplier.entity.ts` | Agregar `@Column` para `website` (varchar 500, nullable) |
| `src/infrastructure/database/repositories/supplier.repository.impl.ts` | Actualizar `mapToDomain`, `mapToEntity` y `findSupplierById` |
| Migration generada | `AddWebsiteToSuppliers` — ALTER TABLE suppliers ADD COLUMN website |

**Archivos que NO requieren cambios:**
- `src/application/supplier/dto/update-supplier.dto.ts` — hereda el campo via `extends CreateSupplierDto`
- `src/infrastructure/adapters/api/controllers/supplier.controller.ts` — ya usa los DTOs correctos, ningun cambio de firma
- `src/application/supplier/use-cases/update-supplier.use-case.impl.ts` — no opera sobre campos individuales, delega al repositorio
- `src/application/supplier/use-cases/create-supplier.use-case.impl.ts` — idem
- `src/application/supplier/dto/supplier-list-response.dto.ts` — el listado resumido no incluye `website` (decision deliberada; si se requiere, agregar el mismo patron de `@ApiPropertyOptional` + `@Expose()` y seleccionarlo en `findSuppliersByFilters`)

---

## Verificacion final (checklist del skill)

- [ ] `npm run start:dev` — sin errores de compilacion TypeScript
- [ ] `npm run lint` — sin errores de lint
- [ ] Swagger en `/api/docs` — endpoint GET `/supplier/:id` muestra `website` en el schema de `SupplierDetailResponseDto` como campo opcional
- [ ] Revisar la migracion generada antes de ejecutarla — debe ser solo un `ADD COLUMN`, sin `ADD CONSTRAINT` no deseados
- [ ] `npm run typeorm:migration:run` — migracion aplicada correctamente

---

## Nota sobre `SupplierListResponseDto`

El campo `website` NO se agrego a `SupplierListResponseDto` porque ese DTO representa una vista resumida del proveedor en listados paginados. Agregar `website` ahi requeriria tambien seleccionarlo en `findSuppliersByFilters`, lo que aumenta el payload de todos los listados. Si el equipo decide que el listado tambien debe incluirlo, los pasos son:

1. Agregar en `SupplierListResponseDto`: `@ApiPropertyOptional` + `@Expose()` + `website?: string`
2. En `findSuppliersByFilters` dentro del repositorio, agregar `website: true` en el bloque `select`
