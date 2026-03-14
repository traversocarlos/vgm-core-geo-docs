# Convenciones de la API REST

**Versión:** 1.0
**Fecha:** 2026-03-14
**Estado:** Activo

> Convenciones idénticas a VGM Core (Mauricio). Todo el equipo las aplica igual en ambos productos.

---

## Base path

```
/api/v1
```

---

## Headers en todo request autenticado

| Header | Ejemplo | Obligatorio | Descripción |
|---|---|---|---|
| `Authorization` | `Bearer eyJ...` | Sí | JWT del usuario |
| `X-Empresa-Id` | `10` | Condicional | Empresa activa. Opcional si el usuario tiene una sola. |
| `X-Sucursal-Id` | `3` | Condicional | Sucursal activa. Opcional si el usuario tiene una sola. |
| `X-Correlation-Id` | `uuid-v4` | Recomendado | Generado por el frontend. El backend lo propaga en logs y respuestas de error. |

---

## Formato de respuesta de error

Todos los errores del sistema usan este formato:

```json
{
  "codigo": "EMPLEADO_NO_ENCONTRADO",
  "mensaje": "No se encontró el empleado con id 42",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-03-14T10:30:00Z"
}
```

| Campo | Descripción |
|---|---|
| `codigo` | Constante en SCREAMING_SNAKE_CASE — identificable por el frontend |
| `mensaje` | Descripción legible por humanos |
| `correlationId` | El mismo UUID del header `X-Correlation-Id` del request |
| `timestamp` | Momento del error en UTC |

---

## Paginación

### Parámetros de request (query params)

| Parámetro | Default | Descripción |
|---|---|---|
| `pagina` | `0` | Número de página — 0-indexed |
| `tamanio` | `20` | Elementos por página — máximo 100 |
| `ordenar` | varía por endpoint | Campo de ordenamiento |
| `direccion` | `asc` | `asc` o `desc` |

### Formato de respuesta paginada

```json
{
  "contenido": [...],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 150,
  "totalPaginas": 8
}
```

---

## Convenciones de nombres en JSON

Las propiedades JSON usan **camelCase** — igual que VGM Core.

| BD (snake_case) | JSON (camelCase) |
|---|---|
| `id_empleado` | `idEmpleado` |
| `de_nombre` | `deNombre` |
| `co_tipo` | `coTipo` |
| `fe_alta` | `feAlta` |
| `sn_activo` | `snActivo` |

Los IDs que se exponen en la API son siempre los **UUID públicos** (`id_publico`) — nunca los IDs internos bigint.

---

## Bajas

Nunca se eliminan registros físicamente. Siempre es **baja lógica**:

- `DELETE /recurso/{id}` → marca `sn_activo = false`
- Response: `204 No Content` sin body

---

## Operaciones y códigos HTTP

| Operación | Método | Código éxito | Body response |
|---|---|---|---|
| Listar | `GET` | `200 OK` | Array o PaginaResponse |
| Obtener uno | `GET /{id}` | `200 OK` | Objeto completo |
| Crear | `POST` | `201 Created` + header `Location` | Objeto creado |
| Actualizar | `PUT /{id}` | `200 OK` | Objeto actualizado |
| Baja lógica | `DELETE /{id}` | `204 No Content` | Sin body |

---

## Estrategia OpenAPI

La especificación OpenAPI se **genera automáticamente** con `springdoc-openapi` a partir de las anotaciones en los controllers. No se escribe a mano.

- Este documento y `02-endpoints-fase1.md` son referencia de diseño
- La fuente de verdad es el código anotado + el JSON en `/v3/api-docs`
- En desarrollo local, Swagger UI está en `http://localhost:8080/swagger-ui.html`

---

## Rutas excluidas de autenticación

| Ruta | Motivo |
|---|---|
| `GET /actuator/health` | Health check para infraestructura |
| `GET /actuator/info` | Info del servicio |
| `GET /swagger-ui/**` | Documentación API |
| `GET /v3/api-docs/**` | Especificación OpenAPI |
| `POST /api/v1/auth/login` | Login — no tiene JWT todavía |
| `POST /api/v1/posiciones` | Ingesta GPS — autenticación por API Key, no JWT |
