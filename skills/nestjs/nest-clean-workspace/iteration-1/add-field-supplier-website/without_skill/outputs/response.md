# Add `website` field to Supplier entity

## Overview

The field `website` is optional (`string | undefined`). It needs to flow through every layer of the Clean Architecture stack:

1. Domain entity (`supplier.domain.ts`)
2. Infrastructure DB entity (`supplier.entity.ts`)
3. Infrastructure DB migration (new file)
4. Infrastructure repository impl (`supplier.repository.impl.ts`) — `mapToEntity`, `mapToDomain`, `findSupplierById`, and `findSuppliersByFilters`
5. Application DTOs — `create-supplier.dto.ts`, `supplier-detail-response.dto.ts`, `supplier-list-response.dto.ts`
6. Application mapper (`supplier.mapper.ts`)

`update-supplier.dto.ts` inherits from `CreateSupplierDto`, so it picks up the field automatically. The use-case implementations do not need to change because they work with the domain entity, and the field carries through via the mapper.

---

## File changes (in order)

### 1. Domain entity — `src/domain/supplier/entities/supplier.domain.ts`

Add `website` to `SupplierProps` and the class body.

```diff
 interface SupplierProps {
   ...
   customObligationIds?: string[];
+  website?: string;
 }

 export class Supplier {
   ...
   private _customObligationIds?: string[];
+  private _website?: string;

   constructor(props: SupplierProps) {
     ...
     this._customObligationIds = Supplier.uniqueIds(props.customObligationIds);
+    this._website = props.website;

     Supplier.validateNoDuplicateEconomicActivities({ ... });
   }

   ...

   get customObligationIds(): string[] | undefined { ... }
   set customObligationIds(value: string[] | undefined) { ... }

+  get website(): string | undefined {
+    return this._website;
+  }
+
+  set website(value: string | undefined) {
+    this._website = value;
+  }
 }
```

Full updated sections only (no changes to existing methods):

```typescript
// Inside SupplierProps interface
website?: string;

// Inside Supplier class private fields
private _website?: string;

// Inside constructor, after customObligationIds line
this._website = props.website;

// New getter/setter (add after customObligationIds setter)
get website(): string | undefined {
  return this._website;
}

set website(value: string | undefined) {
  this._website = value;
}
```

---

### 2. Infrastructure DB entity — `src/infrastructure/database/entities/supplier.entity.ts`

Add the column after `legalRepresentativeIdentity`:

```diff
   @Column({
     type: 'varchar',
     length: 100,
     nullable: true,
     name: 'legal_representative_identity',
   })
   legalRepresentativeIdentity?: string;

+  @Column({
+    type: 'varchar',
+    length: 500,
+    nullable: true,
+    name: 'website',
+  })
+  website?: string;

   // === Relations ===
```

---

### 3. Migration — `src/infrastructure/database/migrations/<timestamp>-add-website-to-supplier.ts`

Create a new migration file. Use the next available timestamp (example: `1772700000000`).

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddWebsiteToSupplier1772700000000 implements MigrationInterface {
  name = 'AddWebsiteToSupplier1772700000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "suppliers" ADD "website" character varying(500)`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "suppliers" DROP COLUMN "website"`,
    );
  }
}
```

> The timestamp must be unique and greater than all existing migration timestamps. The latest in the repo is `1772663013023`. Pick any value greater than that — e.g., `1772700000000`.

---

### 4. Infrastructure repository impl — `src/infrastructure/database/repositories/supplier.repository.impl.ts`

Four locations need updating:

#### 4a. `mapToEntity` — persist the field

```diff
   mapToEntity(domain: Supplier): SupplierEntity {
     ...
     entity.legalRepresentativeIdentity = domain.legalRepresentativeIdentity;
+    entity.website = domain.website;
     return entity;
   }
```

#### 4b. `mapToDomain` — reconstruct the domain object from DB row

```diff
   mapToDomain(entity: SupplierEntity): Supplier {
     return new Supplier({
       ...
       legalRepresentativeIdentity: entity.legalRepresentativeIdentity ?? undefined,
+      website: entity.website ?? undefined,
       firstEconomicActivityId: entity.firstEconomicActivityId,
       ...
     });
   }
```

#### 4c. `findSupplierById` — include the field in `select` and in the `plainToInstance` call

```diff
   select: {
     ...
     legalRepresentativeIdentity: true,
+    website: true,
     firstEconomicActivityId: true,
     ...
   }

   return plainToInstance(SupplierDetailResponseDto, {
     ...
     legalRepresentativeIdentity: supplier.legalRepresentativeIdentity ?? null,
+    website: supplier.website ?? null,
     firstEconomicActivityId: supplier.firstEconomicActivityId,
     ...
   });
```

#### 4d. `findSuppliersByFilters` — `website` is NOT included in the list response DTO (see decision note below), so no changes are needed here unless you decide to expose it in the list view too. See section 5c.

---

### 5. Application DTOs

#### 5a. `src/application/supplier/dto/create-supplier.dto.ts`

Add the field after `legalRepresentativeIdentity`:

```diff
+import {
+  IsNotEmpty,
+  IsOptional,
+  IsString,
+  IsUUID,
+  IsUrl,
+  MaxLength,
+  ValidateNested,
+} from 'class-validator';

   @ApiProperty({
     description: 'Número de identificación del representante legal',
     example: '80123456',
     maxLength: 100,
     required: false,
   })
   @IsString()
   @IsOptional()
   @MaxLength(100)
   legalRepresentativeIdentity?: string;

+  @ApiProperty({
+    description: 'Sitio web del proveedor',
+    example: 'https://www.empresa.com',
+    maxLength: 500,
+    required: false,
+  })
+  @IsUrl({}, { message: 'website debe ser una URL válida' })
+  @IsOptional()
+  @MaxLength(500)
+  website?: string;

   @ApiProperty({
     description: 'Perfil tributario del proveedor',
```

> `@IsUrl()` from `class-validator` validates the format. If you prefer a plain string without URL validation, replace it with `@IsString()`. Either way, `@IsOptional()` ensures the field is not required.

#### 5b. `src/application/supplier/dto/supplier-detail-response.dto.ts`

Add the field after `legalRepresentativeIdentity`:

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
+    example: 'https://www.empresa.com',
+    required: false,
+  })
+  @Expose()
+  website?: string;

   @ApiProperty({
     description: 'ID de la primera actividad económica',
```

#### 5c. `src/application/supplier/dto/supplier-list-response.dto.ts` (optional)

The list DTO currently omits `website`. If the product requirement is to show it in the list view too, add:

```diff
   @ApiPropertyOptional({
     description: 'Teléfonos de la sucursal principal',
     type: [String],
     example: ['3001234567'],
   })
   @Expose()
   phones?: string[];

+  @ApiPropertyOptional({
+    description: 'Sitio web del proveedor',
+    example: 'https://www.empresa.com',
+  })
+  @Expose()
+  website?: string;
 }
```

If it is only needed in the detail view, skip this change and also skip updating `findSuppliersByFilters` in the repository.

---

### 6. Application mapper — `src/application/supplier/mapper/supplier.mapper.ts`

```diff
   static toDomain(dto: CreateSupplierDto, supplierId?: string): Supplier {
     return new Supplier({
       ...
       legalRepresentativeIdentity: dto.legalRepresentativeIdentity,
+      website: dto.website,
       firstEconomicActivityId: dto.taxProfile.firstEconomicActivityId,
       ...
     });
   }
```

---

## Summary table

| File | Change type | What changes |
|---|---|---|
| `src/domain/supplier/entities/supplier.domain.ts` | Edit | Add `website` to `SupplierProps`, private field, constructor assignment, getter/setter |
| `src/infrastructure/database/entities/supplier.entity.ts` | Edit | Add `@Column` for `website` |
| `src/infrastructure/database/migrations/<ts>-add-website-to-supplier.ts` | Create | `ALTER TABLE suppliers ADD website varchar(500)` |
| `src/infrastructure/database/repositories/supplier.repository.impl.ts` | Edit | `mapToEntity`, `mapToDomain`, `findSupplierById` (select + plainToInstance) |
| `src/application/supplier/dto/create-supplier.dto.ts` | Edit | Add `website` field with `@IsUrl @IsOptional @MaxLength` + `@ApiProperty` |
| `src/application/supplier/dto/supplier-detail-response.dto.ts` | Edit | Add `@Expose() @ApiProperty` for `website` |
| `src/application/supplier/dto/supplier-list-response.dto.ts` | Edit (optional) | Add `@Expose() @ApiPropertyOptional` for `website` in list view |
| `src/application/supplier/mapper/supplier.mapper.ts` | Edit | Pass `website: dto.website` to `new Supplier(...)` |
| `src/application/supplier/dto/update-supplier.dto.ts` | No change | Inherits from `CreateSupplierDto` automatically |

---

## Notes

- **Migration timestamp**: use any integer larger than `1772663013023` (the current last migration). A safe value is `Date.now()` at the time you generate the file, or simply pick `1772700000000`.
- **`full_name` column**: not affected by this change.
- **`UpdateSupplierDto`**: already extends `CreateSupplierDto`, so `website` is inherited with no extra work.
- **Use-case implementations**: `CreateSupplierUseCaseImpl` and `UpdateSupplierUseCaseImpl` receive a `Supplier` domain object from the mapper and pass it straight to the repository. Because `website` is carried on the domain object, neither use-case needs to change.
- **`@IsUrl` vs `@IsString`**: `@IsUrl()` (from `class-validator`) performs RFC-compliant URL validation. If the field can hold arbitrary strings (e.g. without scheme), replace with `@IsString()`.
