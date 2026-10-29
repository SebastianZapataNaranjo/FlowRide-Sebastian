# ADR-0013: telemetría GPS mediante adaptador y tabla particionada en PostgreSQL (Diseñado V1)

| Campo | Valor |
|---|---|
| Estado | **Proposed** · Diseñado V1 |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0005 del xlsx, ADR-09 del tradeoff, `adr-previos/serie-11-adr/adr-0009` |

**Contexto.** V1 incluye telemetría para monitorear viajes en curso y detectar desviaciones del ETA (RF-55, RF-56; `PAS-CONF-01`, `TEL-INTEROP-01`). Dimensionamiento de las fuentes: 50 vehículos cada 30 s, unos 100 eventos/min por instancia, con retención de 6 meses. La telemetría **no** participa en la asignación (diferenciador 5). La fuente (hardware, Live Location de WhatsApp o app propia) no está decidida.

**Decisión (dirección, se conserva de las fuentes).** Puerto `ITelemetryProviderAdapter` en `Infraestructura.Output.ServiciosExternos.Telemetry` (hoy no se crea), normalizado a `RegistroUbicacion`; tabla con particionamiento declarativo mensual en la base del operador; retención de 6 meses eliminando particiones. Ingesta con E/S asíncrona y `Channel<T>` antes de considerar un modelo reactivo (ADR-0003).

**Pendiente en V1.** Elegir la fuente de ubicación; verificar los límites de Live Location y su soporte en la Cloud API; PoC 5 (≥ 100 eventos/min, ≥ 99 % aceptados) y PoC 6 (desviación > 5 min detectada en < 60 s). Habilita la verificación de la promesa de presentación (ADR-0007, riesgo de la matriz estática).

**Escenarios.** `PAS-CONF-01`, `TEL-INTEROP-01` (Diseñados).
