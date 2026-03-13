# Seguridad y autenticación

**Versión:** 1.0
**Fecha:** 2026-03-13
**Estado:** Activo

---

## Estrategia en dos etapas

### Etapa 1 — JWT propio (arranque)

VGM Go genera sus propios tokens JWT internamente.

- Endpoint de login: `POST /api/v1/auth/login`
- VGM Go valida usuario/contraseña contra su propia tabla `usuarios_go`
- Emite un JWT firmado con clave propia (`vgmgo.jwt.secret`)
- El token tiene expiración configurable (`vgmgo.jwt.expiration: 3600`)

**Ventaja:** funciona de forma completamente independiente, sin depender de Auth0 ni de VGM Core.

### Etapa 2 — Auth0 como IdP (futuro)

Cuando haya necesidad (cliente que usa VGM Core + VGM Go, o requerimiento de SSO):

- El backend deja de generar tokens y pasa a ser **Resource Server**
- Solo valida tokens emitidos por Auth0
- El cambio en código es mínimo: 4 líneas en `application.yml`
- Todo lo demás (filtros, RLS, servicios) queda igual

---

## Estructura del token JWT

El token lleva estos claims:

```json
{
  "sub": "usuario@empresa.com",
  "https://vgmgo.com/empresa_id": 5,
  "https://vgmgo.com/rol": "ADMIN",
  "exp": 1234567890
}
```

> **⚠️ PENDIENTE URGENTE:** El namespace `https://vgmgo.com/` debe acordarse con Mauricio antes de escribir código de autenticación. Si en el futuro Auth0 emite tokens para VGM Core y VGM Go, el namespace tiene que ser compatible. Ver [ADR-011 pendiente].

---

## Flujo de autenticación (Etapa 1)

```
1. Usuario abre VGM Go → no tiene sesión → pantalla de login
2. Ingresa usuario y contraseña
3. POST /api/v1/auth/login → VGM Go valida y emite JWT
4. Frontend guarda el token EN MEMORIA (nunca en localStorage)
5. Cada request envía: Authorization: Bearer <token> + X-Empresa-Id: <id>
6. TenantContextFilter intercepta, valida JWT, carga contexto
7. Controller procesa con empresa ya resuelta
```

---

## Componentes de seguridad

Copiados y adaptados del código de Mauricio (VGM Core):

| Componente | Origen | Cambio |
|---|---|---|
| `TenantContextFilter` | VGM Core | Eliminar lógica de `id_cliente_saas` |
| `TenantConnectionPreparer` | VGM Core | Copiar sin cambios |
| `TenantExceptions` | VGM Core | Copiar sin cambios |
| `GlobalExceptionHandler` | VGM Core | Copiar sin cambios |
| `TenantResolver` | VGM Core | Cambiar claim de `https://vgmcore.com/tenant_id` a namespace acordado |
| `SecurityConfig` | VGM Core | Etapa 1: generar JWT. Etapa 2: Resource Server |

---

## Diferencia con VGM Core

| | VGM Core | VGM Go |
|---|---|---|
| Niveles de tenancy | 3 (clientes_saas → empresas → sucursales) | 1 (empresas) |
| RLS | Obligatorio en todas las tablas | No en Etapa 1 |
| IdP | Auth0 desde el inicio | JWT propio en Etapa 1, Auth0 en Etapa 2 |
