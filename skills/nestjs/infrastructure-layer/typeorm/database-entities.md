# Database Entities

Database entities define the structure of data as stored in the database. They are TypeORM classes that map to tables and have no business logic — that belongs in domain entities.

## Rules

1. **Always extend `BaseEntity`**: provides `id` (UUID), `createdAt`, `updatedAt` automatically.
2. **Table name**: `snake_case`, plural (e.g., `@Entity('fixed_assets')`).
3. **Column names**: TypeORM maps `camelCase` to `snake_case` by default via the global naming strategy. Only use `name` option in `@Column` when you need a custom column name.
4. **Nullable columns**: add `nullable: true` explicitly — default is `NOT NULL`.
5. **Relations go at the end** of the class, after all `@Column` fields.
6. **No business logic**: no methods, no validation, no domain rules.

## Column Type Reference

| TypeScript type      | TypeORM column type                      |
| -------------------- | ---------------------------------------- |
| `string`             | `'varchar'` with `length`                |
| `number` (integer)   | `'int'`                                  |
| `number` (decimal)   | `'decimal'` with `precision` and `scale` |
| `boolean`            | `'boolean'`                              |
| `Date`               | `'timestamp'` or `'date'`                |
| `string` (long text) | `'text'`                                 |
| enum                 | `'enum'` with `enum: MyEnum`             |

## Full Example

```typescript
// src/infrastructure/database/entities/fixed-asset.entity.ts
import { FixedAssetStatusEnum } from '@/common/constants/enums/fixed-asset-status.enum';

import { BaseEntity } from '../base.entity';
import { CompanyEntity } from './company.entity';
import { DepreciationEntity } from './depreciation.entity';
import { Column, Entity, JoinColumn, ManyToOne, OneToMany } from 'typeorm';

@Entity('fixed_assets')
export class FixedAssetEntity extends BaseEntity {
  // ─── Basic columns ───────────────────────────────────────────────────────────

  @Column({ type: 'varchar', length: 200 })
  name: string;

  @Column({ type: 'varchar', length: 100, unique: true })
  code: string;

  @Column({ type: 'text', nullable: true })
  description: string | null;

  @Column({ type: 'decimal', precision: 15, scale: 2 })
  acquisitionCost: number;

  @Column({ type: 'date' })
  acquisitionDate: Date;

  @Column({ type: 'boolean', default: true })
  active: boolean;

  @Column({
    type: 'enum',
    enum: FixedAssetStatusEnum,
    default: FixedAssetStatusEnum.IN_USE,
  })
  status: FixedAssetStatusEnum;

  // ─── Foreign key columns (before relations) ──────────────────────────────────

  @Column({ type: 'varchar', length: 36, name: 'company_id' })
  companyId: string;

  @Column({ type: 'varchar', length: 36, name: 'category_id', nullable: true })
  categoryId: string | null;

  // ─── Relations (always at the end) ───────────────────────────────────────────

  @ManyToOne(() => CompanyEntity)
  @JoinColumn({ name: 'company_id' })
  company?: CompanyEntity;

  @ManyToOne(() => CategoryEntity, { nullable: true })
  @JoinColumn({ name: 'category_id' })
  category?: CategoryEntity;

  @OneToMany(() => DepreciationEntity, (dep) => dep.fixedAsset)
  depreciations?: DepreciationEntity[];
}
```

## Relationship Patterns

### ManyToOne (most common — FK on this table)

```typescript
@Column({ type: 'varchar', length: 36, name: 'company_id' })
companyId: string;

@ManyToOne(() => CompanyEntity)
@JoinColumn({ name: 'company_id' })
company?: CompanyEntity;
```

### OneToMany (no FK on this table — FK is on the other side)

```typescript
@OneToMany(() => DepreciationEntity, (dep) => dep.fixedAsset)
depreciations?: DepreciationEntity[];
```

### ManyToMany (join table)

```typescript
@ManyToMany(() => TagEntity)
@JoinTable({
  name: 'asset_tags',
  joinColumn: { name: 'asset_id' },
  inverseJoinColumn: { name: 'tag_id' },
})
tags?: TagEntity[];
```

### Joining by a non-id column (e.g., code)

```typescript
@ManyToOne(() => CityEntity)
@JoinColumn({ name: 'city_code', referencedColumnName: 'code' })
city?: CityEntity;
```

> If the referenced column has no UNIQUE constraint in the DB, remove the `ADD CONSTRAINT` line from the generated migration — the relation will still work in TypeORM queries.

## After Creating or Modifying an Entity

Always generate a migration:

```bash
npm run typeorm:migration:generate --name=DescribeChange
```

Review the generated file before running it. Then:

```bash
npm run typeorm:migration:run
```

**Never modify the database schema manually** — always go through migrations.
