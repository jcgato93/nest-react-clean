---
name: domain-layer
description: Use when creating or modifying any domain layer component in this NestJS project: entities, value objects, repository interfaces, use case interfaces, or domain exceptions. Always implement the domain layer FIRST before working on application or infrastructure layers. Use this skill for any request like "create entity X", "add domain exception Y", "define repository interface for Z", "add use case interface", "create value object". This skill must be used whenever code needs to go in src/modules/{module}/domain/ or src/domain/common/.
---

## Domain Layer Overview

The **Domain Layer** is the core of the application — pure business logic, framework-agnostic. It defines WHAT the system does, not HOW.

Components:
- **Entities**: Business objects with identity and behavior
- **Value Objects**: Immutable, validated primitives
- **Repository Interfaces**: Data access contracts (no implementation)
- **Use Case Interfaces**: Operation contracts (no implementation)
- **Domain Exceptions**: Business rule violations

### File placement

| Component | Location |
|-----------|----------|
| Module-specific entities, repos, use cases, exceptions | `src/modules/{module}/domain/` |
| Shared value objects (Email, Money, etc.) | `src/domain/common/value-objects/` |
| Shared base exceptions | `src/domain/common/exceptions/` |

**Rule**: Nothing in a module's `domain/` may import from `application/` or `infrastructure/`. It may import from `src/domain/common/` (shared domain).

**Language rule**: All file names, class names, code identifiers, and exception/error messages are in **English** — even when the user describes the feature in Spanish. The product/API is English-only (updated 2026-08-08; an earlier version of this rule said exception messages should be Spanish — that was wrong and every exception message in the codebase was corrected). Only internal code comments and JSDoc may stay in Spanish.

---

## 1. Creating an Entity

**File**: `src/modules/{module}/domain/entities/{entity}.domain.ts`

**Step-by-step**:
1. Define an interface `I{Entity}` with all properties (use value object types where applicable)
2. Declare private fields with `_` prefix matching the interface types
3. Constructor assigns each field (generate `id` with `crypto.randomUUID()` if not provided)
4. Add public getters for each field
5. Add domain behavior methods (state mutations, business rules)
6. Implement `static plainToInstance(raw: any): Entity` and `static plainToInstanceList(raw: any[]): Entity[]`

```typescript
// src/modules/{module}/domain/entities/{entity}.domain.ts
import { Email } from '@/domain/common/value-objects/email.value-object';

export interface IUser {
  id?: string;           // optional — generated if not provided
  name: string;
  email: Email;          // use Value Object types, not primitives
}

export class User {
  private _id: string;
  private _name: string;
  private _email: Email; // private field type matches the Value Object

  constructor(props: IUser) {
    this._id = props.id ?? crypto.randomUUID();
    this._name = props.name;
    this._email = props.email;
  }

  get id(): string { return this._id; }
  get name(): string { return this._name; }
  get email(): Email { return this._email; }

  // Domain behavior — mutations use private field, never the getter
  changeEmail(newEmail: Email): void {
    this._email = newEmail; // ✅ assign to private field, not getter
  }

  /**
   * Reconstruye una instancia desde un objeto plano (ej: caché Redis).
   * OBLIGATORIO en todas las entidades.
   */
  static plainToInstance(raw: any): User {
    const instance: User = Object.create(User.prototype);
    Object.assign(instance, raw);
    return instance;
  }

  static plainToInstanceList(rawArray: any[]): User[] {
    return rawArray.map((raw) => this.plainToInstance(raw));
  }
}
```

**Rules**:
- Private field type must match the getter return type (e.g., `private _email: Email`, not `string`)
- Mutations must assign to `this._field`, never `this.field` (which is a getter)
- `id` is always optional in the interface — the entity generates one if absent
- `plainToInstance` and `plainToInstanceList` are MANDATORY in every entity

See [entities.md](./entities.md) for full documentation.

---

## 2. Creating a Value Object

**File**:
- Shared/reusable: `src/domain/common/value-objects/{name}.value-object.ts`
- Module-specific: `src/modules/{module}/domain/value-objects/{name}.value-object.ts`

**Step-by-step**:
1. Create a class with a single `private readonly _value` field
2. Constructor calls `this.validate()` after assigning the value
3. Add a getter `value` returning the primitive
4. Implement `private validate()` that throws a domain exception on invalid input

```typescript
// src/domain/common/value-objects/email.value-object.ts
import { CustomBadRequestException } from '../exceptions/custom-bad-request.exception';

export class Email {
  constructor(private readonly _value: string) {
    this.validate();
  }

  get value(): string { return this._value; }

  private validate(): void {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(this._value)) {
      throw new CustomBadRequestException('El correo electrónico no es válido.');
    }
  }
}
```

See [value-objects.md](./value-objets.md) for full documentation.

---

## 3. Creating a Repository Interface

**File**: `src/modules/{module}/domain/repositories/{module}.repository.ts`

**Step-by-step**:
1. Create an abstract class extending `BaseRepository<DomainEntity>`
2. Declare only the methods NOT already in `BaseRepository` (findById, save, delete are inherited)
3. Add JSDoc explaining expected behavior — these comments guide the infrastructure implementer
4. For queries with joins/aggregations, place them after a `// =========== QUERIES ============` comment and return DTOs, not domain entities

```typescript
// src/modules/{module}/domain/repositories/{module}.repository.ts
import { BaseRepository } from '@/infrastructure/database/base.repository';
import { Product } from '../entities/product.domain';

export abstract class ProductRepository extends BaseRepository<Product> {
  /**
   * Busca un producto por su código SKU.
   * Retorna null si no existe.
   */
  abstract findBySku(sku: string): Promise<Product | null>;

  /**
   * Verifica si existe un producto con el SKU dado (excluyendo un ID).
   * Útil para validar unicidad en actualizaciones.
   */
  abstract existsBySku(sku: string, excludeId?: string): Promise<boolean>;

  // =========== QUERIES ============
  /**
   * Retorna lista paginada con información de categoría incluida.
   * Ver: infrastructure-layer skill → custom-queries.md
   */
  abstract getListWithCategory(
    companyId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<ProductListResponseDto>>;
}
```

**Rules**:
- Do NOT redeclare methods already in `BaseRepository` (e.g., `findById`, `save`, `findAll`)
- JSDoc comments serve as instructions for the implementer — be explicit about behavior
- Queries that return DTOs go in the `QUERIES` section at the bottom

---

## 4. Creating a Use Case Interface

**File**: `src/modules/{module}/domain/use-cases/{action}.use-case.ts`

**Step-by-step**:
1. Create an `interface` (not class) with a single `execute` method
2. Add JSDoc listing the business steps the implementation must follow
3. Parameters and return type use domain entities or primitives — NOT DTOs (except for complex queries)

```typescript
// src/modules/{module}/domain/use-cases/create-product.use-case.ts
import { Product } from '../entities/product.domain';

export interface CreateProductUseCase {
  /**
   * 1. Verificar que la compañía exista
   * 2. Verificar que el SKU no esté duplicado
   * 3. Crear y guardar el producto
   * 4. Invalidar caché relacionado
   * @param companyId - ID de la compañía propietaria
   * @param product - Entidad del dominio con los datos del producto
   * @returns ID del producto creado
   */
  execute(companyId: string, product: Product): Promise<string>;
}
```

**Rules**:
- Interface only — no implementation, no `@Injectable()`
- JSDoc must describe the business flow step by step
- Return type is domain entity or primitive, NEVER a DTO (unless it's a query use case returning complex joined data)

See [use-cases.md](./use-cases.md) for full documentation.

---

## 5. Creating a Domain Exception

**File**: `src/modules/{module}/domain/exceptions/{entity}-{reason}.exception.ts`

**Step-by-step**:
1. Create a class extending the appropriate base exception from `src/domain/common/exceptions/`
2. Constructor calls `super(message)` with a descriptive English message
3. Name the class clearly reflecting the business rule violated

Available base exceptions:
- `CustomNotFoundException` → 404
- `CustomBadRequestException` → 400
- `CustomUnauthorizedException` → 401
- `CustomConflictException` → 409

```typescript
// src/modules/{module}/domain/exceptions/product-not-found.exception.ts
import { CustomNotFoundException } from '@/domain/common/exceptions/custom-not-found.exception';

export class ProductNotFoundException extends CustomNotFoundException {
  constructor(productId: string) {
    super(`Product with ID ${productId} not found.`);
  }
}
```

**Rules**:
- Error messages are ALWAYS in English (project convention, updated 2026-08-08 — the product/API is English-only; the earlier version of this skill said Spanish, that was wrong)
- Name must reflect the specific business rule, not a generic HTTP error
- Never expose technical details (stack traces, SQL errors) in the message

See [exceptions.md](./exceptions.md) for full documentation.

---

## Checklist Before Moving to Application Layer

- [ ] Entity has all required fields with correct private types matching getters
- [ ] Entity has `plainToInstance` and `plainToInstanceList` static methods
- [ ] Value objects validate in constructor and throw domain exceptions
- [ ] Repository interface extends `BaseRepository` and only declares new methods
- [ ] Repository interface has JSDoc on every method
- [ ] Use case interface has step-by-step JSDoc and correct return type
- [ ] Domain exceptions extend the correct base and have English messages
- [ ] No imports from `application/` or `infrastructure/` anywhere in the module's `domain/`
