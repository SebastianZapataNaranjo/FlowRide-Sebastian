# ADR-0005: integrar WhatsApp directamente con Meta Cloud API e identificar a pasajero y conductor por su número

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0005 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando la PoC del webhook (PoC 4 de las fuentes) pase en el tenant A con el número de prueba de Meta: firma verificada, duplicado ignorado y respuesta interactiva recibida en el teléfono (`CANAL-SEG-01`, `CANAL-CONF-01`, `COND-SEG-01`). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-05 del tradeoff (Twilio, "Aprobado" en DOO); ADR-0008 del xlsx ("Sandbox para el alcance académico"); `adr-previos/adr-0005-…meta-cloud-api` (corrige cinco errores, ver final); la parte de identidad del canal del ADR-0003 del xlsx |
| Estado de implementación | Diseñado |

## Contexto

WhatsApp es el único canal de pasajero y conductor (diferenciador 6). Esta decisión condiciona la disponibilidad, la identidad, la experiencia en la madrugada y el costo variable. Lo verificado (consulta 2026-10-28):

| Hecho | Fuente |
|---|---|
| Precio por mensaje entregado; los mensajes que el usuario envía a la empresa no se cobran | [Meta Pricing](https://developers.facebook.com/docs/whatsapp/pricing) |
| Desde el **1-oct-2026** Meta cobra los mensajes de servicio por encima de **1.000 gratis al mes por número**, y también los utility dentro de la ventana | [Updates to pricing](https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/) |
| Colombia tiene tarifa propia y facturación en COP desde el 1-abr-2026. La cifra oficial está en un CSV que no se pudo leer; el rango de terceros para utility es USD 0,0008–0,0014 | Mismas fuentes; cifra en `preguntas-abiertas.md` |
| Twilio cobra **USD 0,005 por mensaje, entrante y saliente**, además de Meta | [Twilio WhatsApp pricing](https://www.twilio.com/en-us/whatsapp/pricing) |
| El sandbox de Twilio usa un número **compartido**, exige `join <código>`, la sesión expira a los **3 días** y envía 1 mensaje cada 3 s | [Twilio Sandbox](https://www.twilio.com/docs/whatsapp/sandbox) |
| Meta **no ofrece SLA comercial** de uptime ni de latencia; reporta 99,9 % observado | [Support FAQ](https://developers.facebook.com/docs/whatsapp/cloud-api/support/), [On-Prem sunset](https://developers.facebook.com/docs/whatsapp/on-premises/sunset) |
| Reintentos del webhook durante hasta 7 días, con posibles duplicados | [Set up webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks) |
| Override de la URL del webhook **por número** | [Webhook overrides](https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override) |
| Botones de respuesta: máximo 3, título ≤ 20 caracteres, `id` ≤ 256 que vuelve en el webhook. Listas: hasta 10 filas | [Reply buttons](https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/interactive-reply-buttons-messages), [List messages](https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/interactive-list-messages) |
| Mensajes libres e interactivos solo dentro de la ventana de 24 h abierta por el usuario; fuera de ella, plantillas aprobadas (revisión de hasta 24 h) | [Template review](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/template-review/) |
| El número de prueba admite **hasta 5 destinatarios** registrados (según fuentes de terceros; no está en la documentación oficial vigente) | [dev.to](https://dev.to/dainik_times_363e0b9e9b18/whatsapp-cloud-api-the-limits-and-error-codes-nobody-documents-5g4g), [oncloudapi](https://oncloudapi.com/blogs/whatsapp-cloud-api-test-number-vs-production-number-how-to-go-live) |
| Servir a operadores con su propio número exige ser **Tech Provider**: verificación del negocio, App Review con acceso avanzado y **Embedded Signup v4** (v2 y v3 deprecados el 15-oct-2026) | [Become a Tech Provider](https://developers.facebook.com/documentation/business-messaging/whatsapp/solution-providers/get-started-for-tech-providers) |
| Límite de mensajería: 250 usuarios únicos en 24 h fuera de la ventana, **por portafolio** del operador | [Messaging limits](https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits) |

## Decisión

### 1. Proveedor: Meta Cloud API directo, sin BSP

- Adaptador propio en `Infraestructura.Output.ServiciosExternos.NotificationGateway.WhatsApp` con un **`HttpClient` tipado** sobre la Graph API, que implementa el puerto `IMensajeriaWhatsApp` de `Servicios.Output.Ports`. Solo hacen falta dos operaciones: enviar un mensaje (texto o interactivo) y enviar una plantilla.
- No se usa la librería `WhatsAppBusiness.CloudApi`. Está mantenida (v1.0.104 del 20-oct-2026, compatible con net10.0, MIT; [NuGet](https://www.nuget.org/packages/WhatsAppBusiness.CloudApi)), pero el cliente propio permite **cambiar la URL base hacia el simulador** sin código condicional y controlar reintentos e idempotencia (ADR-0006).
- El webhook entra por `Infraestructura.Input.Webhooks.WhatsAppWebhookController`: verifica la firma sobre el **cuerpo crudo** antes de deserializar, comprueba que `metadata.phone_number_id` coincida con el número configurado (ADR-0002), inserta en el inbox y responde 200.

### 2. Identidad en el canal: teléfono → usuario, sin token

En WhatsApp no existe sesión ni JWT. La identidad se valida en tres capas:

1. **Autenticidad del mensaje**: la firma HMAC prueba que lo envió Meta (o el simulador del stack B).
2. **Identificación del actor**: el `wa_id` (teléfono E.164) se busca en `usuario.telefono_e164` **dentro de la base del operador**. Si no existe o está inactivo, el remitente recibe una respuesta genérica del catálogo (`canal.remitente_desconocido`), no se crea nada y se audita el intento.
3. **Autorización por recurso**: cada acción verifica la pertenencia. Al aceptar, el `wa_id` debe ser el del conductor de esa asignación; el `id` del botón no basta (`COND-SEG-01`).

El repositorio `IUsuarioPorTelefonoRepositorio` vive en `Servicios.Output.Ports` y se implementa en `Infraestructura.Output.Repositorios.Estructurados`. **No se crea el paquete `ServiciosExternos.IdentityProvider`**: no hay proveedor externo de identidad en la ruta E2E. Aparece en los diagramas como Diseñado (OIDC, ADR-0011).

Reconocer al usuario por su número cumple `COND-PORT-01` (cambiar de celular sin perder el acceso). El riesgo inverso (una línea reasignada por el operador telefónico a otra persona) se mitiga desactivando al usuario desde la administración, y queda declarado.

### 3. Conversación y ventana de 24 h

- F1 usa **botones y listas**, nunca texto libre interpretado: es determinista, auditable y no requiere NLP. El único texto libre es la hora, validada con un formato estricto (`HH:mm`).
- Para que la demo de F3 no dependa de una plantilla aprobada, **el conductor abre la ventana** con el check-in de turno (RF-58, "INICIO TURNO"). Lo mismo para el número del operador que recibe alertas.
- Si la ventana del destinatario está cerrada, la ruta es la plantilla `asignacion_servicio`, `alerta_operativa` o `confirmacion_pasajero` (textos en `diseno-detallado.md`). Queda **Diseñada**; pasa a E2E si Meta las aprueba antes del 4-nov.

### 4. Doble de prueba `whatsapp-sim` (solo en compose)

- Un contenedor que imita la Graph API (`POST /{phone_number_id}/messages`, responde con un `wamid` y genera webhooks de estado) y envía webhooks de mensajes firmados al stack B. **El adaptador de Meta es el mismo código para los dos tenants; solo cambia la URL base.**
- **Nunca se despliega en Azure.** Está marcado como doble de prueba en el C4 y en el diagrama de despliegue.
- **Secretos de firma**: el stack B tiene un secreto propio con el simulador, mientras que en producción **todos los números de la app de Meta comparten un único app secret**. Ese contraste se declara como riesgo: comprometer el app secret permite forjar webhooks hacia todos los operadores. Se mitiga con Key Vault de plataforma, rotación con período de gracia (Meta admite el secreto anterior durante X horas al rotar) y validación del `phone_number_id` en cada stack (ADR-0014).

### 5. Costo del canal (argumento secundario)

Supuestos del caso ancla: 40 servicios/día × 10 salientes por servicio (F1 = 7 según `diseno-detallado.md` §1.1, F3 = 3) × 30 días = 12.000 salientes al mes, menos 1.000 gratis. Entrantes: ~8 por servicio (7 del pasajero + 1 del conductor) = 9.600 al mes.

| Concepto | USD/mes por operador |
|---|---|
| Meta, 11.000 × 0,0008–0,0014 (rango de terceros) | **9–15** |
| Ahorro si F1 baja de 7 a 5 salientes (por ejemplo, con WhatsApp Flows) | ~2–3 |
| Twilio adicional: (12.000 salientes + 9.600 entrantes) × 0,005 | **~108** |

La cifra de Meta es baja, así que **el argumento para tener pocos pasos es la experiencia de uso en la madrugada** (`PAS-UX-01`), no el costo. En cambio, el sobrecosto de Twilio por sí solo **duplica el techo de infraestructura** de 20–40 USD.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| **Twilio (BSP)** | ~108 USD/mes extra por operador; sandbox de número compartido que no permite demostrar el enrutamiento por número (ADR-0002); intermediario sin SLA propio encima de Meta. |
| 360dialog, Bird, Gupshup | Mismo papel de intermediario sin funciones que FlowRide aproveche (se conserva el análisis del ADR-0005 previo). |
| Twilio Sandbox como "entorno académico" (ADR-0008 del xlsx) | Justificación académica descartada por regla del proyecto; además, sus límites (join cada 3 días, número compartido) chocan con la demo multi-tenant. |
| Interpretar texto libre (NLP o IA) | No determinista; la IA está excluida del producto (contexto v10 §6.1). |
| WhatsApp Flows (formulario en un solo mensaje) | Reduce pasos; queda **Diseñado** como evolución de F1, porque agrega un artefacto que Meta también tiene que publicar y aprobar. |
| App propia, SMS, Telegram, correo | Excluidos por el canal único (contexto v10 §6.1). |

## Consecuencias

**Positivas**
- Un salto HTTP menos que con un BSP; el piso de costo es solo el de Meta.
- La demo multi-tenant prueba de verdad el adaptador y la firma en los dos stacks.
- La identidad del canal no depende de un proveedor externo y se prueba con datos.

**Negativas**
- Dependencia de un proveedor sin SLA comercial; el impacto en la disponibilidad compuesta se calcula en ADR-0009.
- Para producción, FlowRide tiene que convertirse en Tech Provider (verificación del negocio y App Review) y cada operador tiene que verificar su negocio en Meta.
- El app secret compartido es un punto de compromiso común.
- El límite de 5 destinatarios del número de prueba obliga a usar 2 teléfonos reales (pasajero y conductor) en la demo del tenant A.
- Si la ventana del conductor está cerrada y no hay plantilla aprobada, la asignación expira y escala al operador (ADR-0007). Es un camino degradado, no una pérdida.

## Correcciones al ADR-0005 previo (`adr-previos/adr-0005-…`)

1. Decía "5 números de prueba gratuitos". Lo correcto es **un número de prueba que puede escribir a un máximo de 5 destinatarios** (fuentes de terceros).
2. Mencionaba un "canal secundario de contingencia (SMS o correo) ya declarado". Contradice la exclusión de contexto v10 §6.1 y se elimina; la contingencia es el reintento de la outbox más la alerta al operador.
3. Usaba "saca al proyecto del alcance del semestre" en la Alternativa E. Se reemplaza por el argumento de canal único y fricción de instalación.
4. Citaba "ADR-0003: JWT HMAC propio". Ese ADR fue reabierto: la identidad del canal se resuelve aquí y la del panel en ADR-0011.
5. Afirmaba que Twilio representa "20–40 % del costo" sin cálculo. Se reemplaza por el cálculo de §5.

## Tácticas de Bass

| Táctica | Dónde |
|---|---|
| Autenticar actores | Firma HMAC y búsqueda teléfono → usuario |
| Autorizar actores | Verificación de pertenencia por recurso |
| Adaptar la interfaz / intermediario | Puerto `IMensajeriaWhatsApp`; el modelo de Meta no sale de `Infraestructura` |
| Reintentar, *circuit breaker* | Resiliencia del `HttpClient` y outbox (ADR-0006) |
| Detectar intrusiones | Rechazo de firma inválida y de `phone_number_id` ajeno, con auditoría |

## Escenarios y requisitos relacionados

`CANAL-SEG-01`, `CANAL-CONF-01`, `COND-SEG-01`, `PAS-UX-01`, `PAS-DISP-01`, `COND-DISP-01`, `PAS-DISP-03` (adaptado), `COND-ACC-01`, `PLAT-AIS-01`. RF-08 (corregido), RF-09, RF-14, RF-15, RF-16, RF-17, RF-45, RF-57 y RF-58.

## Condiciones de revisión

- Si Meta publica un SLA comercial, recalcular ADR-0009.
- Si un BSP ofrece una capacidad que FlowRide necesite y no pueda construir (por ejemplo, failover multi-número), reevaluar.
- Si las plantillas se aprueban antes del 4-nov, la ruta fuera de ventana pasa de Diseñado a E2E y se actualiza este ADR.

## Referencias

Todas las URL de la tabla de contexto, consultadas el 2026-10-28.
