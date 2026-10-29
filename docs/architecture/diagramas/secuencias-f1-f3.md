# Diagramas de secuencia — F1, F2 y F3

**Propósito**: mostrar el recorrido exacto de cada funcionalidad, incluidos los caminos de falla que ponen a prueba los escenarios de calidad.
**Alcance**: el stack de un operador. Los participantes son contenedores (C4 nivel 2) y componentes internos (`componentes.md`).
**Estado**: Diseñado. Cada flecha indica su protocolo. Las líneas `TX` marcan el inicio y el fin de una transacción PostgreSQL.

---

## S1 — F1: registro de itinerario (camino normal, último paso)

Los pasos 1–12 de la conversación (`diseno-detallado.md` §1.1) siguen el mismo patrón: webhook → inbox → Worker → `conversacion` → respuesta. Aquí se muestra el paso que crea la solicitud.

```mermaid
sequenceDiagram
    autonumber
    actor P as Pasajero
    participant M as Meta Cloud API
    participant API as API (WhatsAppWebhookController)
    participant DB as BD del operador
    participant W as Worker (ProcesadorInbox)
    participant UC as RegistrarItinerario (caso de uso)
    participant NG as NotificationGateway

    P->>M: toca [Confirmar] (app WhatsApp)
    M->>API: POST /webhooks/whatsapp [HTTPS, X-Hub-Signature-256]
    API->>API: verifica la firma sobre el cuerpo crudo y phone_number_id
    API->>DB: INSERT mensaje_entrante ON CONFLICT DO NOTHING [TCP/Npgsql]
    API-->>M: 200 OK (sin procesar todavía)
    W->>DB: SELECT inbox pendiente … FOR UPDATE SKIP LOCKED
    W->>DB: busca usuario por telefono_e164 (pasajero activo)
    W->>UC: Confirmar(conversacion)
    Note over UC,DB: TX inicio
    UC->>DB: INSERT itinerario (UNIQUE pasajero, código, fecha)
    UC->>DB: INSERT solicitud_transporte (Pendiente, recogida, franja)
    UC->>DB: INSERT estado_solicitud_historial, auditoria
    UC->>DB: INSERT outbox_evento SolicitudRegistrada
    UC->>DB: UPDATE conversacion → Inactiva, mensaje_entrante.procesado_en
    Note over UC,DB: TX fin (commit)
    W->>NG: Notificar(itinerario.confirmado) — sesión abierta
    NG->>DB: INSERT notificacion_saliente (clave de idempotencia)
    NG->>M: POST /{phone_number_id}/messages [HTTPS, Bearer, biz_opaque_callback_data]
    M-->>NG: 200 {wamid}
    M->>P: "Listo. Tu itinerario quedó registrado como SR-000123…"
```

## S2 — F1: falla — webhook duplicado y firma inválida (`CANAL-CONF-01`, `CANAL-SEG-01`)

```mermaid
sequenceDiagram
    autonumber
    participant M as Meta Cloud API
    participant X as Atacante
    participant API as API
    participant DB as BD del operador

    M->>API: POST webhook wamid=W1 (reintento: el 200 anterior no llegó a Meta)
    API->>DB: INSERT mensaje_entrante W1 ON CONFLICT DO NOTHING
    DB-->>API: 0 filas (ya existía)
    API->>API: métrica webhook_duplicados_total +1
    API-->>M: 200 OK — no se procesa dos veces

    X->>API: POST webhook con firma inválida o ausente
    API->>API: HMAC no coincide
    API->>DB: INSERT auditoria (firma_invalida, actor Desconocido)
    API-->>X: 401 — no se escribe en el inbox
```

## S3 — F2: asignación automática, incluido el conflicto de concurrencia (`OPLOG-CONF-03`, `OPLOG-CONF-04`)

```mermaid
sequenceDiagram
    autonumber
    participant W as Worker (ProcesadorOutbox)
    participant UC as AsignarConductor (caso de uso)
    participant MA as MotorAsignacion (Dominio)
    participant PC as ParameterCatalog
    participant DB as BD del operador

    W->>DB: SELECT outbox SolicitudRegistrada … FOR UPDATE SKIP LOCKED
    W->>UC: Ejecutar(solicitudId, correlationId)
    UC->>DB: carga la solicitud (Pendiente, xmin)
    UC->>PC: lee T, X, margen, N, alistamiento, retorno
    UC->>DB: candidatos: conductores con turno que cubre la franja y check-in si ya empezó
    UC->>MA: Evaluar(solicitud, candidatos, reglas)
    MA->>MA: Nivel 1: licencia C vigente, SOAT y RTM vigentes (fijas)
    MA->>MA: Nivel 2: activo, capacidad, no rechazó antes
    MA->>MA: Nivel 3: puntaje balance_carga_dia y desempate determinista
    MA-->>UC: elegido + razones (descartes, puntajes)
    Note over UC,DB: TX inicio
    UC->>DB: INSERT asignacion (Propuesta, franja, expira_en, razones)
    alt otro proceso tomó al conductor en esa franja
        DB-->>UC: SQLSTATE 23P01 (violación de la restricción de exclusión)
        Note over UC,DB: TX rollback
        UC->>MA: Evaluar de nuevo excluyendo a ese conductor
    else la inserción pasa
        UC->>DB: UPDATE solicitud → AsignacionPropuesta (xmin)
        UC->>DB: INSERT historial, auditoria (razones)
        UC->>DB: INSERT outbox NotificarConductor
        Note over UC,DB: TX fin
    end
    alt no hay candidatos
        UC->>DB: solicitud → SinCandidato, INSERT alerta, outbox AlertaOperador
    end
```

## S4 — F3: el conductor acepta (`COND-SEG-01`, `COND-ACC-01`)

```mermaid
sequenceDiagram
    autonumber
    actor C as Conductor
    actor P as Pasajero
    participant M as Meta Cloud API
    participant API as API
    participant W as Worker
    participant NG as NotificationGateway
    participant UC as ResponderAsignacion (caso de uso)
    participant DB as BD del operador

    Note over C,M: Antes: el conductor escribió "INICIO TURNO" y abrió la ventana de 24 h (RF-58)
    W->>NG: NotificarConductor (outbox)
    NG->>DB: ¿conversacion.ultimo_entrante_en < 24 h? → sí: mensaje de sesión
    NG->>M: POST /messages interactivo [Aceptar][Rechazar] (id asg:{id}:A/R)
    M->>C: Propuesta SR-000123
    C->>M: toca [Aceptar]
    M->>API: POST webhook (button id = asg:{id}:A) [HTTPS, firma]
    API->>DB: INSERT inbox, 200
    W->>UC: Aceptar(asignacionId, wa_id)
    UC->>DB: carga la asignación y el conductor por teléfono
    alt wa_id ≠ conductor de la asignación, o asignación no Propuesta o vencida
        UC->>DB: INSERT auditoria (rechazo de la acción)
        UC-->>W: responder asignacion.no_disponible
    else válido
        Note over UC,DB: TX inicio
        UC->>DB: asignacion → Aceptada, solicitud → Confirmada (xmin)
        UC->>DB: historial, auditoria, outbox NotificarPasajero
        Note over UC,DB: TX fin
    end
    W->>NG: NotificarPasajero → sesión o plantilla confirmacion_pasajero
    NG->>M: POST /messages
    M->>P: "Tu servicio SR-000123 está confirmado. Conductor Juan R., placa ABC123…"
```

Un segundo toque de *Aceptar* (mala señal) llega con otro `wamid`. El caso de uso encuentra la asignación ya `Aceptada`, responde `asignacion.no_disponible` y no hay una segunda transición.

## S5 — F3: rechazo o expiración → reencolado → escalamiento al operador (`PAS-CONF-03`)

```mermaid
sequenceDiagram
    autonumber
    participant W as Worker (barrido cada minuto)
    participant UC as GestionarFallaAsignacion
    participant DB as BD del operador
    participant NG as NotificationGateway
    participant M as Meta Cloud API
    actor O as Operador logístico

    alt el conductor tocó [Rechazar]
        W->>UC: Rechazar(asignacionId, wa_id) — con la misma validación que en S4
    else expira_en < now() sin respuesta
        W->>DB: SELECT asignacion Propuesta con expira_en vencido
        W->>UC: Expirar(asignacionId)
    end
    Note over UC,DB: TX inicio
    UC->>DB: asignacion → Rechazada o Expirada, solicitud.intentos +1
    alt intentos < N y la hora límite no pasó
        UC->>DB: solicitud → Pendiente, outbox SolicitudRegistrada (vuelve a S3, excluyendo a ese conductor)
    else intentos = N, o sin tiempo
        UC->>DB: solicitud → SinCandidato, INSERT alerta, outbox AlertaOperador
    end
    UC->>DB: historial, auditoria
    Note over UC,DB: TX fin
    W->>NG: AlertaOperador → número con recibe_alertas = true
    NG->>M: POST /messages (sesión si la ventana está abierta, si no, plantilla alerta_operativa [Diseñado])
    M->>O: "Alerta operativa SR-000123: sin conductor tras 3 intentos…"
```

## S6 — Falla del proveedor al enviar (`PAS-DISP-03` adaptado)

```mermaid
sequenceDiagram
    autonumber
    participant W as Worker (ProcesadorOutbox)
    participant NG as NotificationGateway
    participant DB as BD del operador
    participant M as Meta Cloud API

    W->>NG: NotificarConductor (intento k)
    NG->>DB: notificacion_saliente Registrada (clave)
    NG->>M: POST /messages
    alt 5xx o 429
        M-->>NG: error
        NG->>DB: outbox.intentos +1, disponible_en = now + min(2^k, 300) s
    else timeout sin respuesta
        NG->>DB: notificacion → Incierta
        Note over NG,M: espera un webhook de estado con biz_opaque_callback_data = clave (60 s)
        alt llega el estado sent o delivered
            NG->>DB: notificacion → Enviada (no reenvía)
        else no llega
            NG->>M: reenvía una sola vez (riesgo residual de duplicado declarado)
        end
    end
    alt intentos = outbox.max_intentos
        NG->>DB: outbox → Fallido, alerta NotificacionFallida, outbox AlertaOperador
    end
    Note over W,M: Varios fallos seguidos abren el circuit breaker del HttpClient, y los eventos quedan Pendiente hasta que se cierre.
```
