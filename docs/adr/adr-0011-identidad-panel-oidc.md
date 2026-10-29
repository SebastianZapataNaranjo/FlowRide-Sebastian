# ADR-0011: identidad del panel web con OIDC (Diseñado V1)

| Campo | Valor |
|---|---|
| Estado | **Proposed** · Diseñado V1, no forma parte de la entrega E2E |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0003 del xlsx (OIDC + Keycloak con un realm por operador), ADR-03 del tradeoff, `adr-previos/serie-11-adr/adr-0003` |

**Contexto.** El administrador de empresa y el operador logístico usarán un panel web (RF-01, RF-02, RF-12, RF-21). Ahí sí hay sesión de navegador, y aplican la revocación en ≤ 60 s (`ADM-SEG-01`) y la auditoría por persona (`OPLOG-REND-01`, `OPLOG-AUD-02`). En la línea base, la API de administración usa una API key por stack que **no identifica personas** (ADR-0003 §4).

**Decisión (dirección).** OpenID Connect con Authorization Code + PKCE contra un proveedor externo o autoalojado. Validación JWT en la API, con el rol de la base del operador, y un adaptador de identidad en `Infraestructura.Output.ServiciosExternos.IdentityProvider` (paquete que hoy **no se crea**). Revocación: además de validar el token, se consulta en cada petición el estado `activo` del usuario en la base del operador (sin caché, o con una caché de TTL ≤ 60 s). Alertas al panel por SSE (ADR-0003).

**Alternativas por evaluar en V1.** Keycloak autoalojado (un realm por operador: costo de cómputo por calcular frente al techo de ADR-0009); Microsoft Entra External ID; Auth0; ASP.NET Core Identity propio (el comparador).

**Consecuencias.** Hasta V1, la reasignación manual y la auditoría por persona no existen. La API key queda restringida al rol de Administrador técnico.

**Escenarios.** `ADM-SEG-01`, `OPLOG-REND-01`, `OPLOG-AUD-02`, `OPLOG-SEG-02`, `ADM-SEG-02` (todos Diseñados).

**Condición para activarlo.** Cuando entre el panel web, o antes si un operador exige trazabilidad por persona en los cambios de parámetros.
