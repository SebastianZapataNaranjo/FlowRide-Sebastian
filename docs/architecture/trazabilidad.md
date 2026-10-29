# Trazabilidad F1–F3

Cadena: **requisito funcional → escenario de calidad → táctica → ADR → componente → prueba**. Si un eslabón falta, el elemento no está sustentado.

## 1. Requisitos funcionales de F1–F3 (versión vigente)

Base: catálogo RF-01..RF-56 de `docs/_fuentes/DriversArquitectonicosFlowRide_ACTUALIZADO.md`. Lo marcado **corregido** o **nuevo** se explica en `registro-de-cambios-vs-fuentes.md`.

| RF | Nombre | Actor | Estado del texto | Resumen vigente |
|---|---|---|---|---|
| RF-08 | Registro de itinerario por WhatsApp | **Pasajero** (antes: Operador logístico desde el panel) | **Corregido** | El pasajero registra código de servicio o vuelo, fecha, hora de presentación en destino, origen y destino (lugares frecuentes). Cada itinerario genera una solicitud de un pasajero. La variante del coordinador con varios pasajeros queda Diseñada (V1) |
| RF-09 | Recepción de solicitudes de transporte | Sistema | Ajustado | La solicitud se crea a partir del itinerario confirmado, con la recogida calculada y la presentación comprometida |
| RF-14 | Recepción de mensajes desde WhatsApp | Meta | Ajustado | Webhook HTTPS con firma verificada, inbox idempotente y validación de `phone_number_id` |
| RF-15 | Procesamiento conversacional | Sistema | Ajustado | Conversación guiada con botones y listas; sin interpretación de texto libre salvo código y hora |
| RF-16 | Confirmación al pasajero | Sistema | Vigente | Al registrar (F1) y al confirmar la asignación (F3) |
| RF-10 | Asignación automática | Motor | Vigente | Por evento, sobre la solicitud `Pendiente` |
| RF-11 | Reglas de asignación | Motor | **Ajustado** | Tres niveles: eliminatorias legales fijas, filtros operativos y puntaje configurable |
| RF-17 | Notificación al conductor | Sistema | Vigente | Interactivo con [Aceptar] [Rechazar] (sesión) o plantilla `asignacion_servicio` (Diseñado) |
| RF-21 | Configuración de reglas | Administrador técnico (línea base) / Administrador de empresa (V1) | Ajustado | Por `PUT /admin/reglas/{codigo}` y `PUT /admin/parametros/{clave}` |
| RF-45 | Alerta operativa | Sistema → Operador logístico | **Ajustado** | Por WhatsApp al número del operador con `recibe_alertas`, además de registrarse; `GET /admin/alertas` solo consulta |
| RF-57 | Confirmación de asignación por el conductor | Conductor | **Nuevo** | Acepta o rechaza con botones; validación de identidad; expiración relativa; reencolado hasta N y escalamiento |
| RF-58 | Check-in de turno del conductor | Conductor | **Nuevo** | "INICIO TURNO" abre la ventana de 24 h y confirma que se presentó. Detecta la **no presentación**, no el abandono a mitad del turno |
| RF-19 | Aislamiento físico entre tenants | Sistema | Ajustado | El tenant lo fija el despliegue (stack por operador), no un claim |
| RF-01 | Autenticación basada en token | — | **Ajustado** | Aplica al panel (Diseñado, ADR-0011). Pasajero y conductor **no** usan token: se identifican por su teléfono (ADR-0005) |

## 2. Matriz de trazabilidad

| RF | Escenarios | Tácticas principales | ADR | Componentes | Pruebas |
|---|---|---|---|---|---|
| RF-14 | `CANAL-SEG-01`, `CANAL-CONF-01`, `PLAT-AIS-01` | Autenticar actores, validar la entrada, idempotencia | 0002, 0003, 0005, 0006 | `FiltroFirmaWebhook`, `WhatsAppWebhookController`, `EscritorInbox` | Contrato del webhook (6 casos de firma, 3 reenvíos); script de aislamiento |
| RF-08, RF-15 | `PAS-UX-01`, `PAS-DISP-01` | Iniciativa del sistema, encolar, reintento del proveedor | 0005, 0006, 0008 | `EnrutarMensajeEntrante`, `AvanzarConversacion`, `MessageCatalog`, `ParameterCatalog` | E2E de 7 interacciones; prueba con 5 personas; k6 + caída del Worker |
| RF-09 | `PAS-DISP-01`, `CANAL-CONF-01` | Transacciones, outbox | 0004, 0006 | `RegistrarItinerario`, repositorio de solicitudes, `EscritorOutbox` | Integración: `UNIQUE` del itinerario; inbox → solicitud |
| RF-16 | `PAS-DISP-01` | Intermediario (gateway), reintentar | 0008, 0006 | `Notificar`, `NotificationGateway`, `ClienteGraphApi` | k6 (tiempo hasta la confirmación) |
| RF-10, RF-11 | `OPLOG-CONF-03`, `OPLOG-CONF-04`, `COND-CONF-02`, `EQ-MANT-01` | Restringir la concurrencia, eliminar la excepción, diferir el binding | 0001, 0004, 0006, 0007 | `AsignarConductor`, `MotorAsignacion`, `Reglas.*`, restricción de exclusión | Concurrencia ×1.000; matriz de vencimientos; NetArchTest |
| RF-21 | `EQ-MANT-01` | Diferir el binding, validar la entrada | 0007, 0008 | `Admin*Controller`, `ParameterCatalog`, `regla_asignacion` | Cambio de parámetro sin redespliegue; rechazo fuera de rango |
| RF-17 | `PAS-DISP-03`, `COND-DISP-01` | Reintentar, *circuit breaker*, *timeout* | 0005, 0006, 0008 | `ProcesadorOutbox`, `NotificationGateway` | Falla inyectada en el simulador |
| RF-57 | `COND-SEG-01`, `COND-ACC-01`, `PAS-CONF-03` | Autorizar, validar condiciones previas, *timeout*, escalar | 0005, 0006, 0007 | `ResponderAsignacion`, `GestionarFallaAsignacion`, `BarridoExpiraciones` | 4 casos de identidad; duplicado y tardío; reloj controlado |
| RF-58 | `COND-DISP-01`, `COND-CONF-02` | Una sola fuente de verdad | 0005, 0007 | `RegistrarCheckin`, `turno_conductor` | Concurrencia check-in/asignación ×200 |
| RF-45 | `PAS-CONF-03` | Escalar | 0007, 0008 | `GestionarFallaAsignacion`, `NotificationGateway` | Alerta en el tiempo calculado |
| RF-19 | `PLAT-AIS-01` | Separar entidades, limitar el acceso | 0002, 0009 | Configuración por stack; usuarios de PostgreSQL por base | Script de aislamiento |
| Transversal | `EQ-MANT-01`, `OPLOG-AUD-01` (Diseñado: consulta) | Restringir dependencias, auditar | 0001, 0010 | NetArchTest, `Auditoria`, `CorrelationMiddleware` | NetArchTest en CI; una solicitud reconstruida por `correlation_id` |

## 3. Drivers → decisiones

| Driver | Tipo | ADR que lo atiende |
|---|---|---|
| Disponibilidad (prioridad 11) | Atributo de calidad | 0002, 0006, 0009 |
| Seguridad (10) | Atributo de calidad | 0003, 0005, 0010 |
| Rendimiento (9) | Atributo de calidad | 0001, 0003 |
| Confiabilidad (8) | Atributo de calidad | 0004, 0006, 0007 |
| Aislamiento (7) | Atributo de calidad | 0002 |
| Interoperabilidad (6) | Atributo de calidad | 0005, 0008 |
| Capacidad para ser auditado (4) | Atributo de calidad | 0010 |
| Usabilidad (3) | Atributo de calidad | 0005, 0008 |
| Mantenibilidad (1) | Atributo de calidad | 0001, 0007 |
| C# / .NET 10 | Restricción técnica | 0003 |
| PostgreSQL como motor principal | Restricción técnica | 0004 |
| Silo físico por operador | Restricción de negocio | 0002 |
| 20–40 USD/mes por operador | Restricción de negocio | 0001, 0002, 0009 |
| WhatsApp como canal único | Restricción de producto | 0005, 0008 |
| Ley 769 de 2002 (SOAT, RTM, licencia) | Restricción legal | 0007 |
| Ley 1581 de 2012 (datos personales) | Restricción legal | 0005, 0010 (minimización, auditoría) |
