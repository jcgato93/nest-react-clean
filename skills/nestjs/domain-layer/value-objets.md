### Value Objects for Domain Validation

Use value objects for domain primitives that require validation:

- For common value objects, place them in `src/domain/common/value-objects/`

```typescript
// src/domain/common/value-objects/email.value-object.ts
export class Email {
    constructor(private readonly _value: string) {
        this.validate();
    }
    
    get value(): string { return this._value; }
    
    private validate(): void {
        // Validation logic throws on invalid input
    }
}
```

- For module-specific value objects, place them in `src/domain/<module>/value-objects/`

```typescript
// src/domain/users/value-objects/username.value-object.ts
export class Username {
    constructor(private readonly _value: string) {
        this.validate();
    }

    get value(): string { return this._value; }

    private validate(): void {
        // Validation logic throws on invalid input
    }
}
```