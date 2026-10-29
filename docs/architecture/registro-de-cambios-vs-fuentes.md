# Registro de cambios frente a las fuentes

Las fuentes de `docs/_fuentes/` son trabajo previo a mejorar: ninguna decisión hereda validez automática (contexto v10 §0.1). Cada cambio dice **qué decía la fuente**, **qué dice ahora** y **por qué** (evidencia o criterio). Fecha de todos los cambios: 2026-10-28. Las URL se consultaron en esa fecha.

## 1. Equivalencia de numeraciones de ADR

| Nuevo (`docs/adr`) | tradeoff (ADR-01..10) | xlsx `ADR_…_LISTA` (0001..0017) | Serie de 11 ADR (`adr-previos/serie-11-adr`) | `docs/adr` anterior (`adr-previos/docs-adr-sep2026`) |
|---|---|---|---|---|
| 0001 Monolito modular | ADR-01 | 0001 | 0001 | — |
| 0002 Silo por stack | ADR-02 | 0002 | 0002 | 0010 (en parte) |
| 0003 Stack, ejecución y protocolos | — | 0017 (lenguaje) | — | — |
| 0004 PostgreSQL + EF Core | ADR-04 | 0004 | 0008 | — |
| 0005 WhatsApp Meta + identidad del canal | ADR-05 | 0008 + parte de 0003 | 0005 (y `adr-previos/adr-0005-…_2026-09-02`) | — |
| 0006 Consistencia | ADR-06 | 0007 | 0006 | — |
| 0007 Motor de asignación | — | 0009 | 0004 | — |
| 0008 Notification gateway y catálogos | — | 0015 + 0016 | — | — |
| 0009 Despliegue Azure, disponibilidad y costo | ADR-08 | 0010 | 0010 | 0007 + 0010 |
| 0010 Observabilidad y auditoría | ADR-10 | 0011 | 0011 | — |
| 0011 Identidad del panel (OIDC) — Diseñado | ADR-03 | 0003 | 0003 | — |
| 0012 Caché — Diseñado | ADR-07 | 0006 | 0007 | — |
| 0013 Telemetría GPS — Diseñado | ADR-09 | 0005 | 0009 | — |
| 0014 Secretos — Diseñado | — | 0014 | — | — |
| 0015 Borde común — Diseñado | — | 0012 | — | — |
| 0016 Almacenamiento de objetos — Diseñado | — | 0013 | — | — |

Al inicio de esta versión (oct 2026) se trabajaba con la numeración de la serie de 11 ADR (0005 WhatsApp, 0006 outbox, 0007 caché); se reemplaza por esta tabla.

## 2. Cambios de alcance y requisitos

| # | Fuente | Decía | Dice ahora | Por qué |
|---|---|---|---|---|
| 1 | Drivers ACTUALIZADO, RF-08 | "Registro de itinerarios" por el Operador logístico desde el panel | Registro de itinerario **por WhatsApp**, con el **pasajero** como actor; F1 = RF-08 + RF-09 + RF-14 + RF-15 + RF-16 | Es lo que se presentó al profesor como F1; el actor real del caso ancla es el tripulante. La variante del coordinador queda Diseñada |
| 2 | Drivers, RF-17 | "El conductor puede aceptar o rechazar" sin definir qué pasa después | **RF-57 nuevo**: confirmación del conductor con expiración relativa, reencolado hasta N y escalamiento | Faltaba el comportamiento ante rechazo o silencio, justo el incidente del caso ancla |
| 3 | — | No existía | **RF-58 nuevo**: check-in de turno | Abre la ventana de 24 h (la demo no depende de plantillas) y detecta la no presentación. **No** detecta el abandono a mitad del turno |
| 4 | Drivers, RF-45 / contexto v10 §15 | Alertas operativas al panel; C4 con SMTP | Alerta por WhatsApp al número del operador, además de registrarse | El panel es V1; contexto v10 §15 ya incluía "alertas operativas" en WhatsApp; el SMTP contradecía la exclusión de §6.1 |
| 5 | Drivers, RF-01 | Todos los usuarios, incluidos pasajero y conductor, se autentican con correo, contraseña y token | El token aplica al panel (Diseñado). Pasajero y conductor se identifican por su teléfono | En el canal WhatsApp no hay sesión ni token |
| 6 | Drivers, RF-11 | Reglas configurables, incluidas las "restricciones regulatorias" | Tres niveles; las eliminatorias legales (licencia C, SOAT, RTM) **no** son configurables | Una obligación legal no puede desactivarse por configuración |
| 7 | — | No definido | La hora que da el pasajero es la de **presentación en destino**; la recogida se calcula con una matriz estática | Ataca el dolor de los horarios estrictos. Riesgo declarado: la matriz no considera el tráfico; la promesa queda registrada pero no es verificable sin RF-18 |
| 8 | Planteamiento inicial de esta versión (oct 2026) | Una API que atiende dos bases de tenant; F1 = RF-08 | Un stack (API + Worker + BD) por operador; compose con `api-a`, `worker-a`, `api-b`, `worker-b`, `whatsapp-sim`, `postgres`; F1 mapeada como en el punto 1 | Decisión P3 de la ronda 1 (ADR-0002) |

## 3. Cambios de arquitectura y decisiones

| # | Fuente | Decía | Dice ahora | Por qué |
|---|---|---|---|---|
| 9 | ADR-0002 del xlsx | El tenant se resuelve con un claim `tenantid` en el JWT | El tenant lo fija el despliegue; no hay código de resolución de tenant | Un claim es una resolución propia de Pool; en Silo por stack sobra y abre una superficie (ADR-0002) |
| 10 | Contexto v10 §8.3, Restricciones técnicas, banco (`PAS-DISP-01`, `COND-DISP-01`) | Canal WhatsApp ≥ 99,9 % | Stack ≥ 99,5 % + **0 mensajes perdidos**; disponibilidad compuesta calculada; HA como opción contractual | Meta no ofrece SLA ([Support FAQ](https://developers.facebook.com/docs/whatsapp/cloud-api/support/)); ACA 99,95 % × PostgreSQL 99,9 % ya deja un techo de 99,85 % (ADR-0009) |
| 11 | ADR-0010 del xlsx | Ya había bajado el canal a 99,5 % (revisión del 22-09) sin el cálculo | Igual, con el cálculo y la medida "0 perdidos" | Coherencia entre fuentes |
| 12 | ADR-05 del tradeoff / ADR-0008 del xlsx | Twilio (DOO) / "Sandbox para el alcance académico" | Meta Cloud API directo; Twilio descartado: ~108 USD/mes extra por operador y sandbox de número compartido | [Twilio pricing](https://www.twilio.com/en-us/whatsapp/pricing), [Twilio Sandbox](https://www.twilio.com/docs/whatsapp/sandbox); regla de no usar justificaciones académicas |
| 13 | `adr-previos/adr-0005-…` (Meta) | (1) "5 números de prueba"; (2) canal de contingencia SMS/correo; (3) "alcance del semestre"; (4) ADR-0003 = JWT HMAC; (5) Twilio "20–40 % del costo" sin cálculo | (1) Un número con hasta 5 destinatarios; (2) eliminado; (3) argumento de canal único; (4) identidad en ADR-0005/0011; (5) cálculo en ADR-0005 §5 | Ver ADR-0005, sección de correcciones |
| 14 | Contexto v10 §8.4 | Los mensajes dentro de la ventana de 24 h son gratis | Desde el 1-oct-2026 se cobran los mensajes de servicio por encima de 1.000/mes por número, y los utility en la ventana; costo estimado de 9–15 USD/mes por operador | [Updates to pricing](https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing/). Argumento secundario frente a la experiencia de usuario |
| 15 | ADR-0012 del xlsx | Borde común con router de webhooks (un callback por app de Meta) | El override de webhook **por número** entrega directo a cada stack; el borde queda Diseñado (ADR-0015) | [Webhook overrides](https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override). Se elimina un punto de falla común |
| 16 | ADR-0006 del xlsx, banco `COND-CONF-02` | Caché local; `COND-CONF-02` prescribía una caché de disponibilidad sincronizada por outbox | Sin caché en la ruta F1–F3; `COND-CONF-02` reescrito con lectura transaccional | La disponibilidad del conductor es justo lo que se decide; no hay driver de rendimiento con ~40 servicios al día |
| 17 | `docs-adr-sep2026/adr-0007` | `minReplicas=2` para el canal; *leader election* con `pg_advisory_lock` | 1 réplica (2 como opción contractual); outbox con `FOR UPDATE SKIP LOCKED` | 2 réplicas fijas cuestan ~40–46 USD/mes (rompe el techo); SKIP LOCKED reparte el trabajo sin elegir líder |
| 18 | ADR-0010 del xlsx | Contenedor con API + hosted services en un proceso | Dos procesos (API y Worker); en Azure, dos contenedores en una misma Container App | Si cae el Worker, la API sigue recibiendo webhooks, sin pagar una segunda app |
| 19 | Banco `ADM-DISP-01` | Balanceador + varias instancias + failover ≤ 30 s como base | Opción contractual de nivel Crítico (~+260 USD/mes) | El nivel Burstable no soporta HA ([HA concepts](https://learn.microsoft.com/en-us/azure/postgresql/high-availability/concepts-high-availability)); no cabe en el techo |
| 20 | Banco `COND-ESC-01`, ADR-0005 previo | "200 conductores" | 50 conductores (techo dimensionado de contexto v10 §8.1) | Inconsistencia con el dimensionamiento |
| 21 | C4 de las fuentes (drawio) | Proveedor externo de rutas/ETA; alertas por SMTP; Keycloak con un realm por operador | Matriz de duración estática (ADR-0007); alerta por WhatsApp; identidad del panel Diseñada (ADR-0011) | Ninguna de esas piezas tenía un ADR que la respaldara, y el SMTP contradecía la exclusión |
| 22 | Contexto v10 §22 | "HTTPS local desactivado en desarrollo" | HTTPS con dev-cert en compose; HTTP no publicado | La entrega es E2E con HTTPS (ADR-0003) |
| 23 | Contexto v10 §19 | Repositorio FlowRide-Backend (DOO) con Pool y JWT propio como estado actual | Referencia histórica **descartada**; la línea base se construye desde cero sobre estos documentos | Su modelo (Pool, `empresaId` como frontera, JWT HMAC) contradice las decisiones vigentes |
| 24 | ADR-0016 del xlsx / fuentes de ADR | Parámetros y textos en catálogos (se conserva) | Se fusiona con ADR-0015 del xlsx en ADR-0008 | Ambos tratan la misma frontera: qué se dice y cómo se envía |
| 25 | ADR-0003 de las fuentes (Keycloak, OIDC) | Identidad desacoplada para todos los actores | Identidad del panel Diseñada (ADR-0011); API key por stack para el Administrador técnico, con su debilidad declarada | Ningún actor de F1–F3 usa navegador; la API key no identifica personas |
| 26 | Varias fuentes (ADR-0001, 0002, 0006, 0007, 0009–0017 del xlsx) | Estado `Accepted` sin PoC ni prueba | Todos `Proposed`, con criterio de aceptación verificable | Regla del proyecto: `Accepted` solo con evidencia |

## 4. Cambios en el marco legal y en las cifras

| # | Fuente | Decía | Dice ahora | Por qué |
|---|---|---|---|---|
| 27 | Contexto v10 §8.5 | "Datos sensibles tratados: nombre, teléfono, ubicación" | El nombre y el teléfono son **datos personales, no sensibles** (Ley 1581, art. 5: los sensibles afectan la intimidad o pueden discriminar). La ubicación de un viaje *personal* sí roza la intimidad (`esPersonal`) | Texto de la [Ley 1581](https://normativa.colpensiones.gov.co/compilacion/docs/ley_1581_2012.htm); clasificación inferida del texto, sin concepto de la SIC |
| 28 | Contexto v10 §8.5 | Multa de 2.000 SMMLV ≈ 2.800 millones COP | ≈ 3.500 millones COP con el SMMLV 2026 de 1.750.905 COP (Decreto 1469 de 2025), **con la advertencia** de una posible suspensión provisional no confirmada | [siemprealdia](https://siemprealdia.co/colombia/derecho-laboral/salario-minimo-2026/), [alegra](https://blog.alegra.com/colombia/salario-minimo-en-colombia-2026/); la cifra anterior usaba el SMMLV de 2025 |
| 29 | Contexto v10 §8.5 | "Bloquear la asignación de un conductor con licencia vencida" (sin cita) | SOAT: Ley 769, art. 42; RTM: Ley 769, art. 51; licencia de categoría C (boletín del Consejo de Estado); la vigencia exacta del art. 22 queda pendiente | ADR-0007, referencias |
| 30 | QA-06 de Drivers | Webhook "≤ 500 ms" sin motivador | Se conserva como cifra **interna**, con el motivador explícito (menos reintentos de Meta) | Regla: ninguna cifra sin motivador |
| 31 | Lluvia de escenarios | Dos versiones (tradeoff con votos y CORREGIDA sin votos) | Se usa la hoja "Escenarios Sei seleccionados(33)" del `tradeoff_FlowRide_ACTUALIZADO.xlsx` como banco | Es la versión que coincide con los CSV y la que trae las seis partes |
