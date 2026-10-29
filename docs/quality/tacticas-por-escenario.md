# Tácticas de Bass por escenario

Regla de este documento: **ninguna táctica sin escenario y ningún escenario sin táctica**. Cada fila dice qué táctica responde a qué escenario, con qué mecanismo concreto y en qué ADR se decide. Las tácticas siguen el catálogo de Bass, Clements y Kazman, *Software Architecture in Practice*, 4.ª ed.

| Escenario | Atributo | Táctica (Bass) | Mecanismo en FlowRide | ADR |
|---|---|---|---|---|
| `PAS-DISP-01` | Disponibilidad | Recuperarse — reintentar (lo hace el proveedor) | Meta reintenta los webhooks durante hasta 7 días | 0005, 0009 |
| | | Prevenir fallos — eliminar del servicio / separar procesos | API y Worker separados: si el Worker cae, la API sigue recibiendo | 0001, 0003 |
| | | Rendimiento — encolar (inbox) | El webhook persiste y responde; procesa el Worker | 0006 |
| | | Detectar fallos — monitorear / *heartbeat* | `/health/live` y `/health/ready`; métrica `outbox_pendientes` | 0010 |
| | | Recuperarse — reiniciar | Reinicio automático del contenedor (compose `restart`, ACA) | 0009 |
| `COND-DISP-01` | Disponibilidad / Rendimiento | Introducir concurrencia | E/S asíncrona con `async/await` | 0003 |
| | | Reducir la sobrecarga de cómputo | Transacción local, sin llamadas entre servicios | 0001 |
| `PAS-UX-01` | Usabilidad | Mantener el modelo de tareas / dar iniciativa al sistema | Conversación guiada por el sistema con botones y listas | 0005 |
| | | Localizar cambios (en los textos) | Catálogo de mensajes | 0008 |
| `COND-SEG-01` | Seguridad | Autenticar actores | Teléfono → usuario en la base del operador | 0005 |
| | | Autorizar actores | Verificación de pertenencia de la asignación | 0005 |
| | | No repudio / auditar | Auditoría de solo inserción con actor y resultado | 0010 |
| `COND-ACC-01` | Confiabilidad ante conectividad | Detectar fallos — validar condiciones previas | Máquina de estados: solo `Propuesta` vigente acepta transición | 0006, 0007 |
| | | Prevenir fallos — transacciones | `xmin` + una transacción por manejador | 0006 |
| `PAS-DISP-03` (adaptado) | Disponibilidad | Recuperarse — reintentar | Backoff exponencial con tope en la outbox | 0006 |
| | | Prevenir fallos — *circuit breaker* | `Microsoft.Extensions.Http.Resilience` sobre `ClienteGraphApi` | 0003, 0005 |
| | | Recuperarse — degradar / dead letter | Estado `Fallido` + `AlertaOperador` | 0006 |
| | | Detectar fallos — *timeout* | `Incierta` + espera del webhook de estado con `biz_opaque_callback_data` | 0006 |
| `COND-CONF-02` (reescrito) | Confiabilidad | Mantener una sola fuente de verdad (no copias) | Sin caché en la ruta F1–F3; lectura transaccional | 0006, 0012 |
| `CANAL-SEG-01` | Seguridad | Autenticar actores | HMAC-SHA256 sobre el cuerpo crudo | 0003, 0005 |
| | | Detectar intrusiones / validar la entrada | Rechazo de firma o de `phone_number_id` ajeno | 0002, 0005 |
| | | Cifrar los datos en tránsito | HTTPS únicamente | 0003 |
| `CANAL-CONF-01` | Confiabilidad | Detectar fallos — validar (idempotencia) | PK `provider_message_id` + `ON CONFLICT DO NOTHING` | 0006 |
| `OPLOG-CONF-03` | Confiabilidad | Prevenir fallos — restringir el acceso concurrente | `EXCLUDE USING gist` sobre la franja | 0004, 0006 |
| | | Recuperarse — reintentar | Reevaluar excluyendo ante `23P01` | 0006 |
| `OPLOG-CONF-04` | Confiabilidad (regulatoria) | Prevenir fallos — eliminar la excepción por diseño | Eliminatorias legales en código, no configurables | 0007 |
| | | Auditar | `asignacion.razones` | 0007, 0010 |
| `PAS-CONF-03` | Confiabilidad / Disponibilidad | Detectar fallos — *timeout* | Expiración relativa de la propuesta | 0007 |
| | | Recuperarse — reintentar y escalar | Reencolado hasta N + alerta por WhatsApp | 0007, 0008 |
| `PLAT-AIS-01` | Aislamiento | Separar entidades | Stack, base y credencial por operador | 0002 |
| | | Limitar el acceso | El proceso solo conoce su cadena de conexión; permisos de PostgreSQL | 0002 |
| | | Contener el fallo | Una falla queda dentro del stack del operador | 0002 |
| `EQ-MANT-01` | Mantenibilidad | Aumentar la cohesión (dividir en módulos) | Cortes verticales en cada capa | 0001 |
| | | Reducir el acoplamiento — restringir dependencias | Reglas de NetArchTest C1–C8, M1–M4 | 0001 |
| | | Diferir el binding | Reglas de puntaje y parámetros en tiempo de ejecución | 0007, 0008 |
| | | Abstraer servicios comunes | `IReglaEliminatoria`, `IReglaPuntaje` | 0007 |

## Tácticas declaradas en los ADR que se ejercitan sin escenario propio

| Táctica | Dónde | Por qué no tiene escenario medido en F1–F3 |
|---|---|---|
| Redundancia activa | ADR-0009 (nivel contractual Plus) | Opción contractual; no se despliega en la base. Escenario asociado: `ADM-DISP-01` (Diseñado) |
| Mantener copias de los datos (HA) | ADR-0009 (nivel Crítico) | Igual que la anterior |
| Incrementar recursos | ADR-0009 | Se activa por medición en producción, no en la línea base |

Las tres están atadas a un escenario Diseñado, así que no quedan huérfanas: quedan diferidas con una condición de activación.
