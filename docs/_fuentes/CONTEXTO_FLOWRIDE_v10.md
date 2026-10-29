# CONTEXTO MAESTRO — FlowRide v10.0
> Documento de referencia para el equipo de desarrollo del proyecto FlowRide.
> Curso actual: Ingeniería del Software 2 — Universidad Católica de Oriente
> Profesor: Wider Farid Sánchez Garzón
> Equipo: Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro
> Última actualización: Septiembre 2026
> Estado: v10 — Capa 1 (base del proyecto) reorganizada con método de investigación honesta: problema aterrizado con caso real, cliente ideal validado, alcance explícito con inclusiones y exclusiones, seis diferenciadores defendibles, contexto de negocio con cifras aterrizadas. Se declara la estrategia de trabajo por capas. Se descarta el argumento "no hay tiempo en el semestre" como justificación de decisiones. Se descarta la referencia a códigos del planeador del profesor (PE-DAN-X, etc.) como estructura de trabajo. Se cambia el modelo de aislamiento multi-tenant de Pool a Silo. JWT propio deja de ser decisión firme y pasa a decisión pendiente de ADR en Capa 3. Twilio deja de ser decisión firme y pasa a decisión pendiente de ADR en Capa 3 (con Meta Cloud API como candidato preferente).

---

## INSTRUCCIONES DE USO

Este documento es la **fuente de verdad** del proyecto FlowRide. Ante cualquier duda sobre problema, cliente, alcance, diferenciadores, contexto de negocio, reglas de dominio, roles, clases o funcionalidades, este documento tiene precedencia sobre cualquier respuesta generada sin este contexto.

Cuando se abre un chat nuevo, se adjunta este archivo junto con la bitácora y se dice: *"Continúo trabajando en FlowRide, aquí están el contexto maestro y la bitácora."*

---

## 0. NOTA DE POSTURA — Reglas de argumentación

### 0.1 Re-evaluación de decisiones heredadas

Las decisiones técnicas de FlowRide se tomaron originalmente en el semestre 1-2026, curso de **Diseño Orientado a Objetos (DOO)**, con un profesor distinto y un foco diferente (correctitud OO, entidades, herencia, JWT funcionando). En el semestre 2-2026, **Ingeniería del Software 2** con el profesor Wider Sánchez, el foco es distinto: drivers arquitectónicos, atributos de calidad, tácticas de Bass, ADRs con alternativas honestas y trade-offs conscientes.

Por eso:

1. **Ninguna decisión hereda validez automática** por haber sido "aprobada" en DOO. Cada una se revisa contra el estado actual del mercado, contra los escenarios de calidad del `tradeoff.xlsx`, y contra los criterios del profesor actual.
2. Los ADRs (Capa 3) arrancan en estado **Proposed** por defecto, no en Accepted. Solo pasan a Accepted cuando el análisis dentro del ADR lo justifica y el equipo lo defiende conscientemente.
3. Si una decisión heredada del semestre anterior no sobrevive el análisis actual, se reescribe o se rechaza — aunque ya esté implementada en código. La implementación previa no es argumento arquitectónico.
4. Las etiquetas "Validada por el profe ✅" que aparecen en secciones heredadas (arquitectura, diagrama de componentes, diagrama de clases) se conservan como registro histórico del semestre anterior, no como argumento defendible en el curso actual.

Esta postura evita el anti-patrón *Fairy Tale* (justificación por autoridad ajena) y *Sales Pitch* (dar por hecho lo que no se ha argumentado con el rigor actual).

### 0.2 Regla de framing — no invocar "el semestre"

**Ninguna decisión o exclusión se justifica con "no hay tiempo en el semestre", "fuera del alcance del curso", "el equipo no alcanza"** o cualquier variante académica. El proyecto se trabaja como producto real con horizonte realista (V1 a 12 meses, V2 a 24, V3 a 36+); los avances se muestran en clase de forma incremental, pero las decisiones se toman con criterios de producto real: dolor del cliente, dominio, arquitectura, contrato de negocio, presupuesto operacional.

Toda exclusión de este documento va acompañada del argumento *decisión → alternativa descartada → por qué (con criterio objetivo)*.

### 0.3 Regla operativa — sin códigos del planeador

El profesor Wider **no usa los códigos del planeador** (PE-DAN-X, PE-DEP-X, PE-DD-X, PE-LB-X) como estructura de sus clases ni de sus preguntas. En consecuencia, el equipo tampoco los usa como estructura de trabajo ni en los entregables. Los códigos existen en el planeador oficial del curso y aparecen en la bitácora histórica del proyecto, pero no organizan el trabajo actual. Se trabaja por temas al ritmo que el equipo defina, siguiendo la estrategia por capas de la sección 0.4.

### 0.4 Estrategia de trabajo por capas

El proyecto se re-planta con una estrategia explícita, ordenada de fundamento a implementación. Cada capa se cierra antes de mover a la siguiente y produce artefactos concretos que quedan versionados en este documento y en la bitácora.

| Capa | Contenido | Estado |
|---|---|---|
| **Capa 1** — Base del proyecto | Problema, visión, cliente, segmentos, alcance explícito, diferenciadores, contexto de negocio | ✅ Cerrada en v10 |
| **Capa 2** — Drivers arquitectónicos | Limpieza del `tradeoff.xlsx`: atributos de calidad como escenarios, utility tree, priorización, corrección de los hallazgos del Punto 3 de la bitácora | 🔄 Siguiente |
| **Capa 3** — ADRs con método de investigación honesta | Diez ADRs, uno por chat: monolito vs. microservicios (re-evaluación), modelo de aislamiento multi-tenant, autenticación, base de datos, caché, escalabilidad horizontal, WhatsApp (re-abrir ADR-05), telemetría GPS, notificaciones, reportes y sincronización asíncrona | ⏳ Pendiente |
| **Capa 4** — Diseño detallado | Diagramas UML de las 4+1 vistas, ajustes al diagrama de clases con las entidades faltantes (`Calificacion`, `CancelacionSolicitud`, `EstadoSolicitudHistorial`, `RegistroUbicacion`, atributo `esPersonal`) | ⏳ Pendiente |
| **Capa 5** — Implementación y línea base | Migración del código a las decisiones nuevas, línea base del repositorio, CI/CD, análisis estático | ⏳ Pendiente |

El método de investigación honesta (estrenado en el ADR-05 de WhatsApp) es el mismo para todas las capas: **contexto real → alternativas reales investigadas → decisión con trade-offs declarados → consecuencias asumidas conscientemente**.

---

## 1. TÍTULO DEL PROYECTO

**FlowRide** — Plataforma SaaS de gestión de transporte corporativo interno para operadores B2B2B.

---

## 2. OBJETIVO GENERAL

Construir una plataforma SaaS que reemplace la coordinación manual del transporte corporativo interno — hoy sostenida en la cabeza de un administrador y en decenas de hilos de WhatsApp — por un sistema que externalice la lógica de asignación, la trazabilidad de servicios y la respuesta a contingencias. La plataforma sirve simultáneamente a múltiples empresas operadoras de transporte (multi-tenant SaaS con aislamiento físico por operador), integra el canal WhatsApp como interfaz única para pasajero y conductor, e incorpora telemetría GPS del vehículo para monitoreo en tiempo real y detección de desviación de ETA.

---

## 3. OBJETIVOS ESPECÍFICOS

1. Diseñar una arquitectura de software que soporte multi-tenancy SaaS con aislamiento físico por operador (modelo Silo) y que sea evolutiva para acomodar decisiones re-evaluadas en la Capa 3.
2. Implementar una API REST en .NET que gestione la lógica de negocio para administración de usuarios, itinerarios, solicitudes, asignaciones, viajes y telemetría.
3. Implementar autenticación centralizada por token con roles y trazabilidad al operador; el proveedor concreto (JWT propio, Auth0, Keycloak, IdentityServer, Cognito) se decide en la Capa 3.
4. Integrar el canal WhatsApp Business como interfaz única para pasajero y conductor; el proveedor concreto (Meta Cloud API directo o intermediario tipo Twilio) se decide en la Capa 3.
5. Implementar un motor de asignación de conductores basado en reglas configurables por operador, pensado para servicios programados por turnos, no para competencia por proximidad en tiempo real.
6. Incorporar ingesta y consulta de telemetría GPS del vehículo, con detección de desviación de ETA respecto a la ventana comprometida al cliente empresarial.
7. Diseñar un modelo de persistencia que garantice integridad y aislamiento por operador (Silo), y que soporte reportes operacionales sobre una base de datos separada.

---

## 4. DESCRIPCIÓN GENERAL Y PROBLEMA QUE RESOLVEMOS

FlowRide es una plataforma SaaS multi-tenant orientada a la gestión del transporte corporativo interno para operadores que atienden bajo contrato B2B a empresas clientes con horarios estrictos: aerolíneas con tripulaciones, minería con cuadrillas entre campamentos, hospitales con personal entre sedes, empresas de seguridad con equipos operativos.

### 4.1 Problema que resolvemos

Hoy, las empresas de transporte corporativo interno operan con un sistema real pero frágil: un administrador coordina toda la operación desde su celular, usando WhatsApp como canal y Excel como registro. Un caso representativo — validado con el administrador de una empresa que atiende tripulaciones aéreas — coordina alrededor de 40 servicios al día para 4 o 5 conductores, con un hilo de WhatsApp separado por cada servicio. La lógica de asignación (qué conductor va con qué pasajero, a qué hora, con qué vehículo) vive en la cabeza del administrador. La trazabilidad de qué se pidió, qué se prometió y qué se cumplió vive dispersa entre esos 40 chats. La contingencia ante imprevistos vive en su capacidad personal de reaccionar.

Ese arreglo funciona en operación normal, pero al costo de mantener a una persona dedicada full-time a coordinar y de que todo el negocio dependa de ella. Y colapsa cuando algo se rompe. En un incidente reportado por el administrador entrevistado, un conductor abandonó su turno en medio del día. La consecuencia inmediata fue que el administrador tuvo que redistribuir a mano los servicios restantes, doblar turno él mismo como conductor para cubrir los que faltaban, y aun así el pasajero afectado llegó tarde a su compromiso. La consecuencia diferida — la que le pone plata al problema — fue pérdida de confianza del cliente empresarial (una aerolínea), del cual depende el contrato que sostiene la operación.

FlowRide no automatiza algo que hoy no existe: reemplaza el sistema actual —una persona con un celular— por uno donde la lógica de asignación, la trazabilidad y la contingencia dejen de depender exclusivamente del administrador.

### 4.2 Capacidades principales

- Pasajeros y conductores interactúan por WhatsApp para solicitar transporte, recibir notificaciones, aceptar servicios, reportar inicio/fin de viaje e incidentes.
- API REST centraliza la lógica de negocio y expone endpoints para el panel web administrativo.
- Panel web administrativo para administrador de empresa y operador logístico, con visibilidad completa de la operación del propio operador (aislamiento por Silo).
- Motor de asignación automática basado en reglas configurables por operador.
- Ingesta de telemetría GPS del vehículo con detección de desviación de ETA respecto a la ventana comprometida.

---

## 5. CLIENTE IDEAL Y SEGMENTOS TARGET

### 5.1 Cliente ideal (V1)

Operadores de transporte corporativo interno bajo contrato B2B con un cliente empresarial más grande que exige horarios estrictos. El caso ancla validado es transporte de tripulaciones aéreas para aerolínea. El diseño se mantiene neutral (nomenclatura genérica: `Pasajero`, `empresaTransporte`, `codigoServicio`) para que otros sectores con la misma forma operacional puedan encajar sin re-arquitectura, pero no se prometen como clientes en V1.

### 5.2 Segmentos que sí — tres condiciones al mismo tiempo

- **B2B2B**: el cliente de FlowRide es un operador de transporte que le vende el servicio a un cliente empresarial más grande bajo contrato mensual o anual. No cobran directamente al pasajero.
- **Horarios estrictos**: perder el horario del servicio cuesta plata al cliente empresarial (una tripulación que no llega hace perder un vuelo; una cuadrilla que no llega retrasa un turno de mina; un médico que no llega deja una guardia sin cubrir). Sin horarios críticos, el motor de asignación y la trazabilidad valen mucho menos.
- **Flota propia o bajo contrato exclusivo**: los conductores son parte de la operación, no proveedores externos que compiten por un viaje.

Ejemplos por vertical (V2 en adelante, no comprometidos como clientes V1): minería con cuadrillas entre campamentos, salud con personal médico entre sedes, seguridad con equipos operativos, industria con turnos rotativos entre plantas.

### 5.3 Segmentos que NO — con argumento defendible

- **Apps de ride-hailing tipo Uber/Cabify/Didi.** *Alternativa descartada:* modelo marketplace con cliente final que paga por viaje. *Por qué no:* el pasajero de FlowRide no elige ni paga, es empleado de la empresa. Ese cambio de comprador rompe modelo de dominio, motor de asignación (programación vs. proximidad), canal (WhatsApp vs. app) y facturación (contrato vs. viaje).
- **Turismo, chárter y limusinas tipo Sengerio.** *Alternativa descartada:* mercado B2C con cotización por evento. *Por qué no:* cliente distinto (persona natural o agencia), operación distinta (evento único vs. recurrencia), cobro distinto (tarifario vs. contrato). Sengerio no es competidor directo — juega otro juego.
- **Transporte escolar.** *Alternativa descartada:* servicio a colegios y padres. *Por qué no:* el pasajero es menor de edad, lo cual añade regulación específica (rutas escolares, escoltas, consentimiento parental) que cambia diseño de identidad y responsabilidad ante incidentes. Fuera del alcance del producto V1; no es "aún no", es "no encaja con el modelo".
- **Delivery y logística de mercancía.** *Alternativa descartada:* transporte de paquetes en vez de personas. *Por qué no:* no hay pasajero, no hay tripulación, no hay obligación de horario personal. Dominio distinto.
- **Transporte público municipal.** *Alternativa descartada:* rutas fijas para ciudadanos sin contrato B2B. *Por qué no:* no hay cliente empresarial, no hay reglas configurables por tenant, no hay contrato de servicio con penalidades — desaparece el gancho del producto.

---

## 6. ALCANCE EXPLÍCITO — QUÉ NO HACEMOS Y QUÉ SÍ

Cada exclusión con la fórmula **decisión → alternativa descartada → por qué (criterio objetivo)**. Ningún argumento invoca "el semestre".

### 6.1 Exclusiones firmes del producto

- **Tarifario y cobro al pasajero.** *Alternativa descartada:* modelo estilo Uber/ToolRides con precio por viaje. *Por qué no:* el pasajero es empleado del cliente empresarial, no comprador. Meter tarifario ensucia el dominio sin resolver problema real del caso de uso.
- **Chat propio in-app.** *Alternativa descartada:* mensajería in-app estilo AllRide o ToolRides. *Por qué no:* WhatsApp es el canal conversacional único; duplicarlo rompe el principio de canal único que sostiene la propuesta.
- **Marca blanca / white-label.** *Alternativa descartada:* rebranding por tenant estilo ToolRides. *Por qué no:* el producto se vende como plataforma común a múltiples operadores; personalización visual por tenant no está en la propuesta de valor y multiplica complejidad de despliegue sin retorno.
- **Red de proveedores externos / subcontratación.** *Alternativa descartada:* modelo de partners estilo Sengerio. *Por qué no:* cada operador cliente opera su propia flota interna. La subcontratación no está en el modelo de negocio del cliente ancla ni de los segmentos target.
- **Facturación al cliente empresarial y pasarelas de pago.** *Alternativa descartada:* módulo de facturación integrado tipo Sengerio con Stripe/PCI-DSS. *Por qué no:* el cobro operador-cliente empresarial es un contrato mensual fijo negociado fuera del sistema. FlowRide genera los datos que alimentan la facturación (reportes de servicios ejecutados); no cobra.
- **Aplicación móvil nativa para pasajero o conductor.** *Alternativa descartada:* apps iOS/Android como AllRide o ToolRides. *Por qué no:* WhatsApp es el canal declarado y validado. Construir apps duplica el trabajo de UX, autenticación de dispositivo y notificaciones push, y agrega fricción de descarga que el producto quiere evitar. El panel web administrativo existe pero no es una app móvil nativa.
- **Predicción de demanda con IA / machine learning.** *Alternativa descartada:* forecasting estilo AllRide. *Por qué no:* requiere volumen histórico que un producto nuevo por definición no tiene, y agrega complejidad arquitectónica (pipeline de datos, entrenamiento, servido de modelos) sin que el pronóstico aparezca como dolor validado del cliente ancla (los servicios son programados por contrato, no impredecibles). Se re-evalúa en V3.
- **Mantenimiento predictivo de vehículos.** *Alternativa descartada:* funcionalidad estilo AllRide. *Por qué no:* es otro producto con otro modelo de dominio (vehículo como activo mecánico vs. vehículo como recurso operativo). Fuera del problema que resolvemos.
- **Ruteo dinámico con optimización de rutas.** *Alternativa descartada:* motor tipo Google Maps o servicios de optimización de flota. *Por qué no:* el caso de uso son servicios programados con origen y destino ya definidos por el cliente empresarial. No hay ruta que optimizar. Cuando aparezca un cliente que sí necesite optimización compleja (minería con múltiples paradas), se evalúa aparte.
- **Integración con sistemas de RRHH / HRMS / nómina.** *Alternativa descartada:* conectores estilo AllRide. *Por qué no:* no aparece en el dolor validado con el cliente ancla; agregarlo sin cliente que lo pida es scope creep. El escenario `ADM-INTEROP-02` del `tradeoff.xlsx` (importador CSV/Excel de RRHH) sale del alcance y del banco de 33 seleccionados — pendiente de corrección en Capa 2.
- **Multi-idioma / internacionalización.** *Alternativa descartada:* soporte a inglés, portugués u otros. *Por qué no:* cliente y usuarios están en Colombia, canal en español. La internacionalización se abre cuando aparezca un cliente que la exija.
- **Notificaciones por SMS, correo electrónico o llamada automatizada hacia pasajero/conductor.** *Alternativa descartada:* multi-canal saliente. *Por qué no:* WhatsApp es el canal único hacia pasajero y conductor. Correo se usa solo para recuperación de contraseña del panel web administrativo (donde WhatsApp no aplica).

### 6.2 Inclusiones firmes del producto

- **Calificación `Pasajero → Conductor` sobre `Viaje`.** Entidad nueva, refuerza el discurso de calidad operativa. Pendiente de propagar al diagrama de clases (Capa 4), al banco de escenarios del `tradeoff.xlsx` (Capa 2) y a la sección 15 de este documento.
- **Reportes operacionales básicos** (servicios ejecutados por periodo, cumplimiento por conductor, incidentes por rango de fechas) sobre base de datos separada `flowride_reporting` con sincronización asíncrona desde `flowride_core`. Dashboards analíticos avanzados y BI quedan fuera de V1.
- **Telemetría GPS del vehículo con detección de desviación de ETA.** Nueva inclusión respecto al v9. Aporta valor concreto al escenario `PAS-CONF-01` del `tradeoff.xlsx` (que en v9 tenía una promesa incumplible sin telemetría) y refuerza el diferenciador 5. Abre trabajo arquitectónico propio en Capa 3 (ver 6.3).

### 6.3 Consecuencias de incluir telemetría GPS — para trabajar en Capa 3

Meter telemetría abre estas decisiones que requieren su propio ADR en la Capa 3. Se declaran aquí para que no aparezcan de sorpresa.

- **Fuente de la ubicación.** Tres caminos: (a) hardware GPS dedicado en el vehículo, integrado vía proveedor tipo Satrack — más robusto, exige hardware y contrato adicional; (b) GPS del celular del conductor vía WhatsApp Live Location — más barato, pero Live Location tiene límites de duración (15 min a 8 horas según fuente) y hay que verificar soporte del proveedor de WhatsApp Business elegido; (c) app propia del conductor solo para reportar ubicación — contradice la exclusión de app móvil de la sección 6.1. La elección se hace en un ADR dedicado, contrastando las tres.
- **Cadencia y volumen de datos.** Con 50 conductores por instancia reportando cada 30 segundos son ~144 000 eventos/día por instancia. Postgres puro maneja pero no es óptimo; hay que evaluar particionamiento nativo, TimescaleDB, o almacén separado para series temporales.
- **Motor de asignación.** Se mantiene la decisión del diferenciador 5: **motor puramente programado por turnos**, telemetría se usa solo para monitoreo y detección de desviación. **No abre reasignación dinámica por cercanía** en V1. Cuando aparezca en V2 o V3 la necesidad se re-evalúa.
- **Habeas Data.** La ubicación en tiempo real de conductor y pasajero es dato personal sensible bajo Ley 1581. El escenario `PAS-SEG-01` (ocultar ubicación en solicitudes personales) deja de ser opcional y se vuelve requisito duro. Esto refuerza el atributo Seguridad en el utility tree de la Capa 2.
- **Modelo de dominio.** Aparece entidad `RegistroUbicacion` (vehiculoId, timestamp, lat, lon) con relación a `Viaje`. Se agrega a la sección 17 en Capa 4.
- **Re-apertura del ADR-05 (WhatsApp).** Si la fuente elegida es WhatsApp Live Location, hay que confirmar soporte del proveedor. Esto se cruza con la decisión pendiente de proveedor de WhatsApp (Meta Cloud vs. Twilio).

---

## 7. DIFERENCIADORES DEL PRODUCTO

Seis diferenciadores defendibles, cada uno anclado en decisión de producto o de arquitectura ya tomada.

1. **Multi-tenant SaaS con aislamiento físico por operador (modelo Silo).** Base de datos física por operador cliente. Aislamiento verdadero de datos y opcionalmente de cómputo. Sengerio es single-tenant real (instalable por cliente); AllRide y ToolRides son multi-tenant en modelo Pool con aislamiento lógico. FlowRide es SaaS multi-tenant con aislamiento físico — más caro de operar que Pool, más seguro contra fallos cruzados y consumo cruzado entre tenants, más barato de operar que single-tenant instalable.
2. **Roles operativos específicos de transporte corporativo interno.** Pasajero empleado + Conductor de nómina + Administrador de operador + Operador logístico. Los competidores modelan roles de marketplace (cliente externo que paga + conductor + operador de flota). No es la misma abstracción con nombres distintos: son estructuras de responsabilidad distintas.
3. **Ausencia deliberada de tarifario y cobro al pasajero.** El comprador es el cliente empresarial vía contrato mensual, no el pasajero por viaje. Modelo de dominio limpio de conceptos de facturación al usuario final — decisión, no carencia.
4. **Reglas de asignación configurables por operador.** Hoy la lógica vive en la cabeza del administrador y cambia según prioridades del operador (uno minimiza cambios de turno, otro prioriza distancia hotel-aeropuerto, otro balancea carga entre conductores). FlowRide externaliza esa lógica y la deja configurable por tenant.
5. **Motor de asignación programado por turnos, con telemetría GPS para monitoreo y detección de desviación (no para reasignación dinámica por cercanía).** El caso de uso no es competencia por proximidad como un app de taxi — son servicios programados con contrato. La telemetría refuerza la trazabilidad del compromiso hecho al cliente empresarial sin cambiar la naturaleza del motor.
6. **WhatsApp como canal único para pasajero y conductor, sin app propia — decisión de diseño anclada en el comportamiento real del cliente.** ToolRides tiene WhatsApp Bot pero también app móvil, portal y chat integrado; WhatsApp es un canal más. FlowRide toma la decisión opuesta: WhatsApp es el único canal para pasajero y conductor. En el mundo del cliente ancla ("el administrador siempre carga el celular, la gente ya usa WhatsApp todo el día"), esto reduce fricción de adopción a cero.

---

## 8. CONTEXTO DE NEGOCIO REALISTA

### 8.1 Volumen aterrizado

Cliente ancla real: ~40 servicios/día, 4-5 conductores, 1-2 personas en administración. Extrapolando a rango de operadores target:

| Dimensión | Rango operador pequeño | Rango operador grande | Techo de dimensionamiento por instancia (V1) |
|---|---|---|---|
| Pasajeros activos | 50-100 | 200-500 | 500 |
| Conductores | 5-15 | 30-50 | 50 |
| Administradores + operadores logísticos | 1-2 | 4-8 | 10 |
| **Total usuarios activos** | ~60 | ~600 | ~600 |
| Concurrencia en pico (arranque de turno) | ~20 | ~200 | **200 concurrentes** |

El "10 000 usuarios concurrentes" del v9 se reemplaza por **~200 concurrentes por instancia**, con el argumento anclado en el caso real. La escala a nivel plataforma se resuelve por Silo: agregar tenants = agregar instancias, no aumentar carga a una instancia existente.

**Proyección de tenants a nivel plataforma:**
- **V1 (0-12 meses):** 5 tenants activos, todos operadores de transporte de tripulaciones aéreas o similar.
- **V2 (12-24 meses):** 20 tenants, extensión a operadores de minería y salud.
- **V3 (24-36+ meses):** 50-100 tenants, evaluación de expansión regional latinoamericana.

### 8.2 Rendimiento por endpoint

El "2 segundos" del v9 se reemplaza por umbrales concretos por tipo de operación:

| Tipo de operación | P95 | P99 |
|---|---|---|
| Operaciones CRUD (crear solicitud, aceptar servicio, actualizar estado) | < 500 ms | < 1.5 s |
| Consultas de estado (ver solicitud, ver asignación) | < 300 ms | < 1 s |
| Reportes operacionales | < 2 s | < 5 s |
| Ingesta de eventos de telemetría GPS | no aplica percentil de respuesta al usuario — se mide por throughput sostenido: ~100 eventos/min por instancia con 50 conductores reportando cada 30 s | — |

### 8.3 Disponibilidad por tier de servicio

Reemplaza el "99% uptime mínimo" del v9 con esquema por tier declarado como decisión consciente:

| Servicio | SLA | Downtime aceptable/año |
|---|---|---|
| Canal WhatsApp (interfaz operativa pasajero/conductor) | 99.9% | ~8h |
| Motor de asignación y operaciones críticas | 99.5% | ~43h |
| Panel web administrativo y operaciones asíncronas | 99% | ~87h |

Anclado en los públicos: WhatsApp es interfaz de operación en vivo, no puede caer en horario operativo; el panel es interno del operador y aguanta más caída.

### 8.4 Presupuesto operacional

Marco de razonabilidad — no son promesas comerciales, son referencia para las decisiones arquitectónicas de la Capa 3.

- **Costo operacional por instancia (Silo) al lanzamiento:** techo de **20-40 USD/mes por tenant** en infraestructura básica (BD gestionada pequeña, cómputo compartido para instancias con baja actividad, storage de logs y telemetría). Determina qué proveedores caben.
- **Costo variable relevante — WhatsApp Business API:** con 40 servicios/día × ~3 mensajes por servicio = ~120 mensajes/día por instancia. La mayoría en ventana de 24h (gratis en Meta Cloud API). Proactivos fuera de ventana (recordatorios, alertas) son costo real: ~50 USD/mes por instancia grande.
- **Pricing al cliente que soporta esos costos:** rango razonable **150-500 USD/mes por instancia** según tamaño del operador. Referencia de mercado: ToolRides cobra ~100-300 USD según fuentes públicas.

Estos números sirven para decir en un ADR: *"elegimos X sobre Y porque a los volúmenes proyectados (V2, 20 tenants) el costo operacional de X es ~800 USD/mes vs. 3200 USD/mes de Y, y el pricing al cliente no aguanta la diferencia"*.

### 8.5 Marco legal — Colombia

- **Ley 1581 de Habeas Data.** FlowRide es **encargado del tratamiento**, el operador cliente es **responsable del tratamiento**. Datos sensibles tratados: nombre, teléfono, ubicación en tiempo real (con telemetría). Multas por incumplimiento hasta **2.000 SMMLV** (~2.800 millones COP al 2026). El escenario `LEGAL-HD-01` del `tradeoff.xlsx` lo cubre; se refuerza con la inclusión de telemetría.
- **Contrato SaaS FlowRide ↔ Operador.** Incluye: confidencialidad, aislamiento (Silo lo refuerza como cláusula defendible), notificación de incidentes, derechos ARCO (Acceso, Rectificación, Cancelación, Oposición) sobre datos de pasajeros y conductores. Sin estas cláusulas ningún cliente empresarial serio firma.
- **Propiedad de los datos.** Los datos son del operador, no de FlowRide. FlowRide puede usar datos agregados y anonimizados para métricas de producto; no datos identificables.
- **Retención propuesta:** datos operacionales activos indefinidos mientras el contrato esté vigente; datos históricos archivados después de 24 meses; telemetría GPS retenida 6 meses por sensibilidad y volumen; datos de auditoría legal 5 años por reclamación laboral o penal.
- **Regulación sector transporte especial (Ministerio de Transporte).** El operador cliente debe tener habilitación, conductores con licencia y cursos vigentes, vehículos con revisión técnico-mecánica al día. FlowRide **no es responsable** de esa regulación pero **la modela como reglas de dominio**: bloquear asignación de conductor con licencia vencida, alertar vehículo con RTM próxima a vencer.

### 8.6 Horizonte del producto

- **V1 (0-12 meses):** 5 tenants activos, foco en estabilizar operación del caso ancla, iterar con feedback real de operadores de tripulaciones aéreas.
- **V2 (12-24 meses):** 20 tenants, extensión a operadores de minería y salud. Se estabiliza telemetría GPS. Se evalúan integraciones con proveedores de telemetría profesional (Satrack u otros) como opción para operadores que ya los tienen contratados.
- **V3 (24-36+ meses):** 50-100 tenants, expansión regional, integraciones con HRMS/nómina, dashboards analíticos avanzados, marca blanca opcional. Aquí entran las cosas que hoy están en la lista de exclusiones como roadmap declarado.

Los ADRs de la Capa 3 se toman pensando en V1 con revisiones planeadas para no cerrar puertas de V2/V3.

---

## 9. DECISIÓN DE ARQUITECTURA — Monolito Modular

*[Sección heredada del semestre 1-2026, pendiente de re-evaluación en Capa 3. Se conserva como estado actual del código y como candidato preferente del ADR de arquitectura de aplicación.]*

**Arquitectura actual: Monolito Modular.**

**Argumento vigente (a re-evaluar en ADR de Capa 3):**

| Criterio | Microservicios | FlowRide V1 |
|---|---|---|
| Independencia de despliegue por módulo | Necesaria | No requerida |
| Escalabilidad diferenciada por módulo | Necesaria | No requerida |
| Tamaño del equipo | Grandes por servicio | Pequeño (3 personas) |
| Cantidad y complejidad de módulos | Muy alta | Acotada — 8 módulos bien definidos |

En Capa 3, el ADR de arquitectura de aplicación va a comparar formalmente Monolito Modular vs. Microservicios vs. arquitectura por capas simples, con ponderación numérica reproducible (no solo cualitativa). El resultado esperado es que el monolito modular sobreviva, pero con argumento nuevo — no por herencia.

**El Motor de Asignación** es un módulo interno dentro del monolito, no un microservicio separado.

---

## 10. ESTRUCTURA DEL PROYECTO .NET — Solución Multi-Proyecto

*[Sección heredada. Estado del código a septiembre 2026. Estructura arquitectónica vigente; los detalles de tecnologías internas (proveedor de auth, proveedor de WhatsApp, motor de BD para telemetría) están sujetos a re-evaluación en Capa 3.]*

**Estructura de la solución:**
```
FlowRide.sln
├── FlowRide.API              ← Capa de entrada
├── FlowRide.Application      ← Lógica de aplicación
├── FlowRide.Domain           ← Núcleo del negocio
└── FlowRide.Infrastructure   ← Implementaciones externas
```

### 10.1 Qué va en cada proyecto

**FlowRide.Domain**
- No depende de ningún otro proyecto.
- Entidades puras del negocio: Pasajero, Conductor, Empresa, Vehiculo, Itinerario, SolicitudTransporte, Asignacion, Viaje, Parada, Incidente, Rol, Usuario.
- Pendientes de agregar (Capa 4): Calificacion, CancelacionSolicitud, EstadoSolicitudHistorial, RegistroUbicacion, atributo `esPersonal` en SolicitudTransporte.
- ENUMs: EstadoConductor, EstadoSolicitud, EstadoAsignacion, EstadoViaje.
- Código C# puro, sin referencias a EF Core ni ningún framework externo.

**FlowRide.Application**
- Depende solo de Domain.
- Interfaces de repositorios y servicios.
- Implementaciones de servicios de aplicación.
- DTOs.
- Define QUÉ hace el sistema sin saber CÓMO.

**FlowRide.Infrastructure**
- Depende de Domain y Application.
- FlowRideDbContext (EF Core).
- Implementaciones concretas de repositorios.
- Servicios de infraestructura (auth, WhatsApp, caché) — implementaciones concretas sujetas a los ADRs de Capa 3.
- Configuración de conexión a PostgreSQL (Npgsql).

**FlowRide.API**
- Depende de Application e Infrastructure.
- Controllers (sin lógica de negocio).
- Program.cs (registro de dependencias).
- Configuración de Swagger.

### 10.2 Flujo de dependencias

```
API → Application → Domain
API → Infrastructure → Domain
Infrastructure → Application (para implementar interfaces)
```

---

## 11. APLICACIONES DEL SISTEMA

### 11.1 API REST en .NET — Backend principal
- **Framework:** .NET 10
- **Solución:** 4 proyectos (Domain, Application, Infrastructure, API)
- **Conexión Panel Web:** HTTP/JSON directo, sin BFF
- **Conexión WhatsApp:** Webhook HTTP entrante

**Módulos internos:**
1. Autenticación y Autorización (proveedor concreto pendiente de ADR en Capa 3).
2. Gestión de Usuarios y Empresas.
3. Gestión Operativa de Solicitudes y Viajes.
4. Motor de Asignación (reglas configurables por operador).
5. Servicio de Notificaciones (WhatsApp saliente).
6. Integración WhatsApp (proveedor concreto pendiente de ADR en Capa 3).
7. Telemetría GPS (fuente y almacenamiento pendientes de ADR en Capa 3).
8. Reportes y sincronización asíncrona a `flowride_reporting`.
9. Auditoría y Trazabilidad.
10. Logging y Observabilidad.

### 11.2 Panel Web Administrativo
Interfaz para administrador de empresa y operador logístico. Cada usuario ve solo los datos de su propio operador (aislamiento por Silo).

### 11.3 Canal WhatsApp
Interfaz única para pasajero y conductor. Proveedor pendiente de ADR en Capa 3 (candidato preferente: Meta Cloud API directo).

---

## 12. DIAGRAMA DE COMPONENTES

*[Heredado del semestre 1-2026 con etiqueta "Validado por el profe ✅" que ya no aplica en este curso. Estructura general vigente, pendiente de actualizar en Capa 4 con: módulo de Telemetría GPS, aislamiento por instancia Silo, y proveedor de WhatsApp abstracto sin nombre concreto.]*

**Estructura en 3 niveles:**
- **Superior (externos):** Usuarios WhatsApp, Panel Web Administrativo.
- **Medio (backend):** Aplicación Monolítica FlowRide .NET con módulos internos.
- **Inferior (datos):** Bases de datos por instancia (Silo).

---

## 13. ROLES DEL SISTEMA

| Aplicación | Roles |
|---|---|
| Canal WhatsApp | Pasajero, Conductor |
| Panel Web Administrativo | Administrador de empresa, Operador logístico |

El Administrador técnico es rol de infraestructura del propio FlowRide, accede vía herramientas externas (consola, Swagger, gestor de BD). No aparece como rol de ninguna aplicación funcional.

---

## 14. MATRIZ DE ROLES Y FUNCIONALIDADES

| Funcionalidad | Pasajero | Conductor | Admin. de empresa | Operador Logístico |
|---|---|---|---|---|
| Iniciar sesión en el sistema | x | x | x | x |
| Recuperar contraseña | x | x | x | x |
| Solicitar transporte | x | | | |
| Consultar estado de la solicitud | x | | | x |
| Cancelar solicitud de transporte | x | | | x |
| Consultar historial de solicitudes/viajes | x | x | x | x |
| Recibir notificaciones del servicio | x | x | | |
| Ver servicios asignados | | x | | |
| Aceptar servicio asignado | | x | | |
| Rechazar servicio asignado | | x | | |
| Registrar inicio de viaje | | x | | |
| Registrar finalización del viaje | | x | | |
| Reportar novedades o incidentes | x | x | | |
| Calificar servicio recibido | x | | | |
| Gestionar usuarios de la empresa | | | x | |
| Gestionar sedes, áreas o dependencias | | | x | |
| Consultar reportes administrativos | | | x | |
| Configurar reglas de asignación del operador | | | x | |
| Configurar parámetros del servicio | | | x | x |
| Crear programación de rutas o servicios | | | | x |
| Asignar conductor a un servicio | | | | x |
| Monitorear operación diaria (con telemetría GPS) | | | | x |
| Validar y dar seguimiento a solicitudes | | | | x |
| Consultar indicadores operativos | | | x | x |

---

## 15. FUNCIONALIDADES — Por módulo y prioridad

*[Se mantiene la lista de v9 con dos ajustes: (a) se agrega bloque de Calificación como funcionalidad incluida; (b) se agrega bloque de Telemetría GPS. La numeración RF-01 a RF-56 mencionada en el tradeoff pero inexistente como catálogo — pendiente de producir en Capa 2.]*

**Autenticación y acceso:** inicio de sesión seguro (Alta), autenticación por token (Alta), autorización basada en roles (Alta), cierre de sesión (Alta), recuperación de credenciales (Media). Proveedor concreto pendiente de ADR en Capa 3.

**Gestión de empresas (operadores):** registro de operador en la plataforma (Alta), configuración inicial (Alta), parametrización (Media).

**Gestión de usuarios:** creación de usuarios por operador (Alta), actualización (Alta), asignación de roles (Alta), activación/inactivación (Media).

**Gestión de conductores:** registro (Alta), actualización de datos (Alta), consulta de disponibilidad (Media), activación/inactivación (Media), validación de licencia y cursos vigentes (Alta — regla de dominio).

**Gestión de itinerarios y solicitudes:** registro de itinerarios (Alta), recepción de solicitudes de transporte (Alta), validación de datos mínimos (Alta), consulta de estado (Alta), historial (Alta), modificación/cancelación (Media), distinguir solicitud personal vs. laboral (Alta — impacta escenario PAS-SEG-01 con telemetría).

**Asignación de transporte:** asignación automática de conductores (Alta), aplicación de reglas configurables por operador (Alta), visualización de asignaciones (Alta), reasignación manual (Alta), actualización de estado (Alta), priorización de solicitudes urgentes (Media).

**Integración con WhatsApp:** recepción de mensajes (Alta), procesamiento de solicitudes (Alta), envío de confirmaciones al pasajero (Alta), envío de notificaciones al conductor (Alta), actualización de estado del viaje desde WhatsApp (Alta), envío de recordatorios automáticos (Media).

**Telemetría GPS:** ingesta de eventos de ubicación del vehículo (Alta), consulta de última ubicación conocida por viaje (Alta), detección de desviación respecto a ETA comprometida (Alta), consulta de trayecto histórico por viaje (Media). Fuente y almacenamiento pendientes de ADR en Capa 3.

**Panel web administrativo:** visualización centralizada de solicitudes y viajes (Alta), monitoreo operativo con telemetría (Alta), consulta de usuarios, conductores y asignaciones (Alta), intervención manual (Alta), trazabilidad de cada viaje (Alta), búsqueda y filtrado (Media), indicadores básicos (Media).

**Notificaciones (solo por WhatsApp):** cambios de estado (Media), confirmación de servicio (Media), reasignación o novedades (Media), alertas operativas para administradores/operadores (Media).

**Calificación:** registro de calificación por pasajero al finalizar el viaje (Media), consulta de calificación promedio por conductor (Media), consulta de calificación promedio por operador (Media).

**Reportes operacionales básicos:** viajes realizados (Media), solicitudes atendidas/pendientes (Media), conductores asignados (Media), histórico de operación por operador (Media).

**Multi-tenant (Silo):** aislamiento físico de la información por operador cliente (Alta), administración independiente por operador (Alta), configuración separada de usuarios y recursos por cliente (Alta).

**Logging y monitoreo:** eventos del sistema (Media), errores y excepciones (Media), acciones de usuarios administrativos (Media), disponibilidad técnica (Baja).

**Configuración avanzada:** reglas de asignación configurables por operador (Alta), parámetros de SLA por servicio (Media).

**Analítica avanzada, forecasting, mantenimiento predictivo, integraciones HRMS/nómina, marca blanca, ruteo dinámico, multi-idioma, apps móviles nativas:** **FUERA DEL ALCANCE.** Ver sección 6.1 con argumento.

---

## 16. MODELO DE AISLAMIENTO MULTI-TENANT — Silo

**Decisión:** modelo Silo — base de datos física por operador cliente.

**Argumento defendible:**
- Aislamiento verdadero de datos por separación física, no por columna filtrada.
- Un fallo en la instancia de un operador no afecta a otros operadores.
- Un consumo excesivo (voluntario o accidental, incluidos ataques tipo DDoS al endpoint público del propio operador) no afecta a otros operadores si se combina con instancia dedicada o quotas duras.
- Sigue siendo SaaS multi-tenant desde la propuesta comercial: FlowRide sirve N operadores con una sola aplicación mantenida por el equipo, no es software instalable por cliente.
- Cláusulas contractuales de aislamiento (sección 8.5) se sostienen con evidencia técnica: "sus datos viven en su propia base física, no compartida".

**Alternativas descartadas:**
- **Pool (una BD, columna `empresaId` filtrada).** *Por qué no:* riesgo estructural de fuga cruzada si algún query olvida el filtro; consumo cruzado entre tenants; fallo del sistema tumba a todos. Es el estándar de Shopify, Notion, Linear y otros SaaS que priorizan costo operacional bajo — pero exige disciplina fuerte de ingeniería que un equipo pequeño no siempre puede sostener sin errores. Trade-off que se elige no correr en V1.
- **Single-tenant real / producto instalable por cliente (estilo Sengerio).** *Por qué no:* cambia el producto por completo — deja de ser SaaS, el pricing pasa a licencia por instalación, el pitch pierde el diferenciador 1, cada cliente nuevo requiere despliegue asistido. Es más simple por dentro pero cambia el modelo de negocio.

**Consecuencia sobre `empresaId`:**
En Silo, `empresaId` cambia de rol. Ya no es filtro de aislamiento entre tenants (lo hace la BD por separación física). Se queda en el modelo como **identificador del cliente empresarial dentro de la BD del operador** — sirve para que un operador atienda a múltiples aerolíneas, mineras u hospitales y los reportes separen por cliente empresarial.

**Consecuencias operacionales (pendientes de ADR de infraestructura en Capa 3):**
- Aprovisionamiento automatizado de nueva instancia al captar tenant.
- Migraciones EF Core corren por instancia; hay que orquestar despliegue.
- Monitoreo por instancia con visibilidad agregada a nivel plataforma para el equipo FlowRide.
- Backup y restauración por instancia.
- Costo operacional escala lineal con número de tenants — cubierto en el marco de la sección 8.4.

---

## 17. DIAGRAMA DE CLASES

*[Heredado del semestre 1-2026 con etiqueta "Validado ✅" que ya no aplica en este curso. Se conserva como base; pendiente de actualizar en Capa 4 con las entidades y atributos declarados a lo largo de este documento.]*

### 17.1 Nomenclatura

Nombres genéricos para múltiples industrias:

| Nombre anterior | Nombre actual | Razón |
|---|---|---|
| Tripulante | Pasajero | Genérico para cualquier empresa de transporte corporativo interno |
| aerolinea (atributo) | empresaTransporte | Aplica a cualquier industria |
| codigoEmpleado | codigoInterno | Aplica a cualquier empresa |
| numeroVuelo (Itinerario) | codigoServicio | Un vuelo es un caso; puede ser ruta o turno |

### 17.2 Clases del dominio (estado actual)

- **Empresa** (operador de transporte, es el tenant): id, nombre, nit, correo, activa.
- **Rol**: id, nombre, descripcion.
- **Usuario** (abstracta): id, nombre, correo, contrasena, telefono, activo, empresaId, rolId + métodos.
- **Pasajero** (hereda Usuario): codigoInterno, empresaTransporte + solicitarTransporte().
- **Conductor** (hereda Usuario): numeroLicencia, estado + aceptarServicio(), rechazarServicio(), registrarInicioViaje(), registrarFinViaje().
- **Vehiculo**: id, placa, marca, modelo, capacidad, activo, empresaId, conductorId.
- **Itinerario**: id, empresaId, codigoServicio, fecha, origen, destino, horaSalida, horaLlegada.
- **SolicitudTransporte**: id, empresaId, pasajeroId, itinerarioId, direccionOrigen, direccionDestino, fechaHoraSolicitud, estado, urgente + cancelar().
- **Asignacion**: id, solicitudId, conductorId, vehiculoId, estado, fechaHoraAsignacion + actualizarEstado().
- **Viaje**: id, asignacionId, empresaId, estado, fechaHoraInicio, fechaHoraFin + registrarIncidente().
- **Parada**: id, viajeId, direccion, orden, horaEstimada.
- **Incidente**: id, viajeId, descripcion, fechaHora, reportadoPor.

### 17.3 Ajustes pendientes al diagrama (Capa 4)

- **Calificacion** (nueva entidad): id, viajeId, pasajeroId, puntaje, comentario, fechaHora. Relación 1..1 con Viaje.
- **CancelacionSolicitud** (nueva entidad, exigida por escenario PAS-AUD-02): id, solicitudId, motivo, timestamp, canceladoPor. Relación 1..1 con SolicitudTransporte.
- **EstadoSolicitudHistorial** (nueva entidad, exigida por escenario PAS-CONF-02): id, solicitudId, estadoAnterior, estadoNuevo, timestamp, actor. Relación 1..* con SolicitudTransporte.
- **RegistroUbicacion** (nueva entidad, exigida por telemetría): id, vehiculoId, viajeId, timestamp, latitud, longitud. Relación 1..* con Vehiculo y 1..* con Viaje.
- **Atributo `esPersonal`** (booleano) en SolicitudTransporte, exigido por escenario PAS-SEG-01 para ocultar ubicación en solicitudes personales.

Estos cambios se aplican en Capa 4 y **se declaran explícitamente frente al profesor por tratarse de una sección validada en el semestre anterior**, no se cuelan en silencio.

---

## 18. MODELO DE DATOS

Con el cambio a Silo, la estructura de bases de datos por tenant queda:

**Por instancia de operador (Silo):**
- **flowride_core** — base transaccional del operador. Contiene todas las entidades del dominio (sección 17).
- **flowride_audit** — auditoría del operador. Pendiente de cláusulas técnicas.
- **flowride_reporting** — reportes del operador. Se alimenta asíncronamente desde `flowride_core`. Pendiente de cláusulas técnicas.
- **flowride_logs** — logs técnicos del operador. Pendiente de cláusulas técnicas.
- **flowride_telemetry** — nueva BD por instancia para almacenar registros de ubicación GPS. Motor concreto (Postgres particionado, TimescaleDB, otro) pendiente de ADR en Capa 3.

**A nivel plataforma (equipo FlowRide):**
- **flowride_control_plane** — nueva BD central para gestión de tenants: catálogo de operadores, configuración de aprovisionamiento, credenciales de conexión a cada instancia, estado de salud por instancia. Pendiente de detallar en ADR de infraestructura.

---

## 19. IMPLEMENTACIÓN TÉCNICA — Estado actual del repositorio

*[Estado a septiembre 2026. Muchas de estas decisiones son estado del código heredado del semestre 1-2026 y están pendientes de re-evaluación en la Capa 3.]*

**Repositorio:** https://github.com/SebastianZapataNaranjo/FlowRide-Backend.git

### 19.1 Stack actual (implementación heredada)

| Elemento | Valor actual | Estado |
|---|---|---|
| Framework | .NET 10 | Vigente |
| Base de datos | PostgreSQL (`flowride_core`) | Vigente (motor); estructura cambia a Silo |
| ORM | Entity Framework Core 9.0.4 + Npgsql 9.0.4 | Vigente |
| Autenticación | JwtBearer 9.0.4 — HMAC-SHA256 | **Pendiente de re-evaluación en Capa 3** |
| IdentityModel | Microsoft.IdentityModel.JsonWebTokens 8.9.0 + System.IdentityModel.Tokens.Jwt 8.9.0 | Vigente para implementación actual |
| Hash contraseñas | BCrypt.Net-Next 4.0.3 | Vigente si se mantiene JWT propio; puede reemplazarse por proveedor externo |
| Caché | IMemoryCache — Cache-Aside, TTL 5 min | Vigente en implementación; política se re-evalúa en Capa 3 |
| Documentación | Swagger — siempre activo | Vigente |
| WhatsApp | Sin integrar aún — planeado con Twilio en ADR-05 | **Pendiente de re-evaluación en Capa 3** — Meta Cloud API directo es candidato preferente |

### 19.2 Estructura actual del código

```text
FlowRide.sln
├── FlowRide.Domain/
│   └── Entities/ (12 entidades vigentes; 5 pendientes de agregar en Capa 4)
├── FlowRide.Application/
│   └── DTOs/ (Auth/, Itinerario/)
├── FlowRide.Infrastructure/
│   ├── Data/FlowRideDbContext.cs
│   ├── Services/JwtService.cs (pendiente de re-evaluación)
│   └── Migrations/ (3 migraciones aplicadas al esquema heredado; el cambio a Silo va a requerir reestructuración)
└── FlowRide.API/
    ├── Controllers/ (AuthController, ItinerarioController)
    └── Program.cs
```

### 19.3 Flujo JWT actual (implementación heredada, sujeta a re-evaluación)

1. `POST /api/Auth/registrar` — registra usuario con BCrypt.
2. `POST /api/Auth/login` — genera token HMAC-SHA256.
3. Token contiene claims: `email`, `rol`, `usuarioId`, `empresaId`.
4. Endpoints con `[Authorize]` verifican el token.

**Nota:** este flujo puede cambiar sustancialmente si el ADR de autenticación de Capa 3 opta por proveedor externo (Auth0, Cognito, Keycloak, IdentityServer).

### 19.4 appsettings.json — NO en Git

Cada desarrollador mantiene su propio archivo local. Referencia en `appsettings.example.json`.

### 19.5 Seed data actual

| Dato | Valor |
|---|---|
| Rol Administrador | `11111111-1111-1111-1111-111111111111` |
| Rol OperadorLogistico | `22222222-2222-2222-2222-222222222222` |
| Rol Pasajero | `33333333-3333-3333-3333-333333333333` |
| Rol Conductor | `44444444-4444-4444-4444-444444444444` |
| Empresa demo | `aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa` |

---

## 20. IN-MEMORY CACHE — Estado actual y política

*[Implementación vigente; política se re-evalúa en Capa 3 en el contexto del cambio a Silo y de la posible incorporación de caché distribuida por instancia si el proveedor de auth y el motor de telemetría lo justifican.]*

**Patrón:** Cache-Aside con IMemoryCache de ASP.NET Core (sin NuGet adicional).

### 20.1 Qué se cachea y qué no

| Dato | Decisión actual | Razón |
|---|---|---|
| Lista de Itinerarios | ✅ Cacheado | Consulta frecuente, datos estables por periodos cortos |
| Roles | ✅ Candidato futuro | Casi no cambian |
| Empresas cliente (dentro de la BD del operador) | ✅ Candidato futuro | Datos estables |
| Conductores y Vehículos | ✅ Candidato futuro | Consulta frecuente, baja volatilidad |
| Estado de solicitudes | ❌ No cachear | Cambia en tiempo real |
| Login / token | ❌ No cachear el token; sí evaluar caché de estado "activo" del usuario en Capa 3 | Impactado por escenario ADM-SEG-01 (revocación en 60s) |
| Viajes en curso | ❌ No cachear | Datos en movimiento constante |
| Telemetría GPS | ❌ No cachear en memoria | Volumen incompatible con cache in-memory |

### 20.2 Detalles técnicos actuales

- **TTL:** 5 minutos.
- **Cache key:** con Silo, el aislamiento por instancia ya es físico; la key puede simplificarse. Se re-define en Capa 3.
- **Invalidación:** automática al POST — el próximo GET va a BD y renueva el caché.
- **Registro:** `AddMemoryCache()` en Program.cs.

---

## 21. ESTRATEGIA DE RAMAS — GitHub

### 21.1 Reglas acordadas

- **main** es la rama estable. Solo recibe código probado y funcionando.
- Ningún desarrollo nuevo se hace directamente en main.
- Cada experimento o feature vive en su propia rama.
- Commits frecuentes en ramas de trabajo aunque el código sea WIP.
- Cuando el experimento funciona → merge a main vía Pull Request.
- Nunca dos personas trabajan en la misma rama al mismo tiempo.

### 21.2 Convención de nombres

```text
main
  ├── feature/gestion-usuarios
  ├── feature/silo-migration
  ├── feature/telemetria-gps
  ├── feature/integracion-whatsapp
  └── refactor/*
```

---

## 22. DECISIONES TÉCNICAS CLAVE — Tabla consolidada

| Decisión | Estado en v10 |
|---|---|
| Lenguaje | C# / .NET 10 — vigente |
| API | REST HTTP/HTTPS + JSON — vigente |
| Autenticación (tipo) | Token con roles y trazabilidad al operador — vigente |
| Autenticación (proveedor) | **Pendiente ADR Capa 3** — candidatos: JWT propio, Auth0, Cognito, Keycloak, IdentityServer |
| Autorización | Roles con `[Authorize]` — vigente |
| Caché | IMemoryCache Cache-Aside TTL 5 min — vigente, política se re-evalúa en Capa 3 |
| Arquitectura de aplicación | Monolito Modular (implementación actual) — pendiente re-evaluación formal en ADR de Capa 3 |
| Estructura del código | Domain / Application / Infrastructure / API — vigente |
| Motor de asignación | Módulo interno programado por turnos, con reglas configurables por operador — vigente |
| Telemetría GPS | Incluida en el producto — fuente y almacenamiento pendientes de ADR en Capa 3 |
| Panel Web → API | Directo HTTP/JSON, sin BFF — vigente |
| Motor de BD | PostgreSQL — vigente |
| Motor de BD para telemetría | Pendiente ADR Capa 3 — candidatos: Postgres particionado, TimescaleDB, otro |
| ORM | Entity Framework Core + Npgsql — vigente |
| **Modelo de aislamiento multi-tenant** | **Silo — BD física por operador** (cambia desde Pool del v9) |
| `empresaId` en modelo | Se conserva como identificador del cliente empresarial dentro de la BD del operador, no como filtro de aislamiento |
| Escalabilidad | Horizontal a nivel plataforma (más tenants = más instancias); vertical dentro de instancia hasta 200 concurrentes |
| Control de versiones | Git — ramas por feature/fase, merge a main vía PR — vigente |
| Swagger | Siempre activo — vigente |
| HTTPS local | Desactivado en desarrollo — vigente |
| appsettings.json | Excluido de Git — vigente |
| Canal WhatsApp | Único canal para pasajero/conductor — vigente |
| Proveedor WhatsApp | **Pendiente ADR Capa 3** — Meta Cloud API directo candidato preferente sobre Twilio |

---

## 23. CALENDARIO DEL PROFESOR — Referencia de fechas

*[El profesor Wider no organiza sus clases ni entregables por los códigos del planeador (PE-DAN-X, etc.). Esta tabla resume solo el calendario y los temas para referencia; el trabajo del equipo se organiza por las capas de la sección 0.4, no por códigos.]*

| Sesión | Fecha | Tema de clase | Hito relevante |
|---|---|---|---|
| 8 | 9 sep 2026 | Seguimiento parcial | — |
| 15 | 28 oct 2026 | GoF, 12-Factor, Spec Driven Development | Entrega del Documento de Arquitectura de Software (DAS) |
| 16 | 4 nov 2026 | GoF, DAO, Well-Architected | Entrega de línea base + Reto Spec Driven Development |
| 17 | 11 nov 2026 | Cierre y retrospectiva | — |

Los códigos del planeador (PE-DAN-X, PE-DEP-X, PE-DD-X, PE-LB-X) siguen existiendo en el archivo oficial del curso y aparecen mencionados en la bitácora histórica del proyecto, pero **no se usan como estructura de trabajo ni de entregables del equipo**.

---

## 24. HISTORIAL DE ENTREGAS Y FEEDBACK

| Ítem | Semestre | Estado | Observaciones |
|---|---|---|---|
| Entregable 1 (DOO) | 1-2026 | ✅ Entregado | Estructura seria. JWT. Monolito validado por profesor DOO. |
| Fase 0 — Multi-proyecto (DOO) | 1-2026 | ✅ Completada | 4 proyectos: Domain, Application, Infrastructure y API. |
| Fase 1 — Dominio completo (DOO) | 1-2026 | ✅ Completada | 12 entidades en PostgreSQL. 3 migraciones. |
| Fix JWT 401 (DOO) | 1-2026 | ✅ Resuelto | IdentityModel 8.9.0 fijado explícitamente. |
| Estrategia por capas + Capa 1 cerrada (Software 2) | 2-2026 | ✅ Cerrado en v10 | Problema aterrizado, cliente ideal, alcance explícito, diferenciadores, contexto de negocio, cambio a Silo, JWT y WhatsApp abiertos como decisiones de Capa 3 |
| Capa 2 — Drivers arquitectónicos | 2-2026 | 🔄 Siguiente | Limpieza del tradeoff con los 13 hallazgos del Punto 3 de la bitácora |
| Capa 3 — 10 ADRs | 2-2026 | ⏳ Pendiente | Uno por chat, con método de investigación honesta |
| Capa 4 — Diseño detallado | 2-2026 | ⏳ Pendiente | 4+1 vistas, ajustes al diagrama de clases |
| Capa 5 — Implementación y línea base | 2-2026 | ⏳ Pendiente | Migración de código, CI/CD, línea base |

---

*Fin del documento CONTEXTO_FLOWRIDE_v10.md*
