# Mappers - Capa de Aplicación

## Descripción General

Los **Mappers** son clases abstractas responsables de la transformación bidireccional entre DTOs (Data Transfer Objects) y entidades del dominio. Actúan como una capa de traducción que mantiene separadas las representaciones externas (API) de las estructuras internas del dominio.

## Principios Fundamentales

1. **Clase Abstracta con Métodos Estáticos**: Los mappers son clases abstractas que no se instancian, solo contienen métodos estáticos de transformación.
2. **Bidireccionalidad**: Transforman en ambas direcciones: DTO → Domain y Domain → DTO.
3. **Responsabilidad Única**: Cada mapper se enfoca en un único tipo de entidad del dominio.
4. **Sin Lógica de Negocio**: Solo realizan transformaciones estructurales, no aplican reglas de negocio.
5. **Value Objects**: Construyen correctamente los value objects del dominio durante la transformación.

## Estructura de Archivos

```plaintext
src/
  application/
    {module}/
      mappers/
        {entity}.mapper.ts          # Mapper para la entidad
```

## Patrón de Implementación

### Estructura Básica de un Mapper

```typescript
import { Email } from '@/domain/common/value-objects/email.value-object';
import { User } from '@/domain/user/entities/user.domain';
import { UserResponseDto } from '../dtos/user-response.dto';
import { CreateUserDto } from '../dtos/create-user.dto';

export abstract class UserMapper {
  /**
   * Transforma un DTO de entrada a una entidad del dominio
   * @param dto - DTO con datos de entrada
   * @returns Entidad del dominio
   */
  static toDomain(dto: CreateUserDto): User {
    return new User({
      email: new Email(dto.email),
      name: dto.name,
    });
  }

  /**
   * Transforma una entidad del dominio a un DTO de respuesta
   * @param entity - Entidad del dominio
   * @returns DTO para la respuesta
   */
  static toDto(entity: User): UserResponseDto {
    return {
      id: entity.id,
      name: entity.name,
      email: entity.email.value, // Extraer el valor del value object
      auth0Id: entity.authProviderId ?? '',
    };
  }

  /**
   * Transforma un array de entidades a un array de DTOs
   * @param entities - Array de entidades del dominio
   * @returns Array de DTOs para la respuesta
   */
  static toDtoList(entities: User[]): UserResponseDto[] {
    return entities.map(entity => UserMapper.toDto(entity));
  }
}
```

## Métodos Comunes en Mappers

### 1. `toDomain(dto)` - DTO a Entidad del Dominio

Convierte un DTO de entrada a una entidad del dominio.

**Responsabilidades**:
- Construir value objects a partir de primitivos
- Mapear propiedades simples
- Transformar fechas de string a Date
- Manejar campos opcionales

```typescript
static toDomain(dto: CreateCompanyDto, id?: string): Company {
  return new Company({
    id,
    document: dto.document,
    verificationDigit: dto.verificationDigit,
    firstName: dto.firstName,
    secondName: dto.secondName,
    lastName: dto.lastName,
    secondLastName: dto.secondLastName,
    countryCode: dto.countryCode,
    societyTypeId: dto.societyTypeId,
    identityTypeId: dto.identityTypeId,
    accountingGroup: dto.accountingGroup,
    balanceDate: new Date(dto.balanceDate), // String → Date
    reportsFiscalInformation: dto.reportsFiscalInformation,
    departmentCode: dto.departmentCode,
    cityCode: dto.cityCode,
    zoneId: dto.zoneId,
    address: dto.address,
    phones: dto.phones,
    mail: dto.mail,
    website: dto.website,
    primaryEconomicActivityId: dto.primaryEconomicActivityId,
    secondaryEconomicActivityId: dto.secondaryEconomicActivityId,
    otherEconomicActivitiesIds: dto.otherEconomicActivitiesIds ?? [], // Manejar opcionales
    taxResponsibilitiesIds: dto.taxResponsibilitiesIds ?? [],
    customsObligationsIds: dto.customsObligationsIds ?? [],
    licenseId: dto.licenseId,
  });
}
```

### 2. `toDto(entity)` - Entidad del Dominio a DTO

Convierte una entidad del dominio a un DTO de respuesta.

**Responsabilidades**:
- Extraer valores de value objects
- Mapear propiedades del dominio a la estructura de respuesta
- Formatear datos para presentación
- Manejar campos nulos/opcionales

```typescript
static toDto(entity: Company): CompanyResponseDto {
  return {
    id: entity.id,
    countryCode: entity.countryCode,
    societyTypeId: entity.societyTypeId,
    identityTypeId: entity.identityTypeId,
    firstName: entity.firstName,
    secondName: entity.secondName,
    lastName: entity.lastName,
    secondLastName: entity.secondLastName,
    document: entity.document,
    verificationDigit: entity.verificationDigit,
    accountingGroup: entity.accountingGroup,
    balanceDate: entity.balanceDate,
    reportsFiscalInformation: entity.reportsFiscalInformation,
    departmentCode: entity.departmentCode,
    cityCode: entity.cityCode,
    zoneId: entity.zoneId,
    address: entity.address,
    phones: entity.phones,
    mail: entity.mail,
    website: entity.website,
    primaryEconomicActivityId: entity.primaryEconomicActivityId,
    secondaryEconomicActivityId: entity.secondaryEconomicActivityId,
    otherEconomicActivitiesIds: entity.otherEconomicActivitiesIds,
    taxResponsibilitiesIds: entity.taxResponsibilitiesIds,
    customsObligationsIds: entity.customsObligationsIds,
    licenseId: entity.licenseId,
    logoUrl: entity.logoUrl,
    active: entity.active,
  };
}
```

### 3. `toDtoList(entities)` - Array de Entidades a Array de DTOs

Convierte múltiples entidades a múltiples DTOs.

```typescript
static toDtoList(entities: Company[]): CompanyResponseDto[] {
  return entities.map(entity => CompanyMapper.toDto(entity));
}
```

### 4. Mappers Especializados para Diferentes Vistas

Múltiples métodos `toDto` para diferentes representaciones:

```typescript
export abstract class CompanyMapper {
  // Vista simplificada para listados
  static toListDto(domain: Company): CompanyListResponseDto {
    return {
      id: domain.id,
      name: domain.firstName,
      identity: domain.document,
      verificationDigit: domain.verificationDigit,
      logoUrl: domain.logoUrl,
      active: domain.active,
    };
  }

  // Vista completa para detalles
  static toDetailDto(domain: Company): CompanyDetailResponseDto {
    return {
      id: domain.id,
      firstName: domain.firstName,
      document: domain.document,
      // ... todas las propiedades
      taxResponsibilities: domain.taxResponsibilities?.map(tax => ({
        id: tax.id,
        code: tax.code,
        name: tax.name,
      })),
      customObligations: domain.customObligations?.map(custom => ({
        id: custom.id,
        code: custom.code,
        description: custom.description,
      })),
    };
  }

  // Conversión de listas con vista específica
  static toListDtoArray(domains: Company[]): CompanyListResponseDto[] {
    return domains.map(item => CompanyMapper.toListDto(item));
  }
}
```

## Ejemplo Completo: Company Mapper

```typescript
import { CompanyListResponseDto } from '@/application/company/dto/company-list-response-dto';
import { CompanyResponseDto } from '@/application/company/dto/company-response-dto';
import { GetCompanyDetailResponseDto } from '@/application/company/dto/get-company-detail-response-dto';
import { CustomObligation } from '@/domain/catalog/entities/custom-obligation.domain';
import { EconomicActivity } from '@/domain/catalog/entities/economic-activity.domain';
import { TaxResponsibility } from '@/domain/catalog/entities/tax-responsibility.domain';
import { Company } from '@/domain/company/entities/company.domain';
import { CreateCompanyDto } from '../dto/create-company.dto';
import { UpdateCompanyDto } from '../dto/update-company.dto';

export abstract class CompanyMapper {
  /**
   * Convierte DTO de creación a entidad del dominio
   */
  static toDomain(dto: CreateCompanyDto, id?: string): Company {
    return new Company({
      id,
      document: dto.document,
      verificationDigit: dto.verificationDigit,
      firstName: dto.firstName,
      secondName: dto.secondName,
      lastName: dto.lastName,
      secondLastName: dto.secondLastName,
      countryCode: dto.countryCode,
      societyTypeId: dto.societyTypeId,
      identityTypeId: dto.identityTypeId,
      accountingGroup: dto.accountingGroup,
      balanceDate: new Date(dto.balanceDate),
      reportsFiscalInformation: dto.reportsFiscalInformation,
      departmentCode: dto.departmentCode,
      cityCode: dto.cityCode,
      zoneId: dto.zoneId,
      address: dto.address,
      phones: dto.phones,
      mail: dto.mail,
      website: dto.website,
      primaryEconomicActivityId: dto.primaryEconomicActivityId,
      secondaryEconomicActivityId: dto.secondaryEconomicActivityId,
      otherEconomicActivitiesIds: dto.otherEconomicActivitiesIds ?? [],
      taxResponsibilitiesIds: dto.taxResponsibilitiesIds ?? [],
      customsObligationsIds: dto.customsObligationsIds ?? [],
      licenseId: dto.licenseId,
    });
  }

  /**
   * Convierte DTO de actualización a entidad del dominio
   */
  static updateToDomain(dto: UpdateCompanyDto, id: string): Company {
    return new Company({
      id,
      document: dto.document,
      verificationDigit: dto.verificationDigit,
      firstName: dto.firstName,
      secondName: dto.secondName,
      lastName: dto.lastName,
      secondLastName: dto.secondLastName,
      countryCode: dto.countryCode,
      societyTypeId: dto.societyTypeId,
      identityTypeId: dto.identityTypeId,
      accountingGroup: dto.accountingGroup,
      balanceDate: new Date(dto.balanceDate),
      reportsFiscalInformation: dto.reportsFiscalInformation,
      departmentCode: dto.departmentCode,
      cityCode: dto.cityCode,
      zoneId: dto.zoneId,
      address: dto.address,
      phones: dto.phones,
      mail: dto.mail,
      website: dto.website,
      primaryEconomicActivityId: dto.primaryEconomicActivityId,
      secondaryEconomicActivityId: dto.secondaryEconomicActivityId,
      otherEconomicActivitiesIds: dto.otherEconomicActivitiesIds ?? [],
      taxResponsibilitiesIds: dto.taxResponsibilitiesIds ?? [],
      customsObligationsIds: dto.customsObligationsIds ?? [],
      licenseId: dto.licenseId,
    });
  }

  /**
   * Convierte entidad del dominio a DTO de respuesta completo
   */
  static toDto(domain: Company): CompanyResponseDto {
    return {
      id: domain.id,
      countryCode: domain.countryCode,
      societyTypeId: domain.societyTypeId,
      identityTypeId: domain.identityTypeId,
      firstName: domain.firstName,
      secondName: domain.secondName,
      lastName: domain.lastName,
      secondLastName: domain.secondLastName,
      document: domain.document,
      verificationDigit: domain.verificationDigit,
      accountingGroup: domain.accountingGroup,
      balanceDate: domain.balanceDate,
      reportsFiscalInformation: domain.reportsFiscalInformation,
      departmentCode: domain.departmentCode,
      cityCode: domain.cityCode,
      zoneId: domain.zoneId,
      address: domain.address,
      phones: domain.phones,
      mail: domain.mail,
      website: domain.website,
      primaryEconomicActivityId: domain.primaryEconomicActivityId,
      secondaryEconomicActivityId: domain.secondaryEconomicActivityId,
      otherEconomicActivitiesIds: domain.otherEconomicActivitiesIds,
      taxResponsibilitiesIds: domain.taxResponsibilitiesIds,
      customsObligationsIds: domain.customsObligationsIds,
      licenseId: domain.licenseId,
      logoUrl: domain.logoUrl,
      active: domain.active,
    };
  }

  /**
   * Convierte a vista simplificada para listados
   */
  static toListDto(domain: Company): CompanyListResponseDto {
    return {
      id: domain.id,
      name: domain.firstName,
      identity: domain.document,
      verificationDigit: domain.verificationDigit,
      logoUrl: domain.logoUrl,
      active: domain.active,
    };
  }

  /**
   * Convierte a vista detallada con relaciones
   */
  static toDetailDto(
    domain: Company,
    taxResponsibilities?: TaxResponsibility[],
    customObligations?: CustomObligation[],
    otherEconomicActivities?: EconomicActivity[],
  ): GetCompanyDetailResponseDto {
    return {
      id: domain.id,
      firstName: domain.firstName,
      secondName: domain.secondName,
      lastName: domain.lastName,
      secondLastName: domain.secondLastName,
      document: domain.document,
      verificationDigit: domain.verificationDigit,
      // ... todas las propiedades base
      taxResponsibilities: taxResponsibilities?.map(tax => ({
        id: tax.id,
        code: tax.code,
        name: tax.name,
        hasCustomObligation: tax.hasCustomObligation,
      })) ?? [],
      customObligations: customObligations?.map(custom => ({
        id: custom.id,
        code: custom.code,
        description: custom.description,
      })) ?? [],
      otherEconomicActivities: otherEconomicActivities?.map(activity => ({
        id: activity.id,
        code: activity.code,
        description: activity.description,
      })) ?? [],
    };
  }

  /**
   * Convierte array de entidades a array de DTOs simplificados
   */
  static toListDtoArray(domains: Company[]): CompanyListResponseDto[] {
    return domains.map(item => CompanyMapper.toListDto(item));
  }
}
```

## Manejo de Value Objects

Los mappers son cruciales para construir y deconstruir value objects:

### Construir Value Objects (DTO → Domain)

```typescript
import { Email } from '@/domain/common/value-objects/email.value-object';
import { Price } from '@/domain/product/value-objects/price.value-object';

static toDomain(dto: CreateProductDto): Product {
  return new Product({
    name: dto.name,
    price: new Price(dto.price, dto.currency), // Construir value object
    contactEmail: new Email(dto.email),        // Construir value object
    stock: dto.stock,
  });
}
```

### Extraer Valores de Value Objects (Domain → DTO)

```typescript
static toDto(entity: Product): ProductResponseDto {
  return {
    id: entity.id,
    name: entity.name,
    price: entity.price.value,              // Extraer valor del value object
    currency: entity.price.currency,        // Extraer propiedad del value object
    formattedPrice: entity.price.formatted, // Usar método del value object
    email: entity.contactEmail.value,       // Extraer valor del value object
    stock: entity.stock,
  };
}
```

## Transformación de Fechas

### String ISO → Date (DTO → Domain)

```typescript
static toDomain(dto: CreateCompanyDto): Company {
  return new Company({
    // ...
    balanceDate: new Date(dto.balanceDate), // String → Date
    createdAt: dto.createdAt ? new Date(dto.createdAt) : new Date(),
  });
}
```

### Date → String ISO (Domain → DTO)

```typescript
static toDto(entity: Company): CompanyResponseDto {
  return {
    // ...
    balanceDate: entity.balanceDate, // Date se serializa automáticamente a ISO
    createdAt: entity.createdAt,
  };
}
```

## Mejores Prácticas

### ✅ DO (Hacer)

1. **Usar clase abstracta con métodos estáticos**
   ```typescript
   export abstract class UserMapper {
     static toDomain(dto: CreateUserDto): User { /* ... */ }
     static toDto(entity: User): UserResponseDto { /* ... */ }
   }
   ```

2. **Crear métodos específicos para diferentes vistas**
   ```typescript
   static toListDto(entity: Company): CompanyListResponseDto { /* ... */ }
   static toDetailDto(entity: Company): CompanyDetailResponseDto { /* ... */ }
   static toDto(entity: Company): CompanyResponseDto { /* ... */ }
   ```

3. **Manejar arrays con métodos dedicados**
   ```typescript
   static toDtoList(entities: User[]): UserResponseDto[] {
     return entities.map(entity => UserMapper.toDto(entity));
   }
   ```

4. **Construir correctamente value objects**
   ```typescript
   static toDomain(dto: CreateUserDto): User {
     return new User({
       email: new Email(dto.email), // Construir value object
       name: dto.name,
     });
   }
   ```

5. **Extraer valores de value objects en DTOs**
   ```typescript
   static toDto(entity: User): UserResponseDto {
     return {
       email: entity.email.value, // Extraer el valor
       name: entity.name,
     };
   }
   ```

6. **Manejar campos opcionales con operador ??**
   ```typescript
   static toDomain(dto: CreateCompanyDto): Company {
     return new Company({
       secondName: dto.secondName,
       otherActivities: dto.otherActivities ?? [],
       taxResponsibilities: dto.taxResponsibilities ?? [],
     });
   }
   ```

7. **Transformar relaciones anidadas**
   ```typescript
   static toDetailDto(company: Company): CompanyDetailDto {
     return {
       id: company.id,
       name: company.name,
       taxResponsibilities: company.taxResponsibilities?.map(tax => ({
         id: tax.id,
         name: tax.name,
       })) ?? [],
     };
   }
   ```

### ❌ DON'T (No Hacer)

1. **No incluir lógica de negocio**
   ```typescript
   // ❌ Mal - La validación de negocio debe estar en el dominio
   static toDomain(dto: CreateProductDto): Product {
     if (dto.price < 0) {
       throw new Error('Price cannot be negative');
     }
     return new Product(dto);
   }
   
   // ✅ Bien - Solo transformación
   static toDomain(dto: CreateProductDto): Product {
     return new Product({
       price: new Price(dto.price), // El value object Price valida
     });
   }
   ```

2. **No hacer cálculos o transformaciones de negocio**
   ```typescript
   // ❌ Mal
   static toDto(entity: Product): ProductResponseDto {
     return {
       price: entity.price.value,
       priceWithTax: entity.price.value * 1.19, // Cálculo de negocio
     };
   }
   
   // ✅ Bien
   static toDto(entity: Product): ProductResponseDto {
     return {
       price: entity.price.value,
       priceWithTax: entity.calculatePriceWithTax(), // Método del dominio
     };
   }
   ```

3. **No acceder directamente a propiedades privadas**
   ```typescript
   // ❌ Mal
   static toDto(entity: User): UserResponseDto {
     return {
       id: entity._id, // Acceso a propiedad privada
     };
   }
   
   // ✅ Bien
   static toDto(entity: User): UserResponseDto {
     return {
       id: entity.id, // Usar getter público
     };
   }
   ```

4. **No instanciar mappers**
   ```typescript
   // ❌ Mal
   const mapper = new UserMapper();
   const dto = mapper.toDto(user);
   
   // ✅ Bien
   const dto = UserMapper.toDto(user);
   ```

5. **No mapear entidades de base de datos directamente**
   ```typescript
   // ❌ Mal
   static toDto(entity: UserEntity): UserResponseDto { /* ... */ }
   
   // ✅ Bien
   static toDto(entity: User): UserResponseDto { /* ... */ }
   ```


## Resumen

- **Mappers transforman** entre DTOs y entidades del dominio
- **Son clases abstractas** con métodos estáticos
- **No contienen lógica de negocio**, solo transformaciones estructurales
- **Construyen value objects** correctamente desde primitivos
- **Extraen valores** de value objects para DTOs
- **Proveen múltiples vistas** con métodos especializados (list, detail, etc.)
- **Se usan en controladores** para separar la API del dominio
- **Facilitan el testing** al aislar las transformaciones
