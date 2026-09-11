# Patrones de Páginas y Routing

## Estructura de una página CRUD completa

```
src/pages/[feature]/
├── [Feature]Page.tsx           # Contenedor principal (datos + estado de UI)
└── components/
    ├── [Feature]List.tsx       # Tabla/lista
    ├── [Feature]Form.tsx       # Formulario de creación/edición
    └── [Feature]Dialog.tsx     # Dialog wrapper (opcional)
```

---

## 1. Página contenedora (patrón estándar)

```typescript
// src/pages/suppliers/SuppliersPage.tsx
import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Plus } from 'lucide-react'
import { useApiGet, useApiSend } from '@/common/react-query/useApi'
import { SupplierService } from '@/services/supplier.service'
import { useAppStore } from '@/store/app.store'
import { toast } from 'sonner'
import SupplierList from './components/SupplierList'
import SupplierDialog from './components/SupplierDialog'
import type { SupplierResponse } from '@/domain/supplier/supplier.types'

const SuppliersPage = () => {
  const [isDialogOpen, setIsDialogOpen] = useState(false)
  const [selectedSupplier, setSelectedSupplier] = useState<SupplierResponse | undefined>()

  const companyId = useAppStore((state) => state.selectedCompany?.id)

  const { data: suppliers, isLoading } = useApiGet<SupplierResponse[]>({
    key: ['suppliers', companyId],
    fn: SupplierService.getAll,
    options: { enabled: Boolean(companyId) },
  })

  const { mutate: deleteSupplier, isPending: isDeleting } = useApiSend<string>({
    fn: SupplierService.delete,
    success: () => toast.success('Proveedor eliminado'),
    error: () => toast.error('Error al eliminar proveedor'),
    invalidateKey: [{ queryKey: ['suppliers'] }],
  })

  const handleEdit = (supplier: SupplierResponse) => {
    setSelectedSupplier(supplier)
    setIsDialogOpen(true)
  }

  const handleCreate = () => {
    setSelectedSupplier(undefined)
    setIsDialogOpen(true)
  }

  const handleDialogClose = () => {
    setIsDialogOpen(false)
    setSelectedSupplier(undefined)
  }

  return (
    <div className="space-y-4">
      {/* Header de sección */}
      <div className="flex items-center justify-between">
        <div>
          <h2 className="text-2xl font-bold tracking-tight">Proveedores</h2>
          <p className="text-muted-foreground">
            Gestión de proveedores de la empresa
          </p>
        </div>
        <Button onClick={handleCreate}>
          <Plus className="mr-2 h-4 w-4" />
          Nuevo proveedor
        </Button>
      </div>

      {/* Contenido */}
      <SupplierList
        suppliers={suppliers ?? []}
        isLoading={isLoading}
        onEdit={handleEdit}
        onDelete={(id) => deleteSupplier(id)}
        isDeleting={isDeleting}
      />

      {/* Dialog de creación/edición */}
      <SupplierDialog
        open={isDialogOpen}
        onOpenChange={handleDialogClose}
        supplier={selectedSupplier}
      />
    </div>
  )
}

export default SuppliersPage
```

---

## 2. Registrar ruta en `AppRouter.tsx`

```typescript
// src/AppRouter.tsx — Agregar dentro de las rutas del dashboard

// 1. Import lazy (al inicio del archivo, junto con los otros lazy imports):
const SuppliersPage = lazy(() => import('@/pages/suppliers/SuppliersPage'))

// 2. Agregar la ruta en la sección correspondiente del router:
{
  path: 'administrative/suppliers',
  element: (
    <Suspense fallback={<LoaderSpin />}>
      <SuppliersPage />
    </Suspense>
  ),
},
```

### Estructura del router (referencia):

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Navigate to="/auth/login" replace />,
  },
  {
    path: '/auth',
    element: <PublicRoute><AuthLayout /></PublicRoute>,
    children: [...rutas de autenticación],
  },
  {
    path: '/dashboard',
    element: <ProtectedRoute><DashboardLayout /></ProtectedRoute>,
    children: [
      // Aquí van las rutas del dashboard
      { path: 'administrative/suppliers', element: <Suspense><SuppliersPage /></Suspense> },
    ],
  },
])
```

---

## 3. Agregar título de ruta

Si la página necesita un título en el topbar, agregar en el mapa de títulos:

```typescript
// src/common/constants/topbar-search-routes.ts (o el archivo de títulos)
// Buscar el objeto de rutas y agregar:

export const ROUTE_TITLES: Record<string, string> = {
  // ... títulos existentes ...
  '/dashboard/administrative/suppliers': 'Proveedores',
};
```

---

## 4. Página con tabs (múltiples vistas)

```typescript
// Para páginas con varias secciones/tabs
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { useSearchParams } from 'react-router'

const FixedAssetsPage = () => {
  const [searchParams, setSearchParams] = useSearchParams()
  const activeTab = searchParams.get('tab') ?? 'property'

  const handleTabChange = (value: string) => {
    setSearchParams({ tab: value })
  }

  return (
    <Tabs value={activeTab} onValueChange={handleTabChange}>
      <TabsList>
        <TabsTrigger value="property">Propiedad, Planta y Equipo</TabsTrigger>
        <TabsTrigger value="intangible">Activos Intangibles</TabsTrigger>
      </TabsList>
      <TabsContent value="property">
        <PropertyEquipmentList />
      </TabsContent>
      <TabsContent value="intangible">
        <IntangibleAssetsList />
      </TabsContent>
    </Tabs>
  )
}
```

---

## 5. Página de detalle (con parámetro de ruta)

```typescript
// src/pages/suppliers/SupplierDetailPage.tsx
import { useParams } from 'react-router'
import { useApiGet } from '@/common/react-query/useApi'
import { SupplierService } from '@/services/supplier.service'
import { LoaderSpin } from '@/components/ui/loader-spin'

const SupplierDetailPage = () => {
  const { id } = useParams<{ id: string }>()

  const { data: supplier, isLoading, error } = useApiGet({
    key: ['supplier', id],
    fn: () => SupplierService.getById(id!),
    options: { enabled: Boolean(id) },
  })

  if (isLoading) return <LoaderSpin />
  if (error || !supplier) return <div>Error al cargar el proveedor</div>

  return (
    <div>
      <h1>{supplier.name}</h1>
      {/* contenido del detalle */}
    </div>
  )
}

export default SupplierDetailPage

// En AppRouter.tsx:
{ path: 'administrative/suppliers/:id', element: <SupplierDetailPage /> },
```

---

## 6. Navegación programática

```typescript
import { ROUTE_PATHS } from '@/common/constants/route-paths';

import { useNavigate } from 'react-router';

const SupplierForm = () => {
  const navigate = useNavigate();

  const { mutate } = useApiSend({
    fn: SupplierService.create,
    success: (data) => {
      toast.success('Proveedor creado');
      navigate(`/dashboard/administrative/suppliers/${data.id}`);
    },
  });
};
```

---

## 7. Página con búsqueda y filtros

```typescript
import { useQueryState } from 'nuqs'  // ← gestión de estado en URL

const SuppliersPage = () => {
  const [search, setSearch] = useQueryState('q')
  const [page, setPage] = useQueryState('page', { defaultValue: '1' })

  const { data } = useApiGet({
    key: ['suppliers', search, page],
    fn: () => SupplierService.getAll({
      q: search ?? undefined,
      page: Number(page),
      take: 10,
    }),
  })

  return (
    <div className="space-y-4">
      <Input
        placeholder="Buscar proveedores..."
        value={search ?? ''}
        onChange={(e) => setSearch(e.target.value || null)}
      />
      {/* lista y paginación */}
    </div>
  )
}
```

---

## 8. Breadcrumb (si la página está anidada)

```typescript
import {
  Breadcrumb, BreadcrumbItem, BreadcrumbLink,
  BreadcrumbList, BreadcrumbSeparator,
} from '@/components/ui/breadcrumb'
import { Link } from 'react-router'

const SupplierDetailPage = () => (
  <div className="space-y-4">
    <Breadcrumb>
      <BreadcrumbList>
        <BreadcrumbItem>
          <BreadcrumbLink asChild>
            <Link to="/dashboard/administrative/suppliers">Proveedores</Link>
          </BreadcrumbLink>
        </BreadcrumbItem>
        <BreadcrumbSeparator />
        <BreadcrumbItem>Detalle</BreadcrumbItem>
      </BreadcrumbList>
    </Breadcrumb>
    {/* contenido */}
  </div>
)
```

---

## Checklist de páginas

- [ ] ¿La página usa lazy loading en `AppRouter.tsx`?
- [ ] ¿La ruta está registrada dentro de la protección de `ProtectedRoute`?
- [ ] ¿El título de la ruta está registrado en el mapa de títulos?
- [ ] ¿La página maneja los estados de `isLoading` y `error`?
- [ ] ¿El estado de UI (dialogs, selecciones) está en `useState` local (no en Zustand)?
- [ ] ¿La navegación usa `useNavigate()` y `Link` de `react-router`?
- [ ] ¿Los parámetros de búsqueda/filtros usan `nuqs` (`useQueryState`)?
- [ ] ¿El `key` de React Query incluye todos los parámetros dinámicos?
