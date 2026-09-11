# AggregateRoot & DomainEvent Base Classes

## DomainEvent Base

**File**: `src/domain/common/domain-event.base.ts`

Every domain event in this project extends `DomainEvent`. The base class stores the `aggregateId` and requires each subclass to declare an `eventName` string.

```typescript
// src/domain/common/domain-event.base.ts
export abstract class DomainEvent {
  abstract readonly eventName: string;

  constructor(public readonly aggregateId: string) {}
}
```

**What to pass as `aggregateId`**: the ID of the root entity that originated the event — always `this._id` from inside the entity constructor or method.

---

## AggregateRoot Base

**File**: `src/domain/common/aggregate-root.base.ts`

Aggregates (entities that can emit domain events) extend `AggregateRoot<T>` instead of `Entity<T>`.

```typescript
export abstract class AggregateRoot<T> extends Entity<T> {
  private _domainEvents: DomainEvent[] = [];

  get domainEvents(): DomainEvent[] {
    return [...this._domainEvents]; // returns a copy — caller cannot mutate the internal list
  }

  protected addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  clearDomainEvents(): void {
    this._domainEvents = [];
  }

  protected hasDomainEvent(
    eventClass: new (...args: unknown[]) => DomainEvent,
  ): boolean {
    return this._domainEvents.some((e) => e instanceof eventClass);
  }
}
```

### API at a Glance

| Method | Visibility | When to call |
|--------|-----------|--------------|
| `addDomainEvent(event)` | `protected` | Inside entity constructor or domain behavior method |
| `domainEvents` (getter) | `public` | In use case, after persistence, to collect events to publish |
| `clearDomainEvents()` | `public` | In use case, immediately before calling `emitAsync` |
| `hasDomainEvent(Class)` | `protected` | Inside the entity to avoid registering the same event twice |

### Preventing Duplicate Events

Use `hasDomainEvent` when a behavior can be called multiple times and must only emit the event once:

```typescript
activate(): void {
  if (this._active) return;
  this._active = true;
  if (!this.hasDomainEvent(AssetActivatedEvent)) {
    this.addDomainEvent(new AssetActivatedEvent(this._id));
  }
}
```

---

## When to Use Entity vs AggregateRoot

| Use | Extends |
|-----|---------|
| Entity that never emits domain events | `Entity<Props>` |
| Entity that can emit domain events | `AggregateRoot<Props>` |

Changing from `Entity` to `AggregateRoot` later is safe — it only adds the domain event API without changing persistence or other behavior.

### Decision Guide

Use **`Entity<Props>`** when:
- The entity only exists as part of another aggregate (e.g., `CompanyAddress`, `AssetDepreciationLine`)
- No side effect in the system needs to react to its creation or mutation
- It is read/written exclusively through its parent aggregate root

Use **`AggregateRoot<Props>`** when:
- Creating the entity should trigger initialization of related data in another module (e.g., creating a `Company` → initialize default asset groups)
- A status change on the entity must trigger a cross-module reaction (e.g., `Asset` transferred → recalculate depreciation)
- The entity is the entry point of its own transactional boundary

---

## Full Entity Example — AggregateRoot

Below is the minimal structure for a domain entity that emits an event on creation:

```typescript
// src/domain/asset/entities/asset.domain.ts
import { AggregateRoot } from '@/domain/common/aggregate-root.base';
import { AssetCreatedEvent } from '../events/asset-created';

export interface AssetProps {
  id?: string;
  companyId: string;
  name: string;
  // ... rest of props
}

export class Asset extends AggregateRoot<AssetProps> {
  private readonly _companyId: string;
  private _name: string;

  constructor(props: AssetProps) {
    super({ id: props.id, props });
    this._companyId = props.companyId;
    this._name = props.name.toLowerCase().trim();

    this.addDomainEvent(new AssetCreatedEvent(this._id, this._companyId));
  }

  get id(): string { return this._id; }
  get companyId(): string { return this._companyId; }
  get name(): string { return this._name; }
}
```

### Emitting from a behavior method (not constructor)

For events triggered by state changes rather than creation, place `addDomainEvent` inside the domain method that causes the change:

```typescript
transfer(newLocationId: string): void {
  this._locationId = newLocationId;
  this.addDomainEvent(new AssetTransferredEvent(this._id, this._companyId, newLocationId));
}
```

The use case calls `asset.transfer(newLocationId)` and then publishes the collected events after `save()` — the same publish pattern as for creation events.
