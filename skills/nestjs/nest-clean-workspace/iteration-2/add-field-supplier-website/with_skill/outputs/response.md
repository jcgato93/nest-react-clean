# Agregar campo `website` a la entidad Supplier

## Escenario identificado

**Scenario B — Modificar funcionalidad existente**: agregar un campo a una entidad existente.

Según la impact table del skill:

| Cambio | Capas afectadas | Migración |
|--------|-----------------|-----------|
| Add field to entity | Domain → Application → Infrastructure (DB entity + mappers + DTO) | Sí |

Orden de aplicación: **Domain → Application → Infrastructure**. No se modifica ninguna capa de use cases porque el campo es puramente de datos y no introduce nueva lógica de negocio.

---

## Archivos leídos antes de proponer cambios

- `src/domain/supplier/entities/supplier.domain.ts`
- `src/application/supplier/dto/create-supplier.dto.ts`
- `src/application/supplier/dto/update-supplier.dto.ts`
- `src/application/supplier/dto/supplier-detail-response.dto.ts`
- `src/application/supplier/dto/supplier-list-response.dto.ts`
- `src/application/supplier/mapper/supplier.mapper.ts`
- `src/infrastructure/database/entities/supplier.entity.ts`
- `src/infrastructure/database/repositories/supplier.repository.impl.ts`

---

## Capa 1 — Domain

### `src/domain/supplier/entities/supplier.domain.ts`

Agregar `website?: string` a la interfaz `SupplierProps`, al campo privado, al constructor, y al getter/setter.

**Diff:**

```diff
 interface SupplierProps {
   id?: string;
   companyId: string;
   countryId: string;
   identity: string;
   societyTypeId: string;
   identityTypeId: string;
   name: string;
   secondName?: string;
   lastName?: string;
   secondLastName?: string;
   initial?: string;
   legalRepresentativeName?: string;
   legalRepresentativeIdentity?: string;
   firstEconomicActivityId: string;
   secondEconomicActivityId?: string;
   otherEconomicActivityIds?: string[];
   taxResponsibilityIds: string[];
   customObligationIds?: string[];
+  website?: string;
 }
```

```diff
 export class Supplier {
   private readonly _id: string;
   private readonly _companyId: string;
   private _countryId: string;
   private _identity: string;
   private _societyTypeId: string;
   private _identityTypeId: string;
   private _name: string;
   private _secondName?: string;
   private _lastName?: string;
   private _secondLastName?: string;
   private _initial?: string;
   private _fullName: string;
   private _firstEconomicActivityId: string;
   private _secondEconomicActivityId?: string;
   private _legalRepresentativeName?: string;
   private _legalRepresentativeIdentity?: string;
   private _otherEconomicActivityIds?: string[];
   private _taxResponsibilityIds: string[];
   private _customObligationIds?: string[];
+  private _website?: string;
```

```diff
   constructor(props: SupplierProps) {
     this._id = props.id ?? crypto.randomUUID().toString();
     this._companyId = props.companyId;
     this._countryId = props.countryId;
     this._identity = props.identity;
     this._societyTypeId = props.societyTypeId;
     this._identityTypeId = props.identityTypeId;
     this._name = props.name;
     this._secondName = props.secondName;
     this._lastName = props.lastName;
     this._secondLastName = props.secondLastName;
     this._initial = props.initial;
     this._firstEconomicActivityId = props.firstEconomicActivityId;
     this._secondEconomicActivityId = props.secondEconomicActivityId;
     this._legalRepresentativeName = props.legalRepresentativeName;
     this._legalRepresentativeIdentity = props.legalRepresentativeIdentity;
+    this._website = props.website;

     this._fullName = this.calculateFullName();
     // ... resto del constructor sin cambios
   }
```

Agregar getter y setter al final del bloque de accesores (antes de `plainToInstance`):

```diff
+  get website(): string | undefined {
+    return this._website;
+  }
+
+  set website(value: string | undefined) {
+    this._website = value;
+  }
```

---

## Capa 2 — Application

### `src/application/supplier/dto/create-supplier.dto.ts`

Agregar el campo opcional `website` al final de los campos escalares (antes de `taxProfile`):

```diff
+  @ApiProperty({
+    description: 'Sitio web del proveedor',
+    example: 'https://www.proveedor.com',
+    maxLength: 500,
+    required: false,
+  })
+  @IsString()
+  @IsOptional()
+  @MaxLength(500)
+  website?: string;
+
   @ApiProperty({
     description: 'Perfil tributario del proveedor',
     type: CreateSupplierTaxProfileDto,
   })
   @ValidateNested()
   @Type(() => CreateSupplierTaxProfileDto)
   @IsNotEmpty()
   taxProfile: CreateSupplierTaxProfileDto;
```

`update-supplier.dto.ts` hereda de `CreateSupplierDto` sin cambios adicionales, por lo que recibe el campo automáticamente.

### `src/application/supplier/dto/supplier-detail-response.dto.ts`

Agregar el campo `website` en el response DTO de detalle. Se ubica después de `legalRepresentativeIdentity` (siguiendo el orden de los campos existentes):

```diff
   @ApiProperty({
     description: 'Identificación del representante legal',
     example: '123456789',
     required: false,
   })
   @Expose()
   legalRepresentativeIdentity?: string;

+  @ApiProperty({
+    description: 'Sitio web del proveedor',
+    example: 'https://www.proveedor.com',
+    required: false,
+  })
+  @Expose()
+  website?: string;
+
   @ApiProperty({
     description: 'ID de la primera actividad económica',
```

> **Nota**: `supplier-list-response.dto.ts` no se modifica. El campo `website` no aparece en el listado paginado — ese endpoint retorna datos resumidos para tablas, y un campo de URL no es relevante en ese contexto. Si en el futuro se necesita en la lista, se agrega siguiendo el mismo patrón.

### `src/application/supplier/mapper/supplier.mapper.ts`

El mapper `toDomain` debe incluir `website` en la construcción de la entidad:

```diff
   static toDomain(dto: CreateSupplierDto, supplierId?: string): Supplier {
     return new Supplier({
       id: supplierId,
       companyId: dto.companyId,
       countryId: dto.countryId,
       identity: dto.identity,
       societyTypeId: dto.societyTypeId,
       identityTypeId: dto.identityTypeId,
       name: dto.name,
       secondName: dto.secondName,
       lastName: dto.lastName,
       secondLastName: dto.secondLastName,
       initial: dto.initial,
       legalRepresentativeName: dto.legalRepresentativeName,
       legalRepresentativeIdentity: dto.legalRepresentativeIdentity,
       firstEconomicActivityId: dto.taxProfile.firstEconomicActivityId,
       secondEconomicActivityId: dto.taxProfile.secondEconomicActivityId,
       otherEconomicActivityIds: dto.taxProfile.otherEconomicActivityIds,
       taxResponsibilityIds: dto.taxProfile.taxResponsibilityIds,
       customObligationIds: dto.taxProfile.customObligationIds,
+      website: dto.website,
     });
   }
```

---

## Capa 3 — Infrastructure

### `src/infrastructure/database/entities/supplier.entity.ts`

Agregar la columna `website` antes del bloque `// === Relations ===`:

```diff
   @Column({
     type: 'varchar',
     length: 100,
     nullable: true,
     name: 'legal_representative_identity',
   })
   legalRepresentativeIdentity?: string;

+  @Column({ type: 'varchar', length: 500, nullable: true })
+  website?: string;
+
   // === Relations ===
```

### `src/infrastructure/database/repositories/supplier.repository.impl.ts`

**`mapToDomain`** — incluir `website`:

```diff
   mapToDomain(entity: SupplierEntity): Supplier {
     return new Supplier({
       id: entity.id,
       companyId: entity.companyId,
       countryId: entity.countryId,
       identity: entity.identity,
       societyTypeId: entity.societyTypeId,
       identityTypeId: entity.identityTypeId,
       name: entity.name,
       secondName: entity.secondName ?? undefined,
       lastName: entity.lastName ?? undefined,
       secondLastName: entity.secondLastName ?? undefined,
       initial: entity.initial ?? undefined,
       legalRepresentativeName: entity.legalRepresentativeName ?? undefined,
       legalRepresentativeIdentity: entity.legalRepresentativeIdentity ?? undefined,
       firstEconomicActivityId: entity.firstEconomicActivityId,
       secondEconomicActivityId: entity.secondEconomicActivityId ?? undefined,
       otherEconomicActivityIds: [],
       taxResponsibilityIds: [],
       customObligationIds: [],
+      website: entity.website ?? undefined,
     });
   }
```

**`mapToEntity`** — incluir `website`:

```diff
   mapToEntity(domain: Supplier): SupplierEntity {
     const entity = new SupplierEntity();
     entity.id = domain.id;
     entity.companyId = domain.companyId;
     entity.countryId = domain.countryId;
     entity.identity = domain.identity;
     entity.societyTypeId = domain.societyTypeId;
     entity.identityTypeId = domain.identityTypeId;
     entity.name = domain.name;
     entity.secondName = domain.secondName;
     entity.lastName = domain.lastName;
     entity.secondLastName = domain.secondLastName;
     entity.fullName = domain.fullName;
     entity.initial = domain.initial;
     entity.firstEconomicActivityId = domain.firstEconomicActivityId;
     entity.secondEconomicActivityId = domain.secondEconomicActivityId;
     entity.legalRepresentativeName = domain.legalRepresentativeName;
     entity.legalRepresentativeIdentity = domain.legalRepresentativeIdentity;
+    entity.website = domain.website;
     return entity;
   }
```

**`findSupplierById`** — agregar `website` en el `select` y en el objeto que se pasa a `plainToInstance`:

```diff
       select: {
         id: true,
         countryId: true,
         identity: true,
         societyTypeId: true,
         identityTypeId: true,
         name: true,
         secondName: true,
         lastName: true,
         secondLastName: true,
         initial: true,
         legalRepresentativeName: true,
         legalRepresentativeIdentity: true,
+        website: true,
         firstEconomicActivityId: true,
         secondEconomicActivityId: true,
         // ... relaciones sin cambios
       },
```

```diff
     return plainToInstance(SupplierDetailResponseDto, {
       countryId: supplier.countryId,
       countryName: supplier.country.name,
       identity: supplier.identity,
       societyTypeId: supplier.societyTypeId,
       societyTypeName: supplier.societyType.name,
       identityTypeId: supplier.identityTypeId,
       identityTypeName: supplier.identityType.name,
       name: supplier.name,
       secondName: supplier.secondName ?? null,
       lastName: supplier.lastName ?? null,
       secondLastName: supplier.secondLastName ?? null,
       initial: supplier.initial ?? null,
       legalRepresentativeName: supplier.legalRepresentativeName ?? null,
       legalRepresentativeIdentity: supplier.legalRepresentativeIdentity ?? null,
+      website: supplier.website ?? null,
       firstEconomicActivityId: supplier.firstEconomicActivityId,
       // ... resto sin cambios
     });
```

### Migración

Después de guardar los cambios en `supplier.entity.ts`, ejecutar:

```bash
npm run typeorm:migration:generate -- -n AddWebsiteToSuppliers
```

La migración generada contendrá algo similar a:

```sql
ALTER TABLE "suppliers" ADD "website" character varying(500) NULL;
```

Verificar el archivo generado en `src/infrastructure/database/migrations/` antes de correrlo. No se esperan `ADD CONSTRAINT` no deseados para esta columna simple.

```bash
npm run typeorm:migration:run
```

---

## Resumen de archivos modificados

| Archivo | Cambio |
|---------|--------|
| `src/domain/supplier/entities/supplier.domain.ts` | Agregar `website?: string` a props, campo privado, constructor, getter y setter |
| `src/application/supplier/dto/create-supplier.dto.ts` | Agregar campo `website` opcional con `@ApiProperty`, `@IsString`, `@IsOptional`, `@MaxLength(500)` |
| `src/application/supplier/dto/supplier-detail-response.dto.ts` | Agregar campo `website` con `@ApiProperty` y `@Expose()` |
| `src/application/supplier/mapper/supplier.mapper.ts` | Pasar `website: dto.website` en `toDomain` |
| `src/infrastructure/database/entities/supplier.entity.ts` | Agregar `@Column({ type: 'varchar', length: 500, nullable: true }) website?: string` |
| `src/infrastructure/database/repositories/supplier.repository.impl.ts` | Agregar `website` en `mapToDomain`, `mapToEntity`, `select` de `findSupplierById`, y objeto de `plainToInstance` |
| Migración nueva | `AddWebsiteToSuppliers` — agrega columna `website varchar(500) nullable` |

---

## Archivos NO modificados (y por qué)

| Archivo | Razón |
|---------|-------|
| `src/application/supplier/dto/update-supplier.dto.ts` | Hereda de `CreateSupplierDto` — recibe `website` automáticamente |
| `src/application/supplier/dto/supplier-list-response.dto.ts` | El listado paginado es una vista resumida; `website` no aplica en ese contexto |
| Use cases (domain e impl) | El campo no introduce nueva lógica de negocio; los use cases existentes de create/update ya persisten lo que llega del mapper sin cambios |
| Controlador | No cambia la firma de ningún endpoint; el campo fluye transparentemente por los DTOs existentes |
| Módulo | No se agregan nuevos providers ni tokens de inyección |

---

## Verificación final

```bash
npm run start:dev   # sin errores de TypeScript
npm run lint        # sin errores de lint
```

En Swagger (`/api/docs`), el endpoint `GET /suppliers/:id` mostrará `website` como campo opcional en la respuesta, y `POST /suppliers` / `PUT /suppliers/:id` lo mostrarán como campo opcional en el body.
