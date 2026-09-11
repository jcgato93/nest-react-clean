# Cambios para el módulo `bank-account`

## Dos objetivos

1. **Campo `swiftCode`**: string, opcional, máximo 11 caracteres.
2. **Endpoint `GET /bank-accounts/by-account-number/:accountNumber`**: busca una cuenta bancaria por número de cuenta, filtrada por `companyId` (del token JWT).

---

## Orden de cambios

Seguimos la arquitectura en capas del proyecto: Domain → Application → Infrastructure.

1. Entidad de dominio (`bank-account.domain.ts`)
2. DTO de creación (`create-bank-account.dto.ts`) — `UpdateBankAccountDto` hereda de él, así que se actualiza solo
3. DTO de respuesta de detalle (`bank-account-response.dto.ts`)
4. Mapper de aplicación (`bank-account.mapper.ts`)
5. Repositorio abstracto de dominio (`bank-account.repository.ts`) — nuevo método `findByAccountNumber`
6. Caso de uso de dominio — interfaz (`get-bank-account-by-account-number.use-case.ts`) [**archivo nuevo**]
7. Caso de uso de aplicación — implementación (`get-bank-account-by-account-number.use-case.impl.ts`) [**archivo nuevo**]
8. Repositorio de infraestructura (`bank-account.repository.impl.ts`) — implementar `findByAccountNumber` + propagar `swiftCode` en mappers
9. Entidad ORM (`bank-account.entity.ts`) — nueva columna `swift_code`
10. Migración de base de datos [**archivo nuevo**]
11. Controlador (`bank-account.controller.ts`) — nuevo endpoint
12. Módulo (`bank-account.module.ts`) — registrar el nuevo use-case

---

## Cambios detallados

### 1. Entidad de dominio

**`src/domain/bank-account/entities/bank-account.domain.ts`**

Añadir `swiftCode?: string` a la interfaz `BankAccountProps`, el campo privado `_swiftCode`, y un getter (sin setter porque es inmutable post-creación, al igual que `accountNumber`). La validación de la longitud máxima de 11 caracteres se delega al DTO de entrada.

```typescript
// En BankAccountProps (interfaz)
swiftCode?: string | null;

// Campo privado
private readonly _swiftCode?: string | null;

// En el constructor, después de las otras asignaciones
this._swiftCode = props.swiftCode ?? null;

// Getter
get swiftCode(): string | null | undefined {
  return this._swiftCode;
}
```

Diff completo del constructor y la interfaz:

```diff
 interface BankAccountProps {
   id?: string;
   companyId: string;
   countryCode: string;
   financialEntityId: string;
   branchName: string;
   branchAddress: string;
   cityCode: string;
   departmentCode: string;
   accountNumber: string;
   accountType: AccountTypeEnum;
   accountId: string;
   fiscalConciliationConceptId?: string | null;
+  swiftCode?: string | null;
 }

 export class BankAccount {
   private readonly _id: string;
   private readonly _companyId: string;
   private readonly _countryCode: string;
   private readonly _financialEntityId: string;
   private _branchName: string;
   private _branchAddress: string;
   private readonly _cityCode: string;
   private readonly _departmentCode: string;
   private readonly _accountNumber: string;
   private _accountType: AccountTypeEnum;
   private _accountId: string;
   private _fiscalConciliationConceptId?: string | null;
+  private readonly _swiftCode?: string | null;

   constructor(props: BankAccountProps) {
     this.validateAccountNumber(props.accountNumber);
     this.validateBranchName(props.branchName);
     this.validateBranchAddress(props.branchAddress);

     this._id = props.id || crypto.randomUUID();
     this._companyId = props.companyId;
     this._countryCode = props.countryCode;
     this._financialEntityId = props.financialEntityId;
     this._branchName = props.branchName.toLowerCase().trim();
     this._branchAddress = props.branchAddress.toLowerCase().trim();
     this._cityCode = props.cityCode;
     this._departmentCode = props.departmentCode;
     this._accountNumber = props.accountNumber.toLowerCase().trim();
     this._accountType = props.accountType;
     this._accountId = props.accountId;
     this._fiscalConciliationConceptId = props.fiscalConciliationConceptId;
+    this._swiftCode = props.swiftCode ?? null;
   }

   // ... getters existentes ...

+  get swiftCode(): string | null | undefined {
+    return this._swiftCode;
+  }
```

---

### 2. DTO de creación

**`src/application/bank-account/dto/create-bank-account.dto.ts`**

```diff
 import {
   IsEnum,
   IsNotEmpty,
   IsOptional,
   IsString,
   IsUUID,
   MaxLength,
   MinLength,
 } from 'class-validator';

 // ...campos existentes...

+  @IsString()
+  @IsOptional()
+  @MaxLength(11)
+  @ApiProperty({
+    description: 'Código SWIFT/BIC de la entidad financiera para transferencias internacionales',
+    example: 'COLBCOBB',
+    maxLength: 11,
+    required: false,
+  })
+  swiftCode?: string;
```

> `UpdateBankAccountDto` extiende `CreateBankAccountDto` sin añadir nada propio, por lo que hereda el campo automáticamente. No requiere cambios.

---

### 3. DTO de respuesta de detalle

**`src/application/bank-account/dto/bank-account-response.dto.ts`**

```diff
   @Expose()
   @ApiProperty({
     description: 'ID del concepto de conciliación fiscal',
     example: '123e4567-e89b-12d3-a456-426614174000',
     required: false,
     nullable: true,
   })
   fiscalConciliationConceptId?: string | null;

+  @Expose()
+  @ApiProperty({
+    description: 'Código SWIFT/BIC de la entidad financiera',
+    example: 'COLBCOBB',
+    required: false,
+    nullable: true,
+  })
+  swiftCode?: string | null;

   @Expose()
   @ApiProperty({ ... })
   @Type(() => AccountDetailDto)
   account?: AccountDetailDto | null;
```

---

### 4. Mapper de aplicación

**`src/application/bank-account/mapper/bank-account.mapper.ts`**

```diff
   static toDomain(dto: CreateBankAccountDto, bankAccountId?: string): BankAccount {
     return new BankAccount({
       id: bankAccountId,
       companyId: dto.companyId,
       countryCode: dto.countryCode,
       financialEntityId: dto.financialEntityId,
       branchName: dto.branchName,
       branchAddress: dto.branchAddress,
       cityCode: dto.cityCode,
       departmentCode: dto.departmentCode,
       accountNumber: dto.accountNumber,
       accountType: dto.accountType,
       accountId: dto.accountId,
       fiscalConciliationConceptId: dto.fiscalConciliationConceptId,
+      swiftCode: dto.swiftCode,
     });
   }
```

---

### 5. Repositorio abstracto de dominio

**`src/domain/bank-account/repositories/bank-account.repository.ts`**

Se añade el método abstracto `findByAccountNumber`. Devuelve el `BankAccountResponseDto` directamente (igual que `findOneByIdWithDetails`) para poder incluir las relaciones de cuenta contable y concepto fiscal.

```diff
   abstract findOneByIdWithDetails(
     id: string,
   ): Promise<BankAccountResponseDto | null>;
+
+  /**
+   * Busca una cuenta bancaria por número de cuenta dentro de una empresa
+   * @param accountNumber - Número de cuenta bancaria
+   * @param companyId - ID de la empresa
+   * @returns DTO de respuesta con detalles o null si no se encuentra
+   */
+  abstract findByAccountNumber(
+    accountNumber: string,
+    companyId: string,
+  ): Promise<BankAccountResponseDto | null>;
 }
```

---

### 6. Interfaz del caso de uso (dominio)

**`src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts`** *(archivo nuevo)*

```typescript
import { BankAccountResponseDto } from '@/application/bank-account/dto/bank-account-response.dto';

/**
 * Caso de uso para obtener una cuenta bancaria por número de cuenta
 */
export interface GetBankAccountByAccountNumberUseCase {
  /**
   * Obtiene una cuenta bancaria por su número de cuenta
   * @param accountNumber - Número de cuenta bancaria
   * @param companyId - ID de la empresa (extraído del token JWT)
   * @returns DTO de respuesta de la cuenta bancaria
   * @throws BankAccountNotFoundException si no se encuentra la cuenta
   */
  execute(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccountResponseDto>;
}
```

---

### 7. Implementación del caso de uso (aplicación)

**`src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts`** *(archivo nuevo)*

El patrón es idéntico al de `GetBankAccountByIdUseCaseImpl`: cache-first, luego repositorio, luego verificación de pertenencia a la empresa.

```typescript
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { BankAccountNotFoundException } from '@/domain/bank-account/exceptions/bank-account-not-found.exception';
import { BankAccountRepository } from '@/domain/bank-account/repositories/bank-account.repository';
import { GetBankAccountByAccountNumberUseCase } from '@/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';

import { BankAccountResponseDto } from '../dto/bank-account-response.dto';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class GetBankAccountByAccountNumberUseCaseImpl
  implements GetBankAccountByAccountNumberUseCase
{
  constructor(
    private readonly bankAccountRepository: BankAccountRepository,
    private readonly cacheService: CacheService,
  ) {}

  async execute(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccountResponseDto> {
    // Construir clave de cache por número de cuenta y compañía
    const cacheKey = this.cacheService.buildKey(
      RedisKeyEnum.BANK_ACCOUNT,
      companyId,
      'by-account-number',
      accountNumber,
    );

    // Intentar obtener del cache
    const cachedData =
      await this.cacheService.get<BankAccountResponseDto>(cacheKey);
    if (cachedData) {
      return plainToInstance(BankAccountResponseDto, cachedData, {
        excludeExtraneousValues: true,
      });
    }

    // Consultar repositorio (ya filtra por companyId internamente)
    const bankAccount =
      await this.bankAccountRepository.findByAccountNumber(
        accountNumber,
        companyId,
      );

    if (!bankAccount) {
      throw new BankAccountNotFoundException();
    }

    // Guardar en cache
    await this.cacheService.set(cacheKey, bankAccount);

    return bankAccount;
  }
}
```

---

### 8. Repositorio de infraestructura

**`src/infrastructure/database/repositories/bank-account.repository.impl.ts`**

Dos cambios:

**a) Propagar `swiftCode` en `mapToDomain` y `mapToEntity`:**

```diff
   mapToDomain(entity: BankAccountEntity): BankAccount {
     return new BankAccount({
       id: entity.id,
       companyId: entity.companyId,
       countryCode: entity.countryCode,
       financialEntityId: entity.financialEntityId,
       branchName: entity.branchName,
       branchAddress: entity.branchAddress,
       cityCode: entity.cityCode,
       departmentCode: entity.departmentCode,
       accountNumber: entity.accountNumber,
       accountType: entity.accountType,
       accountId: entity.accountId,
       fiscalConciliationConceptId: entity.fiscalConciliationConceptId,
+      swiftCode: entity.swiftCode,
     });
   }

   mapToEntity(domain: BankAccount): BankAccountEntity {
     const entity = new BankAccountEntity();
     entity.id = domain.id;
     entity.companyId = domain.companyId;
     entity.countryCode = domain.countryCode;
     entity.financialEntityId = domain.financialEntityId;
     entity.branchName = domain.branchName;
     entity.branchAddress = domain.branchAddress;
     entity.cityCode = domain.cityCode;
     entity.departmentCode = domain.departmentCode;
     entity.accountNumber = domain.accountNumber;
     entity.accountType = domain.accountType;
     entity.accountId = domain.accountId;
     entity.fiscalConciliationConceptId =
       domain.fiscalConciliationConceptId ?? null;
+    entity.swiftCode = domain.swiftCode ?? null;
     return entity;
   }
```

**b) Añadir `swiftCode` al `select` de `findOneByIdWithDetails` e implementar `findByAccountNumber`:**

```diff
   async findOneByIdWithDetails(
     id: string,
   ): Promise<BankAccountResponseDto | null> {
     const entity = await this.bankAccountEntityRepository.findOne({
       where: { id },
       select: {
         id: true,
         companyId: true,
         countryCode: true,
         financialEntityId: true,
         branchName: true,
         branchAddress: true,
         cityCode: true,
         departmentCode: true,
         accountNumber: true,
         accountType: true,
         accountId: true,
         fiscalConciliationConceptId: true,
+        swiftCode: true,
         account: { name: true },
         fiscalConciliationConcept: { name: true },
       },
       relations: { account: true, fiscalConciliationConcept: true },
     });

     if (!entity) return null;

     return plainToInstance(BankAccountResponseDto, entity, {
       excludeExtraneousValues: true,
     });
   }

+  async findByAccountNumber(
+    accountNumber: string,
+    companyId: string,
+  ): Promise<BankAccountResponseDto | null> {
+    const entity = await this.bankAccountEntityRepository.findOne({
+      where: { accountNumber, companyId },
+      select: {
+        id: true,
+        companyId: true,
+        countryCode: true,
+        financialEntityId: true,
+        branchName: true,
+        branchAddress: true,
+        cityCode: true,
+        departmentCode: true,
+        accountNumber: true,
+        accountType: true,
+        accountId: true,
+        fiscalConciliationConceptId: true,
+        swiftCode: true,
+        account: { name: true },
+        fiscalConciliationConcept: { name: true },
+      },
+      relations: { account: true, fiscalConciliationConcept: true },
+    });
+
+    if (!entity) return null;
+
+    return plainToInstance(BankAccountResponseDto, entity, {
+      excludeExtraneousValues: true,
+    });
+  }
```

> Nota: el campo `accountNumber` en base de datos se guarda en minúsculas (ver dominio: `props.accountNumber.toLowerCase().trim()`). La búsqueda con TypeORM es case-sensitive por defecto en PostgreSQL. Si el cliente puede enviar el número en mayúsculas, conviene normalizar el parámetro antes de la consulta. Lo más seguro es hacer `.toLowerCase().trim()` en el use-case o en el repositorio antes del `findOne`. Se recomienda hacerlo en el repositorio para mantener la consistencia:

```typescript
// Dentro de findByAccountNumber, antes del findOne:
const normalizedAccountNumber = accountNumber.toLowerCase().trim();
const entity = await this.bankAccountEntityRepository.findOne({
  where: { accountNumber: normalizedAccountNumber, companyId },
  // ...
});
```

---

### 9. Entidad ORM

**`src/infrastructure/database/entities/bank-account.entity.ts`**

```diff
   @Column({
     type: 'varchar',
     name: 'fiscal_conciliation_concept_id',
     nullable: true,
   })
   fiscalConciliationConceptId: string | null;

+  @Column({
+    type: 'varchar',
+    length: 11,
+    name: 'swift_code',
+    nullable: true,
+  })
+  swiftCode: string | null;

   // === Relations ===
```

---

### 10. Migración de base de datos

**`src/infrastructure/database/migrations/<timestamp>-AddSwiftCodeToBankAccount.ts`** *(archivo nuevo)*

El timestamp debe generarse al crear el archivo (por ejemplo `1773000000000`). Debe ser mayor al de la última migración existente (`1772663013023`).

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddSwiftCodeToBankAccount1773000000000
  implements MigrationInterface
{
  name = 'AddSwiftCodeToBankAccount1773000000000';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "bank_accounts" ADD "swift_code" character varying(11)`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "bank_accounts" DROP COLUMN "swift_code"`,
    );
  }
}
```

---

### 11. Controlador

**`src/infrastructure/adapters/api/controllers/bank-account.controller.ts`**

Importar el nuevo use-case e inyectarlo. Añadir el endpoint `GET /bank-accounts/by-account-number/:accountNumber`.

**Importante sobre el orden de rutas:** En el controlador actual, `GET /:id` está declarado antes que `DELETE /:id`. El nuevo endpoint usa un prefijo fijo `by-account-number`, por lo que no hay colisión con `/:id`. Sin embargo, se debe declarar **antes** de `GET /:id` para evitar que NestJS intente resolver `by-account-number` como un UUID de ID.

```diff
+import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';

 @ApiTags('Bank Accounts')
 @Controller('bank-accounts')
 @UseGuards(Auth0Guard, PermissionsGuard)
 @ApiBearerAuth()
 export class BankAccountController {
   constructor(
     private readonly createBankAccountUseCase: CreateBankAccountUseCaseImpl,
     private readonly updateBankAccountUseCase: UpdateBankAccountUseCaseImpl,
     private readonly deleteBankAccountUseCase: DeleteBankAccountUseCaseImpl,
     private readonly getBankAccountsByFinancialEntityUseCase: GetBankAccountsByFinancialEntityUseCaseImpl,
     private readonly getBankAccountByIdUseCase: GetBankAccountByIdUseCaseImpl,
+    private readonly getBankAccountByAccountNumberUseCase: GetBankAccountByAccountNumberUseCaseImpl,
   ) {}

   // ... endpoints existentes (POST, PUT, GET company/:companyId/...) ...

+  @Get('by-account-number/:accountNumber')
+  @ApiOperation({
+    summary: 'Buscar una cuenta bancaria por número de cuenta',
+    description:
+      'Retorna el detalle completo de una cuenta bancaria identificada por su número de cuenta, dentro de la empresa del usuario autenticado.',
+  })
+  @ApiResponse({
+    status: HttpStatus.OK,
+    description: 'Cuenta bancaria encontrada exitosamente',
+    type: BankAccountResponseDto,
+  })
+  @ApiResponse({
+    status: HttpStatus.NOT_FOUND,
+    description: 'No existe una cuenta bancaria con ese número en la empresa',
+  })
+  @ApiResponse({
+    status: HttpStatus.UNAUTHORIZED,
+    description: 'Token de autenticación inválido o no proporcionado',
+  })
+  @ApiResponse({
+    status: HttpStatus.FORBIDDEN,
+    description: 'No tiene permisos para consultar cuentas bancarias',
+  })
+  @Permissions(PermissionsEnum.BANK_ACCOUNT_READ)
+  async getBankAccountByAccountNumber(
+    @Param('accountNumber') accountNumber: string,
+    @CompanyId() companyId: string,
+  ): Promise<BankAccountResponseDto> {
+    return this.getBankAccountByAccountNumberUseCase.execute(
+      accountNumber,
+      companyId,
+    );
+  }

   @Get(':id')  // <-- este debe ir DESPUÉS del nuevo endpoint
   // ...
```

**Orden final de endpoints en el controlador:**
1. `POST /` — crear
2. `PUT /:id` — actualizar
3. `GET /company/:companyId/financial-entity/:financialEntityId` — listar por entidad financiera
4. `GET /by-account-number/:accountNumber` — **nuevo**
5. `GET /:id` — obtener por ID
6. `DELETE /:id` — eliminar

---

### 12. Módulo

**`src/infrastructure/modules/bank-account.module.ts`**

```diff
+import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';

 @Module({
   imports: [DatabaseModule],
   controllers: [BankAccountController],
   providers: [
     CreateBankAccountUseCaseImpl,
     UpdateBankAccountUseCaseImpl,
     DeleteBankAccountUseCaseImpl,
     GetBankAccountsByFinancialEntityUseCaseImpl,
     GetBankAccountByIdUseCaseImpl,
+    GetBankAccountByAccountNumberUseCaseImpl,
     PermissionsGuard,
     CurrentUserPermissionsService,
     CacheService,
   ],
   exports: [],
 })
 export class BankAccountModule {}
```

---

## Resumen de archivos afectados

| Archivo | Tipo de cambio |
|---|---|
| `src/domain/bank-account/entities/bank-account.domain.ts` | Modificado — campo `swiftCode` |
| `src/application/bank-account/dto/create-bank-account.dto.ts` | Modificado — campo `swiftCode` |
| `src/application/bank-account/dto/bank-account-response.dto.ts` | Modificado — campo `swiftCode` |
| `src/application/bank-account/mapper/bank-account.mapper.ts` | Modificado — mapear `swiftCode` |
| `src/domain/bank-account/repositories/bank-account.repository.ts` | Modificado — método `findByAccountNumber` abstracto |
| `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts` | **Nuevo** |
| `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts` | **Nuevo** |
| `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Modificado — implementar `findByAccountNumber`, propagar `swiftCode` |
| `src/infrastructure/database/entities/bank-account.entity.ts` | Modificado — columna `swift_code` |
| `src/infrastructure/database/migrations/<timestamp>-AddSwiftCodeToBankAccount.ts` | **Nuevo** |
| `src/infrastructure/adapters/api/controllers/bank-account.controller.ts` | Modificado — nuevo endpoint |
| `src/infrastructure/modules/bank-account.module.ts` | Modificado — registrar nuevo use-case |

---

## Notas adicionales

- **`UpdateBankAccountDto`** no requiere cambios porque extiende directamente `CreateBankAccountDto`. El campo `swiftCode` queda disponible en actualizaciones automáticamente.
- **`BankAccountListResponseDto`** (el DTO de la lista paginada) no incluye `swiftCode` intencionalmente: la lista es un resumen y el SWIFT es un dato de detalle. Si se necesitara en la lista, habría que añadirlo también ahí y al `select` de `getBankAccountsByFinancialEntity`.
- El `swiftCode` no tiene validación de dominio adicional más allá del `MaxLength(11)` del DTO de entrada, ya que en la práctica los códigos SWIFT tienen entre 8 y 11 caracteres y no amerita una excepción de dominio propia para un proyecto de gestión de activos.
- La clave de cache para el nuevo endpoint usa el segmento `'by-account-number'` para no colisionar con las claves existentes que usan el UUID del registro como tercer segmento.
