# ADR-0012: sin caché en la ruta de F1–F3; caché local solo con un driver medido (Diseñado)

| Campo | Valor |
|---|---|
| Estado | **Proposed** · Diseñado |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0006 del xlsx (Cache-Aside con IMemoryCache, Accepted), ADR-07 del tradeoff, `adr-previos/serie-11-adr/adr-0007` |

**Contexto.** En F1–F3 se leen la disponibilidad del conductor, la solicitud, la asignación, la conversación y los parámetros. Las cuatro primeras son exactamente el estado que se decide: servirlas desde caché rompe `COND-CONF-02`. Los parámetros cambian por la API de administración y deben aplicarse "a partir de la siguiente solicitud" (RF-21). El volumen del caso ancla (~40 servicios/día) no genera presión de lectura. El curso advierte que "la memoria local no es fuente de verdad para parámetros que deben compartirse" (lineamientos del curso Ingeniería del Software 2).

**Decisión.** Ninguna caché en la ruta E2E. El building block queda **no instanciado**.

**Cuándo se instancia y cómo.** Si una medición muestra que una lectura de baja volatilidad (catálogo de mensajes, lugares frecuentes, reportes del panel en V1) supera su presupuesto de latencia: Cache-Aside con `IMemoryCache` por proceso, TTL explícito e invalidación por `actualizado_en`. Nunca para solicitudes, asignaciones, turnos ni conversaciones. Caché distribuida solo con más de una réplica y si la inconsistencia temporal entre réplicas importa.

**Alternativas.** Cache-Aside local desde ya (la fuente anterior): optimiza sin un driver y agrega riesgo de obsolescencia. Redis por operador: suma ~1 componente y costo por stack.

**Escenarios.** `COND-CONF-02` (justifica no tenerla); `ADM-REND-01` (V1, posible activación).
