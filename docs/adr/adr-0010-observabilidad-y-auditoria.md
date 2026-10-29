# ADR-0010: observabilidad con correlation ID propagado por la outbox y auditoría persistente separada de los logs técnicos

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0010 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando, en la línea base, una solicitud se pueda reconstruir desde el webhook hasta la confirmación al pasajero filtrando por un solo `correlation_id` en los logs, y la auditoría tenga una fila por cada transición de F1–F3 (prueba en `estrategia-de-verificacion.md`). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0011 del xlsx, ADR-10 del tradeoff, `adr-previos/serie-11-adr/adr-0011` |
| Estado de implementación | Diseñado |

## Contexto

- El flujo F1→F2→F3 atraviesa dos procesos (API y Worker), una outbox asíncrona y un proveedor externo. Sin una correlación que sobreviva a la outbox, un log de la API y uno del Worker de la misma solicitud no se pueden unir.
- El curso exige logging estructurado, contexto tipo MDC, correlation ID generado o validado por el servidor y no exponer datos sensibles (lineamientos del curso Ingeniería del Software 2).
- Los conductores y pasajeros piden poder demostrar qué pasó (lluvia de escenarios: "si existe una discusión sobre si acepté o rechacé…"). Eso es **auditoría**, con otra retención (5 años, contexto v10 §8.5) y otro propósito que el log técnico.

## Decisión

### Observabilidad técnica
- **Correlation ID**: lo **genera el servidor** (UUID v7) al recibir el webhook o la petición de administración; nunca se acepta del cliente como dato de confianza. Se guarda en `mensaje_entrante.correlation_id`, se copia en cada `outbox_evento` que deriva de él y el Worker lo **restaura** antes de procesar. Se propaga a la llamada a Meta mediante `biz_opaque_callback_data` junto a la clave de idempotencia (ADR-0006).
- **Logging estructurado**: `ILogger` con `BeginScope` (equivalente a MDC) que agrega `correlation_id`, `operador`, `componente` y `caso_uso`. Formato JSON a la consola en el compose.
- **Trazas y métricas**: OpenTelemetry para .NET. En el compose, exportador de consola; en Azure, OTLP hacia Azure Monitor y Log Analytics (Diseñado). El exportador es intercambiable sin tocar la instrumentación.
- **Métricas mínimas**: `webhook_recibidos_total`, `webhook_duplicados_total`, `webhook_firma_invalida_total`, `outbox_pendientes`, `outbox_fallidos_total`, `asignacion_propuesta_latencia_s`, `notificacion_incierta_total`, `solicitud_sin_candidato_total`.
- **Minimización de datos**: nunca se registran el cuerpo de los mensajes, los tokens ni los secretos. Los teléfonos se enmascaran (`+57******1234`). Los mensajes técnicos no llegan al usuario (ADR-0008).
- **Salud**: `/health/live` (el proceso responde) y `/health/ready` (conexión a la base y migraciones al día).

### Auditoría funcional
- Tabla `auditoria` en la base de **cada operador**: `ocurrido_en`, `actor_tipo` (Pasajero, Conductor, AdministradorTecnico, Sistema), `actor_id`, `accion`, `recurso_tipo`, `recurso_id`, `resultado`, `detalle` jsonb (sin datos personales en claro) y `correlation_id`.
- **Solo inserción**: el usuario de base de la aplicación tiene `INSERT` y `SELECT` sobre `auditoria`, sin `UPDATE` ni `DELETE`.
- **Acciones auditadas en F1–F3**: itinerario registrado o rechazado (duplicado, anticipación), propuesta de asignación con `razones`, aceptación, rechazo, expiración, `SinCandidato`, alerta enviada, check-in, remitente desconocido, firma inválida, `phone_number_id` ajeno y cambio de parámetro.
- `estado_solicitud_historial` registra cada transición de la solicitud (actor, estado anterior y nuevo, instante). Es lo que exige `PAS-CONF-02` del banco.
- Retención de 5 años; el archivo a almacenamiento de objetos queda Diseñado (ADR-0016).

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| Usar los logs técnicos como auditoría | Retención, integridad y propósito distintos; los logs se muestrean y se rotan. |
| APM propietario acoplado al código | Lock-in en la instrumentación; OpenTelemetry permite cambiar el backend. |
| Aceptar el correlation ID que mande el cliente | Lo prohíben los lineamientos del curso Ingeniería del Software 2 y permitiría inyectar correlaciones falsas. |
| Auditoría en un servicio central compartido | Rompería el Silo (ADR-0002). |

## Consecuencias

**Positivas**: una solicitud se reconstruye de punta a punta; la evidencia para disputas existe y no se puede editar desde la aplicación; las métricas alimentan los escenarios de calidad.
**Negativas**: más escrituras por transición (auditoría e historial dentro de la misma transacción); la redacción mal implementada podría filtrar datos personales, por eso hay una prueba que busca patrones de teléfono sin enmascarar en los logs.

## Tácticas de Bass

Monitorear (métricas y salud); registrar eventos y auditar; detectar fallos (readiness); limitar la exposición (redacción); no repudio (auditoría de solo inserción con actor identificado por teléfono).

## Escenarios y requisitos relacionados

`OPLOG-AUD-01` (consulta de auditoría: la consulta del panel queda Diseñada; en la línea base es `GET /admin/auditoria`), `COND-SEG-01`, `CANAL-SEG-01`, `PAS-CONF-03`; restricción técnica de observabilidad; RF-50 a RF-53.

## Condiciones de revisión

Si el volumen de logs supera los 5 GB gratis por mes en Azure, aplicar muestreo de trazas y retención más corta para los logs técnicos (nunca para la auditoría).

## Referencias

Lineamientos del curso ingeniería del software 2; contexto v10 §8.5 (retención); ADR-0011 del xlsx.
