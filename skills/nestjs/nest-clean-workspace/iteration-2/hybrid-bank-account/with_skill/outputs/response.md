# Respuesta: Módulo `bank-account` — Escenario C (Híbrido)

Solicitud recibida: agregar campo `swiftCode` (string, opcional, máx 11 caracteres) **y** nuevo endpoint para buscar una cuenta bancaria por su número de cuenta.

Esto es un **Escenario C — Híbrido**: primero se completa el campo en todas las capas incluyendo la migración, luego se implementa el nuevo endpoint.

---

## PARTE 1 — Agregar campo `swiftCode`

### Archivos afectados (campo)

| Capa | Archivo | Cambio |
|------|---------|--------|
| Domain | `src/domain/bank-account/entities/bank-account.domain.ts` | Agregar `swiftCode?: string \| null` a la interfaz, campo privado, getter y setter |
| Application (DTO request) | `src/application/bank-account/dto/create-bank-account.dto.ts` | Agregar `swiftCode` opcional con `@IsOptional`, `@IsString`, `@MaxLength(11)` |
| Application (DTO response) | `src/application/bank-account/dto/bank-account-response.dto.ts` | Agregar `swiftCode` con `@Expose()` y `@ApiPropertyOptional` |
| Application (Mapper) | `src/application/bank-account/mapper/bank-account.mapper.ts` | Pasar `swiftCode` en `toDomain` |
| Infrastructure (DB entity) | `src/infrastructure/database/entities/bank-account.entity.ts` | Agregar columna `swift_code` nullable |
| Infrastructure (Repo impl) | `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Incluir `swiftCode` en `mapToDomain`, `mapToEntity`, y en el `select` de `findOneByIdWithDetails` |
| Migration | nueva migración | `ALTER TABLE bank_accounts ADD COLUMN swift_code` |

---

### 1.1 Domain — Entidad `BankAccount`

**Archivo**: `src/domain/bank-account/entities/bank-account.domain.ts`

Cambios mínimos — solo se muestran las líneas nuevas o modificadas respecto al archivo actual:

```typescript
// En la interfaz BankAccountProps — agregar al final del bloque de propiedades:
swiftCode?: string | null;

// Campo privado — agregar junto a los demás campos privados:
private _swiftCode?: string | null;

// En el constructor — agregar después de la asignación de fiscalConciliationConceptId:
this._swiftCode = props.swiftCode ?? null;

// Getter — agregar después del getter de fiscalConciliationConceptId:
get swiftCode(): string | null | undefined {
  return this._swiftCode;
}

// Setter — agregar después del setter de fiscalConciliationConceptId:
set swiftCode(value: string | null | undefined) {
  this._swiftCode = value;
}
```

**Archivo completo resultante**:

```typescript
import { AccountTypeEnum } from '../enums/account-type.enum';
import { InvalidAccountNumberException } from '../exceptions/invalid-account-number.exception';
import { InvalidBranchAddressException } from '../exceptions/invalid-branch-address.exception';
import { InvalidBranchNameException } from '../exceptions/invalid-branch-name.exception';

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
  swiftCode?: string | null;
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
  private _swiftCode?: string | null;

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
    this._swiftCode = props.swiftCode ?? null;
  }

  private validateAccountNumber(accountNumber: string): void {
    if (!accountNumber || accountNumber.trim().length === 0) {
      throw new InvalidAccountNumberException();
    }
  }

  private validateBranchName(branchName: string): void {
    if (!branchName || branchName.trim().length === 0) {
      throw new InvalidBranchNameException();
    }
  }

  private validateBranchAddress(branchAddress: string): void {
    if (!branchAddress || branchAddress.trim().length === 0) {
      throw new InvalidBranchAddressException();
    }
  }

  get id(): string { return this._id; }
  get companyId(): string { return this._companyId; }
  get countryCode(): string { return this._countryCode; }
  get financialEntityId(): string { return this._financialEntityId; }
  get branchName(): string { return this._branchName; }
  get branchAddress(): string { return this._branchAddress; }
  get cityCode(): string { return this._cityCode; }
  get departmentCode(): string { return this._departmentCode; }
  get accountNumber(): string { return this._accountNumber; }
  get accountType(): AccountTypeEnum { return this._accountType; }
  get accountId(): string { return this._accountId; }
  get fiscalConciliationConceptId(): string | null | undefined { return this._fiscalConciliationConceptId; }
  get swiftCode(): string | null | undefined { return this._swiftCode; }

  set branchName(value: string) {
    this.validateBranchName(value);
    this._branchName = value.toLowerCase().trim();
  }

  set branchAddress(value: string) {
    this.validateBranchAddress(value);
    this._branchAddress = value.toLowerCase().trim();
  }

  set accountType(value: AccountTypeEnum) { this._accountType = value; }
  set accountId(value: string) { this._accountId = value; }
  set fiscalConciliationConceptId(value: string | null | undefined) { this._fiscalConciliationConceptId = value; }
  set swiftCode(value: string | null | undefined) { this._swiftCode = value; }

  /**
   * Reconstruye una instancia desde un objeto plano (ej: caché Redis).
   */
  static plainToInstance(raw: any): BankAccount {
    const instance: BankAccount = Object.create(BankAccount.prototype);
    Object.assign(instance, raw);
    return instance;
  }

  static plainToInstanceList(rawArray: any[]): BankAccount[] {
    return rawArray.map((raw) => this.plainToInstance(raw));
  }
}
```

---

### 1.2 Application — DTO de creación

**Archivo**: `src/application/bank-account/dto/create-bank-account.dto.ts`

Agregar al final de la clase `CreateBankAccountDto`, antes del cierre `}`:

```typescript
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsString, MaxLength } from 'class-validator';

// ...campos existentes...

  @IsOptional()
  @IsString()
  @MaxLength(11)
  @ApiPropertyOptional({
    description: 'Código SWIFT/BIC de la entidad financiera (máximo 11 caracteres)',
    example: 'BCOLOMCBXXX',
    maxLength: 11,
  })
  swiftCode?: string;
```

**Archivo completo resultante**:

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { AccountTypeEnum } from '@/domain/bank-account/enums/account-type.enum';
import {
  IsEnum,
  IsNotEmpty,
  IsOptional,
  IsString,
  IsUUID,
  MaxLength,
  MinLength,
} from 'class-validator';

export class CreateBankAccountDto {
  @IsUUID()
  @IsNotEmpty()
  @ApiProperty({
    description: 'ID de la empresa propietaria de la cuenta bancaria',
    example: '550e8400-e29b-41d4-a716-446655440000',
  })
  companyId: string;

  @IsString()
  @IsNotEmpty()
  @ApiProperty({
    description: 'Código del país al que pertenece la cuenta bancaria',
    example: '170',
    maxLength: 10,
  })
  countryCode: string;

  @IsUUID()
  @IsNotEmpty()
  @ApiProperty({
    description: 'ID de la entidad financiera (banco) donde está la cuenta',
    example: '550e8400-e29b-41d4-a716-446655440001',
  })
  financialEntityId: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  @MaxLength(255)
  @ApiProperty({
    description: 'Nombre de la sucursal u oficina del banco',
    example: 'Sucursal Centro',
    maxLength: 255,
  })
  branchName: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(5)
  @MaxLength(500)
  @ApiProperty({
    description: 'Dirección física de la sucursal bancaria',
    example: 'Calle 100 # 10-20, Bogotá',
    maxLength: 500,
  })
  branchAddress: string;

  @IsString()
  @IsNotEmpty()
  @MaxLength(50)
  @ApiProperty({
    description: 'Código de la ciudad (elemento nivel 2 de división política)',
    example: '11001',
    maxLength: 50,
  })
  cityCode: string;

  @IsString()
  @IsNotEmpty()
  @MaxLength(50)
  @ApiProperty({
    description: 'Código del departamento (elemento nivel 1 de división política)',
    example: '11',
    maxLength: 50,
  })
  departmentCode: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(5)
  @MaxLength(100)
  @ApiProperty({
    description: 'Número de la cuenta bancaria',
    example: '1234567890',
    maxLength: 100,
  })
  accountNumber: string;

  @IsEnum(AccountTypeEnum)
  @IsNotEmpty()
  @ApiProperty({
    description: 'Tipo de cuenta bancaria',
    enum: AccountTypeEnum,
    example: AccountTypeEnum.SAVINGS,
    default: AccountTypeEnum.SAVINGS,
  })
  accountType: AccountTypeEnum;

  @IsUUID()
  @IsNotEmpty()
  @ApiProperty({
    description: 'ID de la cuenta contable donde se registrarán las transacciones',
    example: '550e8400-e29b-41d4-a716-446655440002',
  })
  accountId: string;

  @IsUUID()
  @IsOptional()
  @ApiProperty({
    description: 'ID del concepto de conciliación fiscal (requerido solo si la empresa reporta información fiscal)',
    example: '550e8400-e29b-41d4-a716-446655440003',
    required: false,
  })
  fiscalConciliationConceptId?: string;

  @IsOptional()
  @IsString()
  @MaxLength(11)
  @ApiPropertyOptional({
    description: 'Código SWIFT/BIC de la entidad financiera (máximo 11 caracteres)',
    example: 'BCOLOMCBXXX',
    maxLength: 11,
  })
  swiftCode?: string;
}
```

> **Nota**: `UpdateBankAccountDto` extiende `CreateBankAccountDto` sin cambios adicionales, por lo que hereda `swiftCode` automáticamente. No se requiere modificar ese archivo.

---

### 1.3 Application — DTO de respuesta

**Archivo**: `src/application/bank-account/dto/bank-account-response.dto.ts`

Agregar al final de la clase `BankAccountResponseDto`, antes de los campos de relación `account` y `fiscalConciliationConcept`:

```typescript
  @Expose()
  @ApiPropertyOptional({
    description: 'Código SWIFT/BIC de la entidad financiera',
    example: 'BCOLOMCBXXX',
    nullable: true,
  })
  swiftCode?: string | null;
```

**Sección añadida** (posición: después de `fiscalConciliationConceptId`, antes de `account`):

```typescript
  @Expose()
  @ApiPropertyOptional({
    description: 'Código SWIFT/BIC de la entidad financiera',
    example: 'BCOLOMCBXXX',
    nullable: true,
  })
  swiftCode?: string | null;
```

---

### 1.4 Application — Mapper

**Archivo**: `src/application/bank-account/mapper/bank-account.mapper.ts`

Agregar `swiftCode` en el método `toDomain`:

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
      swiftCode: dto.swiftCode,          // <-- línea nueva
    });
  }
```

---

### 1.5 Infrastructure — Entidad TypeORM

**Archivo**: `src/infrastructure/database/entities/bank-account.entity.ts`

Agregar la columna `swift_code` después de la columna `fiscal_conciliation_concept_id`, antes del bloque `// === Relations ===`:

```typescript
  @Column({
    type: 'varchar',
    length: 11,
    name: 'swift_code',
    nullable: true,
  })
  swiftCode: string | null;
```

**Archivo completo resultante**:

```typescript
import { AccountTypeEnum } from '@/domain/bank-account/enums/account-type.enum';

import { AccountEntity } from './account.entity';
import { BaseEntity } from './base.entity';
import { CityEntity } from './city.entity';
import { CompanyEntity } from './company.entity';
import { FinancialEntityEntity } from './financial-entity.entity';
import { FiscalConciliationConceptEntity } from './fiscal-conciliation-concept.entity';
import { Column, Entity, Index, JoinColumn, ManyToOne } from 'typeorm';

@Entity('bank_accounts')
@Index(['companyId', 'accountNumber', 'financialEntityId'], { unique: true })
@Index('idx_bank_account_company_financial_entity', [
  'companyId',
  'financialEntityId',
])
@Index('idx_bank_account_company_country', ['companyId', 'countryCode'])
export class BankAccountEntity extends BaseEntity {
  @Column({ type: 'varchar', name: 'company_id', nullable: false })
  companyId: string;

  @Column({ type: 'varchar', length: 10, name: 'country_code', nullable: false })
  countryCode: string;

  @Column({ type: 'varchar', name: 'financial_entity_id', nullable: false })
  financialEntityId: string;

  @Column({ type: 'varchar', length: 255, name: 'branch_name', nullable: false })
  branchName: string;

  @Column({ type: 'varchar', length: 500, name: 'branch_address', nullable: false })
  branchAddress: string;

  @Column({ type: 'varchar', length: 50, name: 'city_code', nullable: false })
  cityCode: string;

  @Column({ type: 'varchar', length: 50, name: 'department_code', nullable: false })
  departmentCode: string;

  @Column({ type: 'varchar', length: 100, name: 'account_number', nullable: false })
  accountNumber: string;

  @Column({ type: 'enum', enum: AccountTypeEnum, name: 'account_type', nullable: false })
  accountType: AccountTypeEnum;

  @Column({ type: 'varchar', name: 'account_id', nullable: false })
  accountId: string;

  @Column({ type: 'varchar', name: 'fiscal_conciliation_concept_id', nullable: true })
  fiscalConciliationConceptId: string | null;

  @Column({ type: 'varchar', length: 11, name: 'swift_code', nullable: true })
  swiftCode: string | null;

  // === Relations ===
  @ManyToOne(() => CompanyEntity, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'company_id' })
  company: CompanyEntity;

  @ManyToOne(() => FinancialEntityEntity, { onDelete: 'RESTRICT' })
  @JoinColumn({ name: 'financial_entity_id' })
  financialEntity: FinancialEntityEntity;

  @ManyToOne(() => AccountEntity, { onDelete: 'RESTRICT' })
  @JoinColumn({ name: 'account_id' })
  account: AccountEntity;

  @ManyToOne(() => FiscalConciliationConceptEntity, { onDelete: 'RESTRICT' })
  @JoinColumn({ name: 'fiscal_conciliation_concept_id' })
  fiscalConciliationConcept: FiscalConciliationConceptEntity;

  @ManyToOne(() => CityEntity)
  @JoinColumn([
    { name: 'city_code', referencedColumnName: 'code' },
    { name: 'department_code', referencedColumnName: 'departmentCode' },
  ])
  city: CityEntity;
}
```

---

### 1.6 Infrastructure — Repositorio implementación

**Archivo**: `src/infrastructure/database/repositories/bank-account.repository.impl.ts`

Tres puntos de cambio:

**`mapToDomain`** — agregar `swiftCode`:
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
      swiftCode: entity.swiftCode,           // <-- línea nueva
    });
  }
```

**`mapToEntity`** — agregar `swiftCode`:
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
    entity.fiscalConciliationConceptId = domain.fiscalConciliationConceptId ?? null;
    entity.swiftCode = domain.swiftCode ?? null;  // <-- línea nueva
    return entity;
  }
```

**`findOneByIdWithDetails`** — incluir `swiftCode` en el `select`:
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
        swiftCode: true,                       // <-- línea nueva
        account: { name: true },
        fiscalConciliationConcept: { name: true },
      },
```

---

### 1.7 Migración

Después de modificar la entidad TypeORM, generar la migración:

```bash
npm run typeorm:migration:generate -- -n AddSwiftCodeToBankAccounts
```

El archivo generado en `src/infrastructure/database/migrations/` contendrá algo similar a:

```typescript
export class AddSwiftCodeToBankAccounts1741234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE \`bank_accounts\` ADD \`swift_code\` varchar(11) NULL`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE \`bank_accounts\` DROP COLUMN \`swift_code\``,
    );
  }
}
```

> Revisar el archivo generado y eliminar cualquier `ADD CONSTRAINT` no deseado antes de ejecutarla.

```bash
npm run typeorm:migration:run
```

---

## PARTE 2 — Nuevo endpoint: buscar cuenta bancaria por número de cuenta

Una vez completada y verificada la Parte 1 (incluida la migración), se implementa el nuevo endpoint.

### Archivos afectados (nuevo endpoint)

| Capa | Archivo | Cambio |
|------|---------|--------|
| Domain (repo interface) | `src/domain/bank-account/repositories/bank-account.repository.ts` | Agregar método `findByAccountNumber` |
| Domain (use case interface) | `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts` | Crear nuevo archivo |
| Application (use case impl) | `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts` | Crear nuevo archivo |
| Infrastructure (repo impl) | `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Implementar `findByAccountNumber` |
| Infrastructure (controller) | `src/infrastructure/adapters/api/controllers/bank-account.controller.ts` | Agregar endpoint `GET /bank-accounts/by-account-number/:accountNumber` |
| Infrastructure (module) | `src/infrastructure/modules/bank-account.module.ts` | Registrar nuevo use case |

---

### 2.1 Domain — Repositorio: nuevo método

**Archivo**: `src/domain/bank-account/repositories/bank-account.repository.ts`

Agregar método después de `findByCompanyId`:

```typescript
  /**
   * Busca una cuenta bancaria por su número de cuenta dentro de una empresa.
   * Retorna null si no existe.
   * @param accountNumber - Número de cuenta bancaria a buscar
   * @param companyId - ID de la empresa propietaria
   * @returns La cuenta bancaria encontrada o null
   */
  abstract findByAccountNumber(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccount | null>;
```

**Archivo completo resultante**:

```typescript
import { BankAccountListResponseDto } from '@/application/bank-account/dto/bank-account-list-response.dto';
import { BankAccountResponseDto } from '@/application/bank-account/dto/bank-account-response.dto';
import { PageOptionsDto } from '@/application/pagination/dtos';
import { BaseRepository } from '@/domain/common/repositories/base.repository';
import { PaginationResponse } from '@/infrastructure/database/interfaces/pagination.response';

import { BankAccount } from '../entities/bank-account.domain';

/**
 * Repositorio de cuentas bancarias
 */
export abstract class BankAccountRepository extends BaseRepository<BankAccount> {
  /**
   * Verifica si existe una cuenta con el mismo número en la misma entidad financiera para una compañía
   */
  abstract existsByAccountNumberAndFinancialEntity(
    accountNumber: string,
    financialEntityId: string,
    companyId: string,
  ): Promise<boolean>;

  /**
   * Busca todas las cuentas bancarias de una compañía
   */
  abstract findByCompanyId(companyId: string): Promise<BankAccount[]>;

  /**
   * Busca una cuenta bancaria por su número de cuenta dentro de una empresa.
   * Retorna null si no existe.
   * @param accountNumber - Número de cuenta bancaria a buscar
   * @param companyId - ID de la empresa propietaria
   * @returns La cuenta bancaria encontrada o null
   */
  abstract findByAccountNumber(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccount | null>;

  // =========== QUERIES ============
  /**
   * Obtiene las cuentas bancarias de una empresa filtradas por entidad financiera.
   */
  abstract getBankAccountsByFinancialEntity(
    companyId: string,
    financialEntityId: string,
    page: PageOptionsDto,
  ): Promise<PaginationResponse<BankAccountListResponseDto>>;

  /**
   * Busca una cuenta bancaria por ID con detalles de cuenta contable y concepto fiscal
   */
  abstract findOneByIdWithDetails(
    id: string,
  ): Promise<BankAccountResponseDto | null>;
}
```

---

### 2.2 Domain — Interfaz del caso de uso

**Archivo nuevo**: `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts`

```typescript
import { BankAccountResponseDto } from '@/application/bank-account/dto/bank-account-response.dto';

/**
 * Caso de uso para obtener una cuenta bancaria por su número de cuenta
 */
export interface GetBankAccountByAccountNumberUseCase {
  /**
   * 1. Buscar la cuenta bancaria por número de cuenta dentro de la empresa
   * 2. Lanzar BankAccountNotFoundException si no se encuentra
   * 3. Retornar el detalle completo de la cuenta
   * @param accountNumber - Número de cuenta bancaria a buscar
   * @param companyId - ID de la empresa propietaria
   * @returns DTO de respuesta con el detalle de la cuenta bancaria
   * @throws BankAccountNotFoundException si no se encuentra la cuenta
   */
  execute(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccountResponseDto>;
}
```

---

### 2.3 Application — Implementación del caso de uso

**Archivo nuevo**: `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts`

```typescript
import { Injectable } from '@nestjs/common';

import { RedisKeyEnum } from '@/common/constants/enums/redis-key.enum';
import { BankAccountNotFoundException } from '@/domain/bank-account/exceptions/bank-account-not-found.exception';
import { BankAccountRepository } from '@/domain/bank-account/repositories/bank-account.repository';
import { GetBankAccountByAccountNumberUseCase } from '@/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case';
import { CacheService } from '@/infrastructure/external/services/cache/cache.service';
import { plainToInstance } from 'class-transformer';

import { BankAccountResponseDto } from '../dto/bank-account-response.dto';

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
    // Construir clave de caché usando número de cuenta y empresa
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

    // Buscar en repositorio por número de cuenta
    const bankAccount = await this.bankAccountRepository.findByAccountNumber(
      accountNumber,
      companyId,
    );

    if (!bankAccount) {
      throw new BankAccountNotFoundException();
    }

    // Obtener el detalle completo con relaciones
    const detail = await this.bankAccountRepository.findOneByIdWithDetails(
      bankAccount.id,
    );

    if (!detail) {
      throw new BankAccountNotFoundException();
    }

    // Guardar en caché
    await this.cacheService.set(cacheKey, detail);

    return detail;
  }
}
```

---

### 2.4 Infrastructure — Repositorio: implementar `findByAccountNumber`

**Archivo**: `src/infrastructure/database/repositories/bank-account.repository.impl.ts`

Agregar el nuevo método después de `findByCompanyId`, antes de `mapToDomain`:

```typescript
  async findByAccountNumber(
    accountNumber: string,
    companyId: string,
  ): Promise<BankAccount | null> {
    // Buscar por número de cuenta normalizado (guardado en minúsculas)
    const entity = await this.bankAccountEntityRepository.findOne({
      where: {
        accountNumber: accountNumber.toLowerCase().trim(),
        companyId,
      },
    });
    return entity ? this.mapToDomain(entity) : null;
  }
```

---

### 2.5 Infrastructure — Controlador: nuevo endpoint

**Archivo**: `src/infrastructure/adapters/api/controllers/bank-account.controller.ts`

Agregar la inyección del nuevo use case en el constructor y el nuevo endpoint.

**Constructor** — agregar la nueva dependencia:
```typescript
  constructor(
    private readonly createBankAccountUseCase: CreateBankAccountUseCaseImpl,
    private readonly updateBankAccountUseCase: UpdateBankAccountUseCaseImpl,
    private readonly deleteBankAccountUseCase: DeleteBankAccountUseCaseImpl,
    private readonly getBankAccountsByFinancialEntityUseCase: GetBankAccountsByFinancialEntityUseCaseImpl,
    private readonly getBankAccountByIdUseCase: GetBankAccountByIdUseCaseImpl,
    private readonly getBankAccountByAccountNumberUseCase: GetBankAccountByAccountNumberUseCaseImpl,  // <-- nuevo
  ) {}
```

**Nuevo endpoint** — agregar antes del endpoint `@Get(':id')` para que la ruta estática tome precedencia sobre el parámetro dinámico:

```typescript
  @Get('by-account-number/:accountNumber')
  @ApiOperation({
    summary: 'Obtener una cuenta bancaria por número de cuenta',
    description:
      'Busca y retorna el detalle completo de una cuenta bancaria usando su número de cuenta. La búsqueda se realiza dentro de la empresa del usuario autenticado.',
  })
  @ApiResponse({
    status: HttpStatus.OK,
    description: 'Cuenta bancaria encontrada exitosamente',
    type: BankAccountResponseDto,
  })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: 'No existe una cuenta bancaria con ese número de cuenta',
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

**Archivo completo resultante** del controlador (con ambas partes integradas):

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  HttpCode,
  HttpStatus,
  Param,
  Post,
  Put,
  Query,
  UseGuards,
} from '@nestjs/common';
import {
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
  ApiTags,
} from '@nestjs/swagger';

import { BankAccountListResponseDto } from '@/application/bank-account/dto/bank-account-list-response.dto';
import { BankAccountResponseDto } from '@/application/bank-account/dto/bank-account-response.dto';
import { CreateBankAccountDto } from '@/application/bank-account/dto/create-bank-account.dto';
import { UpdateBankAccountDto } from '@/application/bank-account/dto/update-bank-account.dto';
import { BankAccountMapper } from '@/application/bank-account/mapper/bank-account.mapper';
import { CreateBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/create-bank-account.use-case.impl';
import { DeleteBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/delete-bank-account.use-case.impl';
import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';
import { GetBankAccountByIdUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-id.use-case.impl';
import { GetBankAccountsByFinancialEntityUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-accounts-by-financial-entity.use-case.impl';
import { UpdateBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/update-bank-account.use-case.impl';
import { PageDto, PageOptionsDto } from '@/application/pagination/dtos';
import { PaginationService } from '@/application/pagination/services/pagination.service';
import { PermissionsEnum } from '@/common/constants/enums/permissions.enum';
import { ApiPaginatedResponse } from '@/common/decorators/api-paginate-response.decorator';
import { CompanyId } from '@/common/decorators/company-id.decorator';
import { Permissions } from '@/common/decorators/permissions.decorator';
import { IdResponseDto } from '@/common/dtos/id-response.dto';
import { Auth0Guard } from '@/common/guards/auth0.guard';
import { PermissionsGuard } from '@/common/guards/permissions.guard';

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
    private readonly getBankAccountByAccountNumberUseCase: GetBankAccountByAccountNumberUseCaseImpl,
  ) {}

  @Post()
  @ApiOperation({ summary: 'Crear una nueva cuenta bancaria' })
  @ApiResponse({ status: HttpStatus.CREATED, description: 'Cuenta bancaria creada exitosamente', type: IdResponseDto })
  @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'Datos inválidos o número de cuenta duplicado' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para crear cuentas bancarias' })
  @Permissions(PermissionsEnum.BANK_ACCOUNT_CREATE)
  async createBankAccount(
    @Body() createDto: CreateBankAccountDto,
  ): Promise<IdResponseDto> {
    const bankAccount = BankAccountMapper.toDomain(createDto);
    const createdBankAccount = await this.createBankAccountUseCase.execute(bankAccount);
    return BankAccountMapper.toIdResponse(createdBankAccount);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Actualizar una cuenta bancaria existente' })
  @ApiResponse({ status: HttpStatus.OK, description: 'Cuenta bancaria actualizada exitosamente' })
  @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'Datos inválidos, número de cuenta duplicado o intento de cambiar la empresa' })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Cuenta bancaria no encontrada' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para actualizar cuentas bancarias' })
  @ApiBearerAuth()
  @Permissions(PermissionsEnum.BANK_ACCOUNT_UPDATE)
  async updateBankAccount(
    @Param('id') id: string,
    @Body() updateDto: UpdateBankAccountDto,
  ): Promise<void> {
    const bankAccount = BankAccountMapper.toDomain(updateDto, id);
    await this.updateBankAccountUseCase.execute(bankAccount);
  }

  @Get('company/:companyId/financial-entity/:financialEntityId')
  @ApiOperation({ summary: 'Obtener cuentas bancarias por entidad financiera' })
  @ApiPaginatedResponse(BankAccountListResponseDto)
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para consultar cuentas bancarias' })
  @Permissions(PermissionsEnum.BANK_ACCOUNT_READ)
  async getBankAccountsByFinancialEntity(
    @Param('companyId') companyId: string,
    @Param('financialEntityId') financialEntityId: string,
    @Query() pageOptionsDto: PageOptionsDto,
  ): Promise<PageDto<BankAccountListResponseDto>> {
    const result = await this.getBankAccountsByFinancialEntityUseCase.execute(
      companyId,
      financialEntityId,
      pageOptionsDto,
    );
    return PaginationService.createPageDto(
      result.items,
      result.count,
      pageOptionsDto,
      BankAccountListResponseDto,
    );
  }

  @Get('by-account-number/:accountNumber')
  @ApiOperation({
    summary: 'Obtener una cuenta bancaria por número de cuenta',
    description:
      'Busca y retorna el detalle completo de una cuenta bancaria usando su número de cuenta. La búsqueda se realiza dentro de la empresa del usuario autenticado.',
  })
  @ApiResponse({ status: HttpStatus.OK, description: 'Cuenta bancaria encontrada exitosamente', type: BankAccountResponseDto })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'No existe una cuenta bancaria con ese número de cuenta' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para consultar cuentas bancarias' })
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

  @Get(':id')
  @ApiOperation({ summary: 'Obtener el detalle completo de una cuenta bancaria por ID' })
  @ApiResponse({ status: HttpStatus.OK, description: 'Cuenta bancaria encontrada exitosamente', type: BankAccountResponseDto })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Cuenta bancaria no encontrada' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para consultar el detalle de cuentas bancarias' })
  @ApiBearerAuth()
  @Permissions(PermissionsEnum.BANK_ACCOUNT_READ)
  async getBankAccountById(
    @Param('id') id: string,
    @CompanyId() companyId: string,
  ): Promise<BankAccountResponseDto> {
    return this.getBankAccountByIdUseCase.execute(id, companyId);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Eliminar una cuenta bancaria' })
  @ApiResponse({ status: HttpStatus.NO_CONTENT, description: 'Cuenta bancaria eliminada exitosamente' })
  @ApiResponse({ status: HttpStatus.BAD_REQUEST, description: 'La cuenta bancaria no puede ser eliminada porque está asociada a registros existentes' })
  @ApiResponse({ status: HttpStatus.NOT_FOUND, description: 'Cuenta bancaria no encontrada' })
  @ApiResponse({ status: HttpStatus.UNAUTHORIZED, description: 'Token de autenticación inválido o no proporcionado' })
  @ApiResponse({ status: HttpStatus.FORBIDDEN, description: 'No tiene permisos para eliminar cuentas bancarias' })
  @Permissions(PermissionsEnum.BANK_ACCOUNT_DELETE)
  async deleteBankAccount(
    @CompanyId() companyId: string,
    @Param('id') id: string,
  ): Promise<void> {
    await this.deleteBankAccountUseCase.execute(id, companyId);
  }
}
```

> **Nota de orden de rutas**: `GET by-account-number/:accountNumber` se declara **antes** de `GET :id` para que NestJS no interprete `by-account-number` como un UUID. Este es un requisito de enrutamiento de Express/NestJS — las rutas estáticas deben preceder a las dinámicas.

---

### 2.6 Infrastructure — Módulo

**Archivo**: `src/infrastructure/modules/bank-account.module.ts`

Agregar `GetBankAccountByAccountNumberUseCaseImpl` al array `providers`:

```typescript
import { Module } from '@nestjs/common';

import { CreateBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/create-bank-account.use-case.impl';
import { DeleteBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/delete-bank-account.use-case.impl';
import { GetBankAccountByAccountNumberUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl';
import { GetBankAccountByIdUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-account-by-id.use-case.impl';
import { GetBankAccountsByFinancialEntityUseCaseImpl } from '@/application/bank-account/use-cases/get-bank-accounts-by-financial-entity.use-case.impl';
import { UpdateBankAccountUseCaseImpl } from '@/application/bank-account/use-cases/update-bank-account.use-case.impl';
import { PermissionsGuard } from '@/common/guards/permissions.guard';

import { BankAccountController } from '../adapters/api/controllers/bank-account.controller';
import { DatabaseModule } from '../database/database.module';
import { CacheService } from '../external/services/cache/cache.service';
import { CurrentUserPermissionsService } from '../services/security/current-user-permissions.service';

@Module({
  imports: [DatabaseModule],
  controllers: [BankAccountController],
  providers: [
    CreateBankAccountUseCaseImpl,
    UpdateBankAccountUseCaseImpl,
    DeleteBankAccountUseCaseImpl,
    GetBankAccountsByFinancialEntityUseCaseImpl,
    GetBankAccountByIdUseCaseImpl,
    GetBankAccountByAccountNumberUseCaseImpl,   // <-- nuevo
    PermissionsGuard,
    CurrentUserPermissionsService,
    CacheService,
  ],
  exports: [],
})
export class BankAccountModule {}
```

---

## Resumen de archivos modificados / creados

### PARTE 1 — Campo `swiftCode`

| Archivo | Tipo de cambio |
|---------|---------------|
| `src/domain/bank-account/entities/bank-account.domain.ts` | Modificado |
| `src/application/bank-account/dto/create-bank-account.dto.ts` | Modificado |
| `src/application/bank-account/dto/bank-account-response.dto.ts` | Modificado |
| `src/application/bank-account/mapper/bank-account.mapper.ts` | Modificado |
| `src/infrastructure/database/entities/bank-account.entity.ts` | Modificado |
| `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Modificado (`mapToDomain`, `mapToEntity`, `findOneByIdWithDetails`) |
| `src/infrastructure/database/migrations/XXXX-AddSwiftCodeToBankAccounts.ts` | Generado con `npm run typeorm:migration:generate` |

### PARTE 2 — Endpoint por número de cuenta

| Archivo | Tipo de cambio |
|---------|---------------|
| `src/domain/bank-account/repositories/bank-account.repository.ts` | Modificado (nuevo método abstracto `findByAccountNumber`) |
| `src/domain/bank-account/use-cases/get-bank-account-by-account-number.use-case.ts` | Creado |
| `src/application/bank-account/use-cases/get-bank-account-by-account-number.use-case.impl.ts` | Creado |
| `src/infrastructure/database/repositories/bank-account.repository.impl.ts` | Modificado (implementación de `findByAccountNumber`) |
| `src/infrastructure/adapters/api/controllers/bank-account.controller.ts` | Modificado (nuevo endpoint + nueva inyección) |
| `src/infrastructure/modules/bank-account.module.ts` | Modificado (registrar nuevo use case) |

---

## Verificación final

```bash
# 1. Sin errores de compilación TypeScript
npm run start:dev

# 2. Sin errores de lint
npm run lint

# 3. Swagger en /api/docs muestra:
#    - Campo swiftCode en el body de POST /bank-accounts y PUT /bank-accounts/:id
#    - Campo swiftCode en la respuesta de GET /bank-accounts/:id
#    - Nuevo endpoint GET /bank-accounts/by-account-number/:accountNumber

# 4. Migración ejecutada
npm run typeorm:migration:run
```

### Puntos de atención

- El campo `accountNumber` se persiste en minúsculas (`toLowerCase().trim()`) en el dominio. El método `findByAccountNumber` en el repositorio normaliza el parámetro de búsqueda de la misma forma para garantizar consistencia.
- El endpoint `GET by-account-number/:accountNumber` debe estar declarado **antes** de `GET :id` en el controlador.
- `UpdateBankAccountDto` hereda `swiftCode` automáticamente de `CreateBankAccountDto` sin cambios adicionales.
- La caché del endpoint de búsqueda por número de cuenta usa una clave compuesta con el prefijo `'account-number'` para evitar colisiones con la caché del endpoint por ID (`BANK_ACCOUNT + companyId + id`). Cuando se invalide la caché por mutaciones, `removeCacheByPartialKey(RedisKeyEnum.BANK_ACCOUNT)` limpiará ambas.
