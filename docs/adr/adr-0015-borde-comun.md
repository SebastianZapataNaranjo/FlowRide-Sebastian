# ADR-0015: sin borde común (gateway, WAF, router) en la ruta de mensajes; se reevalúa con el panel (Diseñado)

| Campo | Valor |
|---|---|
| Estado | **Proposed** · Diseñado |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0012 del xlsx (borde común con YARP, WAF y router de webhooks, Accepted) |

**Contexto.** La fuente anterior justificaba un borde compartido porque Meta tenía "una URL de callback por app" y había que enrutar por `phone_number_id` hacia cada stack. Meta permite hoy **sobrescribir el webhook por número** ([Webhook overrides](https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override), consulta 2026-10-28). Cada número entrega directo a su stack. Un borde compartido pasaría a ser un **punto de falla común** a todos los operadores, justo en la ruta del escenario de disponibilidad.

**Decisión.** No hay borde común en F1–F3. Cada Container App expone su propio ingress HTTPS (ADR-0009). Lo único que sigue llegando a la URL por defecto de la app (estado de plantillas y de cuenta) lo recibe el control plane, fuera de la ruta de mensajes (ADR-0002).

**Cuándo se reevalúa.** Con el panel web (V1): WAF, límites de tasa por IP y dominio por operador (`{operador}.flowride…`) pueden justificar Azure Front Door o un gateway. En ese caso se aplica solo al tráfico del panel, sin interponerse en los webhooks.

**Escenarios.** `PAS-DISP-01` (evita un punto común), `CANAL-SEG-01` (la firma se verifica en cada stack).
