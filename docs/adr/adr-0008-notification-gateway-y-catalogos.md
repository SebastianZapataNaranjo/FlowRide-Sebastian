# ADR-0008: centralizar los envíos en un notification gateway y externalizar textos, plantillas y parámetros en catálogos por operador

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0008 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando F1–F3 funcionen sin ningún texto de usuario escrito en el código (prueba de arquitectura que prohíbe literales de usuario fuera del catálogo) y un cambio de `asignacion.expiracion_min` por la API de administración afecte a la siguiente asignación sin redesplegar. |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0015 y ADR-0016 del xlsx (fusionados) |
| Estado de implementación | Diseñado |

## Contexto

Tres casos de uso envían mensajes: la confirmación al pasajero (F1 y F3), la propuesta al conductor (F3) y la alerta al operador (F2/F3). Cada envío tiene que decidir entre **mensaje de sesión** (ventana de 24 h abierta) y **plantilla** (ventana cerrada), registrar la clave de idempotencia (ADR-0006) y respetar el canal único. Si cada caso de uso llama directamente al adaptador de WhatsApp, esas reglas se repiten tres veces.

Además, el curso exige un catálogo de mensajes, uno de notificaciones y uno de parámetros (lineamientos del curso Ingeniería del Software 2), y los umbrales del motor (ADR-0007) deben poder cambiarse sin un despliegue.

## Decisión

1. **Notification gateway** — puerto `INotificador` en `Servicios.Output.Ports`, implementado en `Infraestructura.Output.ServiciosExternos.NotificationGateway`:
   - Recibe `Notificacion(destinatarioUsuarioId, tipo, variables, claveIdempotencia)`. El caso de uso expresa la *intención*, no el canal.
   - Resuelve el teléfono, consulta `conversacion.ultimo_entrante_en` del destinatario y decide: si el último mensaje entrante tiene menos de 24 h, usa un **mensaje de sesión** (interactivo si el tipo lleva botones); si no, la **plantilla** asociada al tipo; si la plantilla no está `Aprobada`, marca la notificación como `SinCanal` y genera `AlertaOperador` (o expira la asignación por ADR-0007).
   - Registra `notificacion_saliente` antes de llamar a Meta (ADR-0006).
2. **Catálogo de mensajes** (`MessageCatalog`): archivos de recursos versionados por clave estable (`itinerario.confirmado`, `asignacion.propuesta`, `canal.remitente_desconocido`…), con locale `es-CO`. Los errores técnicos nunca llegan al usuario: recibe una clave del catálogo y el detalle va al log.
3. **Catálogo de notificaciones** (`NotificationCatalog`): tabla `plantilla_whatsapp` por operador (nombre en Meta, idioma, categoría, variables, botones, estado de aprobación). Cada operador tiene su propia WABA y somete sus propias plantillas (ADR-0005). Los textos para someter están en `diseno-detallado.md`.
4. **Catálogo de parámetros** (`ParameterCatalog`): tabla `parametro` por operador, con `clave`, `valor`, `tipo`, `minimo`, `maximo`, `actualizado_en` y `actualizado_por`. Los rangos están en código y la API de administración rechaza valores fuera de rango. Se lee en cada operación, sin caché: unas decenas de lecturas por solicitud sobre una tabla mínima, sin un driver de rendimiento (ADR-0012).

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| Cada caso de uso llama al adaptador | Duplica la regla de la ventana, la idempotencia y la selección de plantilla. |
| Plataforma omnicanal externa (CPaaS) | Otro intermediario con margen por mensaje (mismo argumento que Twilio en ADR-0005). |
| Textos en el código | Incumple el estándar del curso; cambiar un texto exige un despliegue. |
| Parámetros en `appsettings` | Cambiar un umbral exige reiniciar; no es por operador en tiempo de ejecución ni queda auditado. |
| Servicio externo de configuración o *feature flags* | Componente adicional por operador sin necesidad; rompería el Silo si fuera compartido. |

## Consecuencias

**Positivas**: la regla de la ventana vive en un solo lugar; el operador ajusta su operación sin un despliegue; cada cambio de parámetro queda auditado.
**Negativas**: las claves del catálogo y las plantillas aprobadas en cada WABA hay que mantenerlas sincronizadas; un parámetro extremo (por ejemplo, `expiracion_min = 1`) generaría ruido, y por eso existen los rangos.

## Tácticas de Bass

Usar un intermediario (gateway); diferir el binding (catálogos en tiempo de ejecución); localizar cambios (un texto o un umbral en un solo lugar); validar la entrada (rangos de parámetros).

## Escenarios y requisitos relacionados

`PAS-UX-01`, `PAS-DISP-03` (adaptado), `PAS-CONF-03`, `EQ-MANT-01`; RF-16, RF-17, RF-21, RF-45 y RF-57.

## Condiciones de revisión

Si se agrega un canal (por ejemplo, alertas del panel por SSE en V1), se agrega una política en el gateway sin tocar los casos de uso. Si las lecturas del catálogo de parámetros aparecen en el perfil de latencia, se evalúa una caché con invalidación por `actualizado_en` (ADR-0012).

## Referencias

ADR-0015 y ADR-0016 del `ADR_Decisiones_Arquitectonicas_FlowRide_LISTA.xlsx`; lineamientos del curso Ingeniería del Software 2 (catálogos de mensajes, notificaciones y parámetros).
