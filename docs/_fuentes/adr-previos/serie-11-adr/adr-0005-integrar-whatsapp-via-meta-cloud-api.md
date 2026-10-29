# ADR-0005: integrar WhatsApp vía Meta Cloud API

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0005 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-05 del tradeoff anterior (Twilio como BSP intermedio) y consolida el ADR-0005 de ejemplo. |

## Contexto

WhatsApp es el único canal funcional para Pasajero y Conductor. El proveedor de integración condiciona disponibilidad, costo, lock-in, seguridad del webhook y capacidad de evolución. La versión anterior había fijado Twilio como BSP, pero el diseño actualizado exige desacoplar la lógica de negocio del proveedor.

El ADR de ejemplo del proyecto ya plantea Meta Cloud API directa como la alternativa que mejor evita intermediación innecesaria. Se conserva esa dirección y se corrigen los puntos que habían quedado incompatibles con los documentos más recientes: no existe canal funcional secundario SMS/correo para Pasajero/Conductor y la telemetría GPS es una capacidad V1 separada.

## Decisión

Integrar WhatsApp Business Platform directamente contra **Meta Cloud API**, sin BSP intermedio como ruta por defecto.

Encapsular el cliente de Meta detrás de un `WhatsAppBusinessAdapter` / Anti-Corruption Layer. El dominio solo conoce comandos, mensajes y eventos propios de FlowRide; nunca payloads específicos de Meta.

El adapter debe implementar validación de webhook, idempotencia, reintentos con backoff, persistencia/cola de mensajes pendientes cuando corresponda y observabilidad. Una librería comunitaria .NET puede utilizarse internamente siempre que pueda sustituirse por cliente HTTP propio sin afectar Application/Domain.

## Alternativas consideradas

### Alternativa A — Twilio como BSP
Descartada como ruta principal por añadir intermediación, costo y lock-in sin necesidad funcional demostrada para FlowRide.

### Alternativa B — 360dialog / Bird / Gupshup
Descartadas por el mismo motivo: un BSP solo se justificaría si una capacidad concreta agregada compensa la intermediación.

### Alternativa C — Meta Cloud API directa
Elegida porque reduce intermediarios y mantiene al adapter como frontera de cambio.

### Alternativa D — App móvil propia
Descartada: contradice la decisión de WhatsApp como canal de baja fricción.

### Alternativa E — SMS/Telegram/correo como canal funcional
Descartadas para Pasajero y Conductor. Pueden existir como mecanismos administrativos externos, pero no forman parte del canal de producto.

## Consecuencias positivas

- Menos intermediarios en el camino crítico.
- Menor lock-in comercial con BSP.
- El adapter permite reemplazar cliente/librería sin tocar dominio.
- Se mantiene una única experiencia de usuario para Pasajero y Conductor.

## Consecuencias negativas

- El equipo asume más lógica propia de webhook, plantillas, errores e idempotencia.
- Meta sigue siendo dependencia crítica externa.
- Los cambios de políticas o contratos de la plataforma deben absorberse en el adapter.

## Tácticas de Bass aplicadas

- **Adaptar interfaz** — traducir payload de Meta al lenguaje de FlowRide.
- **Reintento con backoff** — recuperar fallos transitorios.
- **Encolar** — preservar mensajes pendientes/fallidos.
- **Autenticar actores** — verificar autenticidad del webhook y remitente.
- **Reducir acoplamiento** — evitar dependencias del SDK fuera del adapter.

## Escenarios y requisitos relacionados

- `COND-DISP-01` — disponibilidad ≥99,9% del canal.
- `PAS-DISP-01` — recepción confiable de solicitudes.
- `PAS-DISP-03` — reintentos ante interrupciones transitorias.
- `COND-ESC-01` — absorción de picos de mensajes.
- `PAS-UX-01` — solicitud sin app adicional.
- `RF-14` — recepción de mensajes.
- `RF-15` — procesamiento conversacional.
- `RF-16 / RF-17 / RF-18` — confirmaciones, notificaciones y estados.

## ADRs relacionados

- **ADR-0003** — define identidad/autorización del usuario detrás del canal.
- **ADR-0006** — protege consistencia e idempotencia del procesamiento.
- **ADR-0011** — instrumenta disponibilidad, errores y trazas.

## Notas de implementación

- Exponer `POST /api/whatsapp/webhook` solo en API/Infrastructure.
- Normalizar mensajes entrantes a comandos internos antes de invocar Application.
- Usar un identificador idempotente del mensaje externo para evitar doble procesamiento.
- Las credenciales de Meta se administran como secretos por entorno y nunca se versionan.
- Una interrupción del canal genera alerta operativa; FlowRide no define otro canal funcional para Pasajero/Conductor.

## Referencias

- Meta for Developers — WhatsApp Business Platform / Cloud API.
- `adr-0005-integrar-whatsapp-via-meta-cloud-api.md` proporcionado como estructura y antecedente.
- `RestriccionesNegocio_FlowRide_ACTUALIZADO.xlsx`.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx`.
