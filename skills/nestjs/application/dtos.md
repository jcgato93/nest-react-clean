# DTOs (Data Transfer Objects) - Capa de Aplicación

## Descripción General

Los **DTOs (Data Transfer Objects)** son objetos simples que transportan datos entre las diferentes capas de la aplicación. Su propósito principal es definir el contrato de entrada y salida de las APIs, separando la representación externa de las entidades del dominio.

## Principios Fundamentales

1. **Separación de Responsabilidades**: Los DTOs definen cómo se exponen los datos externamente, mientras que las entidades del dominio definen las reglas de negocio.
2. **Validación de Entrada**: Los DTOs de entrada utilizan decoradores de `class-validator` para validar datos.
3. **Documentación API**: Usar decoradores de `@nestjs/swagger` para generar documentación automática.
4. **Solo Datos**: Los DTOs no deben contener lógica de negocio, solo propiedades y validaciones.
5. **Inmutabilidad**: Los DTOs de respuesta usan `@Expose()` de `class-transformer` para controlar qué propiedades se serializan.

## Tipos de DTOs

### 1. DTOs de Entrada (Request DTOs)
Validan y transforman datos recibidos de las peticiones HTTP.

### 2. DTOs de Respuesta (Response DTOs)
Definen la estructura de datos que se envía al cliente.

## Estructura de Archivos

```plaintext
src/
  application/
    {module}/
      dtos/
        create-{entity}.dto.ts        # DTO para crear entidades
        update-{entity}.dto.ts        # DTO para actualizar entidades
        {entity}-response.dto.ts      # DTO para respuestas
        {entity}-list-response.dto.ts # DTO para listados
```

## DTOs de Entrada (Request)

### Ejemplo Básico: Login DTO

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsNotEmpty, IsString } from 'class-validator';

export class LoginDto {
  @ApiProperty({
    description: 'User email',
    example: 'user@example.com',
  })
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @ApiProperty({
    description: 'User password',
    example: 'StrongPassword123',
  })
  @IsString()
  @IsNotEmpty()
  password: string;
}
```

### Ejemplo Complejo: Create Company DTO

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { AccountingGroupEnum } from '@/common/constants/enums/accounting-group.enum';
import {
  IsArray,
  IsBoolean,
  IsDateString,
  IsEnum,
  IsNotEmpty,
  IsOptional,
  IsString,
} from 'class-validator';

export class CreateCompanyDto {
  @ApiProperty({
    description: 'Código del país',
    example: 'CO',
  })
  @IsString()
  @IsNotEmpty()
  countryCode: string;

  @ApiProperty({
    description: 'ID del tipo de sociedad',
    example: '123e4567-e89b-12d3-a456-426614174000',
  })
  @IsString()
  @IsNotEmpty()
  societyTypeId: string;

  @ApiProperty({
    description: 'ID del tipo de identificación',
    example: '123e4567-e89b-12d3-a456-426614174001',
  })
  @IsString()
  @IsNotEmpty()
  identityTypeId: string;

  @ApiProperty({
    description: 'Primer nombre o razón social',
    example: 'Empresa',
  })
  @IsString()
  @IsNotEmpty()
  firstName: string;

  @ApiPropertyOptional({
    description: 'Segundo nombre (opcional)',
    example: 'S.A.S',
  })
  @IsOptional()
  @IsString()
  secondName?: string;

  @ApiPropertyOptional({
    description: 'Primer apellido (opcional)',
  })
  @IsOptional()
  @IsString()
  lastName?: string;

  @ApiPropertyOptional({
    description: 'Segundo apellido (opcional)',
  })
  @IsOptional()
  @IsString()
  secondLastName?: string;

  @ApiProperty({
    description: 'Número de documento de identificación',
    example: '900123456',
  })
  @IsString()
  @IsNotEmpty()
  document: string;

  @ApiProperty({
    description: 'Dígito de verificación',
    example: '7',
  })
  @IsString()
  @IsNotEmpty()
  verificationDigit: string;

  @ApiProperty({
    enum: AccountingGroupEnum,
    description: 'Grupo contable de la empresa',
    example: AccountingGroupEnum.GROUP_1,
  })
  @IsEnum(AccountingGroupEnum)
  accountingGroup: AccountingGroupEnum;

  @ApiProperty({
    description: 'Fecha de corte de balance',
    example: '2024-12-31',
  })
  @IsDateString()
  balanceDate: string;

  @ApiProperty({
    description: 'Indica si reporta información fiscal',
    example: true,
  })
  @IsBoolean()
  reportsFiscalInformation: boolean;

  @ApiProperty({
    description: 'Código del departamento',
    example: '11',
  })
  @IsString()
  @IsNotEmpty()
  departmentCode: string;

  @ApiProperty({
    description: 'Código de la ciudad',
    example: '11001',
  })
  @IsString()
  @IsNotEmpty()
  cityCode: string;

  @ApiPropertyOptional({
    description: 'IDs de responsabilidades tributarias',
    type: [String],
    example: ['123e4567-e89b-12d3-a456-426614174002'],
  })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  taxResponsibilitiesIds?: string[];

  @ApiPropertyOptional({
    description: 'IDs de obligaciones aduaneras',
    type: [String],
    example: ['123e4567-e89b-12d3-a456-426614174003'],
  })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  customsObligationsIds?: string[];

  @ApiProperty({
    description: 'ID de la licencia/suscripción',
    example: '123e4567-e89b-12d3-a456-426614174004',
  })
  @IsString()
  @IsNotEmpty()
  licenseId: string;
}
```

### Validadores Comunes

| Decorador | Uso | Ejemplo |
|-----------|-----|---------|
| `@IsString()` | Valida que sea string | `@IsString() name: string;` |
| `@IsNumber()` | Valida que sea número | `@IsNumber() age: number;` |
| `@IsEmail()` | Valida formato de email | `@IsEmail() email: string;` |
| `@IsBoolean()` | Valida que sea booleano | `@IsBoolean() active: boolean;` |
| `@IsEnum()` | Valida que sea valor del enum | `@IsEnum(Status) status: Status;` |
| `@IsArray()` | Valida que sea array | `@IsArray() tags: string[];` |
| `@IsDateString()` | Valida formato de fecha ISO | `@IsDateString() date: string;` |
| `@IsNotEmpty()` | Valida que no esté vacío | `@IsNotEmpty() name: string;` |
| `@IsOptional()` | Marca como opcional | `@IsOptional() @IsString() middleName?: string;` |
| `@MinLength(n)` | Longitud mínima | `@MinLength(3) name: string;` |
| `@MaxLength(n)` | Longitud máxima | `@MaxLength(100) name: string;` |
| `@Min(n)` | Valor numérico mínimo | `@Min(0) price: number;` |
| `@Max(n)` | Valor numérico máximo | `@Max(100) discount: number;` |

## DTOs de Respuesta (Response)

### Ejemplo Básico: User Response DTO

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

export class UserResponseDto {
  @ApiProperty({
    description: 'ID del usuario',
    example: '123e4567-e89b-12d3-a456-426614174000',
  })
  @Expose()
  id: string;

  @ApiProperty({
    description: 'Nombre completo del usuario',
    example: 'Juan Pérez',
  })
  @Expose()
  name: string;

  @ApiProperty({
    description: 'Correo electrónico',
    example: 'juan.perez@example.com',
  })
  @Expose()
  email: string;

  @ApiProperty({
    description: 'ID de Auth0',
    example: 'auth0|123456789',
  })
  @Expose()
  auth0Id: string;
}
```

### Ejemplo con Propiedades Anidadas

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose, Type } from 'class-transformer';

export class CompanyDetailResponseDto {
  @ApiProperty()
  @Expose()
  id: string;

  @ApiProperty()
  @Expose()
  firstName: string;

  @ApiProperty()
  @Expose()
  document: string;

  @ApiProperty()
  @Expose()
  verificationDigit: string;

  @ApiProperty()
  @Expose()
  active: boolean;

  @ApiProperty({ type: [TaxResponsibilityByCompanyResponseDto] })
  @Expose()
  @Type(() => TaxResponsibilityByCompanyResponseDto)
  taxResponsibilities: TaxResponsibilityByCompanyResponseDto[];

  @ApiProperty({ type: [CustomObligationByCompanyResponseDto] })
  @Expose()
  @Type(() => CustomObligationByCompanyResponseDto)
  customObligations: CustomObligationByCompanyResponseDto[];
}

export class TaxResponsibilityByCompanyResponseDto {
  @ApiProperty()
  @Expose()
  id: string;

  @ApiProperty()
  @Expose()
  code: string;

  @ApiProperty()
  @Expose()
  name: string;
}

export class CustomObligationByCompanyResponseDto {
  @ApiProperty()
  @Expose()
  id: string;

  @ApiProperty()
  @Expose()
  code: string;

  @ApiProperty()
  @Expose()
  description: string;
}
```

## Mejores Prácticas

### ✅ DO (Hacer)

1. **Usar decoradores de validación en DTOs de entrada**
   ```typescript
   export class CreateUserDto {
     @IsEmail()
     @IsNotEmpty()
     email: string;
     
     @IsString()
     @MinLength(8)
     password: string;
   }
   ```

2. **Documentar con Swagger en todas las propiedades**
   ```typescript
   @ApiProperty({
     description: 'Email del usuario',
     example: 'user@example.com',
   })
   @IsEmail()
   email: string;
   ```

3. **Usar @Expose() en DTOs de respuesta**
   ```typescript
   export class UserResponseDto {
     @Expose()
     id: string;
     
     @Expose()
     name: string;
     
     // La propiedad 'password' no se expondrá
   }
   ```

4. **Separar DTOs de entrada y salida**
   ```typescript
   // ✅ Correcto
   export class CreateCompanyDto { /* ... */ }
   export class CompanyResponseDto { /* ... */ }
   
   // ❌ Incorrecto - No reutilizar el mismo DTO
   export class CompanyDto { /* ... */ }
   ```

5. **Usar @IsOptional() para campos opcionales**
   ```typescript
   @ApiPropertyOptional()
   @IsOptional()
   @IsString()
   middleName?: string;
   ```

6. **Validar arrays con { each: true }**
   ```typescript
   @IsArray()
   @IsString({ each: true })
   tags: string[];
   ```

7. **Usar enums para valores restringidos**
   ```typescript
   @ApiProperty({ enum: AccountingGroupEnum })
   @IsEnum(AccountingGroupEnum)
   accountingGroup: AccountingGroupEnum;
   ```

### ❌ DON'T (No Hacer)

1. **No incluir lógica de negocio en DTOs**
   ```typescript
   // ❌ Mal
   export class CreateProductDto {
     price: number;
     
     getTotalWithTax(): number {
       return this.price * 1.19;
     }
   }
   
   // ✅ Bien - Los DTOs solo tienen propiedades
   export class CreateProductDto {
     @IsNumber()
     price: number;
   }
   ```

2. **No exponer propiedades sensibles**
   ```typescript
   // ❌ Mal
   export class UserResponseDto {
     @Expose()
     password: string; // ¡Nunca exponer contraseñas!
   }
   
   // ✅ Bien
   export class UserResponseDto {
     @Expose()
     id: string;
     
     @Expose()
     email: string;
     // password no se incluye
   }
   ```

3. **No usar `any` como tipo**
   ```typescript
   // ❌ Mal
   @ApiProperty()
   data: any;
   
   // ✅ Bien
   @ApiProperty({ type: CompanyDetailDto })
   data: CompanyDetailDto;
   ```

4. **No omitir decoradores de Swagger**
   ```typescript
   // ❌ Mal
   export class UserDto {
     id: string;
     name: string;
   }
   
   // ✅ Bien
   export class UserDto {
     @ApiProperty()
     @Expose()
     id: string;
     
     @ApiProperty()
     @Expose()
     name: string;
   }
   ```

5. **No validar en el controlador, usar DTOs**
   ```typescript
   // ❌ Mal
   @Post()
   create(@Body() data: any) {
     if (!data.email) throw new Error('Email required');
     // ...
   }
   
   // ✅ Bien
   @Post()
   create(@Body() data: CreateUserDto) {
     // La validación ya se hizo automáticamente
   }
   ```

## Transformación con Class Transformer

### @Type para tipos complejos

```typescript
import { Type } from 'class-transformer';

export class CompanyResponseDto {
  @ApiProperty()
  @Expose()
  id: string;

  @ApiProperty()
  @Expose()
  @Type(() => Date)
  balanceDate: Date;

  @ApiProperty({ type: [String] })
  @Expose()
  @Type(() => String)
  phones: string[];
}
```

## Relación con Mappers

Los DTOs se transforman desde/hacia entidades del dominio usando **Mappers**:

```typescript
// DTO → Domain Entity (en el mapper)
static toDomain(dto: CreateCompanyDto): Company {
  return new Company({
    firstName: dto.firstName,
    document: dto.document,
    // ... más propiedades
  });
}

// Domain Entity → DTO (en el mapper)
static toDto(entity: Company): CompanyResponseDto {
  return {
    id: entity.id,
    firstName: entity.firstName,
    document: entity.document,
    // ... más propiedades
  };
}
```

## Resumen

- **DTOs definen contratos** de entrada y salida de las APIs
- **Validan datos** automáticamente usando decoradores de `class-validator`
- **Documentan APIs** usando decoradores de `@nestjs/swagger`
- **No contienen lógica de negocio**, solo propiedades y validaciones
- **DTOs de entrada** validan y sanitizan datos del cliente
- **DTOs de salida** controlan qué datos se exponen usando `@Expose()`
- **Se transforman** a/desde entidades del dominio usando Mappers
- **Separan la API** de la implementación interna del dominio
