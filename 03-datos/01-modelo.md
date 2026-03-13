# Modelo de datos

**Versión:** 1.0
**Fecha:** 2026-03-13
**Estado:** Activo

---

## Principio

VGM Go tiene su **propia base de datos PostgreSQL**. No comparte datos con VGM Core ni con ningún otro sistema. El modelo es simple y específico para el caso de uso de geolocalización — sin Party Model, sin RLS obligatorio.

Multi-tenancy: campo `id_empresa` en cada tabla + filtro en queries. Sin complejidad adicional.

---

## Tablas principales

### `empresas`
Cada empresa que usa VGM Go.

```sql
CREATE TABLE empresas (
    id_empresa      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico      uuid NOT NULL DEFAULT gen_random_uuid(),
    co_empresa      varchar(50) NOT NULL,
    de_empresa      varchar(200) NOT NULL,
    sn_activo       boolean NOT NULL DEFAULT true,
    fe_alta         timestamptz NOT NULL DEFAULT now()
);
```

---

### `empleados`
Vendedores y repartidores que hacen trabajo en campo.

```sql
CREATE TABLE empleados (
    id_empleado         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    co_empleado         varchar(50) NOT NULL,
    de_nombre           varchar(200) NOT NULL,
    co_tipo             varchar(20) NOT NULL,  -- 'VENDEDOR', 'REPARTIDOR'
    sn_registra_coords  boolean NOT NULL DEFAULT true,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),

    -- Campo puente para integración futura con VGM Core
    -- NULL hasta que VGM Core tenga empleados cargados
    id_publico_core     uuid NULL
);
```

> **Nota:** `id_publico_core` es el UUID del empleado en VGM Core. Vacío por ahora. Cuando VGM Core tenga su Party Model con empleados, este campo es el puente de integración sin tocar nada en producción.

---

### `posiciones`
Registro de coordenadas GPS recibidas.

```sql
CREATE TABLE posiciones (
    id_posicion         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_empresa          bigint NOT NULL,
    id_empleado         bigint NOT NULL REFERENCES empleados(id_empleado),
    nu_latitud          numeric(10,7) NOT NULL,
    nu_longitud         numeric(10,7) NOT NULL,
    nu_precision        numeric(8,2),
    nu_velocidad        numeric(8,2),
    co_tipo_operacion   varchar(30),  -- 'VENTA', 'COBRO', 'NO_ATENCION', 'PERIODICO'
    fe_posicion         timestamptz NOT NULL,
    fe_recibida         timestamptz NOT NULL DEFAULT now()
);
```

---

### `puntos_venta`
Comercios / puntos de venta con coordenadas.

```sql
CREATE TABLE puntos_venta (
    id_punto_venta      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    co_punto_venta      varchar(50) NOT NULL,
    de_nombre           varchar(200) NOT NULL,
    nu_latitud          numeric(10,7),
    nu_longitud         numeric(10,7),
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),

    -- Campo puente para integración futura con VGM Core (Party Model)
    -- NULL hasta que VGM Core tenga puntos de venta en entidades_direcciones
    id_publico_core     uuid NULL
);
```

> **Nota:** mismo concepto que en empleados. El ADR-009 de VGM Core define que los "comercios" del legacy se convierten en `entidades_direcciones` con coordenadas en Fase 2+. Cuando eso esté listo, `id_publico_core` es el puente.

---

### `zonas`
Zonas geográficas de cobertura (polígonos).

```sql
CREATE TABLE zonas (
    id_zona         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_empresa      bigint NOT NULL REFERENCES empresas(id_empresa),
    de_nombre       varchar(200) NOT NULL,
    de_color        varchar(7),   -- hex color para visualización en mapa
    de_coordenadas  jsonb NOT NULL,  -- array de puntos del polígono
    sn_activo       boolean NOT NULL DEFAULT true,
    fe_alta         timestamptz NOT NULL DEFAULT now()
);
```

---

### `usuarios_go` y `usuarios_go_empresas`
Usuarios del sistema y sus accesos por empresa.

```sql
CREATE TABLE usuarios_go (
    id_usuario      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico      uuid NOT NULL DEFAULT gen_random_uuid(),
    de_email        varchar(255) NOT NULL UNIQUE,
    de_nombre       varchar(200),
    co_sub_oidc     varchar(255),  -- para Auth0 en Etapa 2
    sn_activo       boolean NOT NULL DEFAULT true,
    fe_alta         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE usuarios_go_empresas (
    id_usuario      bigint NOT NULL REFERENCES usuarios_go(id_usuario),
    id_empresa      bigint NOT NULL REFERENCES empresas(id_empresa),
    co_rol          varchar(30) NOT NULL,  -- 'ADMIN', 'OPERADOR', 'READONLY'
    sn_activo       boolean NOT NULL DEFAULT true,
    PRIMARY KEY (id_usuario, id_empresa)
);
```

---

## Convenciones de nombres

Igual que VGM Core:

| Prefijo | Significado | Ejemplo |
|---|---|---|
| `id_` | Identificador interno | `id_empleado` |
| `co_` | Código funcional | `co_tipo` |
| `de_` | Descripción o texto | `de_nombre` |
| `nu_` | Número o medida | `nu_latitud` |
| `sn_` | Sí/No (boolean) | `sn_activo` |
| `fe_` | Fecha | `fe_alta` |
