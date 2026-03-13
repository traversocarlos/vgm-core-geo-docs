# ADR-003 — JWT propio en Etapa 1, Auth0 en Etapa 2

**Estado:** Aceptado
**Fecha:** 2026-03-13
**Decisores:** Equipo VGM

---

## Contexto

VGM Core usa Auth0 desde el inicio. VGM Go necesita autenticación propia para poder funcionar y venderse de forma independiente, sin requerir que el cliente tenga Auth0 configurado.

## Decisión

VGM Go arranca con **JWT propio** generado internamente. El diseño interno ya está preparado para conectar Auth0 en Etapa 2 sin reescribir código.

## Condición para Etapa 2

Antes de implementar cualquier código de autenticación, el equipo debe reunirse y acordar el **namespace de los JWT claims** con Mauricio. Si en el futuro Auth0 emite tokens para VGM Core y VGM Go, el namespace tiene que ser compatible desde el día uno.

**Pregunta a resolver:** ¿namespace compartido `https://vgm.com/` o separado por producto `https://vgmgo.com/`?

## Consecuencias

**Positivas:**
- VGM Go funciona sin depender de Auth0 ni de VGM Core
- La migración a Auth0 en Etapa 2 es un cambio de configuración, no de código

**Trade-offs:**
- En Etapa 1 el equipo mantiene su propia lógica de login y refresh de tokens

**Mitigaciones:**
- Copiar el patrón de seguridad de Mauricio (TenantContextFilter, TenantConnectionPreparer) para no reinventar la rueda
