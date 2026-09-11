# Event Handlers (Listeners)

## Folder & Naming

```
src/infrastructure/adapters/event-handlers/
  {module}.handlers.ts         ← one file per side-effect module
```

Class: `{Module}Listener` → e.g., `AssetsGroupListener`, `DepreciationListener`
Method: `handle{EventName}` → e.g., `handleCompanyCreatedEvent`, `handleAssetTransferredEvent`

---

## Full Example — Multiple Events in One Handler

```typescript
// src/infrastructure/adapters/event-handlers/depreciation.handlers.ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';

import { InitializeDepreciationUseCase } from '@/domain/depreciation/use-cases/initialize-depreciation.use-case';
import { RecalculateDepreciationUseCase } from '@/domain/depreciation/use-cases/recalculate-depreciation.use-case';
import {
  COMPANY_CREATED_EVENT,
  CompanyCreatedEvent,
} from '@/domain/company/events/company-created';
import {
  ASSET_TRANSFERRED_EVENT,
  AssetTransferredEvent,
} from '@/domain/asset/events/asset-transferred';

@Injectable()
export class DepreciationListener {
  constructor(
    private readonly initializeDepreciation: InitializeDepreciationUseCase,
    private readonly recalculateDepreciation: RecalculateDepreciationUseCase,
  ) {}

  @OnEvent(COMPANY_CREATED_EVENT)
  async handleCompanyCreatedEvent(payload: CompanyCreatedEvent): Promise<void> {
    await this.initializeDepreciation.execute(payload.companyId);
  }

  @OnEvent(ASSET_TRANSFERRED_EVENT)
  async handleAssetTransferredEvent(
    payload: AssetTransferredEvent,
  ): Promise<void> {
    await this.recalculateDepreciation.execute(
      payload.companyId,
      payload.assetId,
    );
  }
}
```

---

## Module Registration

```typescript
// src/infrastructure/modules/depreciation.module.ts
import { Module } from '@nestjs/common';

import { InitializeDepreciationUseCaseImpl } from '@/application/depreciation/use-cases/initialize-depreciation.use-case.impl';
import { RecalculateDepreciationUseCaseImpl } from '@/application/depreciation/use-cases/recalculate-depreciation.use-case.impl';
import { InitializeDepreciationUseCase } from '@/domain/depreciation/use-cases/initialize-depreciation.use-case';
import { RecalculateDepreciationUseCase } from '@/domain/depreciation/use-cases/recalculate-depreciation.use-case';
import { DepreciationListener } from '@/infrastructure/adapters/event-handlers/depreciation.handlers';

import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  providers: [
    { provide: InitializeDepreciationUseCase, useClass: InitializeDepreciationUseCaseImpl },
    { provide: RecalculateDepreciationUseCase, useClass: RecalculateDepreciationUseCaseImpl },
    DepreciationListener,
  ],
  exports: [InitializeDepreciationUseCase, RecalculateDepreciationUseCase],
})
export class DepreciationModule {}
```

---

## Common Mistakes

| Mistake | Why it fails | Fix |
|---------|-------------|-----|
| Listener not in `providers` | NestJS never instantiates it → `@OnEvent` is never registered | Add to `providers` array |
| Using `emit` instead of `emitAsync` | Handler's Promise is not awaited → errors are silently lost | Use `emitAsync` in the use case |
| Injecting `Impl` directly (e.g., `InitializeDepreciationUseCaseImpl`) | Creates a hidden coupling to implementation | Inject by abstract class token |
| Importing `EventEmitter2` in domain | Breaks the dependency rule (domain must be framework-agnostic) | Only use `EventEmitter2` in application/infrastructure layers |
| Publishing events before `save()` | If the DB transaction fails, the side effect already ran | Always publish after the persistence call succeeds |

---

## AppModule Requirement

`EventEmitterModule.forRoot()` must be in `AppModule` imports exactly once. This is already configured in this project — do not add it again.

The listener's module (e.g., `DepreciationModule`) must appear in `AppModule` imports so NestJS bootstraps the listener and registers its `@OnEvent` decorators.
