# ADR-0005: integrar whatsapp vía meta cloud api

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0005 |
| Fecha | 2026-09-02 |
| Estado | **Proposed** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-05 del `tradeoff_v6.xlsx` (Twilio como BSP intermedio), no formalizado previamente como .md |

## Contexto

FlowRide define WhatsApp como el único canal de entrada para los roles Pasajero y Conductor: no existe app móvil propia ni panel web para esos dos roles. Por eso la decisión sobre cómo se integra el backend con WhatsApp Business Platform condiciona directamente la disponibilidad, la seguridad de identidad, la experiencia de uso y el modelo de costos operativos del sistema completo. No es una decisión puntual del módulo de mensajería, es una decisión de arquitectura del canal principal.

En el semestre 1-2026 (curso de Diseño Orientado a Objetos, otro profesor) se registró en el tradeoff una decisión inicial de integrar vía Twilio como BSP intermedio, con dos argumentos: la lentitud de Meta para aprobar cuentas empresariales y la velocidad de desarrollo apoyada en el SDK oficial de Twilio para .NET. Al re-evaluar esa decisión en el marco de Software 2, con datos del mercado de 2026, el primero de esos argumentos no se sostiene:

- Meta deprecó la On-Premise API en octubre de 2025. Toda integración nueva corre sobre Cloud API, hospedada por Meta, gratis a nivel de API y abierta a cualquier desarrollador.
- Todos los BSPs (Twilio, 360dialog, Bird, Gupshup, Wati) revenden acceso a la misma plataforma subyacente y comparten el mismo piso de costo: los precios de Meta.
- La verificación de negocio de Meta es requisito para cualquier despliegue productivo, sin importar si se va directo o a través de un BSP. Pasar por Twilio no evita ese paso — solo agrega un sandbox de desarrollo alterno.

El valor real de un BSP en 2026 son las herramientas propias que añaden encima (inbox, campaign builder, chatbot low-code, dashboards, integración con Shopify). Como FlowRide construye su propio dominio, motor de asignación, módulo de notificaciones y adapter de integración, ninguna de esas herramientas aplica al caso de uso. El BSP quedaría como pura intermediación técnica — con costo mensual y lock-in — sin aportar valor arquitectónico.

## Decisión

Integrar WhatsApp Business Platform directamente contra **Meta Cloud API**, sin BSP intermedio. Encapsular la integración en un módulo *Integración WhatsApp* con un adapter interno (patrón Adapter / Anti-Corruption Layer) que aísle al resto del sistema del cliente HTTP específico de Meta.

Consumir como cliente inicial la librería comunitaria **`WhatsAppBusiness.CloudApi`** (NuGet, licencia MIT), con la posibilidad de reemplazarla por un cliente HTTP propio dentro del mismo adapter si en el futuro esa librería queda sin mantenimiento — sin impacto en el resto del sistema.

Durante el desarrollo del semestre, usar los **5 números de prueba gratuitos** que Meta provee a cada cuenta de desarrollo Cloud API. Para producción, verificar cuenta Meta Business Manager con documentos legales de la empresa cliente. Esta verificación es requisito común a cualquier ruta y no es factor diferenciador entre alternativas.

## Alternativas consideradas

### Alternativa A — Twilio como BSP intermedio (segundo lugar)
SDK .NET oficial, quickstart maduro, sandbox de desarrollo con código de join, comunidad grande. Reduce fricción de aprendizaje para un equipo sin experiencia previa. Descartada porque agrega un intermediario técnico y comercial que no aporta features aprovechables por FlowRide, cobra ~$0.005 USD adicionales por mensaje sobre el rate card de Meta (representa 20–40% del costo total), y genera lock-in que después habría que desmontar. El beneficio del SDK se mitiga en Meta directa con la librería comunitaria `WhatsAppBusiness.CloudApi`.

### Alternativa B — 360dialog como BSP con licencia plana
Modelo de licencia plana (~€49/mes) más transparente que Twilio en volumen alto. Sin markup por mensaje. Descartada por las mismas razones que Twilio (intermediario sin valor agregado para el diseño de FlowRide), agravadas por menor comunidad para .NET, sin SDK oficial y ecosistema orientado a Europa.

### Alternativa C — Bird (ex-MessageBird) como BSP multicanal
Perfil comparable a Twilio en modelo (~$0.005 markup, multicanal). Descartada por ausencia de ventaja diferencial frente a Meta directa o Twilio, menor madurez de documentación para .NET y menor comunidad en español.

### Alternativa D — Gupshup como BSP low-cost
Markup más bajo del mercado (~$0.001 por mensaje). Descartada porque el enfoque geográfico, el dashboard, el soporte y la comunidad están orientados a India y Sudeste Asiático. Fricción operativa alta para un equipo colombiano atendiendo clientes locales.

### Alternativa E — App móvil propia (Android/iOS)
Descartada. Obliga a Pasajero y Conductor a descargar, instalar, aprender y mantener una app adicional. FlowRide se apoya deliberadamente en un canal ya presente en el teléfono de sus usuarios (>90% de penetración de WhatsApp en Colombia). Construir y sostener dos apps móviles nativas multiplica el esfuerzo del equipo y saca al proyecto del alcance del semestre.

### Alternativa F — SMS
Descartada. Formato pobre de interacción, sin soporte para plantillas ricas, botones ni multimedia. Costo por mensaje frecuentemente mayor que WhatsApp. Menor familiaridad de los usuarios con flujos conversacionales estructurados sobre SMS.

### Alternativa G — Telegram
Descartada. Penetración en Colombia muy inferior a WhatsApp en el segmento corporativo. Aunque su API es abierta y gratuita, obliga al usuario a instalar y aprender una segunda app — misma fricción que la app propia.

### Alternativa H — Correo electrónico
Descartada. Formato asincrónico que no encaja con el timing operativo de solicitudes de transporte en madrugada y ventanas cortas. Baja fiabilidad de entrega en tiempo real y tasas de apertura variables según el perfil del usuario.

## Consecuencias positivas

- **Cero markup por mensaje.** El costo operativo es únicamente el rate card de Meta. En un volumen medio proyectado de 50.000 mensajes/mes, se ahorran ~$250 USD/mes frente a Twilio en puro costo de intermediación.
- **Sin lock-in con un BSP.** Meta es la plataforma final; no hay proveedor intermedio del que salir después. Añadir un BSP más adelante (por ejemplo, para acceso a features de campaign builder) sería una decisión aditiva, no un reemplazo forzado.
- **Alineación con la ruta estándar del mercado 2026.** Cloud API es la ruta recomendada por Meta desde 2023 y la única viable para integraciones nuevas desde octubre de 2025.
- **Menos capas técnicas en el camino crítico.** Un salto HTTP hacia Meta en vez de HTTP → Twilio → Meta. Menos puntos de falla, menos latencia añadida por intermediación.
- **Adapter interno preservado como pieza de diseño.** El módulo Integración WhatsApp se mantiene igual: aísla al resto del sistema del cliente HTTP específico. Si en algún momento se decide adoptar un BSP, la migración se limita a reemplazar la implementación del adapter — sin tocar dominio ni lógica de aplicación.

## Consecuencias negativas

- **Curva de aprendizaje más alta para el equipo.** Ningún integrante ha trabajado antes contra Meta Cloud API. No hay SDK oficial .NET publicado por Meta; se depende de la librería comunitaria `WhatsAppBusiness.CloudApi`. Twilio hubiera reducido esa fricción con SDK propio, quickstart oficial y ejemplos abundantes.
- **Menos ejemplos en español y menor soporte comunitario para .NET** que Twilio. La resolución de problemas específicos toma más tiempo de búsqueda y experimentación.
- **Más código propio a construir**: validación de firma de webhook, manejo de plantillas aprobadas, gestión de la ventana de servicio de 24 horas, mapeo de payload al dominio. Todo esto Twilio lo abstraía parcialmente.
- **Dependencia crítica de un único proveedor operativo.** Si Meta interrumpe servicio, cambia política o deprecia features, no hay ruta alterna vía BSP. Se mitiga con el canal secundario de contingencia (SMS o correo) ya declarado en las restricciones de negocio del contexto maestro, y con la política de reintentos + dead-letter queue del adapter.
- **Aprobación de cuenta Meta obligatoria para producción**, con tiempos de 2 horas a 2 semanas. Requiere planificación anticipada y documentación legal de la empresa cliente. Con Twilio el sandbox de desarrollo permitía posponer esa aprobación; con Meta hay que gestionarla temprano en la línea de tiempo del piloto real.
- **Riesgo de que la librería comunitaria quede sin mantenimiento.** Mitigación: el adapter interno permite reemplazarla por un cliente HTTP propio sin afectar al resto del sistema.

## Tácticas de Bass aplicadas

- **Abstraer aspectos específicos de plataforma** (Modificabilidad) — el adapter interno aísla al resto del sistema del cliente HTTP específico de Meta.
- **Reducir acoplamiento con intermediarios externos** (Modificabilidad) — al eliminar el BSP se elimina un punto de acoplamiento con un tercero.
- **Reintento con backoff exponencial** (Disponibilidad) — para absorber interrupciones transitorias del proveedor o de la red.
- **Encolar mensajes fallidos en dead-letter queue** (Disponibilidad / Fiabilidad) — sustento directo del escenario `PAS-DISP-03`.
- **Autenticar actores** (Seguridad) — validación de firma del webhook entrante contra el token compartido con Meta.
- **Adaptar interfaz (tailor interface)** (Modificabilidad) — el adapter traduce el modelo de Meta al lenguaje del dominio de FlowRide.

## Escenarios y requisitos relacionados

Escenarios del banco de 33 seleccionados que esta decisión sustenta:

- `COND-DISP-01` — el conductor recibe el detalle de su servicio por WhatsApp en la franja de arranque de turno (03:00–05:00).
- `PAS-DISP-01` — el pasajero solicita transporte por WhatsApp a cualquier hora, incluida la madrugada.
- `PAS-DISP-03` — los recordatorios automáticos resisten una interrupción de proveedor de hasta 5 minutos.
- `COND-ESC-01` — hasta 200 conductores enviando mensajes en una ventana de 15 minutos.
- `COND-SEG-01` — las acciones críticas del conductor se validan contra su número registrado.
- `COND-ACC-01` — reintentos ante señal de red intermitente del conductor.
- `PAS-UX-01` — el pasajero completa su primera solicitud sin capacitación previa y sin instalar una app adicional.

Requisitos funcionales relacionados: RF-14, RF-15, RF-16, RF-17, RF-18.

> **Nota de trazabilidad:** el catálogo de Requisitos Funcionales RF-01 a RF-56 todavía no existe como documento propio, es un pendiente registrado en la bitácora del proyecto (Punto 3). Los códigos RF citados corresponden a la numeración del `tradeoff_v6.xlsx` y deben cerrarse como documento independiente antes de considerar este ADR completamente trazable.

## ADRs relacionados

- **ADR-0001** (adoptar monolito modular) — el módulo Integración WhatsApp vive dentro del monolito, no como microservicio.
- **ADR-0003** (autenticación propia con JWT HMAC-SHA256) — la identidad de Pasajero y Conductor en WhatsApp se resuelve por número de teléfono asociado al usuario en el modelo de dominio, no por JWT. JWT aplica a los roles administrativos que usan panel web.
- **ADR-0006** (patrón Outbox para consistencia bajo concurrencia) — sostiene la consistencia del estado bajo mensajes concurrentes que llegan por el webhook.

## Notas de implementación

- El módulo Integración WhatsApp expone un endpoint `POST /api/whatsapp/webhook` que valida la firma del payload contra el token compartido con Meta antes de procesar el mensaje.
- Los DTOs de entrada/salida mapean el payload de Meta al lenguaje del dominio (`SolicitudTransporte`, confirmación de `Asignacion`) sin exponer conceptos de la Cloud API al resto del sistema.
- Las credenciales de Meta (Phone Number ID, WhatsApp Business Account ID, System User Access Token, Webhook Verify Token) se manejan igual que el resto de secretos del proyecto: `appsettings.json` no versionado, separado por entorno, según la sección 15 del contexto maestro.
- Durante el semestre se usan los 5 números de prueba gratuitos de Meta. La verificación de cuenta Business para producción se solicita únicamente cuando exista un piloto acordado con una empresa cliente real.

## Referencias

- Meta for Developers — WhatsApp Business Platform (Cloud API). https://developers.facebook.com/docs/whatsapp
- Deprecación On-Premise API — Meta, octubre 2025.
- `WhatsAppBusiness.CloudApi` — NuGet, licencia MIT. https://www.nuget.org/packages/WhatsAppBusiness.CloudApi
- Benchmark de 12 BSPs de WhatsApp Business API, 2026 — Kanal.
- Comparativas de precios BSP 2026 — Telnyx, Ominiflow, Zernio, EZContact, SetSmart.
- `CONTEXTO_FLOWRIDE_v9.md`, sección 0 (Nota de postura sobre re-evaluación), sección 7.3 (Canal WhatsApp), sección 15 (Implementación técnica).
- `tradeoff_v6.xlsx`, hoja `ADR - Decisiones Arq.`, fila ADR-05 (versión previa reemplazada por este documento).
