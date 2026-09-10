# Arquitectura multi-tenant

Este documento describe la arquitectura de referencia para una aplicación SaaS multi-tenant construida con .NET y PostgreSQL.

El objetivo principal es garantizar que el aislamiento entre organizaciones sea una propiedad estructural del sistema y no dependa únicamente de convenciones de desarrollo.

## Flujo de una solicitud

```text
Cliente
  │
  ▼
API
  │
  ▼
Autenticación
  │
  ▼
Resolución de tenant
  │
  ▼
Validación de membresía
  │
  ▼
Autorización
  │
  ▼
Application
  │
  ▼
Infrastructure
  │
  ▼
PostgreSQL + RLS
```

Cada solicitud debe operar dentro de un tenant explícito, previamente validado contra la identidad autenticada.

## Identidad y membresía

La autenticación responde quién es el usuario. La membresía responde dentro de qué organizaciones puede operar.

```text
User
  │
  ├── Membership ── Tenant A
  ├── Membership ── Tenant B
  └── Membership ── Tenant C
```

El hecho de que un usuario esté autenticado no implica acceso automático a todos los tenants.

## Tenant Context

Una vez validada la membresía, la aplicación construye un contexto confiable para la operación:

```csharp
public interface ITenantContext
{
    Guid TenantId { get; }
}
```

El resto de la aplicación consume ese contexto en lugar de confiar directamente en identificadores entregados por el cliente.

## Persistencia

Las entidades multi-tenant deben incorporar una clave de tenant:

```text
Resource
├── Id
├── TenantId
├── ...
```

Las restricciones e índices deben considerar esa dimensión cuando corresponda.

Por ejemplo, una clave de negocio que sólo deba ser única dentro de una organización puede representarse mediante un índice único compuesto:

```sql
UNIQUE (tenant_id, external_code)
```

## RLS como barrera adicional

PostgreSQL Row-Level Security permite aplicar políticas de aislamiento directamente en la base de datos.

```text
Application filter
        +
Authorization
        +
Database RLS
        =
Defense in depth
```

Una política conceptual puede restringir cada operación al tenant activo.

```sql
CREATE POLICY tenant_isolation
ON resources
USING (tenant_id = current_tenant_id())
WITH CHECK (tenant_id = current_tenant_id());
```

La aplicación debe establecer `current_tenant_id()` mediante un mecanismo controlado y asociado al ciclo de vida de la conexión o transacción.

## Riesgos principales

### IDOR entre tenants

Un cliente modifica un identificador y trata de acceder a un recurso perteneciente a otra organización.

La autorización debe validar simultáneamente identidad, membresía, permiso y pertenencia del recurso al tenant activo.

### Consultas sin filtro

Una consulta de aplicación omite accidentalmente `tenant_id`.

RLS permite reducir el impacto potencial de este error.

### Contexto de tenant contaminado

En pools de conexiones, el contexto de tenant no debe persistir accidentalmente entre solicitudes.

La implementación debe establecer y limpiar correctamente el contexto dentro de una transacción o ámbito conocido.

### Enumeración de recursos

El sistema no debe revelar mediante respuestas distintas que un identificador pertenece a otro tenant.

Las estrategias HTTP y de errores deben evitar fugas de información innecesarias.

## Estrategia inicial recomendada

Para una aplicación SaaS que comienza con un número moderado de organizaciones, la opción de referencia es:

```text
Una base PostgreSQL
        +
Esquema compartido
        +
tenant_id
        +
Autorización de aplicación
        +
RLS
```

Esta alternativa mantiene bajo control la complejidad operacional y permite evolucionar posteriormente si aparecen requisitos reales de aislamiento físico, volumen o regulación.

## Evolución

Si determinados tenants exigen aislamiento superior, la arquitectura puede evolucionar hacia modelos híbridos:

```text
Tenants estándar
      │
      ▼
Shared Database

Tenant regulado / gran volumen
      │
      ▼
Dedicated Database
```

Esto evita asumir desde el inicio el costo operacional de una base de datos por organización.

## Principio de diseño

La multi-tenencia no se considera sólo una característica del modelo de datos. Afecta autenticación, autorización, persistencia, caching, jobs en segundo plano, observabilidad, archivos, integraciones y pruebas.

Toda nueva capacidad del sistema debe responder explícitamente a una pregunta: **¿en qué tenant se ejecuta esta operación y cómo se garantiza su aislamiento?**
