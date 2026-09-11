# Patrones de Servicios y API

## Arquitectura de la capa de servicios

```
src/common/axios.ts          → Definición centralizada de endpoints
src/services/api.service.ts  → Singleton de Axios con interceptores
src/services/*.service.ts    → Clases estáticas por dominio
```

---

## 1. Definir endpoints en `src/common/axios.ts`

```typescript
// src/common/axios.ts (agregar a la sección correspondiente)

// Patrón existente en el archivo:
export const API_ENDPOINTS = {
  AUTH: {
    LOGIN: '/auth/login',
    REFRESH: '/auth/refresh-token',
    // ...
  },
  USERS: {
    GET_BY_LICENSE: (subscriptionId: string) => `/users/license/${subscriptionId}`,
    INVITE: '/users/invite',
    UPDATE: (userId: string) => `/users/${userId}`,
    DELETE: (userId: string) => `/users/${userId}`,
  },
  // Agregar nueva sección:
  SUPPLIERS: {
    GET_ALL: '/suppliers',
    GET_BY_ID: (id: string) => `/suppliers/${id}`,
    CREATE: '/suppliers',
    UPDATE: (id: string) => `/suppliers/${id}`,
    DELETE: (id: string) => `/suppliers/${id}`,
  },
};
```

### Reglas para endpoints:

- Endpoints estáticos: string literal `'/ruta'`
- Endpoints dinámicos: función `(param: string) => \`/ruta/${param}\``
- Agrupar por dominio en un objeto nombrado con UPPER_SNAKE_CASE
- Mantener coherencia con la nomenclatura del backend

---

## 2. Crear el servicio en `src/services/[feature].service.ts`

```typescript
// src/services/supplier.service.ts
import { API_ENDPOINTS } from '@/common/axios';
import type { PageDto, PageOptionsDto } from '@/domain/common/pagination.types';
import type { CreateSupplierDto, SupplierResponse, UpdateSupplierDto } from '@/domain/supplier/supplier.types';
import { apiService } from '@/services/api.service';

export class SupplierService {
  // GET — Lista paginada
  static async getAll(options?: PageOptionsDto): Promise<PageDto<SupplierResponse>> {
    const response = await apiService.get<PageDto<SupplierResponse>>(API_ENDPOINTS.SUPPLIERS.GET_ALL, {
      params: options,
    });
    return response.data;
  }

  // GET — Por ID
  static async getById(id: string): Promise<SupplierResponse> {
    const response = await apiService.get<SupplierResponse>(API_ENDPOINTS.SUPPLIERS.GET_BY_ID(id));
    return response.data;
  }

  // POST — Crear
  static async create(data: CreateSupplierDto): Promise<SupplierResponse> {
    const response = await apiService.post<SupplierResponse>(API_ENDPOINTS.SUPPLIERS.CREATE, data);
    return response.data;
  }

  // PATCH — Actualizar parcialmente
  static async update(id: string, data: UpdateSupplierDto): Promise<SupplierResponse> {
    const response = await apiService.patch<SupplierResponse>(API_ENDPOINTS.SUPPLIERS.UPDATE(id), data);
    return response.data;
  }

  // DELETE — Eliminar
  static async delete(id: string): Promise<void> {
    await apiService.delete(API_ENDPOINTS.SUPPLIERS.DELETE(id));
  }
}
```

### Reglas para servicios:

1. **Clase estática** — todos los métodos son `static`
2. **Tipado explícito** en parámetros y retorno
3. **Sin lógica de negocio** — solo llamadas HTTP
4. **Retornar `response.data`** — no el objeto Axios completo
5. **Importar desde `@/services/api.service`** (el singleton, no Axios directamente)

---

## 3. Cómo funciona `ApiService` (referencia, no modificar)

```typescript
// src/services/api.service.ts — Singleton con interceptores
// Interceptor de REQUEST (automático):
// ✓ Agrega Authorization: Bearer [token] desde localStorage
// ✓ Agrega x-api-companyid desde app-store
// ✓ Agrega x-api-subscriptionid desde app-store
// Interceptor de RESPONSE (automático):
// ✓ En 401: intenta refresh del token
// ✓ Reintenta la petición original con el nuevo token
// ✓ En fallo de refresh: redirige a login
// Uso correcto en servicios:
import { apiService } from '@/services/api.service';

// apiService.get(), apiService.post(), apiService.patch(), apiService.delete()
```

---

## 4. Servicio con parámetros del store (companyId, subscriptionId)

```typescript
// Cuando el endpoint necesita el ID de la compañía/licencia actual
import { useAppStore } from '@/store/app.store'

// En el SERVICIO (capa estática) — Acceder directamente al store:
import { useAppStore } from '@/store/app.store'

export class FixedAssetService {
  static async getByCompany(): Promise<FixedAssetResponse[]> {
    // El interceptor de Axios ya agrega x-api-companyid automáticamente
    // El backend lo lee desde el header, no como parámetro de URL
    const response = await apiService.get<FixedAssetResponse[]>(
      API_ENDPOINTS.FIXED_ASSETS.GET_ALL,
    )
    return response.data
  }

  // Si se necesita en la URL (no en header):
  static async getByCompanyId(companyId: string): Promise<FixedAssetResponse[]> {
    const response = await apiService.get<FixedAssetResponse[]>(
      API_ENDPOINTS.FIXED_ASSETS.BY_COMPANY(companyId),
    )
    return response.data
  }
}

// En el COMPONENTE — Obtener del store y pasar al hook:
const companyId = useAppStore((state) => state.selectedCompany?.id)

const { data } = useApiGet({
  key: ['fixed-assets', companyId],
  fn: () => FixedAssetService.getByCompanyId(companyId!),
  options: { enabled: Boolean(companyId) },
})
```

---

## 5. Servicio con paginación

```typescript
export class UserService {
  static async getUsersByLicenseId(
    subscriptionId: string,
    options: PageOptionsDto,
  ): Promise<PageDto<UserByLicenseResponse>> {
    const response = await apiService.get<PageDto<UserByLicenseResponse>>(
      API_ENDPOINTS.USERS.GET_BY_LICENSE(subscriptionId),
      { params: options },
    );
    return response.data;
  }
}

// En componente:
const { data } = useApiGet<PageDto<UserByLicenseResponse>>({
  key: ['users', subscriptionId, page, limit],
  fn: () => UserService.getUsersByLicenseId(subscriptionId, { page, limit }),
  options: { enabled: Boolean(subscriptionId) },
});
```

---

## 6. Servicio con upload de archivos

```typescript
export class DocumentService {
  static async uploadFile(formData: FormData): Promise<DocumentResponse> {
    const response = await apiService.post<DocumentResponse>(API_ENDPOINTS.DOCUMENTS.UPLOAD, formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    });
    return response.data;
  }
}
```

---

## Checklist antes de crear un servicio

- [ ] ¿Los endpoints están definidos en `src/common/axios.ts`?
- [ ] ¿La clase usa métodos estáticos?
- [ ] ¿Los tipos de retorno están explícitamente declarados?
- [ ] ¿Se importa `apiService` desde `@/services/api.service`?
- [ ] ¿El servicio retorna `response.data` y no el objeto Axios?
- [ ] ¿Los métodos tienen un solo nivel de responsabilidad (solo HTTP)?
- [ ] ¿Existe ya un servicio similar que pueda extenderse?

---

## Anti-patrones a evitar

```typescript
// ❌ NO: Usar axios directamente en componentes
import axios from 'axios'
const { data } = await axios.get('/api/suppliers')

// ✓ SÍ: Usar el servicio desde el hook
const { data } = useApiGet({ fn: SupplierService.getAll, ... })

// ❌ NO: Lógica de negocio en el servicio
static async createAndNotify(data: CreateSupplierDto) {
  const result = await apiService.post(...)
  showNotification('created')  // ← lógica de UI en el servicio
  return result.data
}

// ✓ SÍ: La notificación va en el componente, en el callback `success`
const { mutate } = useApiSend({
  fn: SupplierService.create,
  success: () => toast.success('Creado'),  // ← UI aquí
})

// ❌ NO: Hardcodear URLs en servicios
const response = await apiService.get('/suppliers')

// ✓ SÍ: Siempre desde API_ENDPOINTS
const response = await apiService.get(API_ENDPOINTS.SUPPLIERS.GET_ALL)
```
