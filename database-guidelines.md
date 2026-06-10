# Estandarización de Roles y Permisos — Eurekant LLC

> **Versión:** 1.1.0 (conceptual — sin SQL)
> **Fecha:** 10/06/2026
> **Estado:** Borrador para validación interna
> **Alcance:** Todos los proyectos de software desarrollados por Eurekant

| Versión | Fecha | Cambios |
|---|---|---|
| 1.0.0 | 09/06/2026 | Versión inicial. |
| 1.1.0 | 10/06/2026 | Aclaración del término RBAC; referencias cruzadas y sinónimos en el glosario; fusión de RN-03 y RN-04 (regla Owner-admin) con renumeración; justificación del formato de permisos `modulo.accion` y de la tabla `ROLE_PERMISSIONS`. |

---

## 1. Propósito y alcance

Este documento define el **modelo estándar de multi-tenancy, roles y permisos** que debe implementarse en **todos los proyectos de Eurekant**, sin importar el rubro o la finalidad del software.

El objetivo es que cualquier desarrollador del equipo pueda abrir cualquier proyecto de la empresa y encontrar **la misma estructura, los mismos nombres de tablas y los mismos flujos**, reduciendo la curva de aprendizaje, los errores de seguridad y el costo de mantenimiento.

Esta versión es **puramente conceptual**: define entidades, relaciones, flujos y reglas de negocio. Una vez validada, se generará la versión 2 con el código SQL definitivo (tablas, constraints, funciones, triggers y políticas RLS), siguiendo la [Naming Convention Guide](https://app.clickup.com/9002039309/v/dc/8c90e0d-10194/8c90e0d-6114) de Eurekant.

> 💡 **Ejemplo práctico — ¿por qué estandarizar?**
> Eurekant desarrolla un sistema de turnos para una clínica y un sistema de stock para una distribuidora. Son rubros totalmente distintos, pero ambos necesitan: usuarios, empresas, sucursales, roles, invitaciones y aislamiento de datos. Si ambos usan este estándar, un desarrollador que pasa del proyecto "clínica" al proyecto "distribuidora" ya sabe cómo funciona el 40% del sistema antes de leer una línea de código.

---

## 2. Principios de diseño

1. **Multi-tenant siempre.** Todo sistema soporta múltiples empresas y múltiples sucursales por empresa, **aunque el cliente actual no lo necesite**. Si el software se vende a un solo cliente con una sola sucursal, internamente igual existen `COMPANIES` y `BRANCHES` con un único registro. Esto garantiza escalabilidad sin migraciones traumáticas.
2. **Aislamiento por RLS, no por filtros.** El código de aplicación **nunca** filtra por empresa/sucursal en sus queries. La base de datos (Row Level Security) devuelve únicamente los datos a los que el usuario tiene acceso según su contexto activo.
3. **Roles a nivel empresa, asignaciones a nivel sucursal.** Un rol se define una vez por empresa y se reutiliza en todas sus sucursales. La asignación concreta de un usuario es siempre `usuario + rol + sucursal`.
4. **Permisos granulares.** Un rol no es una etiqueta mágica que el código interpreta: es un **conjunto de permisos** tomados de un catálogo definido por cada sistema. Este es el modelo **RBAC** (*Role-Based Access Control*, control de acceso basado en roles): los permisos nunca se asignan directamente a los usuarios, sino a roles, y los usuarios obtienen sus permisos al recibir roles. Es el modelo clásico que usan Slack, Notion o AWS.
5. **Identidad global única.** Una persona tiene **una sola cuenta** (un email) y con ella puede pertenecer a N empresas y N sucursales con distintos roles.
6. **Nada se borra, se desactiva.** Usuarios, roles y vínculos se desactivan (soft delete) para preservar el historial y la auditoría.
7. **El superadmin vive fuera del modelo de empresas.** Es la capa de los dueños del software, con su propio panel y su propia parametrización global.

> 💡 **Ejemplo práctico — principio 1 (multi-tenant aunque no haga falta)**
> Un cliente pide un sistema interno solo para su ferretería. Se desarrolla igual con el modelo completo: el *initial setup* crea la empresa "Ferretería López" y la sucursal "Principal". Dos años después el cliente abre una segunda sucursal y quiere vender el sistema a un colega. **No hay que tocar la arquitectura**: solo se crea otra sucursal y otra empresa.

---

## 3. Glosario

| Término | Definición |
|---|---|
| **Usuario** | Identidad global de una persona (email único). Existe una sola vez en todo el sistema. |
| **Empresa (Company)** | Tenant principal. Unidad de aislamiento de datos y dueña de los roles. |
| **Sucursal (Branch)** | Subdivisión operativa de una empresa. Toda empresa tiene al menos una. |
| **Rol** | Conjunto de permisos, definido a nivel empresa, reutilizable en todas sus sucursales (ver §5). |
| **Permiso** | Capacidad atómica de hacer algo (ej: `products.create`). Catálogo fijo por sistema (ver §5.1). |
| **Asignación (User Role)** | Vínculo `usuario + rol + sucursal`. Es la unidad central del modelo (ver §5.1). |
| **Owner** | El usuario que creó la empresa. Es admin, pero además es el dueño (único). Diferencias con admin en §5.2. |
| **Admin** | Rol por defecto, inmutable, con todos los permisos de la empresa. Puede haber varios **usuarios** con este rol en la misma empresa (ver §5.2). |
| **Superadmin** | Dueño del software (Eurekant o el cliente que lo comercializa). Vista global del sistema (ver §11). |
| **Contexto activo** | La combinación empresa + sucursal en la que el usuario está operando en este momento (ver §9). También llamado *tenant context* en la literatura; equivale a decir "empresa/sucursal activa". |
| **Initial setup** | Función interna que popula los datos iniciales al crear una empresa (ver §7). |

---

## 4. Modelo de tenancy: Usuario → Empresa → Sucursal

La jerarquía base de todo sistema Eurekant:

```mermaid
flowchart TD
    SYS["🌐 Sistema (instancia del software)"] --> C1["🏢 Empresa A"]
    SYS --> C2["🏢 Empresa B"]
    C1 --> B1["📍 Sucursal Centro"]
    C1 --> B2["📍 Sucursal Norte"]
    C2 --> B3["📍 Sucursal Única"]
    B1 --> U1["👤 Juan — rol Cajero"]
    B2 --> U2["👤 Juan — rol Encargado"]
    B2 --> U3["👤 Ana — rol Cajero"]
    B3 --> U4["👤 Ana — rol Admin"]
```

Puntos clave del diagrama:

- **Juan** tiene **dos roles distintos en dos sucursales** de la Empresa A. Esto es válido y esperado.
- **Ana** pertenece a **dos empresas distintas** con la misma cuenta. También es válido.
- Los roles "Cajero" y "Encargado" de la Empresa A **no existen** en la Empresa B; cada empresa define los suyos (excepto `admin`, que existe en todas).

> 💡 **Ejemplo práctico — multi-rol y multi-empresa**
> María es contadora. El estudio contable donde trabaja usa un sistema de Eurekant, y además dos de sus clientes (una pizzería y una farmacia) usan el mismo software. María tiene **una sola cuenta** con su email. Dentro del sistema: en "Estudio Contable Pérez" es *Admin*; en "Pizzería Don Carlo" tiene el rol *Contador externo* en la sucursal Centro; y en "Farmacia Vital" tiene el rol *Auditor* en las 3 sucursales. Cuando entra al sistema, elige en qué empresa/sucursal va a trabajar (ver §10, contexto activo).

### 4.1 Reglas del modelo de tenancy

- Toda empresa nace con **exactamente una sucursal** (creada por el *initial setup*). Aunque el negocio no use el concepto de "sucursal", existe una llamada "Principal" (u otro nombre por defecto definido por el sistema).
- Una sucursal pertenece a **una y solo una** empresa. No hay sucursales compartidas.
- Los datos operativos del sistema (productos, ventas, turnos, etc.) siempre cuelgan de la empresa, y cuando aplica, también de la sucursal.

---

## 5. Modelo de roles y permisos

### 5.1 Cómo se compone el acceso

```mermaid
flowchart LR
    P["📋 Catálogo de PERMISOS<br/>(fijo, definido por el sistema)"] -->|se agrupan en| R["🎭 ROLES<br/>(definidos por cada empresa)"]
    R -->|se asignan a| UR["🔗 ASIGNACIÓN<br/>usuario + rol + sucursal"]
    U["👤 USUARIO<br/>(identidad global)"] --> UR
    B["📍 SUCURSAL"] --> UR
    UR -->|determina| A["✅ Qué puede hacer<br/>y qué datos ve"]
```

**Catálogo de permisos.** Cada sistema define su catálogo de permisos atómicos con un código estandarizado `modulo.accion`. Por ejemplo: `products.create`, `products.read`, `products.update`, `products.delete`, `users.invite`, `roles.manage`, `reports.view`, `sales.refund`. El catálogo **no es editable por las empresas**: lo define el equipo de desarrollo y se carga como dato semilla. Es la frontera entre lo que el software *puede* hacer y lo que cada rol *permite* hacer.

**Por qué el formato `modulo.accion`.** Es la convención de la industria (los *scopes* de OAuth, los permisos de Google Cloud IAM como `storage.objects.create`) y aporta cuatro ventajas concretas:
1. **Namespacing** — `create` solo no dice nada, `products.create` es inequívoco y evita colisiones entre módulos;
2. **Agrupación automática** — la pantalla de edición de roles agrupa los checkboxes por módulo (todo lo que empieza con `sales.` va junto) sin necesidad de estructura extra;
3. **Legibilidad en código y logs** — un error `permission denied: sales.refund` se entiende al instante;
4. **Consistencia entre proyectos** — las acciones usan siempre el mismo vocabulario (`create`, `read`, `update`, `delete`, más verbos específicos como `refund` o `invite`), así el catálogo de cualquier sistema Eurekant se lee igual.

**Roles.** Un rol pertenece a una empresa y agrupa N permisos del catálogo. El usuario con permiso `roles.manage` (típicamente el admin) puede crear, modificar y eliminar roles de su empresa marcando/desmarcando permisos.

**Asignación.** La unidad central del modelo: `usuario + rol + sucursal`. Reglas:

- Un usuario puede tener **distintos roles en distintas sucursales** de la misma empresa.
- La sucursal de la asignación **debe pertenecer a la misma empresa** que el rol (regla de integridad obligatoria, ver §12).
- Para dar acceso a todas las sucursales, se crea una asignación por sucursal (la UI puede ofrecer un atajo "aplicar a todas las sucursales", pero internamente son N asignaciones).

> 💡 **Ejemplo práctico — armado de un rol**
> La Pizzería Don Carlo crea el rol **"Cajero"** con los permisos: `sales.create`, `sales.read`, `products.read`. Luego crea **"Encargado"** con todo lo del cajero más `sales.refund`, `products.update` y `reports.view`. Cuando contratan a Juan para la sucursal Centro, le asignan *Cajero en Centro*. Seis meses después lo ascienden en la sucursal Norte: se agrega la asignación *Encargado en Norte*, sin tocar la de Centro. Si la pizzería abre una tercera sucursal, los roles "Cajero" y "Encargado" **ya existen** y están listos para usarse: no hay que recrearlos.

### 5.2 Roles por defecto: `admin` y el concepto de Owner

- Al crear una empresa, el *initial setup* crea automáticamente el rol **`admin`**, marcado como rol por defecto (`is_default = true`).
- El rol `admin` **no se puede modificar ni eliminar**: siempre tiene todos los permisos del catálogo, incluidos los permisos nuevos que se agreguen en futuras versiones del software (regla: admin = unión de todo el catálogo, evaluada dinámicamente, no una lista congelada).
- Puede haber **varios usuarios admin** en una empresa: el admin puede asignar el rol admin a otros.
- El usuario que creó la empresa queda marcado como **Owner** (campo en la empresa que apunta a su usuario). El Owner es único, es admin como cualquier otro, pero:
  - No puede ser desactivado ni removido de la empresa por otros admins.
  - Es el único que puede transferir la propiedad (cambiar el Owner a otro usuario admin).
- **Regla Owner-admin:** el Owner siempre tiene el rol admin y nadie —**ni él mismo**— puede quitárselo; la única forma de que deje de ser admin es transferir el ownership a otro admin. Como consecuencia, una empresa **nunca puede quedar sin administrador** (siempre está el Owner como respaldo), y los demás admins sí pueden renunciar a su rol o ser removidos sin restricción.
- Cada sistema puede definir **roles plantilla adicionales** en su *initial setup* (ej: "Vendedor", "Supervisor") que, a diferencia de `admin`, **sí** son editables y eliminables por la empresa. Nacen como sugerencia, no como imposición.

> 💡 **Ejemplo práctico — owner vs admin**
> Carlos crea la cuenta de "Pizzería Don Carlo" → es Owner y admin. Luego le da rol admin a su socio Diego. Diego puede hacer todo lo que hace Carlos (crear roles, invitar gente, ver reportes), pero **no puede** sacarle el acceso a Carlos ni transferir la empresa. Si Carlos vende el negocio, él mismo transfiere el ownership a Diego desde la configuración.

### 5.3 Ciclo de vida de los roles

```mermaid
stateDiagram-v2
    [*] --> Activo : crear rol
    Activo --> Activo : editar nombre / permisos
    Activo --> BloqueoEliminacion : intentar eliminar con usuarios asignados
    BloqueoEliminacion --> Activo : reasignar usuarios primero
    Activo --> Inactivo : eliminar (sin usuarios activos)
    Inactivo --> [*]
    note right of BloqueoEliminacion
        El sistema bloquea la eliminación
        de un rol en uso. Primero hay que
        reasignar a los usuarios afectados.
    end note
    note right of Inactivo
        Soft delete - el registro queda
        para auditoría e historial.
    end note
```

- **No se puede eliminar un rol con usuarios activos asignados.** El sistema informa cuántos usuarios lo usan y exige reasignarlos primero. Esto evita usuarios "huérfanos" sin acceso de un día para el otro.
- La eliminación es siempre **soft delete**: el rol queda inactivo pero su registro persiste (los reportes históricos pueden seguir mostrando "venta cargada por Juan, rol Cajero" aunque el rol ya no exista).
- El rol `admin` nunca entra en este flujo: no es editable ni eliminable.

---

## 6. Modelo de entidades (conceptual)

Entidades nombradas según la [Naming Convention Guide](https://app.clickup.com/9002039309/v/dc/8c90e0d-10194/8c90e0d-6114): tablas en `UPPERCASE`, columnas en `lowercase` con prefijo descriptivo de su tabla, FKs con el mismo nombre que la PK referenciada.

```mermaid
erDiagram
    USERS ||--o{ USER_ROLES : "tiene asignaciones"
    COMPANIES ||--o{ BRANCHES : "posee"
    COMPANIES ||--o{ ROLES : "define"
    COMPANIES ||--o{ INVITATIONS : "emite"
    USERS ||--o{ COMPANIES : "es owner de"
    ROLES ||--o{ USER_ROLES : "se asigna en"
    BRANCHES ||--o{ USER_ROLES : "alcanza a"
    ROLES ||--o{ ROLE_PERMISSIONS : "agrupa"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "compone"
    ROLES ||--o{ INVITATIONS : "propone"
    BRANCHES ||--o{ INVITATIONS : "destina"
    USERS ||--o| SUPERADMINS : "puede ser"

    USERS {
        uuid user_id PK
        citext user_email UK "global, única"
        citext first_name
        citext last_name
        varchar avatar_url "nullable"
        boolean is_active
        timestamptz last_seen
        timestamptz created_at
    }
    COMPANIES {
        uuid company_id PK
        citext company_name
        uuid owner_id FK "USERS - dueño único"
        boolean is_active
        timestamptz created_at
    }
    BRANCHES {
        uuid branch_id PK
        uuid company_id FK
        citext branch_name "única por empresa"
        boolean is_active
        timestamptz created_at
    }
    ROLES {
        uuid role_id PK
        uuid company_id FK
        citext role_name "única por empresa"
        varchar descriptions "nullable"
        boolean is_default "true = admin, inmutable"
        boolean is_active
        timestamptz created_at
    }
    PERMISSIONS {
        uuid permission_id PK
        varchar permission_code UK "ej products.create"
        varchar permission_module "ej products"
        varchar descriptions
        boolean is_active
    }
    ROLE_PERMISSIONS {
        uuid role_permission_id PK
        uuid role_id FK
        uuid permission_id FK
    }
    USER_ROLES {
        uuid user_role_id PK
        uuid user_id FK
        uuid role_id FK
        uuid branch_id FK "misma empresa que el rol"
        boolean is_active
        timestamptz created_at
    }
    INVITATIONS {
        uuid invitation_id PK
        uuid company_id FK
        uuid branch_id FK
        uuid role_id FK
        citext invitation_email
        citext first_name "precargado por quien invita"
        citext last_name "precargado por quien invita"
        uuid invited_by FK "USER_ROLES de quien invita"
        varchar invitation_status "pending-accepted-expired-revoked"
        timestamptz expires_at
        timestamptz created_at
    }
    SUPERADMINS {
        uuid superadmin_id PK
        uuid user_id FK "única - un user es o no superadmin"
        boolean is_active
        timestamptz created_at
    }
    SYSTEM_SETTINGS {
        uuid system_setting_id PK
        varchar setting_key UK "ej mercadopago.commission"
        jsonb setting_value
        varchar descriptions
        uuid updated_by FK "SUPERADMINS"
        timestamptz updated_at
    }
```

### 6.1 Notas por entidad

**`USERS`** — Identidad global. Una fila por persona, vinculada al sistema de autenticación (en Supabase, referencia a `auth.users`). No contiene información de empresa: la pertenencia se expresa solo a través de `USER_ROLES`. El email es único y case-insensitive (`CITEXT`).

**`COMPANIES`** — El tenant. `owner_id` marca al dueño único (ver §5.2). Casi todas las tablas operativas de cada sistema (productos, ventas, etc.) referencian `company_id` directa o indirectamente, porque es la columna sobre la que pivota el RLS.

**`BRANCHES`** — Siempre existe al menos una por empresa. Las tablas operativas que tienen alcance de sucursal (ej: stock, cajas) referencian `branch_id`.

**`ROLES`** — Pertenecen a la empresa, no a la sucursal: se definen una vez y se usan en todas las sucursales. `is_default = true` identifica al rol `admin` (protegido contra edición/eliminación). `role_name` es único dentro de cada empresa (dos empresas distintas pueden tener cada una su rol "Cajero").

**`PERMISSIONS`** — Catálogo global del sistema (sin `company_id`). Se carga como dato semilla en cada deploy/migración. `permission_code` sigue el formato `modulo.accion`.

**`ROLE_PERMISSIONS`** — Tabla puente rol ↔ permiso. Única por combinación (un rol no puede tener el mismo permiso dos veces). El rol `admin` no necesita filas aquí: sus permisos son "todo el catálogo" por definición (evita tener que actualizarlo cuando se agregan permisos nuevos).

> **¿Por qué una tabla puente y no columnas booleanas en `ROLES`?** La relación rol ↔ permiso es muchos-a-muchos: un rol tiene N permisos y un mismo permiso está en N roles. Modelarlo como columnas (`order_c`, `order_u`, `menu_d`, …) implica que agregar un permiso nuevo requiere un `ALTER TABLE` + migración + tocar la UI, que la tabla `ROLES` sea estructuralmente distinta en cada proyecto (se rompe el estándar) y que la verificación de permisos no pueda ser una función genérica reutilizable. Con la tabla puente, agregar un permiso es un INSERT en el catálogo (la UI de roles lo muestra sola), las tablas son **idénticas en todos los proyectos** (solo cambia el contenido del catálogo) y `fn_has_permission('orders.create')` sirve igual en todos los sistemas. El costo de los joins se absorbe materializando los permisos efectivos en los claims del JWT al armar el contexto activo (§9): se calculan una vez por sesión, no en cada query.

**`USER_ROLES`** — El corazón del modelo. Combinación única de `user_id + role_id + branch_id`. Regla de integridad crítica: **la sucursal y el rol deben pertenecer a la misma empresa** (se validará con trigger/función en la v2). Es también la tabla que otras tablas referencian en campos de auditoría como `created_by` (según la convención de nombres, apuntando a `user_role_id`, lo que registra no solo *quién* sino *con qué rol y en qué sucursal* hizo la acción).

**`INVITATIONS`** — Registro completo del flujo de invitación (ver §8). Guarda el rol y la sucursal propuestos, los datos precargados de la persona y el estado del ciclo de vida.

**`SUPERADMINS`** — Lista blanca de usuarios con acceso global (ver §11). Deliberadamente fuera del modelo de roles de empresa.

**`SYSTEM_SETTINGS`** — Parametrización global del software, editable solo desde el panel superadmin (ver §11.2).

> 💡 **Ejemplo práctico — `created_by` apuntando a `USER_ROLES`**
> En el sistema de la pizzería, la tabla `ORDERS` tiene `created_by` → `USER_ROLES.user_role_id`. Cuando se audita una venta sospechosa, no solo se sabe que la hizo Juan: se sabe que la hizo **Juan actuando como Cajero en la sucursal Centro**, aunque hoy Juan ya sea Encargado. El contexto histórico queda congelado.

---

## 7. Flujo 1 — Registro inicial y *initial setup*

Cuando una persona se registra **por cuenta propia** (sin invitación), se asume que está creando una empresa nueva y será su administrador y owner.

```mermaid
flowchart TD
    A["Persona entra a la pantalla de registro"] --> B["Ingresa su email"]
    B --> C["Sistema envía código de 6 dígitos al email"]
    C --> D{"¿Código válido?"}
    D -- No --> C2["Reintentar / reenviar código"] --> C
    D -- Sí --> E["Completa datos personales:<br/>nombre, apellido, contraseña"]
    E --> F["Completa datos de la empresa:<br/>nombre y datos según el sistema"]
    F --> G["⚙️ initial setup (transacción única en BD)"]
    G --> G1["Crea USERS"]
    G --> G2["Crea COMPANIES con owner_id"]
    G --> G3["Crea BRANCHES 'Principal'"]
    G --> G4["Crea rol admin (is_default = true)"]
    G --> G5["Crea USER_ROLES: usuario + admin + sucursal"]
    G --> G6["Carga datos propios del sistema:<br/>plantillas de roles, categorías, datos demo, etc."]
    G1 & G2 & G3 & G4 & G5 & G6 --> H["✅ Usuario dentro del sistema<br/>como Owner / Admin de su empresa"]
```

Características del *initial setup*:

- Es una **función única en la base de datos** que se ejecuta como transacción: o se crea todo, o no se crea nada. Nunca puede quedar una empresa sin sucursal, sin rol admin o sin asignación.
- Su **núcleo es estándar** en todos los proyectos (empresa + sucursal + admin + asignación). Su **cola es específica** de cada sistema: cada software agrega ahí su carga inicial (categorías de ejemplo, configuración por defecto, datos de demo, roles plantilla).
- La validación del email con código de 6 dígitos es **obligatoria también en el registro propio**, no solo en el flujo de invitación. El flujo de validación es el mismo en ambos casos.

> 💡 **Ejemplo práctico — initial setup específico por sistema**
> En el sistema de turnos para clínicas, el *initial setup* además crea: los roles plantilla "Recepcionista" y "Profesional", una agenda de ejemplo y los horarios de atención por defecto. En el sistema de stock, crea: el rol plantilla "Depósito", una categoría "General" y un producto de ejemplo. El núcleo (empresa, sucursal, admin) es idéntico en ambos.

---

## 8. Flujo 2 — Invitación de usuarios

### 8.1 Visión general: los dos escenarios

Quien tenga el permiso `users.invite` puede invitar personas a una sucursal de su empresa. El sistema distingue dos escenarios según el email ingresado:

```mermaid
flowchart TD
    A["Usuario con permiso users.invite<br/>ingresa el email de la persona"] --> B{"¿El email ya tiene<br/>cuenta en el sistema?"}
    B -- "Sí (usuario existente)" --> C["Pantalla: seleccionar<br/>sucursal y rol"]
    C --> D["Se crea INVITATIONS<br/>(sin pedir datos personales)"]
    B -- "No (usuario nuevo)" --> E["Formulario: datos mínimos<br/>nombre, apellido y esenciales"]
    E --> F["Seleccionar sucursal y rol"]
    F --> G["Se crea INVITATIONS<br/>con datos precargados"]
    D & G --> H["📧 Email a la persona invitada:<br/>quién invita, a qué empresa,<br/>a qué sucursal y con qué rol"]
    H --> I{"¿Tenía cuenta?"}
    I -- Sí --> J["Acepta la invitación<br/>(login si hace falta)"]
    J --> K["Se crea USER_ROLES<br/>✅ acceso inmediato"]
    I -- No --> L["Botón 'Registrarme'<br/>→ flujo de registro por invitación (8.2)"]
```

Notas:

- En ambos escenarios el resultado intermedio es el mismo: una fila en `INVITATIONS` con empresa, sucursal, rol, email y quién invita.
- La **verificación de existencia** del email debe hacerse de forma segura: el sistema responde internamente si existe o no para ajustar el formulario, pero **no debe exponer** a cualquier usuario una API que permita enumerar qué emails tienen cuenta (punto de fuga clásico, ver §13).
- Para el usuario existente **no se piden datos personales** (ya los tiene su cuenta); solo confirma la invitación.

### 8.2 Registro por invitación (usuario nuevo) — paso a paso

```mermaid
sequenceDiagram
    actor I as Invitador (ej. admin)
    participant S as Sistema
    actor N as Persona invitada
    I->>S: Ingresa email + nombre/apellido + rol + sucursal
    S->>S: Crea INVITATIONS (status: pending, expira en 7 días)
    S->>N: 📧 Email: "Carlos te invita a Pizzería Don Carlo,<br/>sucursal Centro, como Cajero" + botón Registrarse
    N->>S: Click en "Registrarse"
    S->>N: Pide el email al que fue invitado
    N->>S: Ingresa el email y toca "Validar correo"
    S->>N: 📧 Código de 6 dígitos
    N->>S: Ingresa el código
    S->>S: ✅ Email validado y coincide con la invitación
    S->>N: Muestra datos precargados (nombre, apellido)
    N->>S: Confirma o corrige sus datos
    N->>S: Crea su contraseña
    S->>S: Crea USERS + USER_ROLES (usuario + rol + sucursal)
    S->>S: INVITATIONS → status: accepted
    S->>N: ✅ Cuenta creada, acceso a la sucursal con el rol asignado
```

Detalles importantes:

- El registro por invitación es **el mismo flujo** que el registro por cuenta propia, con tres diferencias: (a) llega por email en vez de iniciarse solo, (b) los datos personales vienen **precargados** por quien invitó y la persona puede corregirlos, y (c) **no se ejecuta el initial setup** ni se piden datos de empresa — la persona entra a una empresa existente, no crea una.
- El email que la persona valida con el código **debe coincidir** con el email de la invitación. Si no coincide, no se vincula la invitación.
- La persona invitada es la dueña final de sus datos: lo que el invitador escribió (nombre, apellido) es solo una sugerencia editable.

### 8.3 Ciclo de vida de una invitación

```mermaid
stateDiagram-v2
    [*] --> Pendiente : se crea y envía
    Pendiente --> Aceptada : la persona completa el flujo
    Pendiente --> Revocada : el invitador la cancela
    Pendiente --> Expirada : pasan 7 días
    Expirada --> Pendiente : reenviar (nueva expiración)
    Aceptada --> [*]
    Revocada --> [*]
```

Reglas:

- **Expiración:** 7 días (valor parametrizable desde `SYSTEM_SETTINGS`). Una invitación expirada puede reenviarse, lo que renueva la fecha.
- **Revocación:** quien tenga `users.invite` puede revocar una invitación pendiente (ej: se equivocó de email o la persona ya no se incorpora). Una invitación revocada o aceptada no puede reutilizarse.
- **Unicidad:** solo puede existir **una invitación pendiente por email + empresa**. Si se quiere cambiar el rol o la sucursal antes de que la acepte, se revoca y se crea una nueva.
- **Validaciones al crear:** no se puede invitar a un email que ya tiene asignación activa en esa misma sucursal con ese mismo rol; y el rol y la sucursal de la invitación deben pertenecer a la empresa del invitador.
- **Validaciones al aceptar:** si entre el envío y la aceptación el rol o la sucursal fueron desactivados, la invitación se considera inválida y se informa a la persona (y al invitador) para que se genere una nueva.

> 💡 **Ejemplo práctico — invitación con typo**
> El admin invita a `jaun@gmail.com` en lugar de `juan@gmail.com`. Se da cuenta al día siguiente: revoca la invitación pendiente y crea una nueva con el email correcto. Si el dueño real de `jaun@gmail.com` intentara usar el link viejo, vería "invitación revocada" y no podría acceder a nada.

---

## 9. Contexto activo: en qué empresa y sucursal estoy parado

Como un usuario puede tener N asignaciones, el sistema necesita saber **cuál está usando ahora**. Ese es el **contexto activo**: una combinación `empresa + sucursal` elegida por el usuario.

```mermaid
flowchart TD
    A["👤 Login de María"] --> B{"¿Cuántas asignaciones<br/>activas tiene?"}
    B -- "Una sola" --> C["Contexto activo automático:<br/>esa empresa + sucursal"]
    B -- "Varias" --> D["Selector: lista de empresas<br/>y sucursales disponibles"]
    D --> E["María elige:<br/>Pizzería Don Carlo / Centro"]
    C & E --> F["El contexto activo viaja en la sesión<br/>(claims del token JWT)"]
    F --> G["RLS filtra TODO según ese contexto"]
    G --> H["María puede cambiar de contexto<br/>desde el menú, sin re-loguearse"]
    H --> F
```

Reglas:

- Si el usuario tiene **una sola** asignación activa, el contexto se establece solo, sin pantalla intermedia (el caso más común: empleados de una sola empresa no deben enterarse de que el sistema es multi-tenant).
- El contexto activo (y los permisos efectivos del usuario en ese contexto) se materializa en los **claims del token de sesión (JWT)**, para que el RLS pueda evaluarlo sin subconsultas costosas. Este es el patrón recomendado por Supabase y la industria.
- Cambiar de contexto refresca el token con los nuevos claims. No requiere cerrar sesión.
- El sistema recuerda el último contexto usado para preseleccionarlo en el próximo login.

---

## 10. RLS: aislamiento de datos sin filtros en el código

### 10.1 El principio

**El código de aplicación nunca filtra por empresa o sucursal.** Cuando el frontend o el backend consulta una tabla, escribe la query "ingenua" (`select * from PRODUCTS`) y la base de datos, mediante Row Level Security, devuelve **solo** las filas que el contexto activo del usuario puede ver.

```mermaid
flowchart LR
    subgraph APP["Aplicación (frontend / API)"]
        Q["select * from PRODUCTS<br/>(sin ningún filtro)"]
    end
    subgraph DB["Base de datos"]
        RLS["🛡️ Políticas RLS<br/>comparan company_id / branch_id<br/>de cada fila contra el contexto<br/>activo del token"]
        T["Tabla PRODUCTS<br/>100 productos de 5 empresas"]
    end
    Q --> RLS --> T
    T --> RLS
    RLS -->|"solo las 12 filas<br/>de MI empresa"| Q
```

> 💡 **Ejemplo práctico — el caso de los 100 productos**
> En la tabla `PRODUCTS` hay 100 productos de 5 empresas distintas. Juan (Cajero de la Pizzería, sucursal Centro) consulta la tabla **sin ningún filtro** y recibe exactamente los 12 productos de la pizzería. No hay forma de que reciba otros: aunque un desarrollador se olvide de todo, aunque la query venga de un script externo con el token de Juan, el RLS está en la base y es la última línea de defensa. El clásico bug de "me olvidé el `where company_id = ...`" **deja de existir como categoría de bug**.

### 10.2 Cómo funcionará (conceptual, el detalle va en la v2)

- Toda tabla operativa lleva `company_id` (y `branch_id` cuando su alcance es por sucursal). Estas columnas estarán **indexadas** siempre — es el principal factor de performance del RLS.
- Las políticas comparan esas columnas contra el **contexto activo en los claims del JWT** (empresa, sucursal y permisos), evitando subconsultas pesadas en cada fila.
- El RLS cubre los cuatro verbos: lectura (qué filas veo), inserción (no puedo insertar filas de otra empresa — cláusula de verificación), actualización y borrado.
- Los **permisos** también se evalúan en la base cuando corresponde: por ejemplo, insertar en `PRODUCTS` exige el permiso `products.create` en el contexto activo, no solo pertenecer a la empresa. Habrá funciones auxiliares estándar (`fn_has_permission`, etc.) reutilizables en todos los proyectos.
- **Alcance empresa vs. sucursal según la tabla:** hay tablas donde todos los miembros de la empresa ven lo mismo (ej: catálogo de productos) y tablas donde solo se ve lo de la propia sucursal (ej: cajas, stock). Cada sistema define el alcance de cada tabla; el estándar provee ambos patrones de política.
- **Trade-off conocido:** como los permisos viajan en el JWT, un cambio de rol/permisos tarda hasta la expiración del token en aplicarse (minutos). Para acciones críticas (desactivar un usuario) se complementa con verificación en base, que es inmediata. Este trade-off es estándar en la industria.

### 10.3 Qué ve cada capa

| Capa | Responsabilidad sobre el acceso |
|---|---|
| **Frontend** | Solo UX: oculta botones que el rol no puede usar. **Nunca** es seguridad. |
| **API / backend** | Lógica de negocio (validaciones, flujos). No filtra por tenant. |
| **Base de datos (RLS)** | Fuente única de verdad del aislamiento. Filtra y bloquea siempre. |

---

## 11. Superadmin: la capa del dueño del software

### 11.1 Concepto

El **superadmin** no es un rol de empresa: es una capa completamente separada para los dueños del software (Eurekant o el cliente que comercializa el sistema). Vive en su propia tabla (`SUPERADMINS`), tiene su propio panel (back-office) y **no aparece como miembro de ninguna empresa**.

```mermaid
flowchart TD
    subgraph SA["🛰️ Capa Superadmin (back-office)"]
        M["📊 Métricas globales:<br/>empresas, usuarios totales,<br/>actividad y horarios de uso"]
        P["⚙️ SYSTEM_SETTINGS:<br/>parametrización global"]
        G["🏢 Gestión de tenants:<br/>activar / suspender empresas"]
    end
    subgraph T1["🏢 Empresa A"]
        A1["Admin / Owner"] --> S1["Sucursales, roles, usuarios"]
    end
    subgraph T2["🏢 Empresa B"]
        A2["Admin / Owner"] --> S2["Sucursales, roles, usuarios"]
    end
    SA -.->|"vista global<br/>(lectura y administración)"| T1
    SA -.->|"vista global<br/>(lectura y administración)"| T2
```

Capacidades del superadmin (lista inicial, ampliable por sistema):

- **Métricas y monitoreo:** cantidad de empresas y su estado, usuarios totales y activos, métricas de uso, horarios de mayor actividad, crecimiento.
- **Gestión de tenants:** ver, activar y suspender empresas (ej: por falta de pago).
- **Parametrización global:** todo lo configurable del software se configura acá (ver §11.2).
- El RLS lo trata con **políticas especiales explícitas**: el superadmin ve los datos globales y agregados que su panel necesita. Importante: el acceso superadmin **no es un bypass silencioso**, sino políticas deliberadas y auditables.

### 11.2 Parametrización global (`SYSTEM_SETTINGS`)

Todo valor del sistema que pueda cambiar sin redeploy se guarda como parámetro clave-valor, editable solo desde el panel superadmin: comisiones (ej: `mercadopago.commission`), días de expiración de invitaciones (`invitations.expiration_days`), límites, textos legales, banderas de funcionalidades, etc.

> 💡 **Ejemplo práctico — comisiones de Mercado Pago**
> El sistema de la pizzería cobra las ventas online con Mercado Pago. MP cambia su comisión del 6,4% al 6,8%. Sin este modelo, habría que tocar código y redeployar. Con `SYSTEM_SETTINGS`, el superadmin entra al back-office, edita `mercadopago.commission` y el cambio aplica al instante en todos los cálculos. Queda registrado quién lo cambió y cuándo.

> 💡 **Ejemplo práctico — métricas de uso**
> Eurekant quiere saber si conviene programar los mantenimientos a la madrugada. El panel superadmin muestra que el 80% de la actividad ocurre entre 9 y 21 hs, y que los domingos el uso cae al 5%. Decisión tomada con datos, sin tocar la base a mano.

---

## 12. Reglas de negocio e integridad (resumen normativo)

Estas reglas son **obligatorias** en todos los proyectos. En la v2, cada una se implementará con constraints, triggers o funciones.

| # | Regla |
|---|---|
| RN-01 | Toda empresa tiene al menos una sucursal, siempre (la crea el *initial setup*). |
| RN-02 | El rol `admin` existe en toda empresa, tiene todos los permisos del catálogo (evaluación dinámica) y no puede modificarse ni eliminarse. |
| RN-03 | **Regla Owner-admin:** toda empresa tiene exactamente un Owner, que siempre tiene el rol admin. Nadie —ni él mismo— puede quitarle el rol ni desactivarlo; la única forma de dejar de ser admin es transferir la propiedad (a otro admin). Como consecuencia, una empresa nunca queda sin administrador; los demás admins sí pueden renunciar o ser removidos. |
| RN-04 | En `USER_ROLES`, la sucursal y el rol deben pertenecer a la **misma empresa**. |
| RN-05 | La combinación `usuario + rol + sucursal` es única (no se duplica una asignación). |
| RN-06 | No se puede eliminar un rol con asignaciones activas; primero se reasignan los usuarios. Toda eliminación es soft delete. |
| RN-07 | Una invitación pendiente es única por `email + empresa`; expira (parametrizable, default 7 días); es revocable; y al aceptarse valida que el rol y la sucursal sigan activos. |
| RN-08 | La baja de un usuario de una empresa es soft delete: pierde acceso de inmediato, el historial queda intacto y puede reactivarse. |
| RN-09 | El email de un usuario es único y case-insensitive a nivel global del sistema. |
| RN-10 | Los nombres de rol y de sucursal son únicos **dentro de su empresa** (case-insensitive). |
| RN-11 | Toda tabla operativa lleva `company_id` (y `branch_id` si su alcance es por sucursal), con RLS activo e índices sobre esas columnas. Sin excepciones. |
| RN-12 | El catálogo `PERMISSIONS` solo lo modifica el equipo de desarrollo (datos semilla); las empresas no lo editan. |
| RN-13 | El acceso superadmin se define en políticas explícitas y auditables, nunca como bypass genérico. |
| RN-14 | Los campos de auditoría (`created_by`, `updated_by`) referencian `USER_ROLES`, no `USERS`, para congelar el contexto (quién, con qué rol, en qué sucursal). |

---

## 13. Casos borde y puntos de fuga analizados

Análisis de escenarios problemáticos y cómo el modelo los resuelve:

| Escenario | Riesgo | Resolución en el modelo |
|---|---|---|
| Verificar si un email existe al invitar | Enumeración de cuentas: cualquiera podría descubrir qué emails usan el sistema | La verificación ocurre del lado del servidor solo para usuarios con `users.invite`, con límite de intentos. La respuesta pública nunca confirma existencia de cuentas. |
| Invitación aceptada después de que el rol/sucursal fue eliminado | Asignación rota o acceso a algo inexistente | Validación al aceptar (RN-07): la invitación se invalida y se notifica para regenerarla. |
| Todos los admins se van de la empresa | Empresa inaccesible para siempre | RN-03 (regla Owner-admin): el Owner siempre es admin y nadie puede quitarle el rol, así que siempre hay al menos un admin. |
| Eliminar un rol en uso | Usuarios sin acceso de un día para el otro | RN-06: eliminación bloqueada hasta reasignar. |
| Usuario desvinculado conserva token JWT vigente | Acceso residual por minutos | Trade-off conocido (§10.2): tokens de vida corta + verificación en base para acciones críticas. |
| Dos roles distintos del mismo usuario en la misma sucursal | Ambigüedad de permisos | Permitido: los permisos efectivos son la **unión** de ambos roles. (Decisión a validar, ver §14.2.) |
| Sistemas "chicos" que no usan sucursales | Tentación de simplificar el modelo y romper el estándar | Prohibido por principio 1: siempre existen `COMPANIES` y `BRANCHES`, aunque tengan una fila. La UI puede ocultar el concepto. |
| Empresa suspendida (ej: falta de pago) | Usuarios siguen operando | `COMPANIES.is_active = false` corta el acceso vía RLS a todos sus miembros de inmediato, sin tocar sus asignaciones. |
| Invitador escribe mal los datos del invitado | Datos incorrectos permanentes | La persona invitada revisa y corrige sus datos al registrarse (§8.2). |
| Sucursal desactivada con usuarios asignados | Asignaciones colgando de algo inactivo | Las asignaciones de esa sucursal quedan inactivas en cascada lógica; si un usuario queda sin ninguna asignación activa, no puede ingresar a esa empresa. |
| Borrado físico de usuarios | Historial y auditoría rotos | RN-08 y RN-14: soft delete + auditoría sobre `USER_ROLES`. |
| Mismo nombre de rol en empresas distintas | Colisión de nombres | No hay colisión: la unicidad es por empresa (RN-10). |

---

## 14. Decisiones de diseño y preguntas abiertas

### 14.1 Decisiones confirmadas (validadas con Franco, 09/06/2026)

1. **Permisos granulares** con catálogo por sistema; los roles agrupan permisos.
2. **Contexto activo seleccionable**: una cuenta global, selector de empresa/sucursal, cambio sin re-login.
3. **Eliminación de roles bloqueada** si hay usuarios asignados; todo soft delete.
4. **Superadmin como modelo separado** (tabla y panel propios, fuera del modelo de empresas).
5. **Multi-admin con Owner**: varios admins posibles; el creador queda marcado como Owner único.
6. **Asignación solo por sucursal** (la UI puede ofrecer "aplicar a todas").
7. **Invitaciones que expiran (7 días) y son revocables**, con estados auditables.
8. **Baja de usuarios por desactivación** (soft delete) con historial intacto.

### 14.2 Preguntas abiertas (a definir antes de la v2)

1. **Multi-rol en la misma sucursal:** ¿confirmamos que un usuario puede tener 2+ roles en la misma sucursal (permisos = unión)? Propuesta: sí, simplifica casos como "Cajero" + "Responsable de caja fuerte". Alternativa: limitar a un rol por sucursal.
2. **Transferencia de ownership:** ¿hace falta un flujo con confirmación del receptor (acepta ser Owner) o alcanza con la acción unilateral del Owner actual?
3. **Permisos a nivel empresa vs. sucursal en `SYSTEM_SETTINGS`:** ¿algunos parámetros podrán tener override por empresa (ej: comisión especial negociada con un cliente grande)? Propuesta: contemplar un alcance opcional por empresa en la v2.
4. **Auditoría formal:** ¿incluimos en el estándar una tabla de log de eventos sensibles (cambios de roles, invitaciones, transferencias de ownership)? Propuesta: sí, como entidad estándar en la v2.
5. **Invitaciones masivas:** ¿se necesita invitar por lote (CSV / múltiples emails)? Puede diseñarse después sin tocar el modelo.

---

## 15. Inspiración y referencias

El modelo sigue los patrones de la industria para SaaS multi-tenant:

- Organización → espacios → roles con permisos granulares y miembros multi-organización: patrón de **Slack, Notion y Google Workspace**.
- RBAC multi-tenant con roles tenant-scoped y catálogo de permisos atómicos expuesto como "bundles": [WorkOS — How to design multi-tenant RBAC for SaaS](https://workos.com/blog/how-to-design-multi-tenant-rbac-saas), [Aserto — Multi-tenant RBAC](https://www.aserto.com/blog/authorization-101-multi-tenant-rbac), [AWS Prescriptive Guidance — Multi-tenant access control](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/avp-mt-abac-examples.html).
- Aislamiento por columna de tenant + RLS como última línea de defensa, con claims en JWT e índices sobre las columnas de tenant: [Permit.io — Best practices for multi-tenant authorization](https://www.permit.io/blog/best-practices-for-multi-tenant-authorization), [Clerk — Multi-tenant SaaS architecture](https://clerk.com/blog/how-to-design-multitenant-saas-architecture), [Supabase — Custom claims & RLS](https://github.com/orgs/supabase/discussions/1148).
- Nomenclatura de base de datos: [Eurekant Naming Convention Guide](https://app.clickup.com/9002039309/v/dc/8c90e0d-10194/8c90e0d-6114).

## 16. Próximos pasos

1. Validar este documento con el equipo (especialmente §14.2).
2. **v2:** DDL completo en SQL — tablas, constraints, índices, funciones (`fn_initial_setup`, `fn_has_permission`), triggers de integridad (RN-03, RN-04) y políticas RLS, todo según la Naming Convention Guide.
3. **v3:** kit reutilizable (migraciones base + seeds del catálogo de permisos) para iniciar cualquier proyecto nuevo de Eurekant con este cimiento ya instalado.
