# Respuesta: Agregar `swiftCode` + endpoint buscar por número de cuenta (bank-account)

## Escenario detectado: C — Híbrido (campo nuevo + nuevo endpoint)

La solicitud toca dos cambios independientes:

1. **Agregar campo `swiftCode`** — afecta todas las capas + requiere migración (Scenario B: Add field to entity)
2. **Nuevo endpoint buscar por número de cuenta** — requiere nueva interfaz de caso de uso, implementación, método en repositorio y endpoint en el controlador (Scenario B: Add new operation/endpoint)

Se resuelve completamente el campo primero (Parte 1), luego el endpoint (Parte 2).

---

## Parte 1 — Agregar campo `swiftCode`

### Archivos afectados (según Impact Table)

| Capa | Archivo | Cambio |
|------|---------|--------|
| Domain | `src/domain/bank-account/entities/bank-account.domain.ts` | Agregar `_swiftCode` private field + getter + setter |
| Application | `src/application/bank-account/dto/create-bank-account.dto.ts` | Agregar `swiftCode` opcional con validaciones |
| Application | `src/application/bank-account/dto/bank-account-response.dto.ts` | Agregar `swiftCode` con `@Expose()` |
| Application | `src/application/bank-account/mapper/bank-account.mapper.ts` | Incluir `swiftCode` en `toDomain` |
| Infrastructure | `src/infrastructure/database/entities/bank-account.entity.ts` | Agregar columna `swift_code` |
| Infrastructure | `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Actualizar `mapToDomain`, `mapToEntity`, `findOneByIdWithDetails` |
| Infrastructure | Migración | `AddSwiftCodeToBankAccounts` |

---

### 1.1 — Domain: entidad `BankAccount`

**Archivo:** `src/domain/bank-account/entities/bank-account.domain.ts`

Agregar el campo `swiftCode` como opcional (máx 11 caracteres). No requiere value object propio — la validación de longitud se maneja en el DTO. En dominio sólo se guarda el valor normalizado (trim + uppercase, convención SWIFT).

**Diff conceptual — cambios a aplicar:**

```typescript
// En la interfaz BankAccountProps, agregar:
swiftCode?: string | null;

// Nuevos campos privados (junto a _fiscalConciliationConceptId):
private _swiftCode?: string | null;

// En el constructor, asignar después de _fiscalConciliationConceptId:
this._swiftCode = props.swiftCode
  ? props.swiftCode.toUpperCase().trim()
  : props.swiftCode;

// Nuevo getter (después del getter de fiscalConciliationConceptId):
get swiftCode(): string | null | undefined {
  return this._swiftCode;
}

// Nuevo setter (después del setter de fiscalConciliationConceptId):
set swiftCode(value: string | null | undefined) {
  this._swiftCode = value ? value.toUpperCase().trim() : value;
}
```

El resto del archivo (`plainToInstance`, `plainToInstanceList`) no cambia — `Object.assign` ya copia el campo `_swiftCode` automáticamente desde el objeto plano.

---

### 1.2 — Application: `CreateBankAccountDto`

**Archivo:** `src/application/bank-account/dto/create-bank-account.dto.ts`

Agregar al final de la clase, antes del cierre:

```typescript
@IsString()
@IsOptional()
@MaxLength(11)
@ApiPropertyOptional({
  description: 'Código SWIFT/BIC de la entidad financiera (máximo 11 caracteres)',
  example: 'COLOCOBM',
  maxLength: 11,
})
swiftCode?: string;
```

Importaciones adicionales requeridas:
- `ApiPropertyOptional` de `@nestjs/swagger`
- `MaxLength` ya está importado

---

### 1.3 — Application: `BankAccountResponseDto`

**Archivo:** `src/application/bank-account/dto/bank-account-response.dto.ts`

Agregar antes del cierre de la clase `BankAccountResponseDto`, después de `fiscalConciliationConcept`:

```typescript
@Expose()
@ApiProperty({
  description: 'Código SWIFT/BIC de la entidad financiera',
  example: 'COLOCOBM',
  required: false,
  nullable: true,
})
swiftCode?: string | null;
```

---

### 1.4 — Application: `BankAccountMapper`

**Archivo:** `src/application/bank-account/mapper/bank-account.mapper.ts`

En `static toDomain`, agregar `swiftCode` al objeto de construcción de `BankAccount`:

```typescript
static toDomain(
  dto: CreateBankAccountDto,
  bankAccountId?: string,
): BankAccount {
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
    swiftCode: dto.swiftCode,          // <-- agregar esta línea
  });
}
```

---

### 1.5 — Infrastructure: `BankAccountEntity`

**Archivo:** `src/infrastructure/database/entities/bank-account.entity.ts`

Agregar la columna después de `fiscalConciliationConceptId` y antes del bloque `// === Relations ===`:

```typescript
@Column({
  type: 'varchar',
  length: 11,
  name: 'swift_code',
  nullable: true,
})
swiftCode: string | null;
```

---

### 1.6 — Infrastructure: `BankAccountRepositoryImpl`

**Archivo:** `src/infrastructure/database/repositories/bank-account.repository.impl.ts`

**En `mapToDomain`**, agregar `swiftCode`:

```typescript
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
    swiftCode: entity.swiftCode,        // <-- agregar esta línea
  });
}
```

**En `mapToEntity`**, agregar `swiftCode`:

```typescript
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
  entity.swiftCode = domain.swiftCode ?? null;   // <-- agregar esta línea
  return entity;
}
```

**En `findOneByIdWithDetails`**, agregar `swiftCode: true` dentro del `select` y en el `plainToInstance` el campo se mapea automáticamente gracias a `@Expose()` en el DTO:

```typescript
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
  swiftCode: true,                    // <-- agregar esta línea
  account: { name: true },
  fiscalConciliationConcept: { name: true },
},
```

---

### 1.7 — Migration

Después de modificar la entidad, generar la migración:

```bash
npm run typeorm:migration:generate -- -n AddSwiftCodeToBankAccounts
```

La migración generada producirá algo equivalente a:

```sql
ALTER TABLE `bank_accounts` ADD `swift_code` varchar(11) NULL;
```

> **Verificar antes de aplicar:** revisar el archivo generado en `src/infrastructure/database/migrations/` y confirmar que no contenga líneas `ADD CONSTRAINT` inesperadas relacionadas con la columna `city_code` o `department_code` (ya existe esa relación multi-columna). Si aparecen, eliminarlas manualmente — la relación TypeORM seguirá funcionando.

```bash
npm run typeorm:migration:run
```

---

## Parte 2 — Nuevo endpoint: buscar por número de cuenta

### Archivos afectados

| Capa | Archivo | Cambio |
|------|---------|--------|
| Domain | `src/domain/bank-account/repositories/bank-account.repository.ts` | Agregar `findByAccountNumber` |
| Domain | `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts` | **Nuevo archivo** — interfaz del caso de uso |
| Application | `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts` | **Nuevo archivo** — implementación |
| Infrastructure | `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Implementar `findByAccountNumber` |
| Infrastructure | `src/infrastructure/adapters/api/controllers/bank-account.controller.ts` | Agregar endpoint `GET /bank-accounts/by-account-number/:accountNumber` |
| Infrastructure | `src/infrastructure/modules/bank-account.module.ts` | Registrar `GetBankAccountByAccountNumberUseCaseImpl` |

No requiere migración — no hay cambios en la estructura de la base de datos.

---

### 2.1 — Domain: repositorio `BankAccountRepository`

**Archivo:** `src/domain/bank-account/repositories/bank-account.repository.ts`

Agregar el nuevo método abstracto después de `findByCompanyId` y antes del bloque `// =========== QUERIES ============`:

```typescript
/**
 * Busca una cuenta bancaria por su número de cuenta dentro de una empresa.
 * Retorna null si no se encuentra ninguna cuenta con ese número.
 * @param accountNumber - Número de cuenta bancaria a buscar
 * @param companyId - ID de la empresa propietaria
 * @returns Entidad de dominio o null
 */
abstract findByAccountNumber(
  accountNumber: string,
  companyId: string,
): Promise<BankAccount | null>;
```

---

### 2.2 — Domain: interfaz del caso de uso

**Archivo nuevo:** `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts`

```typescript
import { BankAccountResponseDto } from '@/application/bank-account/dto/bank-account-response.dto';

/**
 * Caso de uso para obtener una cuenta bancaria por su número de cuenta
 */
export interface GetBankAccountByAccountNumberUseCase {
  /**
   * Busca una cuenta bancaria por su número de cuenta
   * 1. Verificar que existe una cuenta con ese número para la empresa
   * 2. Retornar el detalle completo de la cuenta bancaria
   * @param accountNumber - Número de cuenta bancaria
   * @param companyId - ID de la empresa
   * @returns DTO de respuesta con detalles de la cuenta bancaria
   * @throws BankAccountNotFoundException si no se encuentra la cuenta
   */
  execute(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccountResponseDto>;
}
```

---

### 2.3 — Application: implementación del caso de uso

**Archivo nuevo:** `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts`

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
    // Construir clave de caché
    const cacheKey = this.cacheService.buildKey(
      RedisKeyEnum.BANK_ACCOUNT,
      companyId,
      'account-number',
      accountNumber,
    );

    // Intentar obtener desde caché
    const cachedData =
      await this.cacheService.get<BankAccountResponseDto>(cacheKey);
    if (cachedData) {
      return plainToInstance(BankAccountResponseDto, cachedData, {
        excludeExtraneousValues: true,
      });
    }

    // Consultar repositorio por número de cuenta
    const bankAccount =
      await this.bankAccountRepository.findByAccountNumber(
        accountNumber,
        companyId,
      );

    if (!bankAccount) {
      throw new BankAccountNotFoundException(accountNumber);
    }

    // Obtener detalles completos (con account y fiscalConciliationConcept)
    const bankAccountDetails =
      await this.bankAccountRepository.findOneByIdWithDetails(bankAccount.id);

    if (!bankAccountDetails) {
      throw new BankAccountNotFoundException(accountNumber);
    }

    // Guardar en caché y retornar
    await this.cacheService.set(cacheKey, bankAccountDetails);

    return bankAccountDetails;
  }
}
```

> **Nota de diseño:** `findByAccountNumber` devuelve la entidad de dominio (para verificar existencia y obtener el `id`), y luego se reutiliza `findOneByIdWithDetails` — que ya resuelve las relaciones con `account` y `fiscalConciliationConcept` — para construir la respuesta completa. Esto evita duplicar la lógica de join en el repositorio.

---

### 2.4 — Infrastructure: implementar `findByAccountNumber` en el repositorio

**Archivo:** `src/infrastructure/database/repositories/bank-account.repository.impl.ts`

Agregar el método después de `findByCompanyId` y antes de `mapToDomain`:

```typescript
async findByAccountNumber(
  accountNumber: string,
  companyId: string,
): Promise<BankAccount | null> {
  const entity = await this.bankAccountEntityRepository.findOne({
    where: { accountNumber, companyId },
  });
  return entity ? this.mapToDomain(entity) : null;
}
```

---

### 2.5 — Infrastructure: nuevo endpoint en el controlador

**Archivo:** `src/infrastructure/adapters/api/controllers/bank-account.controller.ts`

**Agregar import:**
```typescript
import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';
```

**Agregar al constructor:**
```typescript
private readonly getBankAccountByAccountNumberUseCase: GetBankAccountByAccountNumberUseCaseImpl,
```

**Agregar el endpoint** — debe ir antes de `@Get(':id')` para que NestJS no lo interprete como un ID:

```typescript
@Get('by-account-number/:accountNumber')
@ApiOperation({
  summary: 'Buscar una cuenta bancaria por número de cuenta',
  description:
    'Retorna el detalle completo de una cuenta bancaria identificada por su número de cuenta dentro de la empresa autenticada.',
})
@ApiResponse({
  status: HttpStatus.OK,
  description: 'Cuenta bancaria encontrada exitosamente',
  type: BankAccountResponseDto,
})
@ApiResponse({
  status: HttpStatus.NOT_FOUND,
  description: 'No existe una cuenta bancaria con ese número para esta empresa',
})
@ApiResponse({
  status: HttpStatus.UNAUTHORIZED,
  description: 'Token de autenticación inválido o no proporcionado',
})
@ApiResponse({
  status: HttpStatus.FORBIDDEN,
  description: 'No tiene permisos para consultar cuentas bancarias',
})
@Permissions(PermissionsEnum.BANK_ACCOUNT_READ)
async getBankAccountByAccountNumber(
  @Param('accountNumber') accountNumber: string,
  @CompanyId() companyId: string,
): Promise<BankAccountResponseDto> {
  return this.getBankAccountByAccountNumberUseCase.execute(
    accountNumber,
    companyId,
  );
}
```

> **Orden de rutas importante:** `GET /bank-accounts/by-account-number/:accountNumber` debe declararse **antes** que `GET /bank-accounts/:id` en el controlador, de lo contrario NestJS enruta `by-account-number` como si fuera un valor de `:id`.

---

### 2.6 — Infrastructure: registrar en el módulo

**Archivo:** `src/infrastructure/modules/bank-account.module.ts`

**Agregar import:**
```typescript
import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';
```

**Agregar en el array `providers`:**
```typescript
GetBankAccountByAccountNumberUseCaseImpl,
```

El módulo queda:

```typescript
@Module({
  imports: [DatabaseModule],
  controllers: [BankAccountController],
  providers: [
    CreateBankAccountUseCaseImpl,
    UpdateBankAccountUseCaseImpl,
    DeleteBankAccountUseCaseImpl,
    GetBankAccountsByFinancialEntityUseCaseImpl,
    GetBankAccountByIdUseCaseImpl,
    GetBankAccountByAccountNumberUseCaseImpl,   // <-- agregar
    PermissionsGuard,
    CurrentUserPermissionsService,
    CacheService,
  ],
  exports: [],
})
export class BankAccountModule {}
```

---

## Checklist de verificación final

### Parte 1 — Campo `swiftCode`

- [ ] `BankAccountProps` interface incluye `swiftCode?: string | null`
- [ ] `BankAccount` tiene `private _swiftCode` con getter y setter
- [ ] Constructor normaliza: `toUpperCase().trim()` cuando no es nulo
- [ ] `CreateBankAccountDto` tiene `@IsOptional()`, `@IsString()`, `@MaxLength(11)` y `@ApiPropertyOptional`
- [ ] `BankAccountResponseDto` tiene `swiftCode` con `@Expose()` y `@ApiProperty`
- [ ] `BankAccountMapper.toDomain` pasa `swiftCode: dto.swiftCode`
- [ ] `BankAccountEntity` tiene columna `swift_code` tipo `varchar(11) nullable`
- [ ] `mapToDomain` incluye `swiftCode: entity.swiftCode`
- [ ] `mapToEntity` incluye `entity.swiftCode = domain.swiftCode ?? null`
- [ ] `findOneByIdWithDetails` incluye `swiftCode: true` en `select`
- [ ] Migración `AddSwiftCodeToBankAccounts` generada y revisada
- [ ] `npm run typeorm:migration:run` ejecutado

### Parte 2 — Endpoint por número de cuenta

- [ ] `BankAccountRepository` declara `findByAccountNumber` con JSDoc
- [ ] Interfaz `GetBankAccountByAccountNumberUseCase` creada con JSDoc de pasos
- [ ] `GetBankAccountByAccountNumberUseCaseImpl` implementa la interfaz con `@Injectable()`
- [ ] Usa caché con clave `BANK_ACCOUNT + companyId + 'account-number' + accountNumber`
- [ ] Lanza `BankAccountNotFoundException` (no `HttpException`)
- [ ] `BankAccountRepositoryImpl.findByAccountNumber` implementado
- [ ] Endpoint `GET /bank-accounts/by-account-number/:accountNumber` declarado **antes** de `GET /bank-accounts/:id`
- [ ] Endpoint protegido con `@Permissions(PermissionsEnum.BANK_ACCOUNT_READ)`
- [ ] `GetBankAccountByAccountNumberUseCaseImpl` registrado en `BankAccountModule.providers`
- [ ] `npm run start:dev` sin errores de compilación
- [ ] `npm run lint` sin errores
- [ ] Swagger en `/api/docs` muestra los endpoints nuevos con tipos correctos

---

## Resumen de archivos tocados

### Parte 1 — `swiftCode` (6 archivos + 1 migración)

| Archivo | Tipo de cambio |
|---------|---------------|
| `src/domain/bank-account/entities/bank-account.domain.ts` | Modificar |
| `src/application/bank-account/dto/create-bank-account.dto.ts` | Modificar |
| `src/application/bank-account/dto/bank-account-response.dto.ts` | Modificar |
| `src/application/bank-account/mapper/bank-account.mapper.ts` | Modificar |
| `src/infrastructure/database/entities/bank-account.entity.ts` | Modificar |
| `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Modificar |
| `src/infrastructure/database/migrations/*-AddSwiftCodeToBankAccounts.ts` | Generar |

### Parte 2 — endpoint por número de cuenta (5 archivos)

| Archivo | Tipo de cambio |
|---------|---------------|
| `src/domain/bank-account/repositories/bank-account.repository.ts` | Modificar |
| `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts` | **Nuevo** |
| `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts` | **Nuevo** |
| `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Modificar |
| `src/infrastructure/adapters/api/controllers/bank-account.controller.ts` | Modificar |
| `src/infrastructure/modules/bank-account.module.ts` | Modificar |
