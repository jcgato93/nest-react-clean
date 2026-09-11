# Patrones de Componentes

## Anatomía de un componente funcional

```typescript
// src/components/functional/select-supplier.tsx
import { SelectList } from '@/components/helisa/select-list'
import { SelectSkeleton } from '@/components/ui/skeleton'
import { useApiGet } from '@/common/react-query/useApi'
import { SupplierService } from '@/services/supplier.service'
import type { SupplierResponse } from '@/domain/supplier/supplier.types'

interface SelectSupplierProps {
  value: string | null | undefined
  onChange: (value: string | null, item: SupplierResponse | null) => void
  disabled?: boolean
}

const SelectSupplier = ({ value, onChange, disabled }: SelectSupplierProps) => {
  const { data, isLoading, error } = useApiGet<SupplierResponse[]>({
    key: ['suppliers'],
    fn: SupplierService.getAll,
  })

  if (isLoading) return <SelectSkeleton labelText="Cargando proveedores..." />
  if (error || !data) return <div>Error al cargar proveedores</div>

  return (
    <SelectList
      data={data}
      value={value}
      onChange={onChange}
      labelKey="name"
      valueKey="id"
      disabled={disabled}
    />
  )
}

export default SelectSupplier
```

### Reglas del patrón:

1. Interface de props **siempre arriba** del componente
2. Props **destructuradas** en la firma de la función
3. Estados de `isLoading` y `error` manejados **antes** del render principal
4. Exportación `default` al final (no inline con la función)
5. Nombre del archivo en **kebab-case**, nombre del componente en **PascalCase**

---

## Componente de tabla/lista (página CRUD)

```typescript
// src/pages/suppliers/components/SupplierList.tsx
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from '@/components/ui/table'
import { Pencil, Trash2 } from 'lucide-react'
import type { SupplierResponse } from '@/domain/supplier/supplier.types'

interface SupplierListProps {
  suppliers: SupplierResponse[]
  onEdit: (supplier: SupplierResponse) => void
  onDelete: (id: string) => void
  isLoading?: boolean
}

const SupplierList = ({ suppliers, onEdit, onDelete, isLoading }: SupplierListProps) => {
  if (isLoading) {
    return <div className="flex justify-center p-8"><LoaderSpin /></div>
  }

  if (suppliers.length === 0) {
    return (
      <div className="flex flex-col items-center gap-2 p-8 text-muted-foreground">
        <p>No hay proveedores registrados</p>
      </div>
    )
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Nombre</TableHead>
          <TableHead>Tipo</TableHead>
          <TableHead>Estado</TableHead>
          <TableHead className="text-right">Acciones</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {suppliers.map((supplier) => (
          <TableRow key={supplier.id}>
            <TableCell className="font-medium">{supplier.name}</TableCell>
            <TableCell>{supplier.type}</TableCell>
            <TableCell>
              <Badge variant={supplier.active ? 'default' : 'secondary'}>
                {supplier.active ? 'Activo' : 'Inactivo'}
              </Badge>
            </TableCell>
            <TableCell className="text-right">
              <div className="flex justify-end gap-2">
                <Button variant="ghost" size="icon" onClick={() => onEdit(supplier)}>
                  <Pencil className="h-4 w-4" />
                </Button>
                <Button variant="ghost" size="icon" onClick={() => onDelete(supplier.id)}>
                  <Trash2 className="h-4 w-4" />
                </Button>
              </div>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  )
}

export default SupplierList
```

---

## Formulario con React Hook Form + Zod

```typescript
// src/pages/suppliers/components/SupplierForm.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from '@/components/ui/form'
import { useApiSend } from '@/common/react-query/useApi'
import { SupplierService } from '@/services/supplier.service'
import { CreateSupplierSchema, type CreateSupplierDto } from '@/domain/supplier/supplier.types'
import { toast } from 'sonner'

interface SupplierFormProps {
  onSuccess?: () => void
  defaultValues?: Partial<CreateSupplierDto>
}

const SupplierForm = ({ onSuccess, defaultValues }: SupplierFormProps) => {
  const form = useForm<CreateSupplierDto>({
    resolver: zodResolver(CreateSupplierSchema),
    defaultValues: {
      name: '',
      email: '',
      type: '',
      ...defaultValues,
    },
  })

  const { mutate, isPending } = useApiSend<CreateSupplierDto>({
    fn: SupplierService.create,
    success: () => {
      toast.success('Proveedor creado correctamente')
      form.reset()
      onSuccess?.()
    },
    error: () => toast.error('Error al crear el proveedor'),
    invalidateKey: [{ queryKey: ['suppliers'] }],
  })

  const onSubmit = (data: CreateSupplierDto) => mutate(data)

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Nombre</FormLabel>
              <FormControl>
                <Input placeholder="Nombre del proveedor" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" placeholder="correo@empresa.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={isPending} className="w-full">
          {isPending ? 'Guardando...' : 'Crear proveedor'}
        </Button>
      </form>
    </Form>
  )
}

export default SupplierForm
```

---

## Dialog/Modal para creación/edición

```typescript
// Patrón: Dialog controlado por estado booleano en el padre
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog'

interface SupplierDialogProps {
  open: boolean
  onOpenChange: (open: boolean) => void
  supplier?: SupplierResponse  // undefined = crear, definido = editar
}

const SupplierDialog = ({ open, onOpenChange, supplier }: SupplierDialogProps) => {
  const isEditing = Boolean(supplier)

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-md">
        <DialogHeader>
          <DialogTitle>
            {isEditing ? 'Editar proveedor' : 'Nuevo proveedor'}
          </DialogTitle>
        </DialogHeader>
        <SupplierForm
          defaultValues={supplier}
          onSuccess={() => onOpenChange(false)}
        />
      </DialogContent>
    </Dialog>
  )
}
```

---

## Componente con permisos

```typescript
// Verificar permisos con el store de app
import { useAppStore } from '@/store/app.store'
import { hasPermission } from '@/utils/permissions.utils'
import { PERMISSIONS } from '@/common/constants/enums/permissions'

const SupplierActions = ({ supplier }: { supplier: SupplierResponse }) => {
  const permissions = useAppStore((state) => state.permissions)

  const canEdit = hasPermission(permissions, PERMISSIONS.SUPPLIER_EDIT)
  const canDelete = hasPermission(permissions, PERMISSIONS.SUPPLIER_DELETE)

  return (
    <div className="flex gap-2">
      {canEdit && (
        <Button variant="ghost" size="icon" onClick={() => onEdit(supplier)}>
          <Pencil className="h-4 w-4" />
        </Button>
      )}
      {canDelete && (
        <Button variant="ghost" size="icon" onClick={() => onDelete(supplier.id)}>
          <Trash2 className="h-4 w-4" />
        </Button>
      )}
    </div>
  )
}
```

---

## Componente de confirmación de eliminación

```typescript
// Patrón estándar con AlertDialog de shadcn-ui
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel,
  AlertDialogContent, AlertDialogDescription, AlertDialogFooter,
  AlertDialogHeader, AlertDialogTitle,
} from '@/components/ui/alert-dialog'

interface DeleteConfirmDialogProps {
  open: boolean
  onOpenChange: (open: boolean) => void
  onConfirm: () => void
  itemName: string
  isPending?: boolean
}

const DeleteConfirmDialog = ({
  open, onOpenChange, onConfirm, itemName, isPending,
}: DeleteConfirmDialogProps) => (
  <AlertDialog open={open} onOpenChange={onOpenChange}>
    <AlertDialogContent>
      <AlertDialogHeader>
        <AlertDialogTitle>¿Eliminar {itemName}?</AlertDialogTitle>
        <AlertDialogDescription>
          Esta acción no se puede deshacer.
        </AlertDialogDescription>
      </AlertDialogHeader>
      <AlertDialogFooter>
        <AlertDialogCancel>Cancelar</AlertDialogCancel>
        <AlertDialogAction onClick={onConfirm} disabled={isPending}>
          {isPending ? 'Eliminando...' : 'Eliminar'}
        </AlertDialogAction>
      </AlertDialogFooter>
    </AlertDialogContent>
  </AlertDialog>
)
```

---

## Utility: cn() para clases condicionales

```typescript
import { cn } from '@/lib/utils'

// Uso correcto:
<div className={cn(
  'base-class flex items-center',
  isActive && 'bg-primary text-primary-foreground',
  disabled && 'opacity-50 cursor-not-allowed',
  className,  // siempre al final para permitir override externo
)} />
```

---

## Anti-patrones a evitar

```typescript
// ❌ NO: Props drilling profundo
<GrandParent>
  <Parent onEdit={onEdit} onDelete={onDelete} isLoading={isLoading}>
    <Child onEdit={onEdit} onDelete={onDelete} isLoading={isLoading}>
      <GrandChild onEdit={onEdit} />

// ✓ SÍ: Pasar el dato al hijo y que él llame el hook
const GrandChild = ({ supplierId }: { supplierId: string }) => {
  const { mutate } = useApiSend({ fn: SupplierService.delete, ... })
  return <Button onClick={() => mutate(supplierId)}>Eliminar</Button>
}

// ❌ NO: any implícito
const handleData = (data: any) => { ... }

// ✓ SÍ: Tipado explícito siempre
const handleData = (data: SupplierResponse) => { ... }

// ❌ NO: Rutas relativas largas
import { Button } from '../../../components/ui/button'

// ✓ SÍ: Alias @/
import { Button } from '@/components/ui/button'
```
