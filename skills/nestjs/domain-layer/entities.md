# Entities

Entities are core objects that encapsulate both data and behavior related to a specific domain concept. They are characterized by having a unique identity that persists over time, regardless of changes to their attributes.

## Guidelines for Defining Entities

- **Unique Identity**: Each entity must have a unique identifier (e.g., ID) that distinguishes it from other entities, even if their attributes are identical.
- **Behavior and State**: Entities should encapsulate both data (attributes) and behavior (methods) that operate on that data.
- **Lifecycle Management**: Entities should manage their own lifecycle, including creation, updates, and deletion.
- **Domain Logic**: Business rules and domain logic should be implemented within entities to ensure consistency and integrity.
- **Persistence Ignorance**: Entities should not be concerned with how they are stored or retrieved from a database; this responsibility lies with repositories.
- **Equality**: Two entities are considered equal if they share the same unique identity, regardless of their attribute values.
- **Cache Reconstruction Methods**: **TODAS** las entidades del dominio **DEBEN** implementar los métodos estáticos `plainToInstance` y `plainToInstanceList` para facilitar la reconstrucción de instancias desde objetos planos (caché, respuestas serializadas, etc.).

## Critical Rules for Private Fields

- **Private field type must match the getter return type**: if the getter returns `Email`, the private field must be `private _email: Email`, not `private _email: string`.
- **Mutations must assign to the private field**: use `this._email = newEmail`, never `this.email = newEmail` (which tries to assign to a read-only getter and will fail at runtime).
- **`id` is always optional in the interface**: the entity generates a UUID if `id` is not provided.

### Example of an Entity

```typescript
// src/domain/users/entities/user.entity.ts
import { Email } from '../value-objects/email.value-object';
import { Username } from '../value-objects/username.value-object';

export interface IUser {
  id?: string;         // optional — generated if not provided
  username: Username;
  email: Email;
}

export class User {
  private _id: string;
  private _username: Username; // ✅ type matches getter return type
  private _email: Email;       // ✅ type matches getter return type

  constructor(props: IUser) {
    this._id = props.id ?? crypto.randomUUID();
    this._username = props.username;
    this._email = props.email;
  }

  get id(): string { return this._id; }
  get username(): Username { return this._username; }
  get email(): Email { return this._email; }

  // ✅ Assign to private field, NOT to the getter
  changeEmail(newEmail: Email): void {
    this._email = newEmail;
  }

  // Additional domain logic methods

  /**
   * Convierte un objeto plano en una instancia de User.
   * Útil para reconstruir instancias desde caché o respuestas serializadas.
   *
   * IMPORTANTE: Este método es OBLIGATORIO en todas las entidades del dominio.
   * Permite que los datos almacenados en caché (objetos planos sin métodos)
   * sean reconstruidos como instancias completas con todos sus métodos.
   *
   * @param raw - Objeto plano con las propiedades de User
   * @returns Instancia de User con todos sus métodos disponibles
   *
   * @example
   * const cachedData = await cache.get('user:123');
   * const user = User.plainToInstance(cachedData);
   * user.changeEmail(newEmail); // Ahora los métodos están disponibles
   */
  static plainToInstance(raw: any): User {
    const instance: User = Object.create(User.prototype);
    Object.assign(instance, raw);
    return instance;
  }

  /**
   * Convierte un array de objetos planos en un array de instancias de User.
   * Útil para reconstruir listas desde caché o respuestas paginadas.
   *
   * @param rawArray - Array de objetos planos
   * @returns Array de instancias de User
   *
   * @example
   * const cachedList = await cache.get('users:all');
   * const users = User.plainToInstanceList(cachedList);
   * users.forEach(user => console.log(user.email.value));
   */
  static plainToInstanceList(rawArray: any[]): User[] {
    return rawArray.map((raw) => this.plainToInstance(raw));
  }
}
```

## Common Mistakes to Avoid

```typescript
// ❌ WRONG — private field type doesn't match getter
export class User {
  private _email: string;  // string, but getter returns Email
  get email(): Email { return this._email; } // TypeScript error
}

// ✅ CORRECT — private field type matches getter
export class User {
  private _email: Email;
  get email(): Email { return this._email; }
}

// ❌ WRONG — assigning to getter (runtime error)
changeEmail(newEmail: Email): void {
  this.email = newEmail; // Cannot assign to read-only getter
}

// ✅ CORRECT — assigning to private field
changeEmail(newEmail: Email): void {
  this._email = newEmail;
}
```
