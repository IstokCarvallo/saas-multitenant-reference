# SaaS Multi-Tenant Reference

Arquitectura de referencia para aplicaciones SaaS multi-tenant con .NET y PostgreSQL, enfocada en aislamiento de datos, seguridad y escalabilidad.

Este repositorio documenta una estrategia reutilizable para diseñar aplicaciones multi-tenant sin acoplar la solución a un producto o dominio específico. El objetivo es mostrar decisiones arquitectónicas aplicables a sistemas reales, con especial énfasis en identificación de tenant, aislamiento de datos, autorización, persistencia y evolución operativa.

## Objetivos

- Definir una arquitectura clara para aplicaciones SaaS multi-tenant.
- Separar identidad de usuario, organización y contexto de tenant.
- Evitar filtraciones de datos entre tenants por diseño.
- Combinar controles en aplicación y base de datos.
- Mantener una arquitectura escalable sin introducir complejidad prematura.
- Documentar trade-offs entre distintas estrategias de aislamiento.
- Servir como referencia reutilizable para APIs .NET con PostgreSQL.

## Concepto de tenant

Un tenant representa una organización aislada lógicamente dentro de una misma plataforma.

```text
Usuario
  │
  ▼
Membresía
  │
  ▼
Tenant / Organización
  │
  ▼
Datos y operaciones autorizadas
```

Un usuario puede pertenecer a uno o más tenants, pero cada operación debe ejecutarse dentro de un contexto de tenant explícito y validado.

## Estrategia de aislamiento

La referencia parte de un modelo de base de datos compartida con aislamiento lógico por `tenant_id`.

```text
Database
├── Tenant A
│   ├── Users
│   ├── Records
│   └── Transactions
│
├── Tenant B
│   ├── Users
│   ├── Records
│   └── Transactions
│
└── Tenant C
    ├── Users
    ├── Records
    └── Transactions
```

Físicamente los registros pueden residir en las mismas tablas, pero toda entidad multi-tenant incorpora una clave de tenant y las políticas de acceso impiden que una organización consulte o modifique datos de otra.

## Defensa en profundidad

El aislamiento no debe depender de un único `WHERE tenant_id = ...` disperso por la aplicación.

La estrategia considera varias capas:

```text
Request autenticado
        │
        ▼
Resolución de identidad
        │
        ▼
Resolución de tenant
        │
        ▼
Validación de membresía
        │
        ▼
Autorización de operación
        │
        ▼
Application / Domain
        │
        ▼
Persistencia con tenant
        │
        ▼
PostgreSQL / RLS
```

Esto permite aplicar defensa en profundidad: aunque exista un error en una consulta de aplicación, una política de base de datos puede impedir el acceso cruzado entre tenants.

## Resolución del tenant

El tenant activo nunca debe aceptarse ciegamente desde el cliente.

Una estrategia típica consiste en:

1. autenticar al usuario;
2. obtener su identidad confiable;
3. recibir o resolver el tenant solicitado;
4. validar que el usuario tenga una membresía activa en ese tenant;
5. construir un `TenantContext` para la operación;
6. propagar el identificador de tenant de forma controlada hacia persistencia.

Un contrato conceptual puede ser:

```csharp
public interface ITenantContext
{
    Guid TenantId { get; }
}
```

La implementación concreta podrá obtener el tenant desde claims, routing, headers u otro mecanismo, siempre que exista validación de pertenencia antes de utilizarlo.

## Modelo conceptual

```text
Tenant
  │
  ├── Membership ───── User
  │
  ├── Resource
  ├── Transaction
  └── Configuration
```

La membresía establece qué usuarios pueden operar dentro de cada organización y puede incorporar roles o permisos específicos del tenant.

## Row-Level Security

PostgreSQL permite reforzar el aislamiento mediante Row-Level Security (RLS).

Conceptualmente:

```sql
CREATE POLICY tenant_isolation
ON resources
USING (tenant_id = current_tenant_id());
```

La implementación final deberá establecer el contexto de tenant de forma segura en cada conexión o transacción antes de ejecutar consultas protegidas.

RLS se plantea como una capa adicional de seguridad, no como sustituto de autorización y validación en la aplicación.

## Alternativas de aislamiento

No existe una única estrategia correcta para todos los SaaS.

### Base compartida + esquema compartido

```text
Database
└── Shared schema
    └── Tables with tenant_id
```

Ventajas:
- menor complejidad operacional;
- despliegues y migraciones centralizados;
- eficiente para un número alto de tenants pequeños o medianos.

Costos:
- exige disciplina estricta de aislamiento;
- una consulta mal construida puede representar un riesgo si no existe defensa adicional.

### Base compartida + esquema por tenant

```text
Database
├── tenant_a.*
├── tenant_b.*
└── tenant_c.*
```

Aumenta separación lógica, pero también incrementa complejidad de migraciones, observabilidad y operación.

### Base de datos por tenant

```text
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
```

Entrega mayor aislamiento físico, pero aumenta significativamente aprovisionamiento, costos, conexiones, migraciones y mantenimiento.

La elección debe responder a requisitos reales de seguridad, regulación, volumen, operación y costo; no a una preferencia arquitectónica abstracta.

## Estructura prevista

```text
saas-multitenant-reference/
├── src/
│   ├── MultiTenantReference.Api/
│   ├── MultiTenantReference.Application/
│   ├── MultiTenantReference.Domain/
│   └── MultiTenantReference.Infrastructure/
│
├── tests/
│   ├── MultiTenantReference.UnitTests/
│   └── MultiTenantReference.IntegrationTests/
│
├── docs/
│   ├── architecture.md
│   └── isolation-strategies.md
│
├── .gitignore
├── README.md
└── MultiTenantReference.sln
```

La estructura se implementará de manera incremental. Cada proyecto se añadirá cuando exista una responsabilidad concreta que justifique su presencia.

## Seguridad

La referencia considerará progresivamente:

- autenticación independiente del tenant;
- validación de membresías;
- autorización por roles y permisos dentro del tenant;
- aislamiento por `tenant_id`;
- RLS en PostgreSQL;
- manejo seguro del contexto de tenant;
- logging sin filtración de información entre organizaciones;
- pruebas explícitas de aislamiento;
- protección frente a manipulación de identificadores de tenant.

## Pruebas de aislamiento

Un sistema multi-tenant debe probar específicamente que un usuario del Tenant A no pueda:

- consultar registros del Tenant B;
- modificar registros del Tenant B;
- inferir su existencia mediante errores o diferencias de respuesta;
- utilizar identificadores válidos de otro tenant para elevar acceso.

Estas pruebas forman parte de la arquitectura, no son sólo casos funcionales adicionales.

## Documentación

Las decisiones principales se desarrollan en [`docs/architecture.md`](docs/architecture.md).

Las estrategias de aislamiento y sus trade-offs se documentarán posteriormente en `docs/isolation-strategies.md`.

## Estado

Este repositorio está en construcción. La primera etapa está centrada en documentar la arquitectura y los mecanismos de aislamiento antes de incorporar una implementación ejecutable.

## Autor

**Istok Carvallo**  
Arquitectura de Software · Gestión de Proyectos TI · Desarrollo de Productos Digitales
