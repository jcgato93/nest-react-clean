# Patrones de Modelos y Tipos

## Organización en `src/domain/`

Cada dominio tiene su propia carpeta bajo `src/domain/`:

```
src/domain/[feature]/
├── [feature].types.ts          # Tipos, interfaces, schemas Zod principales
├── [feature].enum.ts           # Enumeraciones del dominio (si aplica)
└── dto/
    ├── create-[feature].dto.ts
    └── update-[feature].dto.ts
```

---

## 1. Interfaces de respuesta del servidor

```typescript
// src/domain/supplier/supplier.types.ts

// Respuesta que llega del servidor (snake_case → camelCase según el backend)
export interface SupplierResponse {
  id: string;
  name: string;
  email: string;
  phone?: string;
  type: SupplierType;
  active: boolean;
  companyId: string;
  createdAt: string;
  updatedAt: string;
}

// Respuesta paginada (patrón reutilizable)
export interface PageDto<T> {
  data: T[];
  meta: {
    page: number;
    take: number;
    itemCount: number;
    pageCount: number;
    hasPreviousPage: boolean;
    hasNextPage: boolean;
  };
}
```

---

## 2. Schemas Zod para formularios (con tipos inferidos)

```typescript
// src/domain/supplier/supplier.types.ts (continuación)
import { z } from 'zod';

// Schema de creación — base de verdad para validación de formulario
export const CreateSupplierSchema = z.object({
  name: z.string().min(1, 'El nombre es requerido').max(100),
  email: z.email('Email inválido'),
  phone: z.string().optional(),
  type: z.enum(['INDIVIDUAL', 'COMPANY'], {
    required_error: 'El tipo es requerido',
  }),
});

// Tipo inferido del schema — usar en los componentes
export type CreateSupplierDto = z.infer<typeof CreateSupplierSchema>;

// Schema de actualización — todos los campos opcionales excepto los identificadores
export const UpdateSupplierSchema = CreateSupplierSchema.partial();
export type UpdateSupplierDto = z.infer<typeof UpdateSupplierSchema>;
```

### Reglas para Zod schemas:

- Siempre agregar mensajes de error en español
- Usar `z.email()` para emails (no `z.string().email()` — en zod v4 hay atajo)
- Usar `.optional()` para campos no requeridos
- Usar `.partial()` para DTOs de actualización
- Inferir tipos con `z.infer<>` — nunca duplicar la definición manualmente

---

## 3. Enumeraciones

```typescript
// src/domain/supplier/supplier.enum.ts
export enum SupplierType {
  INDIVIDUAL = 'INDIVIDUAL',
  COMPANY = 'COMPANY',
}

// Para constantes de UI (labels de select, badges):
export const SUPPLIER_TYPE_LABELS: Record<SupplierType, string> = {
  [SupplierType.INDIVIDUAL]: 'Persona natural',
  [SupplierType.COMPANY]: 'Empresa',
};
```

---

## 4. DTOs separados (para DTOs complejos)

```typescript
// src/domain/supplier/dto/create-supplier.dto.ts
// Usar cuando el DTO es suficientemente complejo para archivo propio
import { z } from 'zod';

export const CreateSupplierSchema = z.object({
  name: z.string().min(1, 'Requerido').max(100, 'Máximo 100 caracteres'),
  email: z.email('Email inválido'),
  phone: z
    .string()
    .regex(/^\+?[0-9]{7,15}$/, 'Teléfono inválido')
    .optional(),
  type: z.enum(['INDIVIDUAL', 'COMPANY']),
  address: z
    .object({
      street: z.string().min(1),
      city: z.string().min(1),
      country: z.string().length(2),
    })
    .optional(),
});

export type CreateSupplierDto = z.infer<typeof CreateSupplierSchema>;
```

---

## 5. Tipos utilitarios de paginación

```typescript
// src/domain/common/pagination.types.ts (archivo existente o a crear si no existe)
export interface PageOptionsDto {
  page?: number;
  take?: number;
  order?: 'ASC' | 'DESC';
  q?: string; // query de búsqueda
}
```

---

## 6. Tipos de formulario vs tipos de API

```typescript
// Patrón: separar la forma del formulario de lo que se envía al servidor

// Lo que el formulario maneja (puede ser diferente al DTO)
interface SupplierFormValues {
  name: string;
  email: string;
  countryId: string; // El formulario usa el ID del país
  departmentId: string; // El formulario usa el ID del departamento
}

// Lo que se envía al servidor (procesado en onSubmit)
interface CreateSupplierDto {
  name: string;
  email: string;
  locationCode: string; // El server quiere un código combinado
}

// En el componente:
const onSubmit = (values: SupplierFormValues) => {
  const dto: CreateSupplierDto = {
    name: values.name,
    email: values.email,
    locationCode: `${values.countryId}-${values.departmentId}`,
  };
  mutate(dto);
};
```

---

## 7. Tipos para el store de Zustand

```typescript
// src/store/supplier.store.ts (si se necesita estado global de UI)
import { create } from 'zustand';

interface SupplierUIState {
  selectedSupplierId: string | null;
  isFormOpen: boolean;
}

interface SupplierUIActions {
  selectSupplier: (id: string | null) => void;
  toggleForm: (open?: boolean) => void;
}

type SupplierUIStore = SupplierUIState & SupplierUIActions;

export const useSupplierUIStore = create<SupplierUIStore>((set) => ({
  selectedSupplierId: null,
  isFormOpen: false,

  selectSupplier: (id) => set({ selectedSupplierId: id }),
  toggleForm: (open) =>
    set((state) => ({
      isFormOpen: open !== undefined ? open : !state.isFormOpen,
    })),
}));
```

> **Nota:** Usar Zustand para estado global de **UI** (dialogs abiertos, elementos seleccionados).
> Para estado del servidor, usar React Query (no duplicar datos en Zustand).

---

## 8. Tipos de respuesta con errores de API

```typescript
// En el componente:
import type { AxiosError } from 'axios';

// Para manejar errores tipados del servidor
export interface ApiError {
  statusCode: number;
  message: string | string[];
  error: string;
}

const { mutate } = useApiSend({
  fn: SupplierService.create,
  error: (error: AxiosError<ApiError>) => {
    const msg = Array.isArray(error.response?.data?.message)
      ? error.response.data.message.join(', ')
      : (error.response?.data?.message ?? 'Error desconocido');
    toast.error(msg);
  },
});
```

---

## Checklist de modelos

- [ ] ¿Las interfaces de respuesta reflejan exactamente lo que devuelve el servidor?
- [ ] ¿Los schemas Zod tienen mensajes de error en español?
- [ ] ¿Los tipos se infieren del schema (`z.infer<>`) en vez de duplicarse?
- [ ] ¿Las enumeraciones tienen su archivo separado `.enum.ts`?
- [ ] ¿Los DTOs de actualización usan `.partial()` del schema base?
- [ ] ¿Los tipos de paginación reutilizan `PageDto<T>` y `PageOptionsDto`?

---

## Anti-patrones a evitar

```typescript
// ❌ NO: Duplicar tipos que ya existen en domain/
interface User {  // ya existe en src/domain/user/
  id: string
  email: string
}

// ✓ SÍ: Importar del dominio
import type { User } from '@/domain/user/user.types'

// ❌ NO: Usar `any` en interfaces
interface SupplierResponse {
  data: any
  meta: any
}

// ✓ SÍ: Tipado explícito completo
interface SupplierResponse {
  data: SupplierItem[]
  meta: PaginationMeta
}

// ❌ NO: Redefinir manualmente el tipo del schema
const CreateSupplierSchema = z.object({ name: z.string() })
interface CreateSupplierDto {  // duplicado innecesario
  name: string
}

// ✓ SÍ: Inferir del schema
export type CreateSupplierDto = z.infer<typeof CreateSupplierSchema>
```
