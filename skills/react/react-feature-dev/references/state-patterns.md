# Patrones de Estado

## Regla de oro: ¿Qué tipo de estado necesito?

```
¿Es dato que viene del servidor?     → React Query (useApiGet / useApiSend)
¿Es estado de UI compartido global?  → Zustand store
¿Es estado local del componente?     → useState / useReducer
¿Es filtro/paginación en la URL?     → nuqs (useQueryState)
¿Es estado de formulario?            → React Hook Form
```

**No duplicar estado del servidor en Zustand.** React Query es la fuente de verdad para datos de API.

---

## 1. React Query — Hooks personalizados del proyecto

### `useApiGet<T>` — Consultas GET

```typescript
import { useApiGet } from '@/common/react-query/useApi'

// Consulta básica
const { data, isLoading, error, refetch } = useApiGet<SupplierResponse[]>({
  key: ['suppliers'],                    // QueryKey — debe ser único y descriptivo
  fn: SupplierService.getAll,            // Función que retorna Promise<T>
})

// Con parámetros dinámicos (el key debe incluir los parámetros)
const { data } = useApiGet<SupplierResponse[]>({
  key: ['suppliers', companyId, page],
  fn: () => SupplierService.getAll({ companyId, page }),
  options: {
    enabled: Boolean(companyId),         // Solo ejecutar si companyId existe
    staleTime: 5 * 60 * 1000,           // Cache válido por 5 minutos
  },
})

// Consulta individual por ID
const { data: supplier } = useApiGet<SupplierResponse>({
  key: ['supplier', supplierId],
  fn: () => SupplierService.getById(supplierId),
  options: { enabled: Boolean(supplierId) },
})
```

### `useApiSend<T>` — Mutaciones POST/PUT/PATCH/DELETE

```typescript
import { useApiSend } from '@/common/react-query/useApi'

// Crear
const { mutate: createSupplier, isPending: isCreating } = useApiSend<CreateSupplierDto>({
  fn: SupplierService.create,
  success: (data) => {
    toast.success('Proveedor creado')
    setDialogOpen(false)
  },
  error: (err) => toast.error('Error al crear proveedor'),
  invalidateKey: [{ queryKey: ['suppliers'] }],   // Invalidar cache tras éxito
})

// Actualizar (necesita el ID + datos)
const { mutate: updateSupplier } = useApiSend<{ id: string; data: UpdateSupplierDto }>({
  fn: ({ id, data }) => SupplierService.update(id, data),
  success: () => toast.success('Proveedor actualizado'),
  invalidateKey: [
    { queryKey: ['suppliers'] },        // Invalidar lista
    { queryKey: ['supplier', id] },     // Invalidar detalle
  ],
})

// Eliminar
const { mutate: deleteSupplier, isPending: isDeleting } = useApiSend<string>({
  fn: (id: string) => SupplierService.delete(id),
  success: () => toast.success('Proveedor eliminado'),
  error: () => toast.error('No se pudo eliminar el proveedor'),
  invalidateKey: [{ queryKey: ['suppliers'] }],
})

// Uso en JSX:
<Button onClick={() => createSupplier(formData)} disabled={isCreating}>
  {isCreating ? 'Guardando...' : 'Guardar'}
</Button>
```

### `useApiInfinite<T>` — Listas infinitas

```typescript
import { useApiInfinite } from '@/common/react-query/useApi';

const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useApiInfinite<PageDto<SupplierResponse>>({
  key: ['suppliers-infinite', search],
  fn: ({ pageParam = 1 }) => SupplierService.getAll({ page: pageParam, q: search }),
  options: {
    getNextPageParam: (lastPage) => (lastPage.meta.hasNextPage ? lastPage.meta.page + 1 : undefined),
  },
});

// Los datos están en:
const suppliers = data?.pages.flatMap((page) => page.data) ?? [];
```

---

## 2. QueryKey — Convenciones

```typescript
// Patrón: [recurso, ...parámetros]
['suppliers'][('suppliers', companyId)][('suppliers', companyId, page, search)][('supplier', id)]; // lista sin filtros // lista filtrada por empresa // lista con paginación y búsqueda // elemento individual

// Al invalidar, invalidar el nivel correcto:
invalidateKey: [{ queryKey: ['suppliers'] }]; // invalida todas las consultas que empiecen con 'suppliers'
invalidateKey: [{ queryKey: ['supplier', id] }]; // invalida solo la consulta de este ID
```

---

## 3. Zustand — Stores existentes

### `useAuthStore` — Estado de autenticación

```typescript
import { useAuthStore } from '@/store/auth.store';

// Leer estado:
const user = useAuthStore((state) => state.user);
const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
const isLoading = useAuthStore((state) => state.isLoading);

// Acciones:
const login = useAuthStore((state) => state.login);
const logout = useAuthStore((state) => state.logout);

// Uso:
await login(email, password); // retorna boolean
logout();
```

### `useAppStore` — Estado de la aplicación

```typescript
import { useAppStore } from '@/store/app.store';

// Leer estado:
const selectedLicense = useAppStore((state) => state.selectedLicense);
const selectedCompany = useAppStore((state) => state.selectedCompany);
const permissions = useAppStore((state) => state.permissions);
const companyId = useAppStore((state) => state.selectedCompany?.id);

// Acciones:
const setSelectedCompany = useAppStore((state) => state.setSelectedCompany);
const setPermissions = useAppStore((state) => state.setPermissions);
const clearAll = useAppStore((state) => state.clearAll);
```

---

## 4. Crear una nueva Zustand store (cuando sea necesario)

Solo crear una nueva store de Zustand si se necesita estado global de UI que es
compartido por múltiples componentes no relacionados (no padre-hijo).

```typescript
// src/store/[feature]-ui.store.ts
import { create } from 'zustand';

// Interfaz de estado
interface SupplierUIState {
  selectedSupplierId: string | null;
  isFormOpen: boolean;
  formMode: 'create' | 'edit';
}

// Interfaz de acciones
interface SupplierUIActions {
  openCreateForm: () => void;
  openEditForm: (id: string) => void;
  closeForm: () => void;
  selectSupplier: (id: string | null) => void;
}

type SupplierUIStore = SupplierUIState & SupplierUIActions;

// Crear la store
export const useSupplierUIStore = create<SupplierUIStore>((set) => ({
  // Estado inicial
  selectedSupplierId: null,
  isFormOpen: false,
  formMode: 'create',

  // Acciones
  openCreateForm: () => set({ isFormOpen: true, formMode: 'create', selectedSupplierId: null }),
  openEditForm: (id) => set({ isFormOpen: true, formMode: 'edit', selectedSupplierId: id }),
  closeForm: () => set({ isFormOpen: false, selectedSupplierId: null }),
  selectSupplier: (id) => set({ selectedSupplierId: id }),
}));
```

### Con persistencia (solo para estado que debe sobrevivir recargas):

```typescript
import { create } from 'zustand';
import { createJSONStorage, persist } from 'zustand/middleware';

export const useSupplierPrefsStore = create<SupplierPrefsStore>()(
  persist(
    (set) => ({
      preferredView: 'table' as 'table' | 'grid',
      setPreferredView: (view) => set({ preferredView: view }),
    }),
    {
      name: 'supplier-prefs', // Clave en localStorage
      storage: createJSONStorage(() => localStorage),
    },
  ),
);
```

---

## 5. Patrones de selección en componentes (Zustand)

```typescript
// ✓ Siempre seleccionar el mínimo necesario (evita re-renders innecesarios)
const companyId = useAppStore((state) => state.selectedCompany?.id)  // solo el ID
// ❌ const company = useAppStore((state) => state.selectedCompany)   // todo el objeto

// ✓ Separar selectores si necesitas múltiples valores
const permissions = useAppStore((state) => state.permissions)
const licenseId = useAppStore((state) => state.selectedLicense?.id)

// ❌ No combinar en un objeto (crea nuevo objeto en cada render)
const { permissions, license } = useAppStore((state) => ({
  permissions: state.permissions,
  license: state.selectedLicense,
}))
```

---

## 6. Estado de formulario con React Hook Form

```typescript
// No usar useState para campos de formulario — siempre React Hook Form
import { zodResolver } from '@hookform/resolvers/zod';

import { useForm, useWatch } from 'react-hook-form';

const form = useForm<CreateSupplierDto>({
  resolver: zodResolver(CreateSupplierSchema),
  defaultValues: {
    name: '',
    email: '',
    type: 'COMPANY',
  },
  mode: 'onBlur', // validar al perder foco (no en cada keystroke)
});

// Observar un campo para lógica condicional:
const supplierType = useWatch({ control: form.control, name: 'type' });

// Reset al abrir dialog:
useEffect(() => {
  if (supplier) {
    form.reset(supplier); // modo edición: cargar valores
  } else {
    form.reset(); // modo creación: valores por defecto
  }
}, [supplier, form]);
```

---

## 7. Estado en URL con nuqs

```typescript
import { parseAsInteger, parseAsString, useQueryState } from 'nuqs';

// Para filtros y paginación (se persisten en la URL)
const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1));
const [search, setSearch] = useQueryState('q', parseAsString.withDefault(''));
const [status, setStatus] = useQueryState('status'); // null si no está en URL

// El key de React Query debe incluir estos valores:
const { data } = useApiGet({
  key: ['suppliers', page, search, status],
  fn: () => SupplierService.getAll({ page, q: search, status: status ?? undefined }),
});
```

---

## Checklist de estado

- [ ] ¿Los datos del servidor están en React Query (no en useState ni Zustand)?
- [ ] ¿El QueryKey incluye todos los parámetros dinámicos de la consulta?
- [ ] ¿Las mutaciones invalidan el cache correspondiente?
- [ ] ¿El estado de UI local (dialog open, selected item) está en useState?
- [ ] ¿Se creó una Zustand store solo si el estado es realmente global?
- [ ] ¿Los selectores de Zustand obtienen el mínimo necesario (no todo el objeto)?
- [ ] ¿Los formularios usan React Hook Form (no useState por campo)?
- [ ] ¿Los filtros y paginación usan nuqs para persistir en la URL?
