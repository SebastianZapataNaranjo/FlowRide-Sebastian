# Registro de decisiones de arquitectura (ADR)

Numeración canónica de FlowRide desde el 2026-10-28. La equivalencia con las numeraciones anteriores (tradeoff ADR-01..10, xlsx ADR-0001..0017, la serie de 11 ADR de septiembre y los antiguos `docs/adr`) está en `docs/architecture/registro-de-cambios-vs-fuentes.md`.

Regla de estado: todo ADR nace `Proposed`. Pasa a `Accepted` solo cuando se cumple el criterio de aceptación escrito en sus metadatos (una prueba, una PoC o una medición). Ninguno está `Accepted` todavía porque no hay código ejecutado.

## Decisiones que sostienen F1–F3 (entrega E2E)

| ADR | Decisión | Estado | Escenario principal |
|---|---|---|---|
| [0001](adr-0001-monolito-modular.md) | Monolito modular con límites verificados por NetArchTest | Proposed | `EQ-MANT-01` |
| [0002](adr-0002-silo-stack-por-operador.md) | Silo: un stack (API + Worker + BD) por operador; control plane fuera del camino crítico | Proposed | `PLAT-AIS-01` |
| [0003](adr-0003-stack-ejecucion-protocolos.md) | .NET 10, ejecución no reactiva con E/S asíncrona, protocolo por cada par de componentes | Proposed | `CANAL-SEG-01` |
| [0004](adr-0004-postgresql-efcore.md) | PostgreSQL + EF Core/Npgsql por Silo | Proposed | `OPLOG-CONF-03` |
| [0005](adr-0005-whatsapp-meta-cloud-api-e-identidad-canal.md) | WhatsApp vía Meta Cloud API directo (Twilio descartado); identidad teléfono → usuario; simulador como doble de prueba | Proposed | `COND-SEG-01` |
| [0006](adr-0006-consistencia-inbox-outbox-restricciones.md) | Inbox idempotente, outbox con SKIP LOCKED, `xmin` y restricciones de exclusión | Proposed | `CANAL-CONF-01`, `OPLOG-CONF-03` |
| [0007](adr-0007-motor-asignacion-reglas.md) | Motor por evento, tres niveles de reglas, expiración relativa y escalamiento | Proposed | `OPLOG-CONF-04`, `PAS-CONF-03` |
| [0008](adr-0008-notification-gateway-y-catalogos.md) | Notification gateway y catálogos de mensajes, notificaciones y parámetros | Proposed | `PAS-UX-01` |
| [0009](adr-0009-despliegue-azure-disponibilidad-costo.md) | Azure Container Apps + Flexible Server; disponibilidad compuesta; HA como opción contractual | Proposed (Azure Diseñado) | `PAS-DISP-01` |
| [0010](adr-0010-observabilidad-y-auditoria.md) | Correlation ID propagado por la outbox; auditoría de solo inserción | Proposed | `OPLOG-AUD-01` |

## Decisiones Diseñadas (no forman parte de la entrega E2E)

Versión corta, estado `Proposed`. Cada uno dice cuándo se activa.

| ADR | Decisión |
|---|---|
| [0011](adr-0011-identidad-panel-oidc.md) | Identidad del panel con OIDC; la API key queda limitada al Administrador técnico |
| [0012](adr-0012-cache.md) | Sin caché en F1–F3; caché local solo con un driver medido |
| [0013](adr-0013-telemetria-gps.md) | Telemetría GPS con adaptador y tabla particionada |
| [0014](adr-0014-secretos.md) | Secretos por operador y de plataforma; app secret de Meta como punto común (la parte local es E2E) |
| [0015](adr-0015-borde-comun.md) | Sin borde común en la ruta de mensajes (override por número) |
| [0016](adr-0016-almacenamiento-objetos.md) | Almacenamiento de objetos para archivo y exportes |
