---
name: code-review-instructions
description: Use when reviewing Pull Requests, code changes, or any code modification in this NestJS + Clean Architecture project. Produces a structured review with severity ratings (CRITICAL/HIGH/MEDIUM/LOW), a per-layer checklist, merge blockers, and a final verdict (APPROVE/REQUEST_CHANGES). Always use this skill when the user says "review this PR", "review my changes", "check this code", "does this look right", or when they share a diff or code for feedback.
---

# Instrucciones de Code Review

## Objetivo
Realizar revisiones de Pull Request con enfoque en calidad, arquitectura limpia, mantenibilidad, seguridad y consistencia con NestJS + Clean Architecture del repositorio.

## Alcance
- Revisar únicamente el código del diff/selección solicitada.
- No asumir comportamiento no visible en el código.
- Si falta contexto, declararlo explícitamente.
- Marcar cada criterio como: `OK`, `NO`, `N/A`.

## Formato de salida obligatorio
1. **Resumen ejecutivo** (máximo 5 líneas).
2. **Hallazgos por severidad**:
   - `CRITICAL | HIGH | MEDIUM | LOW`
   - Archivo y línea (si aplica)
   - Criterio incumplido
   - Riesgo
   - Recomendación concreta
3. **Checklist de revisión** (completo, con estado por ítem).
4. **Bloqueadores de merge**.
5. **Sugerencias no bloqueantes**.
6. **Veredicto final**: `APPROVE` | `REQUEST_CHANGES`.

## Checklist de Revisión de Pull Request


Tomar de referencia los archivos de instrucciones específicos para cada capa (Domain, Infrastructure, Application) y tipo de componente (Entities, Repositorios, Controladores, etc.) para evaluar cada aspecto del código revisado.
Estos se encuentran en la carpeta `.github/skills/` y deben ser aplicados según corresponda a los archivos modificados en el Pull Request.

### General
- [ ] ¿El código sigue las convenciones de estilo establecidas en el proyecto?
- [ ] ¿Se han agregado pruebas unitarias o de integración para las nuevas funcionalidades, si aplica?
- [ ] ¿El código está debidamente documentado, incluyendo comentarios y documentación de funciones?
- [ ] ¿Se han corregido los errores de linting y formateo?
- [ ] ¿Se han actualizado las dependencias si es necesario?

### Domain

#### Exceptions
- [ ] ¿Se heredan de las excepciones base correctas?
- [ ] ¿Se documentan adecuadamente las excepciones personalizadas?
- [ ] ¿El mensaje es claro y útil para el usuario final?

#### Entities
- [ ] ¿Se siguen las convenciones de nomenclatura establecidas?
- [ ] ¿Se incluyen todos los atributos necesarios?
- [ ] ¿El id se le asigna valor cuando no es suministrado?
- [ ] ¿Se implementan los métodos de plainToInstance?

#### Repositorios
- [ ] ¿Se documentan los métodos del repositorio?

#### Use-cases
- [ ] ¿Se reciben y retornan las entidades correctas, solo dominio?
- [ ] ¿Se siguen las reglas de negocio definidas?
- [ ] ¿Se documentan claramente los parámetros y el valor de retorno?

### Infrastructure

#### Controladores
- [ ] ¿Se validan correctamente los datos de entrada, incluyendo tipos y formatos?
- [ ] ¿Se documentan las rutas y métodos HTTP?
- [ ] ¿Se documenta correctamente con los decoradores de Swagger?
- [ ] ¿Se valida la autenticación y autorización cuando es necesario?
- [ ] ¿Retorna solo DTOs y no entidades de dominio?

#### Database
- [ ] ¿Se utilizan migraciones para los cambios en la base de datos?
- [ ] ¿Se siguen las convenciones de nomenclatura para tablas y columnas?
- [ ] ¿Se implementan índices cuando es necesario para mejorar el rendimiento?
- [ ] ¿Las entidades extienden de BaseEntity?
- [ ] ¿Se documentan las relaciones entre tablas?
- [ ] ¿Los repositorios implementan las interfaces definidas en el dominio?
- [ ] ¿Las consultas están optimizadas y no generan cargas innecesarias?
- [ ] ¿Se evita lógica de negocio en los repositorios?
- [ ] ¿Los seeders inicializan datos necesarios para el funcionamiento de la aplicación?

#### Servicios externos
- [ ] ¿Se manejan adecuadamente los errores de las llamadas a servicios externos?
- [ ] ¿Se documentan las dependencias externas y sus versiones?
- [ ] ¿Se implementan mecanismos de reintento o fallback cuando es necesario?
- [ ] ¿Se validan las respuestas recibidas de los servicios externos?

#### Configuración
- [ ] ¿Se utilizan variables de entorno para configuraciones sensibles?
- [ ] ¿Se documentan las variables de entorno necesarias y sus valores por defecto?
- [ ] ¿Se implementan diferentes configuraciones para distintos entornos (desarrollo, producción, pruebas)?
- [ ] ¿Se evita hardcodear valores en el código fuente?

### Application

#### DTO
- [ ] ¿Se siguen las convenciones de nomenclatura establecidas?
- [ ] ¿Se documentan adecuadamente los atributos del DTO?
- [ ] ¿Se utilizan validadores para asegurar la integridad de los datos?

#### Mappers
- [ ] ¿Se implementan correctamente los métodos de mapeo entre entidades y DTOs?
- [ ] ¿Se manejan adecuadamente los casos especiales durante el mapeo?
- [ ] ¿Se documentan claramente los métodos de mapeo?

#### Use-cases
- [ ] ¿Se reciben y retornan los dominios correctos?
- [ ] ¿Se documentan claramente los parámetros y el valor de retorno?
- [ ] ¿Se implementan las reglas de negocio definidas en el dominio?
- [ ] ¿Se manejan adecuadamente los errores y excepciones?
- [ ] ¿Se evita lógica de presentación o infraestructura en los casos de uso, como validaciones de entrada o manejo de respuestas HTTP?
- [ ] ¿Se tiene un manejo adecuado de transacciones si es necesario?

## Reglas de evaluación
- Priorizar hallazgos que rompan arquitectura por capas o contratos de dominio.
- Si un criterio no aplica, marcar `N/A` con justificación breve.
- Cada hallazgo debe incluir propuesta accionable.
- Evitar recomendaciones genéricas sin evidencia en el código.