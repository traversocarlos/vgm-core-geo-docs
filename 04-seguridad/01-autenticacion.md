# Seguridad y autenticación

**Versión:** 2.2
**Fecha:** 2026-03-14
**Estado:** Activo

---

## Estrategia en dos etapas

### Etapa 1 — JWT propio (arranque)

VGM Core Geo genera sus propios tokens JWT internamente.

- Endpoint de login: `POST /api/v1/auth/login`
- VGM Core Geo valida usuario/contraseña contra `cuentas` + `usuarios_geo`
- Emite un JWT firmado con clave propia (`vgmcoregeo.jwt.secret`)
- Expiración configurable (`vgmcoregeo.jwt.expiration: 3600`)

**Ventaja:** funciona de forma completamente independiente, sin depender de Auth0.

### Etapa 2 — Auth0 (futuro)

Cuando haya necesidad (SSO, cliente enterprise, integración con VGM Core):

- El backend pasa a ser **Resource Server** (igual que VGM Core hoy)
- Solo valida tokens emitidos por Auth0 (`vgm-core-dev.us.auth0.com`)
- El cambio en código es mínimo: 4 líneas en `application.yml`
- Todo lo demás (filtros, servicios, excepciones) queda igual

---

## Auth0 — configuración definida

VGM Core Geo ya está creada como aplicación en Auth0 dentro del tenant de Mauricio.

| Dato | Valor |
|---|---|
| Auth0 Tenant | `vgm-core-dev.us.auth0.com` |
| Aplicación | `VGM Core Geo` (Single Page Application) |
| Namespace de claims | `https://vgmcoregeo.com` |

> El namespace es diferente al de VGM Core (`https://vgmcore.com`) porque son productos independientes.

---

## Estructura del token JWT

Siguiendo el patrón de VGM Core: el token lleva **solo el tenant y el email**. La empresa, la sucursal y el rol se resuelven dinámicamente desde headers HTTP y base de datos — no van embebidos en el token.

### Etapa 1 (JWT propio)
```json
{
  "sub": "usuario@empresa.com",
  "https://vgmcoregeo.com/tenant_id": 1,
  "https://vgmcoregeo.com/email": "usuario@empresa.com",
  "exp": 1234567890
}
```

### Etapa 2 (Auth0)
Mismo formato de claims, emitidos por Auth0 a través de un Action Post Login con namespace `https://vgmcoregeo.com`.

---

## Headers de contexto

Igual que VGM Core, empresa y sucursal activa se resuelven por headers en cada request:

| Header | Obligatorio | Descripción |
|---|---|---|
| `Authorization: Bearer <token>` | Sí | JWT del usuario |
| `X-Empresa-Id` | Condicional | Obligatorio si el usuario tiene acceso a más de una empresa |
| `X-Sucursal-Id` | Condicional | Obligatorio si el usuario tiene acceso a más de una sucursal |
| `X-Correlation-Id` | Recomendado | UUID generado por el frontend para trazabilidad de logs |

Si el usuario solo tiene una empresa/sucursal asignada, el header es opcional — VGM Core Geo la resuelve automáticamente.

El header `X-Empresa-Id` es **por request, no estado del servidor** — stateless, compatible con load balancers, no requiere re-login para cambiar de empresa/sucursal.

---

## Flujo de autenticación

```
1. Usuario abre VGM Core Geo → no tiene sesión → pantalla de login
2. Ingresa usuario y contraseña
3. POST /api/v1/auth/login → VGM Core Geo valida y emite JWT
4. Frontend guarda el token EN MEMORIA (nunca en localStorage)
5. Cada request envía:
   Authorization: Bearer <token>
   X-Empresa-Id: <id>       (si aplica)
   X-Sucursal-Id: <id>      (si aplica)
   X-Correlation-Id: <uuid> (trazabilidad)
6. TenantContextFilter intercepta, valida JWT, carga contexto
   (cliente_saas, empresa, sucursal)
7. Controller procesa con contexto ya resuelto
```

---

## Componentes de seguridad — patrón de tres capas (idéntico a VGM Core)

Mauricio documentó el diseño completo en `01-arquitectura/10-diseno-tenant-filter.md` de vgm-core-docs.

```
Request HTTP con JWT
        │
        ▼
┌─────────────────────────────┐
│  1. TenantContextFilter     │  Servlet Filter (después de Spring Security)
│     Extrae claims del JWT   │  Almacena en TenantContext (ThreadLocal)
│     Resuelve empresa activa │  NO toca la base de datos todavía
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  2. TenantContext            │  ThreadLocal holder
│     idClienteSaas: Long     │  Disponible para todo el hilo del request
│     idEmpresa: Long         │  Se limpia al finalizar (finally obligatorio)
│     idSucursal: Long        │  ← extensión de VGM Core Geo
│     idUsuarioGeo: Long      │
│     email: String           │
│     sub: String             │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  3. TenantConnectionPreparer│  Aspecto AOP alrededor de @Transactional
│     Ejecuta SET LOCAL       │  DENTRO de la transacción real
│     Usa PreparedStatement   │  (previene SQL injection)
│     Misma conexión que JPA  │
└─────────────────────────────┘
```

### TenantContext de VGM Core Geo

```kotlin
data class DatosTenant(
    val idClienteSaas: Long,
    val idEmpresa: Long,
    val idSucursal: Long,      // extensión de VGM Core Geo
    val idUsuarioGeo: Long,
    val email: String,
    val sub: String,
)
```

### TenantConnectionPreparer — SET LOCAL por transacción

```kotlin
// SET LOCAL dentro de la transacción JPA (misma conexión)
connection.prepareStatement("SET LOCAL app.id_cliente_saas = ?").use { stmt ->
    stmt.setLong(1, ctx.idClienteSaas); stmt.execute()
}
connection.prepareStatement("SET LOCAL app.id_empresa = ?").use { stmt ->
    stmt.setLong(1, ctx.idEmpresa); stmt.execute()
}
connection.prepareStatement("SET LOCAL app.id_sucursal = ?").use { stmt ->
    stmt.setLong(1, ctx.idSucursal); stmt.execute()
}
```

### TenantContextProvider — inyectable en servicios

Los servicios no acceden directamente al `TenantContext` estático. Se inyecta `TenantContextProvider` para mantener testabilidad (se puede mockear en tests unitarios):

```kotlin
@Component
class TenantContextProvider {
    fun obtener(): DatosTenant = TenantContext.obtener()
}

// En los servicios:
@Service
class EmpleadosServicio(
    private val tenantCtx: TenantContextProvider,
    private val empleadoRepo: EmpleadoRepo,
) {
    fun listar(): List<EmpleadoResumenDto> {
        val ctx = tenantCtx.obtener()
        return empleadoRepo.buscarTodas().map { it.toResumenDto() }
    }
}
```

---

## Resolución del contexto (paso a paso)

```
1. Spring Security valida el JWT
   - Etapa 1: valida firma con clave propia
   - Etapa 2: valida contra Auth0 (issuer-uri + audience)

2. TenantContextFilter (@Order después de Spring Security) corre

3. TenantResolver extrae del JWT:
   - "https://vgmcoregeo.com/tenant_id" → idClienteSaas
   - "https://vgmcoregeo.com/email"    → email
   - "sub"                              → subject del usuario

4. Busca el usuario en BD (cuentas + usuarios_geo)
   → Cacheado con Caffeine (TTL 5 min) — las membresías cambian muy poco
   - Si no existe → 403 FORBIDDEN

5. Resuelve empresa desde X-Empresa-Id:
   - Si header presente → valida acceso
   - Si una sola empresa → la usa automáticamente
   - Si múltiples y sin header → 400 BAD REQUEST

6. Resuelve sucursal desde X-Sucursal-Id:
   - Igual que empresa, pero dentro de la empresa resuelta

7. Carga rol desde usuarios_geo_sucursales (nunca desde el JWT)

8. TenantContext.establecer(DatosTenant) → guarda en ThreadLocal

9. TenantConnectionPreparer ejecuta SET LOCAL en cada @Transactional:
   - SET LOCAL app.id_cliente_saas = ?
   - SET LOCAL app.id_empresa = ?
   - SET LOCAL app.id_sucursal = ?

10. Controller ejecuta — contexto disponible vía TenantContextProvider

11. finally: TenantContext.limpiar() — SIEMPRE, incluso con excepción
```

---

## Caché de resolución de usuarios

Las consultas del `TenantResolver` a BD (buscar usuario, buscar sucursales habilitadas) se cachean con **Caffeine** para no impactar en cada request:

| Dato cacheado | Clave | TTL |
|---|---|---|
| Membresía usuario en tenant | `sub` + `tenant_id` | 5 min |
| Sucursales habilitadas del usuario | `id_usuario_geo` | 5 min |

Si un administrador cambia accesos, el efecto tarda máximo 5 minutos en reflejarse.

---

## Excepciones mapeadas a HTTP

| Excepción | HTTP | Código |
|---|---|---|
| `TenantNoResueltaException` | 401 | `TENANT_NO_RESUELTO` |
| `UsuarioNoEncontradoException` | 403 | `USUARIO_SIN_MEMBRESIA` |
| `EmpresaNoAsignadaException` | 403 | `EMPRESA_NO_ASIGNADA` |
| `EmpresaInvalidaException` | 400 | `EMPRESA_INVALIDA` |
| `AccesoEmpresaDenegadoException` | 403 | `EMPRESA_ACCESO_DENEGADO` |
| `EmpresaNoSeleccionadaException` | 400 | `EMPRESA_NO_SELECCIONADA` |
| `SucursalNoAsignadaException` | 403 | `SUCURSAL_NO_ASIGNADA` |
| `SucursalNoSeleccionadaException` | 400 | `SUCURSAL_NO_SELECCIONADA` |

---

## Comparación con VGM Core

| | VGM Core | VGM Core Geo |
|---|---|---|
| Claims en JWT | `tenant_id`, `email` | `tenant_id`, `email` |
| Niveles resueltos en contexto | `cliente_saas` + `empresa` | `cliente_saas` + `empresa` + `sucursal` |
| Namespace JWT | `https://vgmcore.com` | `https://vgmcoregeo.com` |
| Rol en JWT | No — se carga de BD | No — se carga de BD |
| Auth0 Application | VGM Core Web / Desktop | VGM Core Geo |
| IdP Etapa 1 | Auth0 desde el inicio | JWT propio |
| IdP Etapa 2 | — | Auth0 (`vgm-core-dev.us.auth0.com`) |
| Header empresa | `X-Empresa-Id` | `X-Empresa-Id` |
| Header sucursal | No aplica | `X-Sucursal-Id` |
| Caché resolución | Caffeine 5 min | Caffeine 5 min |

---

## Auth0 Action para Etapa 2

```javascript
exports.onExecutePostLogin = async (event, api) => {
  const NAMESPACE = 'https://vgmcoregeo.com';

  const orgMetadata = event.organization?.metadata || {};
  const tenantId = orgMetadata.tenant_id;

  if (!tenantId) {
    api.access.deny('No se pudo determinar el tenant.');
    return;
  }

  api.accessToken.setCustomClaim(`${NAMESPACE}/tenant_id`, parseInt(tenantId, 10));

  if (event.user.email) {
    api.accessToken.setCustomClaim(`${NAMESPACE}/email`, event.user.email);
  }
};
```

> Coordinarlo con Mauricio antes de implementar — tiene que vivir en el mismo tenant de Auth0.
