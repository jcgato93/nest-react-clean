---
name: domain-events
description: Use when implementing or connecting domain events in this NestJS + Clean Architecture project. Trigger for phrases like "emit event", "domain event", "publish event", "listen to event", "event handler", "when company is created initialize X", "after creating X trigger Y", "when X happens do Y automatically". Use this skill to: create a DomainEvent class, emit domain events from an AggregateRoot, publish events from a use case after persistence, create event listener classes, and register handlers in a NestJS module.
---

## Domain Events Overview

Domain events communicate that something meaningful happened in the domain. They decouple side effects (e.g., initializing default data, sending notifications) from the core transaction.

**Flow in this project:**

```
Entity constructor → addDomainEvent(new XEvent(id))
  ↓
Use case (after save) → collect events → clear → emitAsync via EventEmitter2
  ↓
@OnEvent listener → execute side-effect use case
```

**Key files:**
- Base event: `src/domain/common/domain-event.base.ts`
- Aggregate root: `src/domain/common/aggregate-root.base.ts`
- Events folder: `src/domain/{module}/events/{event-name}.ts`
- Handlers folder: `src/infrastructure/adapters/event-handlers/{module}.handlers.ts`

**Rule**: Domain events are created in the domain layer. Publishing and listening happen in application/infrastructure layers. Never import `EventEmitter2` inside `src/domain/`.

---

## 1. Creating a Domain Event

**File**: `src/domain/{module}/events/{aggregate}-{past-tense}.ts`

**Step-by-step**:
1. Export a `const` string for the event name — used as the `@OnEvent` key
2. Create a class extending `DomainEvent`
3. Declare `readonly eventName` matching the constant
4. Constructor receives the aggregate ID and any extra data, calls `super(aggregateId)`

```typescript
// src/domain/company/events/company-created.ts
import { DomainEvent } from '@/domain/common/domain-event.base';

export const COMPANY_CREATED_EVENT = 'company.created';

export class CompanyCreatedEvent extends DomainEvent {
  readonly eventName = COMPANY_CREATED_EVENT;

  constructor(public readonly companyId: string) {
    super(companyId); // aggregateId
  }
}
```

**Rules**:
- Event name string follows the pattern `'{aggregate}.{past-tense}'` (e.g., `'company.created'`, `'asset.transferred'`)
- All fields are `readonly` — events are immutable
- Only include data needed by listeners; do not embed entire domain entities

See [aggregate-root.md](./aggregate-root.md) for the base classes.

---

## 2. Emitting a Domain Event from an AggregateRoot Entity

Domain events are registered inside the entity constructor (or a domain method) using `addDomainEvent`.

**Step-by-step**:
1. Entity must extend `AggregateRoot<Props>` (not `Entity`)
2. In the constructor, call `this.addDomainEvent(new XEvent(this._id))`
3. The event is collected in memory — it will be published after persistence

```typescript
// src/domain/company/entities/company.domain.ts
import { AggregateRoot } from '@/domain/common/aggregate-root.base';
import { CompanyCreatedEvent } from '../events/company-created';

export class Company extends AggregateRoot<CompanyProps> {
  constructor(props: CompanyProps) {
    super({ id: props.id, props });
    // ... assign fields ...

    // Register domain event — dispatched after the entity is persisted
    this.addDomainEvent(new CompanyCreatedEvent(this._id));
  }
}
```

**Rules**:
- Only call `addDomainEvent` in constructors or explicit domain behavior methods (not in setters)
- `addDomainEvent` is `protected` — only the aggregate itself can register events
- For conditional events (e.g., only on status change), call `addDomainEvent` inside the domain behavior method that causes the change

See [aggregate-root.md](./aggregate-root.md) for the full `AggregateRoot` API.

---

## 3. Publishing Domain Events from a Use Case

After the aggregate is persisted, collect the domain events, clear them from the entity, and emit each one.

**Step-by-step**:
1. Inject `EventEmitter2` from `@nestjs/event-emitter` in the use case constructor
2. After the persistence call, collect events with `aggregate.domainEvents`
3. Clear the aggregate's event list with `aggregate.clearDomainEvents()`
4. Emit each event with `this.eventEmitter.emitAsync(e.eventName, e)`
5. Use `Promise.all` to emit all events concurrently

```typescript
// src/application/company/use-cases/create-company.use-case.impl.ts
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class CreateCompanyUseCaseImpl implements CreateCompanyUseCase {
  constructor(
    private readonly companyRepository: CompanyRepository,
    private readonly cacheService: CacheService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async execute(authId: string, company: Company): Promise<string> {
    // ... validations ...

    const companyId =
      await this.companyRepository.createCompanyWithRelations(company);

    await this.cacheService.removeCacheByPartialKey(RedisKeyEnum.COMPANY);

    // Publicar domain events después de que la transacción se completa
    const events = company.domainEvents;
    company.clearDomainEvents();
    await Promise.all(
      events.map((e) => this.eventEmitter.emitAsync(e.eventName, e)),
    );

    return companyId;
  }
}
```

**Rules**:
- Always clear events **before** emitting — prevents re-publishing if `execute` is called again
- Use `emitAsync` (not `emit`) so the handler's `Promise` is awaited
- Publish events **after** persistence and cache invalidation — never before saving
- `EventEmitter2` is provided globally by `EventEmitterModule.forRoot()` in `AppModule` — no extra import in the use case's module is needed

---

## 4. Creating an Event Handler (Listener)

**File**: `src/infrastructure/adapters/event-handlers/{module}.handlers.ts`

**Step-by-step**:
1. Create an `@Injectable()` class
2. Inject the use case(s) needed to handle the side effect — use the **abstract class token** from the domain
3. Add a method decorated with `@OnEvent(EVENT_CONSTANT)` — must be `async` and return `Promise<void>`
4. Delegate to the use case inside the handler

```typescript
// src/infrastructure/adapters/event-handlers/assets-group.handlers.ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';

import { InitializeAssetGroupsUseCase } from '@/domain/asset-group/use-cases/initialize-asset-groups.use-case';
import {
  COMPANY_CREATED_EVENT,
  CompanyCreatedEvent,
} from '@/domain/company/events/company-created';

@Injectable()
export class AssetsGroupListener {
  constructor(
    private readonly initializeAssetGroupsUseCase: InitializeAssetGroupsUseCase,
  ) {}

  @OnEvent(COMPANY_CREATED_EVENT)
  async handleCompanyCreatedEvent(payload: CompanyCreatedEvent): Promise<void> {
    await this.initializeAssetGroupsUseCase.execute(payload.companyId);
  }
}
```

**Rules**:
- Handler class name: `{Module}Listener` (e.g., `AssetsGroupListener`, `DepreciationListener`)
- Handler method name: `handle{EventName}` (e.g., `handleCompanyCreatedEvent`)
- Inject use case by the **domain abstract class token**, not the `Impl` class directly
- Keep handlers thin — delegate all logic to use cases
- One handler class per domain module (group multiple `@OnEvent` methods in the same class)

See [event-handlers.md](./event-handlers.md) for full handler documentation.

---

## 5. Registering the Handler in a NestJS Module

The listener **must be a provider** in a NestJS module to be discovered by the event emitter.

**Step-by-step**:
1. Add the listener to the `providers` array of the module that owns the side-effect
2. If the use case token (abstract class) is not already provided in that module, add it with `{ provide: AbstractUseCase, useClass: UseCaseImpl }`
3. The module that triggers the event (e.g., `CompanyModule`) does NOT need to import the handler

```typescript
// src/infrastructure/modules/asset-group.module.ts
import { Module } from '@nestjs/common';

import { InitializeAssetGroupsUseCaseImpl } from '@/application/asset-group/use-cases/initialize-asset-groups.use-case.impl';
import { InitializeAssetGroupsUseCase } from '@/domain/asset-group/use-cases/initialize-asset-groups.use-case';
import { AssetsGroupListener } from '@/infrastructure/adapters/event-handlers/assets-group.handlers';

import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  providers: [
    {
      provide: InitializeAssetGroupsUseCase,
      useClass: InitializeAssetGroupsUseCaseImpl,
    },
    AssetsGroupListener,
  ],
  exports: [InitializeAssetGroupsUseCase],
})
export class AssetGroupModule {}
```

**Rules**:
- `AssetsGroupListener` must be in `providers` — NestJS won't register `@OnEvent` decorators on classes outside the DI container
- `EventEmitterModule.forRoot()` must be in `AppModule` imports (already configured globally)
- The module that emits the event (e.g., `CompanyModule`) does NOT need to import the handler's module — event emitter is global
- The listener's module must be imported in `AppModule` (or the root module) so it is bootstrapped

---

## Checklist

- [ ] Event class extends `DomainEvent`, has `readonly eventName`, and calls `super(aggregateId)`
- [ ] Event name constant follows `'{aggregate}.{past-tense}'` pattern
- [ ] Entity extends `AggregateRoot`, not `Entity`
- [ ] `addDomainEvent` is called in constructor or behavior method — not in setters
- [ ] Use case injects `EventEmitter2` with `private readonly`
- [ ] Events are published **after** persistence and cache invalidation
- [ ] `clearDomainEvents()` is called before `emitAsync` to avoid double publishing
- [ ] `emitAsync` is used (not `emit`)
- [ ] Handler class is `@Injectable()` and injects use cases by abstract class token
- [ ] Handler method is `async`, returns `Promise<void>`, decorated with `@OnEvent(CONSTANT)`
- [ ] Handler is registered in the side-effect module's `providers`
- [ ] No `EventEmitter2` import inside `src/domain/`
