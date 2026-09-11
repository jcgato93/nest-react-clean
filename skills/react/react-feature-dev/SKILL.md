---
name: react-feature-dev
description: >
  Skill para desarrollar funcionalidades y modificaciones en el proyecto reactJs.
  Guía al agente a través de un flujo estructurado: Analizar → Planear → Confirmar → Implementar,
  siguiendo las convenciones del proyecto con React 19, TypeScript estricto, Zustand, React Query
  (wrappers personalizados), React Router v7, Tailwind CSS v4, shadcn-ui, Zod + React Hook Form,
  y principios SOLID/DRY.

  Usa este skill cuando el desarrollador solicite:
  - Crear una nueva página, módulo o funcionalidad
  - Agregar o modificar componentes, servicios, modelos o rutas
  - Integrar un nuevo endpoint de API al frontend
  - Refactorizar código siguiendo las convenciones del proyecto
  - Cualquier cambio que toque la capa de UI, estado, datos o navegación
---

# React Feature Developer

Eres el agente de desarrollo de **ReactJs**. Tu rol es implementar funcionalidades siguiendo
las convenciones exactas del proyecto. Antes de escribir una sola línea de código, debes analizar el
impacto del cambio y obtener confirmación del desarrollador.

## Stack tecnológico

| Capa               | Herramienta                  | Patrón en el proyecto                            |
| ------------------ | ---------------------------- | ------------------------------------------------ |
| UI                 | React 19 + TypeScript strict | Componentes funcionales declarativos             |
| Estado global (UI) | Zustand v5                   | Stores tipadas con `persist` cuando es necesario |
| Estado servidor    | React Query v5               | `useApiGet`, `useApiSend`, `useApiInfinite`      |
| Routing            | React Router v7              | `createBrowserRouter` con lazy loading           |
| Estilos            | Tailwind CSS v4              | Clases + `cn()` para condicionales               |
| Componentes UI     | shadcn-ui (new-york)         | Desde `@/components/ui/`                         |
| Formularios        | React Hook Form + Zod        | Schemas en `domain/*/`                           |
| HTTP               | Axios + ApiService singleton | `@/services/api.service.ts`                      |
| Notificaciones     | Sonner                       | `toast.success/error/warning`                    |
| Aliases            | `@/*` → `src/*`              | Siempre usar alias, nunca rutas relativas largas |

## Alias de importación

```typescript
// SIEMPRE así:
import { Button } from '@/components/ui/button'
import { useApiGet } from '@/common/react-query/useApi'
import { UserService } from '@/services/user.service'

// NUNCA así:
import { Button } from '../../../components/ui/button'
```

---

## Flujo de trabajo obligatorio

Todo cambio — sin importar el tamaño — sigue este flujo de 4 fases. No omitas ninguna fase.

### Fase 1: ANALIZAR

Antes de planear, responde estas preguntas examinando el código real:

**Checklist de análisis:**

- [ ] ¿Qué archivos existentes se verán afectados?
- [ ] ¿Existen tipos/interfaces en `src/domain/` que reutilizar o extender?
- [ ] ¿Hay servicios en `src/services/` que ya manejen este recurso?
- [ ] ¿Los endpoints necesarios están definidos en `src/common/axios.ts`?
- [ ] ¿El estado global (Zustand) está involucrado? ¿Qué store?
- [ ] ¿Se necesita agregar una nueva ruta en `AppRouter.tsx`?
- [ ] ¿Existen componentes reutilizables para este caso en `src/components/`?
- [ ] ¿Hay permisos/guards que aplicar?

Lee los archivos relevantes antes de continuar. No asumas — verifica.

---

### Fase 2: PLANEAR

Presenta el plan al desarrollador con este formato:

```
## Plan de implementación: [Nombre de la funcionalidad]

### Archivos nuevos a crear:
- `src/domain/[feature]/[feature].types.ts` — Interfaces y schemas Zod
- `src/services/[feature].service.ts` — Servicio API
- `src/pages/[feature]/[Feature]Page.tsx` — Página principal
- `src/components/functional/[Feature]Form.tsx` — Formulario (si aplica)

### Archivos existentes a modificar:
- `src/common/axios.ts` — Agregar endpoints del recurso
- `src/AppRouter.tsx` — Registrar nueva ruta

### Dependencias entre archivos (orden de implementación):
1. Tipos → 2. Endpoints → 3. Servicio → 4. Hooks → 5. Componentes → 6. Página → 7. Ruta

### Consideraciones de impacto:
- [Qué otras partes del sistema pueden verse afectadas]
- [Permisos requeridos]
- [Estado global involucrado]
```

---

### Fase 3: CONFIRMAR

**STOP.** Antes de implementar, espera la aprobación explícita del desarrollador.

Pregunta: _"¿Apruebas este plan o quieres ajustar algo antes de implementar?"_

No continúes hasta recibir confirmación.

---

### Fase 4: IMPLEMENTAR

Implementa en el orden establecido en el plan. Usa el checklist de implementación:

**Checklist de implementación:**

- [ ] Tipos e interfaces definidos en `src/domain/[feature]/`
- [ ] Schemas Zod creados y tipos inferidos con `z.infer<>`
- [ ] DTOs de entrada/salida definidos
- [ ] Endpoints agregados en `src/common/axios.ts`
- [ ] Servicio implementado en `src/services/` (clase estática)
- [ ] Página creada con loading, error y estados vacíos
- [ ] Formularios con validación Zod + React Hook Form (si aplica)
- [ ] Ruta registrada con lazy loading en `AppRouter.tsx` (si aplica)
- [ ] Título de ruta agregado en el mapa de títulos (si aplica)
- [ ] Notificaciones con Sonner en acciones CRUD
- [ ] Sin rutas de importación relativas largas (usar `@/`)
- [ ] Sin `any` implícitos — TypeScript strict cumplido

---

## Convenciones de nombrado

| Elemento             | Convención                 | Ejemplo                                 |
| -------------------- | -------------------------- | --------------------------------------- |
| Componentes React    | PascalCase                 | `SupplierForm.tsx`                      |
| Servicios            | PascalCase + "Service"     | `SupplierService.ts`                    |
| Hooks personalizados | camelCase + "use"          | `useSupplierData.ts`                    |
| Tipos e interfaces   | PascalCase                 | `SupplierResponse`, `CreateSupplierDto` |
| Schemas Zod          | PascalCase + "Schema"      | `CreateSupplierSchema`                  |
| Props de componente  | PascalCase + "Props"       | `SupplierFormProps`                     |
| Constantes           | UPPER_SNAKE_CASE           | `API_ENDPOINTS`                         |
| Directorios          | kebab-case                 | `fixed-assets/`, `bank-accounts/`       |
| Archivos de tipos    | kebab-case + `.types.ts`   | `supplier.types.ts`                     |
| Archivos de servicio | kebab-case + `.service.ts` | `supplier.service.ts`                   |

---

## Estructura de directorios para features nuevas

```
src/
├── domain/[feature]/
│   ├── [feature].types.ts          # Interfaces + Zod schemas
│   └── dto/
│       ├── create-[feature].dto.ts
│       └── update-[feature].dto.ts
├── services/
│   └── [feature].service.ts
├── pages/[feature]/
│   ├── [Feature]Page.tsx           # Página contenedora
│   └── components/                 # Componentes locales a esta página
│       ├── [Feature]List.tsx
│       └── [Feature]Form.tsx
└── components/functional/          # Solo si el componente es reutilizable
    └── select-[feature].tsx
```

---

## Referencias disponibles

Consulta estas referencias **solo cuando las necesites** — no las cargues todas al inicio:

| Referencia                         | Cuándo consultarla                                |
| ---------------------------------- | ------------------------------------------------- |
| `references/component-patterns.md` | Crear componentes, formularios, tablas, selects   |
| `references/service-patterns.md`   | Crear servicios API, definir endpoints            |
| `references/model-patterns.md`     | Definir tipos, interfaces, Zod schemas, DTOs      |
| `references/page-patterns.md`      | Crear páginas, CRUD completo, lazy loading, rutas |
| `references/state-patterns.md`     | Zustand stores, React Query hooks personalizados  |

---

## Principios de calidad (SOLID + DRY)

- **S** — Un componente/servicio tiene una sola responsabilidad. `UserService` no maneja catalogs.
- **O** — Extiende tipos con `extends` o `z.extend()`, no modifiques los existentes.
- **L** — Los componentes que reciben callbacks deben funcionar sin conocer su implementación padre.
- **I** — Props interfaces pequeñas y específicas. Evita el prop-drilling con más de 3 niveles.
- **D** — Los componentes dependen de abstracciones (hooks, servicios) no de implementaciones directas.
- **DRY** — Si repites la misma lógica en 2+ lugares, extráela a un hook o utilidad.

---

## Patrones rápidos (para referencia inmediata)

### Llamada a API (GET)

```typescript
const { data, isLoading, error } = useApiGet<SupplierResponse[]>({
  key: ['suppliers', companyId],
  fn: () => SupplierService.getAll(companyId),
});
```

### Mutación (POST/PUT)

```typescript
const { mutate, isPending } = useApiSend<CreateSupplierDto>({
  fn: SupplierService.create,
  success: () => toast.success('Proveedor creado'),
  error: () => toast.error('Error al crear proveedor'),
  invalidateKey: [{ queryKey: ['suppliers'] }],
});
```

### Estado de carga estándar

```typescript
if (isLoading) return <LoaderSpin />
if (error || !data) return <div>Error al cargar datos</div>
```

### Formulario con validación

```typescript
const form = useForm<CreateSupplierDto>({
  resolver: zodResolver(CreateSupplierSchema),
  defaultValues: { name: '', email: '' },
});
```

### Toast notifications

```typescript
toast.success('Operación exitosa');
toast.error('Ocurrió un error');
toast.warning('Atención requerida');
```
