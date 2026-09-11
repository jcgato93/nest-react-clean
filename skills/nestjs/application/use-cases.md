# Use Cases - Implementación en la Capa de Aplicación

## Descripción General

Los **Use Cases (Casos de Uso)** son implementaciones concretas que encapsulan la lógica de orquestación de las operaciones de negocio. Coordinan el flujo entre repositorios, servicios externos y entidades del dominio, sin contener lógica de negocio directamente (que debe estar en las entidades del dominio).

## Principios Fundamentales

1. **Responsabilidad Única**: Cada use case debe tener un único propósito bien definido.
2. **Implementa Interfaces del Dominio**: Todos los use cases implementan interfaces definidas en `src/domain/*/use-cases/`.
3. **Nomenclatura con Sufijo `Impl`**: Las implementaciones siempre terminan en `Impl` (ej: `GetUserUseCaseImpl`).
4. **Inyección de Dependencias**: Utiliza el constructor para inyectar repositorios y servicios necesarios.
5. **Método `execute`**: Toda la lógica del caso de uso se encapsula en el método `execute`.

## Estructura de Archivos

```plaintext
src/
  domain/
    {module}/
      use-cases/
        {action}.use-case.ts              # Interfaz (contrato)
  application/
    {module}/
      use-cases/
        {action}.use-case.impl.ts         # Implementación
```

## Patrón de Implementación

### 1. Definición de la Interfaz (Dominio)

**Ubicación**: `src/domain/{module}/use-cases/{action}.use-case.ts`

```typescript
import { User } from '../entities/user.domain';

export interface GetUserUseCase {
  /**
   * 1. Buscar el usuario por auth0Id
   * 2. Si no existe el usuario, retornar error
   * 3. Retornar la información del usuario
   * @param authId - ID del proveedor de autenticación (Auth0)
   */
  execute(authId: string): Promise<User>;
}
```

**Características**:
- Define el contrato del caso de uso
- Usa JSDoc para documentar el flujo de negocio
- Solo declara el método `execute` con sus parámetros y tipo de retorno

### 2. Implementación del Use Case (Aplicación)

**Ubicación**: `src/application/{module}/use-cases/{action}.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { User } from '@/domain/user/entities/user.domain';
import { UserNotFoundException } from '@/domain/user/exceptions/user-not-found.exception';
import { UserRepository } from '@/domain/user/repositories/user.repository';
import { GetUserUseCase } from '@/domain/user/use-cases/get-user.use-case';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class GetUserUseCaseImpl implements GetUserUseCase {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly cacheService: CacheService,
  ) {}

  /**
   * 1. Buscar el usuario por auth0Id
   * 2. Si no existe el usuario, retornar error
   * 3. Retornar la información del usuario
   * @param authId
   */
  async execute(authId: string): Promise<User> {
    // Construir clave de cache
    const cacheKey = this.cacheService.buildKey(
      RedisKeyEnum.USER_BY_ID,
      authId,
    );

    // Intentar obtener del cache
    const cachedData = await this.cacheService.get(cacheKey);
    if (cachedData) {
      return cachedData as User;
    }

    // Si no está en cache, consultar repositorio
    const user = await this.userRepository.findByAuthProviderId(authId);
    if (!user) {
      throw new UserNotFoundException();
    }

    // Guardar en cache
    await this.cacheService.set(cacheKey, user);

    return user;
  }
}
```

**Características**:
- Decorador `@Injectable()` para habilitar inyección de dependencias
- Implementa la interfaz del dominio
- Constructor con dependencias claramente definidas
- Método `execute` con toda la lógica de orquestación
- Manejo de cache cuando aplique
- Lanza excepciones del dominio cuando corresponda

## Ejemplo Completo: Create Company Use Case

### Interfaz (Dominio)

```typescript
// src/domain/company/use-cases/create-company.use-case.ts
import { Company } from '@/domain/company/entities/company.domain';

export interface CreateCompanyUseCase {
  /**
   * 1. Validar que el usuario exista
   * 2. Validar que la licencia exista
   * 3. Validar obligaciones aduaneras si se requieren
   * 4. Guardar empresa con sus relaciones
   * 5. Invalidar caches relacionados
   */
  execute(authId: string, company: Company): Promise<string>;
}
```

### Implementación (Aplicación)

```typescript
// src/application/company/use-cases/create-company.use-case.impl.ts
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { TaxResponsibilityRepository } from '@/domain/catalog/repositories/tax-responsibility.repository';
import { Company } from '@/domain/company/entities/company.domain';
import { CustomObligationRequiredException } from '@/domain/company/exceptions/custom-obligation-required.exception';
import { LicenseNotFoundException } from '@/domain/company/exceptions/license-not-found.exception';
import { CompanyRepository } from '@/domain/company/repositories/company.repository';
import { CreateCompanyUseCase } from '@/domain/company/use-cases/create-company.use-case';
import { SubscriptionRepository } from '@/domain/subscription/repositories/subscription.repository';
import { UserNotFoundException } from '@/domain/user/exceptions/user-not-found.exception';
import { UserRepository } from '@/domain/user/repositories/user.repository';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

@Injectable()
export class CreateCompanyUseCaseImpl implements CreateCompanyUseCase {
  constructor(
    private companyRepository: CompanyRepository,
    private userRepository: UserRepository,
    private subscriptionRepository: SubscriptionRepository,
    private taxResponsibilityRepository: TaxResponsibilityRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(authId: string, company: Company): Promise<string> {
    // 1. Validar que el usuario exista
    const user = await this.userRepository.findByAuthProviderId(authId);
    if (!user) {
      throw new UserNotFoundException();
    }

    // 2. Validar que la licencia exista
    const license = await this.subscriptionRepository.findById(
      company.licenseId,
    );
    if (!license) {
      throw new LicenseNotFoundException();
    }

    // 3. Validar obligación aduanera si se seleccionaron responsabilidades tributarias
    if (company.taxResponsibilitiesIds?.length > 0) {
      const taxResponsibilities =
        await this.taxResponsibilityRepository.findByIds(
          company.taxResponsibilitiesIds,
        );

      const requiresCustoms = taxResponsibilities.some(
        (tax) => tax.hasCustomObligation === true,
      );

      if (requiresCustoms && !company.customsObligationsIds?.length) {
        throw new CustomObligationRequiredException();
      }
    }

    // 4. Guardar empresa con sus relaciones
    const companyId =
      await this.companyRepository.createCompanyWithRelations(company);

    // 5. Invalidar caches relacionadas con compañías
    await this.cacheService.removeCacheByPartialKey(RedisKeyEnum.COMPANY);

    return companyId;
  }
}
```

## Mejores Prácticas

### ✅ DO (Hacer)

1. **Implementar siempre la interfaz del dominio**
   ```typescript
   export class GetUserUseCaseImpl implements GetUserUseCase {
     // ...
   }
   ```

2. **Usar readonly para dependencias inmutables**
   ```typescript
   constructor(
     private readonly userRepository: UserRepository,
     private readonly cacheService: CacheService,
   ) {}
   ```

3. **Documentar el flujo de negocio con JSDoc**
   ```typescript
   /**
    * 1. Validar datos
    * 2. Consultar repositorio
    * 3. Aplicar lógica de negocio
    * 4. Retornar resultado
    */
   async execute(id: string): Promise<Entity> {
     // ...
   }
   ```

4. **Lanzar excepciones del dominio**
   ```typescript
   if (!user) {
     throw new UserNotFoundException();
   }
   ```

5. **Gestionar cache cuando sea apropiado**
   ```typescript
   const cacheKey = this.cacheService.buildKey(RedisKeyEnum.USER_BY_ID, id);
   const cachedData = await this.cacheService.get(cacheKey);
   if (cachedData) return cachedData;
   
   const data = await this.repository.find(id);
   await this.cacheService.set(cacheKey, data);
   return data;
   ```

6. **Usar `plainToInstance` para reconstruir entidades desde caché**
   
   Cuando se trabaja con caché, los datos almacenados son objetos planos (JSON) que pierden los métodos de las instancias de las entidades del dominio. Para reconstruir correctamente las instancias, todas las entidades deben implementar los métodos estáticos `plainToInstance` y `plainToInstanceList`.
   
   ```typescript
   // En casos de uso que retornan una sola entidad
   const cacheKey = this.cacheService.buildKey(RedisKeyEnum.USER, id);
   const cachedData = await this.cacheService.get(cacheKey);
   if (cachedData) {
     return User.plainToInstance(cachedData);
   }
   
   const user = await this.userRepository.findById(id);
   await this.cacheService.set(cacheKey, user);
   return user;
   ```
   
   ```typescript
   // En casos de uso que retornan listas de entidades
   const cacheKey = this.cacheService.buildKey(
     RedisKeyEnum.COMPANY,
     licenseId,
     `${pageOptions.page}`,
     `${pageOptions.take}`,
   );
   
   const cachedData = await this.cacheService.get(cacheKey);
   if (cachedData) {
     const paginationData = cachedData as PaginationResponse<any>;
     return {
       items: Company.plainToInstanceList(paginationData.items),
       count: paginationData.count,
     };
   }
   
   const result = await this.companyRepository.getList(licenseId, pageOptions);
   await this.cacheService.set(cacheKey, result);
   return result;
   ```
   
   **Razón**: Los datos en caché son objetos planos sin los métodos de la clase. Al usar `plainToInstance`, se reconstruye la instancia correcta con todos sus métodos, permitiendo llamar métodos del dominio como `user.isActive()`, `company.validateRules()`, etc.

7. **Invalidar cache después de mutaciones**
   ```typescript
   await this.repository.update(id, data);
   await this.cacheService.removeCacheByPartialKey(RedisKeyEnum.COMPANY);
   ```


8. **Para las queries retornar el DTO**
- Para simplificar la arquitectura, los casos de uso de tipo consulta (queries) puede retornar directamente el DTO correspondiente en lugar de la entidad del dominio.
también lo debe hacer el repositorio, Esto con el fin de evitar mapeos innecesarios en el controlador ya que las consultas no modifican el estado del dominio.

    ```typescript
    async execute(id: string): Promise<UserResponseDto> {
        return this.userRepository.findDtoById(id);
    }
    ```

### ❌ DON'T (No Hacer)

1. **No incluir lógica de negocio compleja**
   ```typescript
   // ❌ Mal - La lógica de negocio debe estar en la entidad del dominio
   async execute(data: CreateProductDto): Promise<Product> {
     const price = data.price * 1.19; // Cálculo de impuesto
     const product = new Product({ ...data, price });
     return this.repository.save(product);
   }
   
   // ✅ Bien - La lógica está en la entidad
   async execute(data: CreateProductDto): Promise<Product> {
     const product = Product.create(data); // La entidad calcula el precio con impuesto
     return this.repository.save(product);
   }
   ```

2. **No acceder directamente a modelos de Prisma**
   ```typescript
   // ❌ Mal - Usar el tipo generado por Prisma
   async execute(id: string): Promise<PrismaUser> {
     return this.prisma.user.findUnique({ where: { id } });
   }
   
   // ✅ Bien - Usar repositorio del dominio que retorna entidades del dominio
   async execute(id: string): Promise<User> {
     return this.userRepository.findById(id);
   }
   ```

3. **No retornar DTOs directamente para modificaciones del dominio, solo en consultas ni en consultas simples como el getById**
   ```typescript
   // ❌ Mal - El use case retorna DTO
   async execute(id: string): Promise<UserResponseDto> {
     const user = await this.userRepository.findById(id);
     return UserMapper.toDto(user);
   }
   
   // ✅ Bien - El use case retorna entidad del dominio
   async execute(id: string): Promise<User> {
     return this.userRepository.findById(id);
   }
   ```

4. **No gestionar múltiples responsabilidades**
   ```typescript
   // ❌ Mal - Un use case que hace demasiado
   async execute(data: any): Promise<void> {
     await this.createUser(data);
     await this.sendEmail(data);
     await this.createCompany(data);
   }
   
   // ✅ Bien - Separar en use cases independientes
   // CreateUserUseCaseImpl
   // SendWelcomeEmailUseCaseImpl
   // CreateCompanyUseCaseImpl
   ```

## Registro en el Módulo

Los use cases deben registrarse como providers en el módulo correspondiente:

```typescript
// src/infrastructure/modules/user.module.ts
import { Module } from '@nestjs/common';
import { GetUserUseCaseImpl } from '@/application/user/use-cases/get-user.use-case.impl';

@Module({
  providers: [
    GetUserUseCaseImpl,
    // ... otros providers
  ],
  exports: [GetUserUseCaseImpl], // Si se necesita en otros módulos
})
export class UserModule {}
```

## Uso desde Controladores

Los controladores inyectan e invocan los use cases:

```typescript
@Controller('users')
export class UserController {
  constructor(
    private readonly getUserUseCase: GetUserUseCaseImpl,
  ) {}

  @Get(':id')
  async getUser(@Param('id') id: string): Promise<UserResponseDto> {
    const user = await this.getUserUseCase.execute(id);
    return UserMapper.toDto(user); // Mapeo a DTO en el controlador
  }
}
```

## Resumen

- **Los use cases coordinan** flujos de trabajo entre múltiples componentes
- **No contienen lógica de negocio** (esa está en las entidades del dominio)
- **Implementan interfaces** definidas en la capa de dominio
- **Retornan entidades del dominio, excepto en consultas complejas**, solo DTOs cuando es necesario
- **Manejan excepciones del dominio** para casos de error
- **Gestionan cache** cuando es necesario para optimizar rendimiento
- **Son inyectables** y se registran como providers en los módulos de NestJS
