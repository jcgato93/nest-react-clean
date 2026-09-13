# Exceptions

## Domain Exceptions Overview
Domain Exceptions are custom error classes that represent specific business rule violations or domain-related errors. They help in maintaining the integrity of the domain logic by providing meaningful error handling mechanisms.

## Guidelines for Creating Domain Exceptions

- **Custom Exception Classes**: Create custom exception classes that extend a base on one of the 
existing exception from ./domain/common/exceptions, which provides common functionality for all domain exceptions (e.g., `CustomBadRequestException`, `CustomNotFoundException`, `CustomUnauthorizedException`).
- **Meaningful Names**: Name your exception classes clearly to reflect the specific business rule violation they represent (e.g., `UserNotFoundException`, `InvalidOrderStateException`).
- **Detailed Messages**: Provide detailed error messages that explain the reason for the exception and any relevant context, written in the project's resolved exception language — check `exceptionLanguage` in `.claude/nest-clean.config.json` (see `nest-clean`'s Language Configuration step); default to English (`en`) if the config hasn't been resolved yet.
- **Use Cases**: Throw domain exceptions in use cases or domain services when business rules are violated.
- **Avoid Technical Details**: Domain exceptions should focus on business logic and avoid exposing technical implementation details.

### Example of a Domain Exception

```typescript
// src/domain/users/exceptions/user-not-found.exception.ts
import { CustomNotFoundException } from '../../common/exceptions/custom-not-found.exception';
export class UserNotFoundException extends CustomNotFoundException {
    constructor(userId: string) {
        super(`User with ID ${userId} not found.`);
    }
}
```