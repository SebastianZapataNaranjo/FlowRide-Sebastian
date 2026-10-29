# Escenarios de calidad de F1–F3

Formato SEI de seis partes, más el **motivador** (por qué importa al negocio) y la **prueba que lo mide para el 4-nov**. Los códigos `STAKEHOLDER-ATRIBUTO-NN` existentes vienen del banco de 33 escenarios seleccionados (`docs/_fuentes/tradeoff_FlowRide_ACTUALIZADO.xlsx`, hoja "Escenarios Sei seleccionados(33)"). Los nuevos siguen la misma convención. Los cambios frente al banco están en `architecture/registro-de-cambios-vs-fuentes.md`.

**Entorno de medición para el 4-nov**: `docker compose` local (`architecture/diagramas/despliegue.md` D1). Lo que solo puede medirse en producción (disponibilidad mensual) se dice explícitamente y **no** se presenta como medido.

## Resumen

| Código | Atributo | Origen | F | ADR | Prueba |
|---|---|---|---|---|---|
| `PAS-DISP-01` | Disponibilidad | Banco, reformulado | F1 | 0006, 0009 | Carga con k6 + caída del Worker/API con reintentos |
| `COND-DISP-01` | Disponibilidad / Rendimiento | Banco, reformulado | F3 | 0003, 0009 | k6 sobre la respuesta al conductor |
| `PAS-UX-01` | Usabilidad | Banco | F1 | 0005, 0008 | E2E automatizado + prueba con 5 personas |
| `COND-SEG-01` | Seguridad | Banco | F3 | 0005 | Prueba de integración negativa |
| `COND-ACC-01` | Accesibilidad (conectividad) | Banco | F3 | 0006 | Prueba de integración con duplicados y retraso |
| `PAS-DISP-03` | Disponibilidad | Banco, adaptado a F3 | F3 | 0006, 0008 | Falla inyectada en el simulador |
| `COND-CONF-02` | Confiabilidad | Banco, reescrito | F2 | 0006, 0007 | Prueba de concurrencia |
| `CANAL-SEG-01` | Seguridad | Nuevo | F1, F3 | 0003, 0005 | Prueba de contrato del webhook |
| `CANAL-CONF-01` | Confiabilidad | Nuevo | F1, F3 | 0006 | Prueba de contrato con reintento |
| `OPLOG-CONF-03` | Confiabilidad | Nuevo | F2 | 0006, 0004 | Concurrencia con Testcontainers |
| `OPLOG-CONF-04` | Confiabilidad (regulatoria) | Nuevo | F2 | 0007 | Pruebas unitarias del motor |
| `PAS-CONF-03` | Confiabilidad / Disponibilidad | Nuevo | F2, F3 | 0007 | Integración con reloj controlado |
| `PLAT-AIS-01` | Aislamiento | Nuevo | F1–F3 | 0002 | Prueba de aislamiento en compose |
| `EQ-MANT-01` | Mantenibilidad | Nuevo (QA-11 de Drivers) | F2 | 0001, 0007 | NetArchTest + regla nueva de demostración |

---

## Escenarios sustentados (E2E)

### PAS-DISP-01 — Itinerario en la madrugada sin pérdidas
| Parte | Contenido |
|---|---|
| Fuente | Pasajero |
| Estímulo | Confirma un itinerario por WhatsApp a las 03:00 |
| Artefacto | Webhook (API), inbox, Worker, base del operador |
| Entorno | (a) Operación normal con 20 pasajeros confirmando en el mismo minuto. (b) El Worker o la API del stack están caídos hasta 15 minutos |
| Respuesta | Se registra la solicitud y se confirma al pasajero. Si el stack está caído, el mensaje queda pendiente (reintento de Meta o inbox) y se procesa al recuperarse |
| Medida | (a) Confirmación al pasajero P95 < 10 s desde la recepción del webhook. (b) **0 itinerarios confirmados perdidos**; inbox pendiente vaciado ≤ 2 min después de la recuperación. Disponibilidad mensual del stack ≥ 99,5 %, **medible solo en producción** (ADR-0009) |
| Motivador | Banco: "si a las 3am el sistema no me deja pedir el transporte, mi vuelo se me va". El 99,9 % original se reemplaza porque no es alcanzable con un proveedor sin SLA en la cadena (cálculo en ADR-0009) |
| **Prueba** | k6 envía 20 confirmaciones por `whatsapp-sim` en 60 s y mide el tiempo hasta el mensaje de salida (bandeja del simulador). Caída: `docker compose stop worker-b`, se inyectan 20 confirmaciones, se espera 5 min, `start worker-b`, y se verifica que haya 20 solicitudes y 20 confirmaciones. Variante con `api-b` detenida: el simulador reintenta como Meta (cada 30 s) y el resultado es el mismo |

### COND-DISP-01 — El conductor recibe respuesta en el arranque del turno
| Parte | Contenido |
|---|---|
| Fuente | Conductor |
| Estímulo | Envía "INICIO TURNO" o toca Aceptar o Rechazar |
| Artefacto | API, Worker, `ResponderAsignacion` / `RegistrarCheckin` |
| Entorno | Franja 03:00–05:00, con 5 conductores actuando a la vez |
| Respuesta | El sistema confirma la acción al conductor |
| Medida | Respuesta P95 < 5 s de extremo a extremo dentro del stack (sin contar la latencia de Meta); disponibilidad mensual del stack ≥ 99,5 % (solo producción) |
| Motivador | Banco: el WhatsApp es el único canal del conductor; si no responde, "no sé a quién recoger ni a qué hora" |
| **Prueba** | k6 por el simulador: 5 conductores × check-in + aceptar, repetido 20 veces; P95 del tiempo entre la inyección y el mensaje de salida |

### PAS-UX-01 — Primer registro sin capacitación
| Parte | Contenido |
|---|---|
| Fuente | Pasajero nuevo |
| Estímulo | Registra su primer itinerario sin instrucciones previas |
| Artefacto | Conversación guiada (Canal) y catálogo de mensajes |
| Entorno | Celular propio, solo WhatsApp |
| Respuesta | El sistema guía con botones y listas hasta la confirmación |
| Medida | Flujo de 7 interacciones del pasajero como máximo (solo 2 textos libres: código y hora); tiempo promedio < 3 min |
| Motivador | Banco: valida la decisión de canal; "si necesita capacitación para pedir, la decisión de canal fracasa". El argumento es la experiencia en la madrugada, no el costo de los mensajes (9–15 USD/mes, ADR-0005) |
| **Prueba** | (1) E2E automatizado que recorre el flujo por el simulador y verifica 7 salientes y 7 entrantes. (2) Prueba con 5 personas ajenas al proyecto en el tenant A real, con tiempo cronometrado. **Limitación**: 5 participantes es evidencia indicativa, no el ≥ 90 % estadístico del banco; ese porcentaje queda para el piloto con el cliente ancla |

### COND-SEG-01 — Nadie actúa en nombre del conductor
| Parte | Contenido |
|---|---|
| Fuente | Tercero no autorizado (otro número, o un botón reenviado) |
| Estímulo | Envía `asg:{id}:A` sobre una asignación que no es suya |
| Artefacto | `ResponderAsignacion`, repositorio `UsuarioPorTelefono` |
| Entorno | Operación normal |
| Respuesta | Rechaza la acción, no cambia el estado, responde `asignacion.no_disponible` y audita |
| Medida | 100 % de las acciones críticas (aceptar, rechazar, check-in) validadas contra el número registrado; 0 transiciones desde números no asociados |
| Motivador | Banco: "nadie más puede aceptar servicios en mi nombre"; responsabilidad legal del conductor |
| **Prueba** | Integración: 4 casos (número ajeno; conductor de otra asignación; número inactivo; número correcto) → solo el último cambia el estado; la auditoría tiene 3 filas de rechazo |

### COND-ACC-01 — Señal intermitente sin pérdida ni duplicado
| Parte | Contenido |
|---|---|
| Fuente | Conductor con conectividad intermitente |
| Estímulo | Su "Aceptar" llega dos veces (reenvío del cliente) o 8 minutos tarde |
| Artefacto | Inbox, máquina de estados de la asignación |
| Entorno | Zona de baja cobertura |
| Respuesta | Una sola transición; si llega después de `expira_en`, responde "ya no disponible" y no reabre |
| Medida | 0 transiciones duplicadas; 0 pérdidas del estado conversacional ante una interrupción de hasta 10 min (`conversacion.expiracion_min` = 30) |
| Motivador | Banco: "si la app no aguanta con señal baja, quedo perdido" |
| **Prueba** | Integración: dos webhooks con distinto `wamid` y el mismo botón → 1 transición; el mismo botón con el reloj adelantado más allá de `expira_en` → sin transición |

### PAS-DISP-03 (adaptado a F3) — La propuesta al conductor sobrevive a una falla de Meta
| Parte | Contenido |
|---|---|
| Fuente | Proveedor de WhatsApp |
| Estímulo | Responde 5xx o expira durante 5 min al enviar `NotificarConductor` |
| Artefacto | `ProcesadorOutbox`, `NotificationGateway`, `ClienteGraphApi` |
| Entorno | Degradación parcial del proveedor |
| Respuesta | Reintenta con backoff; el *circuit breaker* deja de insistir; al recuperarse, entrega |
| Medida | ≥ 99 % de las notificaciones afectadas entregadas ≤ 2 min después de la recuperación; duplicado visible solo en el caso residual `Incierta` sin webhook de estado (ADR-0006 §5) |
| Motivador | Banco: "los fallos transitorios no deben convertir una notificación en una pérdida silenciosa". Se adapta de recordatorios (Diseñado) a la propuesta al conductor, que sí es E2E |
| **Prueba** | El simulador tiene un modo `falla=5xx|timeout` por 5 min; se encolan 10 propuestas y se mide la entrega tras desactivar la falla. Caso `timeout` con webhook de estado → 0 reenvíos |

### COND-CONF-02 (reescrito) — El motor usa el último estado confirmado del conductor
| Parte | Contenido |
|---|---|
| Fuente | Conductor |
| Estímulo | Hace check-in en el mismo instante en que el motor evalúa candidatos |
| Artefacto | `AsignarConductor`, `turno_conductor` |
| Entorno | Alta demanda en el arranque del turno |
| Respuesta | El motor lee el estado desde la fuente transaccional; si el check-in confirmó antes de la lectura, lo considera; si no, el conductor entra en la siguiente evaluación |
| Medida | 0 asignaciones basadas en un estado servido desde caché (no existe caché en la ruta); 0 asignaciones a conductores que no cumplan el filtro de presencia en el momento del commit |
| Motivador | Banco: servicios que "se esfuman" rompen la confianza del conductor. La versión original prescribía una caché sincronizada por outbox; se elimina (ADR-0012) |
| **Prueba** | Concurrencia: 200 ciclos de check-in y asignación paralelos (Testcontainers) → ninguna asignación propuesta a un conductor sin check-in cuando su turno ya empezó |

### CANAL-SEG-01 — Webhook falsificado rechazado
| Parte | Contenido |
|---|---|
| Fuente | Atacante que conoce la URL pública del webhook |
| Estímulo | POST con firma ausente, inválida o hecha con otro secreto; o con un `phone_number_id` de otro operador |
| Artefacto | `FiltroFirmaWebhook`, `WhatsAppWebhookController` |
| Entorno | Producción (expuesto a Internet) |
| Respuesta | 401 (o 403 si el `phone_number_id` no coincide), sin escribir en el inbox; se audita |
| Medida | 100 % de las peticiones mal firmadas rechazadas; 0 filas en el inbox |
| Motivador | Sin esta defensa, cualquiera podría crear solicitudes o aceptar servicios en nombre de otros |
| **Prueba** | Contrato: 5 casos (sin encabezado, firma alterada, cuerpo alterado tras firmar, secreto del stack A enviado al B, `phone_number_id` ajeno) + 1 válido |

### CANAL-CONF-01 — Reintentos del proveedor sin duplicados
| Parte | Contenido |
|---|---|
| Fuente | Meta (o el simulador) |
| Estímulo | Reenvía el mismo `wamid` 3 veces |
| Artefacto | Inbox |
| Entorno | Operación normal tras un timeout de red |
| Respuesta | Procesa una vez y responde 200 a los tres |
| Medida | 1 solicitud, 1 confirmación; métrica `webhook_duplicados_total` = 2; ack del webhook P95 ≤ 500 ms (cifra interna: cuanto antes responde el webhook, menos reintentos genera Meta) |
| Motivador | Meta reintenta hasta 7 días y advierte que puede duplicar ([Set up webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks), consulta 2026-10-28) |
| **Prueba** | Contrato: el mismo cuerpo firmado enviado 3 veces → 1 fila en `mensaje_entrante` y 1 solicitud |

### OPLOG-CONF-03 — Doble agendamiento imposible
| Parte | Contenido |
|---|---|
| Fuente | Dos ejecuciones del motor |
| Estímulo | Dos solicitudes con franjas solapadas y un solo conductor elegible, procesadas a la vez |
| Artefacto | `AsignarConductor`, restricción de exclusión |
| Entorno | Concurrencia real sobre PostgreSQL |
| Respuesta | Una asignación propuesta; la otra reintenta excluyendo y termina `SinCandidato` con alerta |
| Medida | 0 asignaciones solapadas confirmadas para un conductor o vehículo en 1.000 ejecuciones |
| Motivador | Banco: "no puede pasar que me asignen dos servicios incompatibles" |
| **Prueba** | Testcontainers con PostgreSQL 17: 1.000 pares concurrentes; consulta de verificación de solapes = 0 filas; se cuentan las violaciones `23P01` capturadas |

### OPLOG-CONF-04 — Nunca se asigna con documentos vencidos
| Parte | Contenido |
|---|---|
| Fuente | Motor de asignación |
| Estímulo | El mejor candidato por puntaje tiene la licencia, el SOAT o la RTM vencidos a la fecha del servicio |
| Artefacto | `MotorAsignacion`, reglas de nivel 1 |
| Entorno | Cualquier configuración de puntaje del operador |
| Respuesta | Lo descarta y lo registra en `razones` con la regla que lo eliminó |
| Medida | 0 asignaciones a conductores o vehículos con documentos vencidos; 100 % de los descartes explicados |
| Motivador | Ley 769 de 2002 (art. 42 SOAT, art. 51 RTM) y categoría C para servicio público (ADR-0007) |
| **Prueba** | Unitarias del motor: matriz de casos (vence el día anterior, el mismo día y el siguiente al servicio) × 3 documentos, con el peso de puntaje al máximo; resultado idéntico al repetir (PoC 1) |

### PAS-CONF-03 — Ninguna solicitud queda huérfana
| Parte | Contenido |
|---|---|
| Fuente | Conductores que rechazan o no responden |
| Estímulo | Las N propuestas sucesivas fallan |
| Artefacto | `GestionarFallaAsignacion`, `BarridoExpiraciones`, `NotificationGateway` |
| Entorno | Servicio a las 04:30 asignado a las 22:00 (servicio lejano) y servicio en 40 min (cercano) |
| Respuesta | Reencola hasta N; luego `SinCandidato` y alerta por WhatsApp al operador |
| Medida | 100 % de las solicitudes terminan `Confirmada` o con alerta enviada antes de la hora límite (`recogida − margen_confirmacion`) |
| Motivador | Incidente real del caso ancla: el abandono de un conductor no detectado a tiempo y el pasajero que llega tarde |
| **Prueba** | Integración con `IReloj` controlado: 3 conductores que expiran → la alerta sale en el tiempo calculado; caso lejano → la propuesta expira en la hora límite, no a los 15 min |

### PLAT-AIS-01 — Aislamiento entre operadores
| Parte | Contenido |
|---|---|
| Fuente | Mensaje dirigido al operador B / configuración errónea |
| Estímulo | (a) Un webhook firmado para B llega a A. (b) El proceso de A intenta abrir la base de B |
| Artefacto | Configuración del stack, validación de `phone_number_id`, permisos de PostgreSQL |
| Entorno | Compose con dos stacks |
| Respuesta | (a) Rechazo por firma o `phone_number_id`. (b) PostgreSQL niega el acceso al usuario `app_a` |
| Medida | 0 lecturas o escrituras cruzadas; 100 % de los intentos rechazados |
| Motivador | Cláusula contractual de aislamiento y diferenciador 1 |
| **Prueba** | Script de aislamiento: webhooks cruzados en ambos sentidos; conexión con las credenciales de A a `flowride_tenant_b` → error de permiso; recuento de filas sin cambios en ambas bases |

### EQ-MANT-01 — Una regla nueva no toca otros módulos
| Parte | Contenido |
|---|---|
| Fuente | Equipo de desarrollo |
| Estímulo | Agrega la regla de puntaje `continuidad_vehiculo` |
| Artefacto | `Dominio.Asignacion.Reglas`, `regla_asignacion` |
| Entorno | Desarrollo |
| Respuesta | La regla se agrega como una clase y una fila de configuración |
| Medida | 0 archivos modificados fuera de `Dominio.Asignacion` y su registro en DI; 100 % de las reglas NetArchTest y de las pruebas existentes en verde |
| Motivador | Diferenciador 4 (reglas por operador); QA-11 de Drivers |
| **Prueba** | NetArchTest en CI (reglas C1–C8, M1–M4 de `paquetes.md`) + un PR de demostración cuyo diff muestre solo esos archivos |

---

## Escenarios Diseñados (no se miden el 4-nov)

| Código | Por qué queda Diseñado | Qué lo habilita |
|---|---|---|
| `OPLOG-REND-01` | Reasignación manual desde el panel con auditoría por persona | Panel + OIDC (ADR-0011) |
| `PAS-REND-02` | La consulta "¿cómo va mi solicitud?" no está en la cadena F1→F3 | Opción "Mis servicios" en el canal |
| `ADM-DISP-01` | Failover ≤ 30 s exige el nivel contractual Crítico | ADR-0009 §4 |
| `ADM-SEG-01` | La revocación ≤ 60 s aplica a sesiones del panel; en el canal, desactivar al usuario surte efecto en el siguiente mensaje | ADR-0011 |
| `OPLOG-AUD-01` | La auditoría existe y se consulta por `GET /admin/auditoria`; la vista del panel es V1 | ADR-0011 |
| `OPLOG-SEG-02`, `ADM-SEG-02` | Aislamiento desde el panel | ADR-0011; la parte del canal la cubre `PLAT-AIS-01` |
| `PAS-CONF-01`, `TEL-INTEROP-01` | Telemetría y ETA | ADR-0013 |
| `PAS-AUD-02`, `PAS-AUD-03` | Cancelación y calificación no están en F1–F3 | V1 |
| `OPLOG-DISP-02` | Alertas en el panel (SSE) | ADR-0011 |
| `COND-PORT-01` | Lo cumple la identidad por número (ADR-0005); no se le dedica prueba propia | — |
| Accesibilidad y UX del panel (`OPLOG-ACC-01/02`, `ADM-ACC-01`, `OPLOG-UX-02`, `ADM-UX-02`) | No hay panel en la entrega | ADR-0011 |
| `COND-ESC-01` | El techo se corrige a 50 conductores; la prueba de carga de la línea base usa el volumen del caso ancla | Prueba de carga en el piloto de Azure |
| `OPLOG-CAP-01`, `ADM-CAP-02`, `ADM-REND-01` | Dependen del panel y los reportes | V1 |
