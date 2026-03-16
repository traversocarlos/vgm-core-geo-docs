# Arquitectura visual — VGM Core Geo

> Diagramas de referencia. Se renderizan en GitHub y en cualquier editor compatible con Mermaid.
> Última actualización: 2026-03-16

---

## 1. Contexto del sistema

Quién usa VGM Core Geo, qué sistemas lo rodean y cómo se conectan.

```mermaid
graph TB
    subgraph usuarios [Usuarios]
        admin["👤 Admin / Operador\nWeb browser"]
        gema["📱 GEMA\nApp Android"]
        tracker["📡 GPS Tracker\nHardware"]
    end

    subgraph geo [VGM Core Geo]
        web["Frontend\nReact 18 + TypeScript\nLeaflet + OpenStreetMap"]
        api["Backend\nKotlin 2.1 + Spring Boot 3.5\nJava 21"]
        db[("PostgreSQL 17\nBase de datos propia")]
    end

    subgraph vgmcore [VGM Core — por Mauricio]
        core_api["Backend\nKotlin + Spring Boot"]
        core_db[("PostgreSQL\nBase de datos propia")]
    end

    subgraph legacy [Sistema legacy]
        vgmdis["VGMDIS\nPowerBuilder / SQL Server"]
        bridge["Bridge\nlee v_posiciones_nuevas"]
    end

    auth0["🔐 Auth0\nvgm-core-dev.us.auth0.com\nEtapa 2"]
    osm["🗺️ OpenStreetMap\nTiles gratuitos — sin API key"]

    admin -->|HTTPS| web
    web -->|REST API| api
    gema -->|POST /api/v1/posiciones\nAPI Key| api
    tracker -->|POST /api/v1/posiciones\nAPI Key| api
    vgmdis --> bridge
    bridge -->|POST /api/v1/posiciones\nAPI Key| api
    api --- db
    api <-->|"REST (futuro)\nid_publico_core"| core_api
    core_api --- core_db
    web -->|Tiles HTTP| osm
    api -.->|"Etapa 2\nvalida JWT"| auth0
```

---

## 2. Jerarquía de tenancy

Cómo se organizan los datos dentro de VGM Core Geo. Idéntica a VGM Core.

```mermaid
graph TD
    A["🏢 clientes_saas\n'D OUNI'\nEl cliente que contrató VGM Core Geo"]

    A --> B["🏭 empresas\n'OV'"]
    A --> C["🏭 empresas\n'ZIRKA'"]
    A --> D["🏭 empresas\n'OV2'"]

    B --> E["🏪 sucursales\n'Centro'"]
    B --> F["🏪 sucursales\n'Norte'"]
    C --> G["🏪 sucursales\n'Única'"]
    D --> H["🏪 sucursales\n'Única'"]

    E --> E1["👷 empleados\nVENDEDOR / REPARTIDOR\n/ SUPERVISOR"]
    E --> E2["📍 puntos_venta"]
    E --> E3["🗺️ zonas\n(polígonos GeoJSON)"]

    F --> F1["👷 empleados"]
    F --> F2["📍 puntos_venta"]
    F --> F3["🗺️ zonas"]

    style A fill:#dbeafe,stroke:#3b82f6
    style B fill:#dcfce7,stroke:#22c55e
    style C fill:#dcfce7,stroke:#22c55e
    style D fill:#dcfce7,stroke:#22c55e
    style E fill:#fef9c3,stroke:#eab308
    style F fill:#fef9c3,stroke:#eab308
    style G fill:#fef9c3,stroke:#eab308
    style H fill:#fef9c3,stroke:#eab308
```

---

## 3. Modelo de datos — tablas y relaciones

```mermaid
erDiagram
    clientes_saas {
        bigint id_cliente_saas PK
        uuid id_publico
        varchar co_cliente_saas
        varchar de_cliente_saas
        boolean sn_activo
        timestamptz fe_alta
    }

    empresas {
        bigint id_empresa PK
        uuid id_publico
        bigint id_cliente_saas FK
        varchar co_empresa
        varchar de_empresa
        boolean sn_activo
    }

    sucursales {
        bigint id_sucursal PK
        uuid id_publico
        bigint id_cliente_saas FK
        bigint id_empresa FK
        varchar co_sucursal
        varchar de_sucursal
        boolean sn_activo
    }

    cuentas {
        bigint id_cuenta PK
        uuid id_publico
        varchar de_email
        varchar de_nombre
        varchar co_sub_oidc
        boolean sn_activo
    }

    usuarios_geo {
        bigint id_usuario_geo PK
        bigint id_cuenta FK
        bigint id_cliente_saas FK
        boolean sn_activo
    }

    usuarios_geo_sucursales {
        bigint id_usuario_geo FK
        bigint id_sucursal FK
        varchar co_rol
        boolean sn_activo
    }

    empleados {
        bigint id_empleado PK
        uuid id_publico
        bigint id_sucursal FK
        varchar co_empleado
        varchar de_nombre
        varchar co_tipo
        boolean sn_registra_coords
        uuid id_publico_core
    }

    posiciones {
        bigint id_posicion PK
        bigint id_empleado FK
        numeric nu_latitud
        numeric nu_longitud
        numeric nu_precision
        numeric nu_velocidad
        varchar co_tipo_operacion
        timestamptz fe_posicion
        timestamptz fe_recibida
    }

    puntos_venta {
        bigint id_punto_venta PK
        uuid id_publico
        bigint id_sucursal FK
        varchar co_punto_venta
        varchar de_nombre
        numeric nu_latitud
        numeric nu_longitud
        uuid id_publico_core
    }

    zonas {
        bigint id_zona PK
        bigint id_sucursal FK
        varchar de_nombre
        varchar de_color
        jsonb de_coordenadas
        boolean sn_activo
    }

    clientes_saas ||--o{ empresas : tiene
    clientes_saas ||--o{ sucursales : tiene
    clientes_saas ||--o{ usuarios_geo : tiene
    empresas ||--o{ sucursales : tiene
    sucursales ||--o{ empleados : tiene
    sucursales ||--o{ puntos_venta : tiene
    sucursales ||--o{ zonas : tiene
    sucursales ||--o{ usuarios_geo_sucursales : tiene
    empleados ||--o{ posiciones : registra
    cuentas ||--o{ usuarios_geo : tiene
    usuarios_geo ||--o{ usuarios_geo_sucursales : accede
```

---

## 4. Flujo de autenticación — Etapa 1 (JWT propio)

```mermaid
sequenceDiagram
    actor U as Usuario
    participant W as Frontend React
    participant B as Backend Kotlin
    participant C as Caffeine Cache
    participant DB as PostgreSQL

    U->>W: Abre VGM Core Geo
    W->>W: No tiene sesión
    W->>U: Muestra pantalla de login

    U->>W: email + contraseña
    W->>B: POST /api/v1/auth/login
    B->>DB: busca en cuentas + usuarios_geo
    DB-->>B: usuario válido
    B-->>W: JWT firmado
    Note right of W: {sub, tenant_id, email, exp}
    W->>W: Guarda JWT en memoria
    Note right of W: NUNCA en localStorage

    rect rgb(240, 253, 244)
        Note over W,DB: Requests posteriores (por cada llamada a la API)
        W->>B: GET /api/v1/empleados<br/>Authorization: Bearer JWT<br/>X-Empresa-Id: 10<br/>X-Sucursal-Id: 3
        B->>B: Spring Security valida firma JWT
        B->>C: busca usuario por sub + tenant_id
        alt cache miss
            C-->>B: no encontrado
            B->>DB: busca en cuentas + usuarios_geo_sucursales
            DB-->>B: membresía + sucursales habilitadas
            B->>C: guarda (TTL 5 min)
        else cache hit
            C-->>B: datos cacheados
        end
        B->>B: TenantContext.establecer(DatosTenant)
        B->>DB: @Transactional + SET LOCAL
        Note right of DB: SET LOCAL app.id_cliente_saas = 1<br/>SET LOCAL app.id_empresa = 10<br/>SET LOCAL app.id_sucursal = 3
        DB-->>B: lista de empleados
        B-->>W: 200 OK + JSON
    end
```

---

## 5. Ingesta de posiciones GPS

Seis fuentes distintas, todas apuntan al mismo endpoint con API Key.

```mermaid
flowchart LR
    gema["📱 GEMA\nApp Android"]
    bridge["🔄 Bridge\nVGMDIS legacy"]
    tracker["📡 GPS Tracker\nhardware"]
    externo["🔌 Sistema externo\nAPI-to-API"]
    archivo["📂 Importación\narchivo CSV/JSON"]
    manual["✏️ Carga manual\ndesde web"]

    endpoint["POST /api/v1/posiciones\nAutenticación: API Key\nuna key por origen"]

    db[("posiciones\nid_cliente_saas\nid_empresa\nid_sucursal\nid_empleado\nnu_latitud / nu_longitud\nfe_posicion")]

    gema -->|API Key GEMA| endpoint
    bridge -->|API Key Bridge| endpoint
    tracker -->|API Key Tracker| endpoint
    externo -->|API Key Externo| endpoint
    archivo -->|API Key Archivo| endpoint
    manual -->|sesión del usuario| endpoint

    endpoint --> db

    style endpoint fill:#dbeafe,stroke:#3b82f6
    style db fill:#fef9c3,stroke:#eab308
```

---

## 6. Cadena de filtros por request

Cómo viaja un request HTTP desde que llega hasta que el controller lo procesa.

```mermaid
flowchart TD
    req["Request HTTP\nAuthorization: Bearer JWT\nX-Empresa-Id: 10\nX-Sucursal-Id: 3"]

    ss["1. Spring Security\nvalida firma del JWT\n→ 401 si token inválido"]

    tf["2. TenantContextFilter\nextrae claims del JWT\nresuelve empresa desde X-Empresa-Id\nresuelve sucursal desde X-Sucursal-Id\ncarga rol desde BD / caché\n→ 403 si sin membresía\n→ 400 si empresa/sucursal ambigua"]

    tc["3. TenantContext (ThreadLocal)\nidClienteSaas, idEmpresa, idSucursal\nidUsuarioGeo, email, sub"]

    ctrl["4. Controller\nrecibe request\nusa TenantContextProvider\npara acceder al contexto"]

    tx["5. @Transactional\nTenantConnectionPreparer (AOP)\nejecutaSET LOCAL en la conexión JPA\nantes de cualquier query"]

    db["6. PostgreSQL\ncada query ya tiene\napp.id_cliente_saas, app.id_empresa\napp.id_sucursal disponibles\n(base para RLS futuro)"]

    clean["7. finally\nTenantContext.limpiar()\nsiempre, incluso con excepción"]

    req --> ss
    ss --> tf
    tf --> tc
    tc --> ctrl
    ctrl --> tx
    tx --> db
    db --> clean

    style ss fill:#fee2e2,stroke:#ef4444
    style tf fill:#dbeafe,stroke:#3b82f6
    style tc fill:#dcfce7,stroke:#22c55e
    style tx fill:#fef9c3,stroke:#eab308
    style clean fill:#f3e8ff,stroke:#a855f7
```

---

## 7. Roadmap de implementación

```mermaid
gantt
    title VGM Core Geo — Fases de implementación
    dateFormat  YYYY-MM-DD
    axisFormat  %b %Y

    section Diseño
    Documentación y arquitectura     :done, doc, 2026-03-01, 2026-03-16
    Contrato OpenAPI                 :active, openapi, 2026-03-16, 7d

    section Backend — Fase 1
    Repo + estructura base (copiar VGM Core)  :repo, after openapi, 5d
    Migraciones Flyway (tablas base)          :flyway, after repo, 3d
    Módulo seguridad (tenancy + JWT)          :seg, after flyway, 7d
    POST /api/v1/posiciones                   :pos, after seg, 5d
    CRUD empleados + puntos_venta + zonas     :crud, after pos, 7d

    section Frontend — Fase 1
    Setup React + Leaflet + OpenStreetMap     :fe_setup, after openapi, 5d
    Login + sesión                            :fe_login, after fe_setup, 4d
    Mapa en tiempo real                       :fe_mapa, after fe_login, 10d
    ABM empleados / puntos / zonas            :fe_abm, after fe_mapa, 7d

    section Fase 2
    Auth0 (solo config, sin reescribir)       :auth0, 2026-06-01, 5d
    Row Level Security PostgreSQL             :rls, after auth0, 7d
    Integración REST con VGM Core             :integ, after rls, 14d
```
