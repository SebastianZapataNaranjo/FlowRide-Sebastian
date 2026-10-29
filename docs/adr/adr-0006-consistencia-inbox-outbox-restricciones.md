# ADR-0006: garantizar consistencia con inbox idempotente, outbox con SKIP LOCKED, concurrencia optimista (`xmin`) y restricciones de la base de datos

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0006 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando pasen las pruebas de concurrencia de `OPLOG-CONF-03` y de duplicados de `CANAL-CONF-01` contra PostgreSQL real (Testcontainers). Es el Spike 2 de las fuentes. |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0007 del xlsx ("optimistic locking + outbox", Accepted sin spike), ADR-06 del tradeoff, `adr-previos/serie-11-adr/adr-0006`; corrige el *leader election* con `pg_advisory_lock` de `adr-previos/docs-adr-sep2026/adr-0007` |
| Estado de implementación | Diseñado |

## Contexto

Errores que el negocio no tolera (lluvia de escenarios: "el sistema no me puede asignar dos servicios incompatibles", "mi solicitud no se puede perder ni duplicar"):

| Riesgo concreto | De dónde viene |
|---|---|
| El mismo mensaje de WhatsApp procesado dos veces | Meta reintenta los webhooks que no responden 200 "con frecuencia decreciente durante hasta 7 días" y avisa que puede haber duplicados ([Set up webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks), consulta 2026-10-28) |
| El conductor toca "Aceptar" dos veces, o después de expirar | Red intermitente (`COND-ACC-01`), botón visible en el chat |
| Dos solicitudes simultáneas para la misma franja asignan al mismo conductor | Dos ejecuciones del motor en paralelo |
| Se confirma la asignación pero la notificación se pierde | Caída entre el commit y la llamada a Meta |
| Se reenvía una notificación que Meta ya había aceptado | La llamada a Meta expira sin respuesta; **la API de Meta no ofrece clave de idempotencia para envíos** |

Volumen por operador en el caso ancla: ~40 solicitudes al día, unos 400 eventos internos al día. El problema no es el rendimiento sino la corrección bajo reintentos y concurrencia.

## Decisión

Cinco mecanismos, cada uno responsable de un riesgo. La regla de diseño es que **la última defensa la pone la base de datos**, no el código del motor.

### 1. Inbox idempotente (entrada)
- El webhook verifica la firma, inserta en `mensaje_entrante` con `provider_message_id` como **clave primaria** (`INSERT … ON CONFLICT DO NOTHING`) y responde 200. No procesa en la petición.
- Un reintento de Meta encuentra la fila y no hace nada. El duplicado queda contado en una métrica.
- El Worker procesa `mensaje_entrante` sin procesar en orden de llegada por remitente, y marca `procesado_en` **en la misma transacción** que el efecto de negocio.

### 2. Outbox transaccional (salida y eventos internos)
- Todo efecto posterior a un cambio de estado (`SolicitudRegistrada`, `NotificarConductor`, `NotificarPasajero`, `AlertaOperador`, `ExpirarAsignacion`) se inserta en `outbox_evento` **en la misma transacción** que el cambio.
- El Worker toma lotes con `SELECT … WHERE procesado_en IS NULL AND disponible_en <= now() ORDER BY id LIMIT 20 FOR UPDATE SKIP LOCKED`. Varias réplicas del Worker pueden convivir sin elegir líder y sin tomar la misma fila.
- La entrega es **al menos una vez**. Por eso cada consumidor es idempotente (punto 5).
- Reintento con backoff exponencial y tope: `disponible_en = now() + min(2^intentos, 300) s`. Tras `outbox.max_intentos` (parámetro, por defecto 8) el evento pasa a `estado = 'Fallido'` y se genera `AlertaOperador`. Ese estado cumple el papel de *dead letter*, sin infraestructura extra.
- Cada fila lleva `correlation_id`, que el Worker restaura en el contexto de logging (ADR-0010).

### 3. Concurrencia optimista con `xmin`
- `solicitud_transporte`, `asignacion` y `conversacion` usan la columna de sistema `xmin` como token de concurrencia (`.IsRowVersion()` sobre una propiedad `uint`, [Npgsql — Concurrency tokens](https://www.npgsql.org/efcore/modeling/concurrency.html), consulta 2026-10-28).
- Ante `DbUpdateConcurrencyException`, el manejador recarga la entidad y vuelve a evaluar la máquina de estados. Si la transición ya no es válida (por ejemplo, la asignación ya fue aceptada), termina sin error; si sigue siendo válida, reintenta hasta 3 veces.

### 4. Restricciones de la base de datos (la última defensa)

| Invariante | Restricción |
|---|---|
| Un conductor no tiene dos servicios solapados | `EXCLUDE USING gist (conductor_id WITH =, franja WITH &&) WHERE (estado IN ('Propuesta','Aceptada'))` sobre `asignacion` (extensión `btree_gist`) |
| Un vehículo no está en dos servicios solapados | La misma restricción con `vehiculo_id` |
| Una solicitud tiene como máximo una asignación viva | Índice único parcial `(solicitud_id) WHERE estado IN ('Propuesta','Aceptada')` |
| Un itinerario no se registra dos veces | `UNIQUE (pasajero_id, codigo_servicio, fecha_servicio)` |
| Un mensaje entrante se procesa una vez | PK `mensaje_entrante.provider_message_id` |
| Una notificación lógica se envía una vez | `UNIQUE notificacion_saliente.clave_idempotencia` |

Si el motor elige un conductor que otra transacción acaba de tomar, PostgreSQL rechaza el insert (SQLSTATE `23P01`). El manejador vuelve a ejecutar el motor excluyendo a ese candidato. **El doble agendamiento es imposible aunque el motor tenga un bug.**

### 5. Consumidores idempotentes y envío con incertidumbre controlada
- Cada consumidor de la outbox comprueba el estado antes de actuar. Ejemplo: `NotificarConductor` solo envía si la asignación sigue `Propuesta` y no existe una `notificacion_saliente` con su clave.
- **Clave de idempotencia** = `{tipo}:{agregado_id}:{intento}`. Se guarda en `notificacion_saliente` *antes* de llamar a Meta y se envía en el campo `biz_opaque_callback_data`, que Meta devuelve en los webhooks de estado `sent`, `delivered` y `read` (máximo 512 caracteres; [Messages API](https://developers.facebook.com/docs/whatsapp/cloud-api/reference/messages/), consulta 2026-10-28).
- **Timeout sin respuesta de Meta**: la notificación pasa a `Incierta`. Si en `notificacion.ventana_confirmacion_s` (por defecto 60 s) llega un webhook de estado con esa clave, se marca enviada y **no** se reenvía. Si no llega, se reenvía una sola vez. El duplicado residual (Meta entregó, pero el webhook de estado tardó más que la ventana) se acepta y se declara: es un mensaje repetido al conductor, no un cambio de estado repetido, porque la transición la protegen los puntos 3 y 4.

## Alternativas consideradas

| Alternativa | Qué resuelve | Por qué no |
|---|---|---|
| A. Transacción + llamada directa a Meta (escritura doble) | Simplicidad | Si el proceso cae entre el commit y la llamada, la notificación se pierde sin dejar rastro. Incumple `PAS-DISP-03`. |
| B. Broker externo (RabbitMQ, Azure Service Bus) con outbox | Desacople y fan-out | Unos 400 eventos al día no lo justifican. Agrega un componente por operador (costo, ADR-0009) y además seguiría necesitando la outbox para publicar de forma atómica. |
| C. Bloqueo pesimista (`SELECT … FOR UPDATE` del conductor) en vez de la restricción de exclusión | Evita el doble agendamiento | Depende de que todo camino de código tome el bloqueo en el mismo orden. La restricción lo garantiza aunque un camino nuevo lo olvide, y no abre ventanas de interbloqueo. Puede complementar, no reemplazar. |
| D. *Leader election* con `pg_advisory_lock` para el Worker (fuente anterior) | Evita trabajo duplicado entre réplicas | Serializa todo en un líder y obliga a manejar la pérdida del liderazgo. `SKIP LOCKED` reparte filas sin líder. |
| E. Librería de outbox (MassTransit, Wolverine, CAP) | Menos código propio | MassTransit v9 es comercial ([anuncio](https://masstransit.io/introduction/v9-announcement)). Las demás traen modelos de mensajería que FlowRide no usa. La outbox propia son dos tablas y un `BackgroundService` que se pueden probar. |
| F. `LISTEN/NOTIFY` en vez de sondeo cada segundo | Menor latencia | El sondeo indexado cada `outbox.intervalo_ms` (por defecto 1.000 ms) ya cumple los tiempos de `PAS-DISP-01`. `LISTEN/NOTIFY` queda como optimización si la latencia del motor se acerca al límite. |

## Consecuencias

**Positivas**
- Los tres errores que el negocio no tolera (duplicado, doble agendamiento, pérdida) los cierra un mecanismo con prueba propia.
- El estado se reconstruye desde la base: inbox, outbox, historial y auditoría bastan para explicar qué pasó con una solicitud.
- El Worker se puede escalar a más réplicas sin cambiar el código.

**Negativas**
- Consistencia eventual entre el registro y la asignación: unos segundos en los que la solicitud está `Pendiente`. Es visible para el pasajero ("te aviso cuando tenga conductor") y es aceptable.
- Más tablas y un proceso más (Worker) por operador.
- La restricción de exclusión exige modelar la franja (`tstzrange`) en cada asignación, lo que obliga a conocer la duración del servicio (matriz estática; riesgo declarado en ADR-0007).
- Queda un duplicado residual de notificación, acotado y declarado (punto 5).

## Tácticas de Bass

| Táctica | Mecanismo |
|---|---|
| Detectar fallos — validar condiciones previas | Comprobación de estado antes de cada transición o envío |
| Recuperarse — reintentar | Backoff exponencial en la outbox |
| Recuperarse — degradación / dead letter | Estado `Fallido` y `AlertaOperador` |
| Prevenir fallos — transacciones | Una transacción por manejador: estado, historial, auditoría y outbox |
| Prevenir fallos — restringir el acceso concurrente | `xmin` y la restricción de exclusión |
| Encolar (rendimiento y disponibilidad) | La outbox desacopla el commit de la llamada a Meta; el inbox desacopla Meta del procesamiento |

## Escenarios y requisitos relacionados

- `CANAL-CONF-01` — un webhook repetido por Meta no crea una segunda solicitud.
- `OPLOG-CONF-03` — dos solicitudes concurrentes nunca asignan el mismo conductor a franjas solapadas.
- `PAS-DISP-01` — 0 itinerarios confirmados perdidos aunque el Worker esté caído.
- `PAS-DISP-03` (adaptado a F3) — una falla transitoria de Meta no pierde la notificación al conductor.
- `COND-ACC-01` — un "Aceptar" repetido por mala señal produce una sola transición.
- `COND-CONF-02` (reescrito) — el motor lee la disponibilidad de la fuente transaccional.
- RF-09, RF-10, RF-14, RF-17 y RF-57 (confirmación del conductor).

## Condiciones de revisión

- Si la latencia p95 entre el registro y la asignación propuesta pasa de 5 s, aplicar la alternativa F.
- Si un operador supera unos 50.000 eventos al día o aparece un consumidor fuera del stack, reevaluar la alternativa B.
- Si Meta publica una clave de idempotencia para envíos, eliminar la lógica de `Incierta`.

## Referencias

- Meta — Webhooks (reintentos durante 7 días) y Messages API (`biz_opaque_callback_data`), consulta 2026-10-28.
- Npgsql EF Core — Concurrency tokens con `xmin`, consulta 2026-10-28.
- PostgreSQL — restricciones de exclusión (`EXCLUDE USING gist`) y la extensión `btree_gist`.
- `docs/_fuentes/PoC_y_Spikes_FlowRide_CORREGIDO.xlsx` — Spike 2, "Consistencia bajo concurrencia".
