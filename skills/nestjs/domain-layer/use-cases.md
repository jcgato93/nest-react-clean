# Use cases

**Every business operation** is a use case class with single responsibility:

- Use case interface defined in `src/domain/*/use-cases/*.use-case.ts`
- Implementation in `src/application/*/use-cases/*.use-case.impl.ts`
- Always named with `Impl` suffix (e.g., `LoginUseCaseImpl`)
- Controllers inject and call use cases directly, never repositories
- Leave or ask for comments in the use case interface to guide the implementer about the expected behavior.
- Should only contain method signatures without any implementation.
- Should only receive and return domain entities,value objects or plain objects except for something like dtos.

```typescript
import { Product } from '../entities/products/product.entity';

export interface GetProductUseCase {
  /**
   * All the comments here should guide the implementer about the expected behavior.
   */
  execute(id: string): Promise<Product>;
}
```