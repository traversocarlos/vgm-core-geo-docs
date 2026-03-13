# Stack tecnológico

**Versión:** 1.0
**Fecha:** 2026-03-13
**Estado:** Activo

---

## Backend

| Componente | Tecnología | Versión |
|---|---|---|
| Lenguaje | Kotlin | 2.1.x |
| Framework | Spring Boot | 3.5.x |
| Persistencia | JPA / Hibernate | — |
| Migraciones BD | Flyway | — |
| Seguridad | Spring Security (JWT) | — |
| Documentación API | springdoc-openapi | 2.x |
| Build | Gradle + libs.versions.toml | — |
| Tests | Testcontainers | — |
| CI | GitHub Actions | — |
| Containers | Docker + Docker Compose | — |

**Base de datos Etapa 1:** PostgreSQL 17 (propio, no compartido)
**Base de datos Etapa 2 (futuro):** migración desde SQL Server si se requiere compatibilidad legacy

---

## Frontend

| Componente | Tecnología | Versión |
|---|---|---|
| Framework | React | 18 |
| Lenguaje | TypeScript | 5 |
| Build tool | Vite | — |
| Estilos | Tailwind CSS | — |
| Componentes UI | Shadcn/ui | — |
| Mapas | Leaflet.js + OpenStreetMap | 1.9+ |

**OpenStreetMap reemplaza Google Maps** — sin API key, sin costo de licencia.

---

## Justificación del stack

El stack es intencionalmente idéntico al de VGM Core (desarrollado por Mauricio). Esto permite:

- Reutilizar convenciones, configuración de CI, Docker Compose
- El equipo no tiene que aprender tecnologías nuevas
- Componentes de seguridad (TenantContextFilter, GlobalExceptionHandler) se copian y adaptan directamente

---

## Lo que NO se usa

| Tecnología | Decisión |
|---|---|
| Google Maps | Reemplazado por OpenStreetMap (costo) |
| Progress / OpenEdge | Descartado — sin ventaja técnica, costo de licencias, pool de talento limitado |
| Microservicios | Monolito modular, igual que VGM Core |
| RLS (Row Level Security) | No obligatorio en Etapa 1 — multi-tenancy por campo `id_empresa` |
