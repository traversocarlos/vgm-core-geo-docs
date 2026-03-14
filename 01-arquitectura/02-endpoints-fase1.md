# Contrato de endpoints — Fase 1

**Versión:** 1.0
**Fecha:** 2026-03-14
**Estado:** Activo

> Diseño de referencia. La fuente de verdad es el código + `/v3/api-docs` (generado por springdoc-openapi).
> Ver convenciones generales en `01-arquitectura/01-convenciones-api.md`.

---

## Resumen de endpoints

| Método | Path | Descripción | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Login — emite JWT | Público |
| `GET` | `/api/v1/sesion/yo` | Datos del usuario actual | JWT |
| `GET` | `/api/v1/sesion/sucursales` | Sucursales disponibles del usuario | JWT |
| `POST` | `/api/v1/posiciones` | Ingresar posición GPS | API Key |
| `GET` | `/api/v1/posiciones` | Consultar posiciones con filtros | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/empleados` | Listar empleados | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/empleados/{id}` | Detalle de empleado | JWT + Empresa + Sucursal |
| `POST` | `/api/v1/empleados` | Crear empleado | JWT + Empresa + Sucursal |
| `PUT` | `/api/v1/empleados/{id}` | Actualizar empleado | JWT + Empresa + Sucursal |
| `DELETE` | `/api/v1/empleados/{id}` | Baja lógica de empleado | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/puntos-venta` | Listar puntos de venta | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/puntos-venta/{id}` | Detalle de punto de venta | JWT + Empresa + Sucursal |
| `POST` | `/api/v1/puntos-venta` | Crear punto de venta | JWT + Empresa + Sucursal |
| `PUT` | `/api/v1/puntos-venta/{id}` | Actualizar punto de venta | JWT + Empresa + Sucursal |
| `DELETE` | `/api/v1/puntos-venta/{id}` | Baja lógica | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/zonas` | Listar zonas | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/zonas/{id}` | Detalle de zona | JWT + Empresa + Sucursal |
| `POST` | `/api/v1/zonas` | Crear zona | JWT + Empresa + Sucursal |
| `PUT` | `/api/v1/zonas/{id}` | Actualizar zona | JWT + Empresa + Sucursal |
| `DELETE` | `/api/v1/zonas/{id}` | Baja lógica | JWT + Empresa + Sucursal |
| `GET` | `/actuator/health` | Health check | Público |

---

## Módulo autenticación

### `POST /api/v1/auth/login`

**No requiere autenticación.**

**Request body:**
```json
{
  "email": "usuario@empresa.com",
  "password": "contraseña"
}
```

**Response 200:**
```json
{
  "token": "eyJ...",
  "expiracion": "2026-03-14T11:30:00Z"
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 401 | `CREDENCIALES_INVALIDAS` | Email o contraseña incorrectos |
| 403 | `USUARIO_INACTIVO` | La cuenta existe pero está deshabilitada |

---

## Módulo sesión

### `GET /api/v1/sesion/yo`

Retorna los datos del usuario autenticado y sus sucursales disponibles.

**No requiere `X-Empresa-Id` ni `X-Sucursal-Id`** — opera solo con el `tenant_id` del JWT.

**Response 200:**
```json
{
  "idUsuarioGeo": 1,
  "email": "usuario@empresa.com",
  "sub": "auth0|abc123",
  "sucursales": [
    {
      "idSucursal": 3,
      "deSucursal": "Sucursal Centro",
      "idEmpresa": 10,
      "deEmpresa": "OV",
      "coRol": "ADMIN",
      "activa": true
    },
    {
      "idSucursal": 4,
      "deSucursal": "Sucursal Norte",
      "idEmpresa": 10,
      "deEmpresa": "OV",
      "coRol": "OPERADOR",
      "activa": false
    }
  ]
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 401 | `NO_AUTENTICADO` | JWT ausente o inválido |
| 403 | `USUARIO_SIN_MEMBRESIA` | El `sub` del JWT no tiene membresía en el tenant |

---

### `GET /api/v1/sesion/sucursales`

Retorna las sucursales a las que el usuario tiene acceso.

**No requiere `X-Empresa-Id` ni `X-Sucursal-Id`.**

**Response 200:**
```json
[
  {
    "idSucursal": 3,
    "deSucursal": "Sucursal Centro",
    "idEmpresa": 10,
    "deEmpresa": "OV",
    "coRol": "ADMIN"
  }
]
```

---

## Módulo posiciones

### `POST /api/v1/posiciones`

Ingresa una posición GPS. Autenticado por **API Key** (no JWT). Cada fuente de origen tiene su propia API Key.

**Header:** `X-Api-Key: <clave>`

**Request body:**
```json
{
  "idEmpleado": "uuid-publico-del-empleado",
  "latitud": -27.4516,
  "longitud": -58.9867,
  "precision": 5.2,
  "velocidad": 0.0,
  "tipoOperacion": "VENTA",
  "fechaPosicion": "2026-03-14T09:30:00Z"
}
```

**Response 201:** sin body.

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 401 | `API_KEY_INVALIDA` | API Key ausente o inválida |
| 404 | `EMPLEADO_NO_ENCONTRADO` | El `idEmpleado` no existe o no pertenece al tenant de la API Key |
| 400 | `VALIDACION_ERROR` | Campos obligatorios faltantes o formato inválido |

---

### `GET /api/v1/posiciones`

Consulta posiciones con filtros. Requiere JWT + empresa + sucursal.

**Query params:**

| Param | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `idEmpleado` | uuid | No | Filtrar por empleado |
| `desde` | datetime ISO 8601 | Sí | Fecha/hora inicio |
| `hasta` | datetime ISO 8601 | Sí | Fecha/hora fin |
| `pagina` | int | No | Default `0` |
| `tamanio` | int | No | Default `20`, máximo `100` |

**Response 200:**
```json
{
  "contenido": [
    {
      "idPosicion": 1001,
      "idEmpleado": "uuid-del-empleado",
      "deNombreEmpleado": "Juan García",
      "latitud": -27.4516,
      "longitud": -58.9867,
      "precision": 5.2,
      "velocidad": 0.0,
      "tipoOperacion": "VENTA",
      "fechaPosicion": "2026-03-14T09:30:00Z",
      "fechaRecibida": "2026-03-14T09:30:01Z"
    }
  ],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 450,
  "totalPaginas": 23
}
```

---

## Módulo empleados

### `GET /api/v1/empleados`

**Query params opcionales:**

| Param | Tipo | Descripción |
|---|---|---|
| `buscar` | string | Búsqueda por nombre o código |
| `coTipo` | string | `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR` |
| `activo` | boolean | Default `true` |
| `pagina` | int | Default `0` |
| `tamanio` | int | Default `20` |

**Response 200:**
```json
{
  "contenido": [
    {
      "idEmpleado": "uuid-publico",
      "coEmpleado": "EMP-001",
      "deNombre": "Juan García",
      "coTipo": "VENDEDOR",
      "snRegistraCoords": true,
      "snActivo": true,
      "feAlta": "2026-01-15T10:00:00Z"
    }
  ],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 45,
  "totalPaginas": 3
}
```

---

### `GET /api/v1/empleados/{id}`

`{id}` es el UUID público del empleado.

**Response 200:**
```json
{
  "idEmpleado": "uuid-publico",
  "coEmpleado": "EMP-001",
  "deNombre": "Juan García",
  "coTipo": "VENDEDOR",
  "snRegistraCoords": true,
  "snActivo": true,
  "feAlta": "2026-01-15T10:00:00Z",
  "idPublicoCore": null
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 404 | `EMPLEADO_NO_ENCONTRADO` | No existe o no pertenece a la sucursal activa |

---

### `POST /api/v1/empleados`

**Request body:**
```json
{
  "coEmpleado": "EMP-002",
  "deNombre": "María López",
  "coTipo": "REPARTIDOR",
  "snRegistraCoords": true
}
```

**Response 201** + header `Location: /api/v1/empleados/{uuid}`.

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 400 | `VALIDACION_ERROR` | Campos obligatorios faltantes |
| 409 | `CODIGO_DUPLICADO` | Ya existe un empleado con ese `coEmpleado` en la sucursal |

---

### `PUT /api/v1/empleados/{id}`

Envía el objeto completo (no parcial). Response 200 con el empleado actualizado.

---

### `DELETE /api/v1/empleados/{id}`

Baja lógica (`sn_activo = false`). Response 204 sin body.

---

## Módulo puntos de venta

### `GET /api/v1/puntos-venta`

**Query params opcionales:** `buscar`, `activo`, `pagina`, `tamanio`.

**Response 200** — PaginaResponse con objetos:
```json
{
  "idPuntoVenta": "uuid-publico",
  "coPuntoVenta": "PV-001",
  "deNombre": "Almacén García",
  "latitud": -27.4516,
  "longitud": -58.9867,
  "snActivo": true,
  "feAlta": "2026-01-15T10:00:00Z"
}
```

---

### `GET /api/v1/puntos-venta/{id}`

Response 200 con el objeto completo (incluye `idPublicoCore`).

---

### `POST /api/v1/puntos-venta`

```json
{
  "coPuntoVenta": "PV-002",
  "deNombre": "Almacén López",
  "latitud": -27.4600,
  "longitud": -58.9900
}
```

Response 201 + `Location`.

---

### `PUT /api/v1/puntos-venta/{id}` / `DELETE /api/v1/puntos-venta/{id}`

Igual que empleados — PUT actualiza completo, DELETE hace baja lógica.

---

## Módulo zonas

### `GET /api/v1/zonas`

**Response 200** — array (las zonas son pocas, sin paginación):
```json
[
  {
    "idZona": 1,
    "deNombre": "Zona Norte",
    "deColor": "#3B82F6",
    "coordenadas": [
      [-27.430, -58.970],
      [-27.430, -58.950],
      [-27.450, -58.950],
      [-27.450, -58.970]
    ],
    "snActivo": true
  }
]
```

---

### `GET /api/v1/zonas/{id}`

Response 200 con el objeto completo.

---

### `POST /api/v1/zonas`

```json
{
  "deNombre": "Zona Sur",
  "deColor": "#EF4444",
  "coordenadas": [
    [-27.470, -58.990],
    [-27.470, -58.960],
    [-27.490, -58.960],
    [-27.490, -58.990]
  ]
}
```

Response 201 + `Location`.

---

### `PUT /api/v1/zonas/{id}` / `DELETE /api/v1/zonas/{id}`

Igual que los demás módulos.

---

## Health check

### `GET /actuator/health`

**No requiere autenticación.**

**Response 200:**
```json
{
  "status": "UP"
}
```

---

## Permisos por rol

| Operación | ADMIN | OPERADOR | READONLY |
|---|---|---|---|
| Ver posiciones | ✓ | ✓ | ✓ |
| Ingresar posición (API Key) | ✓ | ✓ | — |
| Ver empleados / puntos de venta / zonas | ✓ | ✓ | ✓ |
| Crear / editar empleados, puntos de venta, zonas | ✓ | ✓ | — |
| Dar de baja empleados, puntos de venta, zonas | ✓ | — | — |
| Gestionar usuarios | ✓ | — | — |

Los permisos se validan en el backend consultando `usuarios_geo_sucursales.co_rol` — nunca desde el JWT.
