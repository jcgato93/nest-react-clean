# Respuesta del Agente — Módulo `mantenimientos`

---

## Confirmación del módulo (Escenario A — Nuevo módulo desde cero)

Before I start, let me confirm the module details:

- **Module name**: `mantenimiento` → DB table: `mantenimientos` (snake_case, plural)
- **Entity fields**:
  - `descripcion` — string — required
  - `costo` — decimal — required
  - `fecha` — date — required
  - `tipo` — enum (`PREVENTIVO`, `CORRECTIVO`, `GARANTIA`) — required
  - `companyId` — string (UUID) — required (foreign key to company)
- **Operations**:
  - Crear mantenimiento (`POST /mantenimientos`)
  - Obtener mantenimiento por ID (`GET /mantenimientos/:id`)
  - Listar mantenimientos por compañía con paginación (`GET /mantenimientos`)
- **Authentication**: `Auth0Guard` on all endpoints — all endpoints are protected
- **Pagination**: list endpoint returns paginated results via `PageOptionsDto` / `PageDto`

Does this look correct? I'll start with the domain layer once confirmed.

---

## Plan completo de archivos

### Step 1 — Domain Layer

> Sub-skill leído: `.github/skills/domain-layer/SKILL.md`

#### `src/domain/mantenimiento/entities/mantenimiento.domain.ts`

- **Clase**: `Mantenimiento`
- **Interfaz**: `IMantenimiento`
- **Campos privados**: `_id`, `_descripcion`, `_costo`, `_fecha`, `_tipo`, `_companyId`
- **Tipos**: `id?: string`, `descripcion: string`, `costo: number`, `fecha: Date`, `tipo: MantenimientoTipoEnum`, `companyId: string`
- `id` es opcional en la interfaz — la entidad genera un UUID si no se provee (`crypto.randomUUID()`)
- Getters públicos para cada campo
- `static plainToInstance(raw: any): Mantenimiento` — obligatorio
- `static plainToInstanceList(raw: any[]): Mantenimiento[]` — obligatorio
- Sin comportamiento de dominio adicional por ahora (el módulo no tiene reglas de mutación en este alcance)

#### `src/domain/mantenimiento/repositories/mantenimiento.repository.ts`

- **Clase abstracta**: `MantenimientoRepository extends BaseRepository<Mantenimiento>`
- NO redeclarar `findById`, `save`, `findAll` (heredados de `BaseRepository`)
- Sección `// =========== QUERIES ============`:
  - `abstract getListByCompany(companyId: string, page: PageOptionsDto): Promise<PaginationResponse<MantenimientoResponseDto>>`
  - JSDoc: describe que retorna lista paginada de mantenimientos para una compañía

#### `src/domain/mantenimiento/use-cases/create-mantenimiento.use-case.ts`

- **Interfaz**: `CreateMantenimientoUseCase`
- `execute(companyId: string, mantenimiento: Mantenimiento): Promise<string>`
- JSDoc pasos:
  1. Verificar que la compañía exista
  2. Crear y guardar el mantenimiento
  3. Invalidar caché relacionado
  - `@returns` ID del mantenimiento creado

#### `src/domain/mantenimiento/use-cases/get-mantenimiento.use-case.ts`

- **Interfaz**: `GetMantenimientoUseCase`
- `execute(id: string): Promise<Mantenimiento>`
- JSDoc pasos:
  1. Intentar obtener desde caché
  2. Consultar repositorio
  3. Lanzar `MantenimientoNotFoundException` si no existe
  4. Guardar en caché y retornar

#### `src/domain/mantenimiento/use-cases/get-mantenimiento-list.use-case.ts`

- **Interfaz**: `GetMantenimientoListUseCase`
- `execute(companyId: string, page: PageOptionsDto): Promise<PaginationResponse<MantenimientoResponseDto>>`
- JSDoc pasos:
  1. Intentar obtener desde caché usando companyId + page + take
  2. Consultar repositorio
  3. Guardar en caché y retornar

#### `src/domain/mantenimiento/exceptions/mantenimiento-not-found.exception.ts`

- **Clase**: `MantenimientoNotFoundException extends CustomNotFoundException`
- Constructor recibe `id: string`
- Mensaje: `El mantenimiento con ID ${id} no fue encontrado.`
- Importa de `@/domain/common/exceptions/custom-not-found.exception`

#### Enum (ubicación compartida)

- **Archivo**: `src/common/constants/enums/mantenimiento-tipo.enum.ts`
- **Nombre**: `MantenimientoTipoEnum`
- **Valores**: `PREVENTIVO = 'PREVENTIVO'`, `CORRECTIVO = 'CORRECTIVO'`, `GARANTIA = 'GARANTIA'`
- Nota: los enums compartidos van en `src/common/constants/enums/`, no dentro de `src/domain/`

#### Checklist — Domain Layer

- [ ] Entidad tiene todos los campos con tipos privados correctos que coinciden con los getters
- [ ] Entidad tiene `plainToInstance` y `plainToInstanceList`
- [ ] Repositorio extiende `BaseRepository` y solo declara métodos nuevos
- [ ] Repositorio tiene JSDoc en cada método
- [ ] Interfaz de caso de uso tiene JSDoc paso a paso y tipo de retorno correcto
- [ ] Excepción extiende la base correcta y tiene mensaje en español
- [ ] Sin importaciones de `@/application` o `@/infrastructure` en `src/domain/`

---

### Step 2 — Application Layer

> Sub-skill leído: `.github/skills/application/SKILL.md`
> Iniciar solo después de completar y verificar el Step 1.

#### `src/application/mantenimiento/dtos/create-mantenimiento.dto.ts`

- **Clase**: `CreateMantenimientoDto`
- Campos y decoradores:
  - `descripcion: string` — `@ApiProperty` + `@IsString()` + `@IsNotEmpty()`
  - `costo: number` — `@ApiProperty` + `@IsNumber()` + `@Min(0)`
  - `fecha: Date` — `@ApiProperty` + `@IsDateString()` (o `@IsDate()` con `@Type(() => Date)`)
  - `tipo: MantenimientoTipoEnum` — `@ApiProperty({ enum: MantenimientoTipoEnum })` + `@IsEnum(MantenimientoTipoEnum)`
- `companyId` NO va en el DTO de request — se extrae del token con el decorator `@CompanyId()` en el controlador

#### `src/application/mantenimiento/dtos/mantenimiento-response.dto.ts`

- **Clase**: `MantenimientoResponseDto`
- Campos con `@ApiProperty` + `@Expose()`:
  - `id: string`
  - `descripcion: string`
  - `costo: number`
  - `fecha: Date`
  - `tipo: MantenimientoTipoEnum`
  - `companyId: string`
- Nota: solo los campos con `@Expose()` aparecen en la respuesta serializada

#### `src/application/mantenimiento/mappers/mantenimiento.mapper.ts`

- **Clase abstracta**: `MantenimientoMapper`
- `static toDomain(dto: CreateMantenimientoDto, companyId: string): Mantenimiento`
  - Construye `new Mantenimiento({ descripcion, costo, fecha: new Date(dto.fecha), tipo, companyId })`
- `static toDto(entity: Mantenimiento): MantenimientoResponseDto`
  - Retorna objeto con todos los campos del getter de la entidad
- `static toDtoList(entities: Mantenimiento[]): MantenimientoResponseDto[]`
- Sin lógica de negocio — solo transformación estructural
- No se usan value objects para los campos de este módulo (son primitivos simples); si se agrega validación compleja en el futuro, crear value objects correspondientes

#### `src/application/mantenimiento/use-cases/create-mantenimiento.use-case.impl.ts`

- **Clase**: `CreateMantenimientoUseCaseImpl implements CreateMantenimientoUseCase`
- `@Injectable()`
- Inyecta: `MantenimientoRepository`, `CacheService`
- Flujo `execute`:
  1. `await this.mantenimientoRepository.save(mantenimiento)`
  2. `await this.cacheService.removeCacheByPartialKey(RedisKeyEnum.MANTENIMIENTO)`
  3. Retorna `mantenimiento.id`
- Lanza excepciones de dominio (nunca `HttpException`)

#### `src/application/mantenimiento/use-cases/get-mantenimiento.use-case.impl.ts`

- **Clase**: `GetMantenimientoUseCaseImpl implements GetMantenimientoUseCase`
- `@Injectable()`
- Inyecta: `MantenimientoRepository`, `CacheService`
- Flujo `execute`:
  1. Construir clave: `cacheService.buildKey(RedisKeyEnum.MANTENIMIENTO, id)`
  2. Si hay caché: retornar `Mantenimiento.plainToInstance(cached)`
  3. Consultar `mantenimientoRepository.findById(id)`
  4. Si no existe: lanzar `MantenimientoNotFoundException(id)`
  5. Guardar en caché y retornar

#### `src/application/mantenimiento/use-cases/get-mantenimiento-list.use-case.impl.ts`

- **Clase**: `GetMantenimientoListUseCaseImpl implements GetMantenimientoListUseCase`
- `@Injectable()`
- Inyecta: `MantenimientoRepository`, `CacheService`
- Flujo `execute`:
  1. Construir clave: `cacheService.buildKey(RedisKeyEnum.MANTENIMIENTO, companyId, page, take)`
  2. Si hay caché: retornar `{ items: Mantenimiento.plainToInstanceList(data.items), count: data.count }`
  3. Consultar `mantenimientoRepository.getListByCompany(companyId, pageOptions)`
  4. Guardar en caché y retornar resultado

#### Redis Key Enum — actualizar archivo existente

- **Archivo**: `src/common/constants/enums/redis-key.enum.ts`
- **Accion**: agregar `MANTENIMIENTO = 'mantenimiento'` al enum `RedisKeyEnum` existente
- No crear un archivo nuevo — modificar el enum existente

#### Checklist — Application Layer

- [ ] Clase de caso de uso tiene `@Injectable()` e implementa la interfaz de dominio
- [ ] Todas las dependencias del constructor usan `private readonly`
- [ ] Casos de uso de lectura verifican caché antes de consultar el repositorio
- [ ] Entidades reconstruidas desde caché usan `plainToInstance`/`plainToInstanceList`
- [ ] Mutaciones invalidan caché con `removeCacheByPartialKey`
- [ ] Solo se lanzan excepciones de dominio (sin `HttpException`)
- [ ] DTOs de request tienen `@ApiProperty` + class-validator en cada campo
- [ ] DTOs de response tienen `@ApiProperty` + `@Expose()` en cada campo
- [ ] Mapper es `abstract class` con todos los métodos `static`

---

### Step 3 — Infrastructure Layer

> Sub-skill leído: `.github/skills/infrastructure-layer/SKILL.md`
> Iniciar solo después de completar y verificar el Step 2.

#### `src/infrastructure/database/entities/mantenimiento.entity.ts`

- **Clase**: `MantenimientoEntity extends BaseEntity`
- `@Entity('mantenimientos')`
- Columnas:
  - `@Column({ type: 'text' }) descripcion: string`
  - `@Column({ type: 'decimal', precision: 10, scale: 2 }) costo: number`
  - `@Column({ type: 'date' }) fecha: Date`
  - `@Column({ type: 'enum', enum: MantenimientoTipoEnum }) tipo: MantenimientoTipoEnum`
  - `@Column({ type: 'varchar', length: 36, name: 'company_id' }) companyId: string`
- Relaciones al final de la clase:
  - `@ManyToOne(() => CompanyEntity) @JoinColumn({ name: 'company_id' }) company?: CompanyEntity`
- `BaseEntity` provee `id`, `createdAt`, `updatedAt` — no redeclararlos
- Generar migración después de crear esta entidad

#### Migración

- Comando: `npm run typeorm:migration:generate -- -n CreateMantenimientosTable`
- Archivo generado en: `src/infrastructure/database/migrations/`
- Revisar el archivo generado y eliminar cualquier `ADD CONSTRAINT` no deseado para relaciones que referencien columnas no únicas

#### `src/infrastructure/database/repositories/mantenimiento.repository.impl.ts`

- **Clase**: `MantenimientoRepositoryImpl extends BaseRepositoryImpl<MantenimientoEntity, Mantenimiento> implements MantenimientoRepository`
- `@Injectable()`
- Inyecta: `@InjectRepository(MantenimientoEntity) private readonly mantenimientoRepo: Repository<MantenimientoEntity>`
- Constructor: `super(mantenimientoRepo)`
- Implementar métodos de mapeo:
  - `mapToDomain(entity: MantenimientoEntity): Mantenimiento` — construye `new Mantenimiento({ id, descripcion, costo, fecha, tipo, companyId })`
  - `mapToEntity(domain: Partial<Mantenimiento>): DeepPartial<MantenimientoEntity>` — retorna `{ descripcion, costo, fecha, tipo, companyId }`
- Sección `// =========== QUERIES ============`:
  - `async getListByCompany(companyId: string, page: PageOptionsDto): Promise<PaginationResponse<MantenimientoResponseDto>>`
  - Usa `findAndCount` con `where: { companyId }`, `order: { fecha: 'DESC' }`, `take: page.take`, `skip: page.skip`
  - Construir instancias tipadas: `const dto = new MantenimientoResponseDto(); dto.id = e.id; ...`
  - Usar `relations: { company: true }` (notación objeto, nunca array de strings)

#### `src/infrastructure/adapters/api/controllers/mantenimiento.controller.ts`

- **Clase**: `MantenimientoController`
- `@ApiTags('Mantenimientos')`
- `@Controller('mantenimientos')`
- `@UseGuards(Auth0Guard)` a nivel de clase — todos los endpoints protegidos
- `@ApiBearerAuth()` a nivel de clase
- Inyectar en constructor (nunca repositorios directamente):
  - `CreateMantenimientoUseCaseImpl`
  - `GetMantenimientoUseCaseImpl`
  - `GetMantenimientoListUseCaseImpl`
- Endpoints:

  **POST /**
  - `@Post()`
  - `@ApiOperation({ summary: 'Crear un nuevo mantenimiento' })`
  - `@ApiResponse({ status: 201, description: 'Mantenimiento creado.', type: String })`
  - `@ApiResponse({ status: 400, description: 'Datos inválidos.' })`
  - Parámetros: `@Body() dto: CreateMantenimientoDto`, `@CompanyId() companyId: string`
  - Retorna: `Promise<string>` (ID del mantenimiento creado)
  - Flujo: `MantenimientoMapper.toDomain(dto, companyId)` → `createMantenimientoUseCase.execute(companyId, entity)`

  **GET /:id**
  - `@Get(':id')`
  - `@ApiOperation({ summary: 'Obtener mantenimiento por ID' })`
  - `@ApiResponse({ status: 200, type: MantenimientoResponseDto })`
  - `@ApiResponse({ status: 404, description: 'Mantenimiento no encontrado.' })`
  - Parámetros: `@Param('id') id: string`
  - Retorna: `Promise<MantenimientoResponseDto>`
  - Flujo: `getMantenimientoUseCase.execute(id)` → `MantenimientoMapper.toDto(entity)`

  **GET /**
  - `@Get()`
  - `@ApiOperation({ summary: 'Listar mantenimientos por compañía con paginación' })`
  - `@ApiResponse({ status: 200, type: PageDto })`
  - Parámetros: `@Query() pageOptions: PageOptionsDto`, `@CompanyId() companyId: string`
  - Retorna: `Promise<PageDto<MantenimientoResponseDto>>`
  - Flujo: `getMantenimientoListUseCase.execute(companyId, pageOptions)` → `PaginationService.createPageDto(items, count, pageOptions, MantenimientoResponseDto)`

#### `src/infrastructure/modules/mantenimiento.module.ts`

- **Clase**: `MantenimientoModule`
- `@Module({})`
- `imports: [TypeOrmModule.forFeature([MantenimientoEntity])]`
- `controllers: [MantenimientoController]`
- `providers`:
  - `{ provide: MantenimientoRepository, useClass: MantenimientoRepositoryImpl }` — vincula la interfaz de dominio con la implementación
  - `CreateMantenimientoUseCaseImpl`
  - `GetMantenimientoUseCaseImpl`
  - `GetMantenimientoListUseCaseImpl`
- `exports: [MantenimientoRepository]`

#### Registro en módulo raíz

- Agregar `MantenimientoModule` al array `imports` de `DatabaseModule` o `AppModule`
- No omitir este paso — el módulo no estará disponible hasta registrarlo

#### Checklist — Infrastructure Layer

- [ ] DB entity extiende `BaseEntity` y tiene `@Entity('mantenimientos')`
- [ ] Todas las columnas tienen tipo explícito y opciones en `@Column()`
- [ ] Relaciones al final de la clase con `@ManyToOne`/`@JoinColumn`
- [ ] Migración generada después de crear la entidad
- [ ] Repository impl extiende `BaseRepositoryImpl` e implementa la interfaz de dominio
- [ ] `mapToDomain` y `mapToEntity` están completamente implementados
- [ ] Métodos de query usan instancias tipadas de DTO (nunca objetos anónimos)
- [ ] Controlador inyecta casos de uso, nunca repositorios
- [ ] Controlador mapea entidades a DTOs antes de retornarlos
- [ ] Cada endpoint tiene `@ApiOperation`, `@ApiResponse` y `@ApiBearerAuth`
- [ ] Módulo vincula interfaz de dominio con implementación
- [ ] Módulo registrado en el módulo raíz

---

## Resumen de archivos a crear

### Nuevos archivos

| Capa | Ruta |
|------|------|
| Domain | `src/domain/mantenimiento/entities/mantenimiento.domain.ts` |
| Domain | `src/domain/mantenimiento/repositories/mantenimiento.repository.ts` |
| Domain | `src/domain/mantenimiento/use-cases/create-mantenimiento.use-case.ts` |
| Domain | `src/domain/mantenimiento/use-cases/get-mantenimiento.use-case.ts` |
| Domain | `src/domain/mantenimiento/use-cases/get-mantenimiento-list.use-case.ts` |
| Domain | `src/domain/mantenimiento/exceptions/mantenimiento-not-found.exception.ts` |
| Common | `src/common/constants/enums/mantenimiento-tipo.enum.ts` |
| Application | `src/application/mantenimiento/dtos/create-mantenimiento.dto.ts` |
| Application | `src/application/mantenimiento/dtos/mantenimiento-response.dto.ts` |
| Application | `src/application/mantenimiento/mappers/mantenimiento.mapper.ts` |
| Application | `src/application/mantenimiento/use-cases/create-mantenimiento.use-case.impl.ts` |
| Application | `src/application/mantenimiento/use-cases/get-mantenimiento.use-case.impl.ts` |
| Application | `src/application/mantenimiento/use-cases/get-mantenimiento-list.use-case.impl.ts` |
| Infrastructure | `src/infrastructure/database/entities/mantenimiento.entity.ts` |
| Infrastructure | `src/infrastructure/database/repositories/mantenimiento.repository.impl.ts` |
| Infrastructure | `src/infrastructure/adapters/api/controllers/mantenimiento.controller.ts` |
| Infrastructure | `src/infrastructure/modules/mantenimiento.module.ts` |
| Infrastructure | `src/infrastructure/database/migrations/{timestamp}-CreateMantenimientosTable.ts` (generado) |

### Archivos a modificar

| Archivo | Cambio |
|---------|--------|
| `src/common/constants/enums/redis-key.enum.ts` | Agregar `MANTENIMIENTO = 'mantenimiento'` |
| `src/infrastructure/modules/app.module.ts` o `database.module.ts` | Agregar `MantenimientoModule` a `imports` |

---

## Convenciones de nomenclatura aplicadas

| Elemento | Valor |
|----------|-------|
| Nombre del módulo (singular) | `mantenimiento` |
| Tabla DB (plural, snake_case) | `mantenimientos` |
| Prefijo de clase | `Mantenimiento` |
| Enum de tipo | `MantenimientoTipoEnum` |
| Valores del enum | `PREVENTIVO`, `CORRECTIVO`, `GARANTIA` |
| Redis key | `mantenimiento` |
| Ruta HTTP | `/mantenimientos` |
| Comentarios | En español |
| Identificadores de código | En inglés (`companyId`, `findById`, etc.) |

---

## Notas importantes

1. **Orden estricto**: No iniciar el Step 2 hasta que todos los archivos del Step 1 estén creados y revisados. No iniciar el Step 3 hasta que el Step 2 esté completo. La arquitectura limpia requiere que las capas internas existan antes de que las externas las implementen.

2. **`companyId` en el controlador**: El campo `companyId` no va en el `CreateMantenimientoDto` — se extrae del JWT con el decorator `@CompanyId()` y se pasa directamente al mapper y al caso de uso. Esto garantiza que el cliente no pueda falsificar la compañía propietaria.

3. **Tipo `fecha` en TypeORM**: Usar `type: 'date'` en la entidad de base de datos para almacenar solo la fecha sin componente de tiempo. Si se requiere timestamp completo, cambiar a `type: 'timestamp'`.

4. **Cache invalidation**: El caso de uso `create` invalida todas las entradas de caché con prefijo `MANTENIMIENTO` usando `removeCacheByPartialKey`. Esto limpia tanto las entradas individuales como las listas paginadas.

5. **Migración**: Correr `npm run typeorm:migration:generate -- -n CreateMantenimientosTable` después de crear `mantenimiento.entity.ts`. Revisar el archivo generado antes de ejecutar `npm run typeorm:migration:run`.

6. **Sin value objects propios**: Los campos de este módulo son primitivos simples (string, number, Date, enum) que no requieren value objects dedicados. Si en el futuro se agrega validación compleja (e.g., `costo` no puede ser negativo con reglas de negocio adicionales), crear `src/domain/mantenimiento/value-objects/costo.value-object.ts`.

7. **Verificación final**:
   - `npm run start:dev` — sin errores de compilación TypeScript
   - `npm run lint` — sin errores de lint
   - Swagger en `/api/docs` — los tres endpoints aparecen bajo la tag `Mantenimientos`
   - Confirmar que `MantenimientoModule` aparece en `DatabaseModule`/`AppModule` imports
