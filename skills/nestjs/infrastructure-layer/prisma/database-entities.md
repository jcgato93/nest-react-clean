# Prisma Schema Models

Prisma models define the structure of data as stored in the database. They live in **one file**, `prisma/schema.prisma` — there is no per-table TS class and no ORM decorators. The generated `@prisma/client` types are what repositories consume; they have no business logic — that belongs in domain entities.

## Rules

1. **Every model declares `id`, `createdAt`, `updatedAt` itself**: Prisma has no model inheritance (unlike TypeORM's `extends BaseEntity`), so these three fields are repeated in every model block. `src/infrastructure/database/base.entity.ts` only exists as a TypeScript-side `BaseEntity` interface (`{ id, createdAt, updatedAt }`) so `BaseRepositoryImpl<T extends BaseEntity, D>` can constrain the Prisma model generic — it has no runtime/schema effect.
2. **Table name**: `snake_case`, plural, set explicitly with `@@map('fixed_assets')` — Prisma does not auto-pluralize or snake_case for you.
3. **Column names**: Prisma keeps the field name you write (`camelCase` in the model) as the TS property, and maps it to a different DB column name only if you add `@map("column_name")` on that field. Add it whenever the DB column isn't the camelCase default.
4. **Nullable columns**: append `?` to the type (e.g. `String?`) — default is required (`NOT NULL`), same intent as TypeORM's `nullable: true` but opposite spelling.
5. **Relations go at the end** of the model, after all scalar fields.
6. **No business logic**: a model block only declares fields/relations — no methods, no validation, no domain rules.

## Field Type Reference

| TypeScript type (as seen by repository code) | Prisma field type |
|-----------------------------------------------|--------------------|
| `string` | `String`, with `@db.VarChar(n)` if you need a length cap |
| `number` (integer) | `Int` |
| `number` (decimal/money) | `Decimal` with `@db.Decimal(precision, scale)` — read it back with `Number(value)` in `mapToDomain`, Prisma returns a `Decimal` instance, not a plain number |
| `boolean` | `Boolean` |
| `Date` | `DateTime`, with `@db.Date` if you only need the date part |
| `string` (long text) | `String` (no `@db.VarChar` cap) or `@db.Text` |
| enum | a Prisma `enum` block, referenced as the field type |

## Full Example

```prisma
// prisma/schema.prisma
enum FixedAssetStatus {
  IN_USE
  UNDER_MAINTENANCE
  DISPOSED
}

model FixedAsset {
  // ─── Basic fields ────────────────────────────────────────────────────────
  id               String            @id @default(uuid()) @db.Uuid
  name             String            @db.VarChar(200)
  code             String            @unique @db.VarChar(100)
  description      String?
  acquisitionCost  Decimal           @db.Decimal(15, 2)
  acquisitionDate  DateTime          @db.Date
  active           Boolean           @default(true)
  status           FixedAssetStatus  @default(IN_USE)
  createdAt        DateTime          @default(now())
  updatedAt        DateTime          @updatedAt

  // ─── Foreign keys (before relations) ────────────────────────────────────
  companyId        String            @db.Uuid
  categoryId       String?           @db.Uuid

  // ─── Relations (always at the end) ──────────────────────────────────────
  company          Company           @relation(fields: [companyId], references: [id])
  category         Category?         @relation(fields: [categoryId], references: [id])
  depreciations    Depreciation[]

  @@map("fixed_assets")
}
```

## Relationship Patterns

### Many-to-one (most common — FK on this model)
```prisma
model FixedAsset {
  companyId String  @db.Uuid
  company   Company @relation(fields: [companyId], references: [id])
}
```

### One-to-many (no FK on this model — FK is on the other side)
```prisma
model FixedAsset {
  depreciations Depreciation[]
}

model Depreciation {
  fixedAssetId String     @db.Uuid
  fixedAsset   FixedAsset @relation(fields: [fixedAssetId], references: [id])
}
```
Both sides of the relation are declared — Prisma needs the `@relation` on the "many" side pointing at the FK, and the inverse array field on the "one" side.

### Many-to-many (implicit join table)
```prisma
model Asset {
  tags Tag[]
}

model Tag {
  assets Asset[]
}
```
Prisma creates and manages the join table automatically for implicit many-to-many (no need to declare a `_AssetToTag` model yourself). Use an explicit join model instead if the relation itself needs extra columns (e.g. `assignedAt`).

### Relating by a non-id column (e.g. `code`)
```prisma
model FixedAsset {
  cityCode String @db.VarChar(250)
  city     City   @relation(fields: [cityCode], references: [code])
}

model City {
  code String @unique @db.VarChar(250)
}
```
> The referenced column (`City.code`) **must** have a `@unique` constraint — Prisma requires it at the schema level (this is stricter than TypeORM, which lets you create a logical-only relation without a DB unique constraint).

## After Creating or Modifying a Model

Always generate a migration:
```bash
pnpm run prisma:migrate:dev --name DescribeChange
```

Review the generated SQL in `prisma/migrations/<timestamp>_describe_change/migration.sql` before it's committed — Prisma sometimes infers a destructive `DROP COLUMN` + `ADD COLUMN` for what was meant to be a rename; if so, edit the migration file to a `RENAME COLUMN` instead. Then, if not already applied by the `dev` command:
```bash
pnpm run prisma:migrate:deploy
```

**Never modify the database schema manually** — always go through migrations. **Never edit the generated `@prisma/client` output** — it's regenerated from `schema.prisma` on every migration.
