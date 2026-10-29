**UNIVERSIDAD CATÓLICA DE ORIENTE**

**Facultad de Ingenierías**

**Programa de Ingeniería de Sistemas**

**INGENIERÍA DEL SOFTWARE 2**

**Semestre 2-2026**

**PROYECTO EXTRACLASE — FLOWRIDE**

Plataforma SaaS de Gestión de Transporte Corporativo Interno

**ENTREGABLE — SESIÓN 5 (19 de agosto de 2026)**

**Drivers Arquitectónicos: Atributos de Calidad**

**Especificación de Requisitos Funcionales**

**Filtro de Significancia Arquitectónica**

**AUTORES:**

**Sebastián Zapata Naranjo**

**Andrés David Ortiz Mejía**

**Jhonny Estiven Benítez Caro**

**DOCENTE:**

**Wider Farid Sánchez Garzón**

**Rionegro, Antioquia**

Septiembre de 2026

**Contenido**

Qué son los atributos de calidad 3

Listado de atributos de calidad aplicables a FlowRide 4

Traducción a lenguaje de usuario final 7

Escenarios de calidad para validar con el usuario 11

Especificación de Requisitos Funcionales 18

Módulo 1 — Autenticación y Autorización 18

Módulo 2 — Gestión de Operadores / Tenants 20

Módulo 3 — Gestión de Usuarios 21

Módulo 4 — Gestión de Conductores 22

Módulo 5 — Gestión de Itinerarios y Solicitudes 23

Módulo 6 — Motor de Asignación 24

Módulo 7 — Integración con WhatsApp 27

Módulo 8 — Multi-tenancy (modelo Silo) 30

Módulo 9 — Configuración Avanzada 31

Capacidades V1 complementarias 32

Catálogo abreviado — Requisitos no significativos para la arquitectura 34

Filtro de significancia arquitectónica 38

Tabla A — Requisitos funcionales SIGNIFICATIVOS para la arquitectura 38

Tabla B — Requisitos funcionales NO significativos (dependencias) 41

# Qué son los atributos de calidad

Un atributo de calidad es una propiedad medible u observable de un sistema que describe qué tan bien cumple sus funciones, no qué hace. La bibliografía base del curso (Bass, Clements y Kazman — Software Architecture in Practice) distingue los requisitos funcionales de los atributos de calidad. En FlowRide, por ejemplo, una solicitud de transporte es una funcionalidad; su comportamiento esperado se mide con objetivos como P95 \< 500 ms y P99 \< 1,5 s para operaciones CRUD, considerando un techo de 200 usuarios concurrentes por instancia. La disponibilidad también se expresa por componente: 99,9% para el canal WhatsApp, 99,5% para el motor de asignación y operaciones críticas, y 99% para el panel administrativo.

1\. No son lo mismo que los RNF sueltos. Un RNF crudo dice 'el sistema debe ser rápido'. Un atributo de calidad se expresa como escenario con seis partes: fuente del estímulo → estímulo → artefacto → entorno → respuesta → medida de respuesta. En FlowRide, cada medida se define con un umbral verificable y coherente con el contexto de negocio vigente: hasta 200 usuarios simultáneos por instancia, métricas P95/P99 por tipo de operación y SLA diferenciados por componente.

2\. Son uno de los cuatro drivers arquitectónicos. Los otros tres son motivadores de negocio, funcionalidades primarias y restricciones técnicas y de negocio. En conjunto justifican las decisiones arquitectónicas del sistema, que se documentan formalmente mediante ADR cuando corresponde.

**3. Se priorizan, no se maximizan todos.** Es imposible optimizar todos al mismo tiempo — mejorar seguridad suele costar rendimiento, mejorar disponibilidad suele costar consistencia. El taller QAW (Quality Attribute Workshop, lectura obligatoria para el 12 ago) sirve justo para eso: identificar cuáles son críticos, cuáles secundarios y cuáles se pueden ceder

**SLA = Service Level Agreement — Acuerdo de Nivel de Servicio.**

Es un **contrato formal** entre el proveedor de un servicio y su cliente, donde se compromete por escrito qué nivel de servicio va a entregar. Y muy importante: **qué pasa si no lo cumple** — típicamente créditos, reembolsos o penalizaciones económicas

**P95 = percentil 95.** Es una forma de medir tiempos de respuesta que reemplaza al "promedio", porque el promedio miente cuando se trata de rendimiento

# Listado de atributos de calidad aplicables a FlowRide

**Atributos de calidad — Aplicabilidad a FlowRide**

| *Atributo*                                     | *¿Aplica a FlowRide?*                                                                                 | *Explícito / Implícito / Mixto* | *¿Asegurar o analizar con usuario?*                                                                                                                                                                          |
|------------------------------------------------|-------------------------------------------------------------------------------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Rendimiento (Performance)*                    | *Sí — crítico*                                                                                        | *Explícito*                     | *Analizar con usuario — el P95 y los endpoints críticos los define el negocio*                                                                                                                               |
| *Escalabilidad (Scalability)*                  | *Sí — crítico*                                                                                        | *Explícito*                     | Analizar con el negocio — el dimensionamiento vigente es hasta 200 usuarios concurrentes por instancia; la escala de plataforma crece por nuevos tenants Silo.                                               |
| *Seguridad (Security)*                         | *Sí — crítico*                                                                                        | *Explícito*                     | Asegurar — autenticación y autorización por roles son obligatorias; el proveedor de identidad se formaliza en ADR-03 mediante comparación de alternativas de proveedor de identidad.                         |
| *Disponibilidad (Availability)*                | *Sí — crítico*                                                                                        | *Explícito*                     | Asegurar según criticidad — SLA vigente por tiers: WhatsApp 99,9%; motor y operaciones críticas 99,5%; panel administrativo 99%.                                                                             |
| *Mantenibilidad (Maintainability)*             | *Sí — crítico*                                                                                        | *Explícito*                     | *Asegurar — decisión interna de arquitectura (multi-proyecto, interfaces)*                                                                                                                                   |
| *Interoperabilidad (Interoperability)*         | *Sí*                                                                                                  | *Explícito*                     | *Asegurar — protocolos técnicos (REST, JSON, Webhook)*                                                                                                                                                       |
| Aislamiento entre tenants (Isolation)          | *Sí — diferenciador*                                                                                  | *Mixto*                         | Asegurar — un tenant = un operador de transporte; el aislamiento entre tenants es físico mediante modelo Silo. Los detalles operativos de aprovisionamiento, migraciones y respaldo se documentan en ADR-02. |
| *Modificabilidad (Modifiability)*              | *Sí*                                                                                                  | *Implícito*                     | *Asegurar — interfaces desacoplan capas por decisión del equipo*                                                                                                                                             |
| *Testabilidad (Testability)*                   | *Sí*                                                                                                  | *Implícito*                     | *Asegurar — decisión interna del equipo*                                                                                                                                                                     |
| *Observabilidad (Observability)*               | *Sí*                                                                                                  | *Mixto*                         | *Analizar con usuario — métricas de negocio y SLAs las define el cliente*                                                                                                                                    |
| *Auditabilidad / Trazabilidad*                 | *Sí*                                                                                                  | *Mixto*                         | *Analizar con usuario — eventos auditables, retención y accesos los define el cliente*                                                                                                                       |
| *Desplegabilidad (Deployability)*              | *Sí — parcial*                                                                                        | *Implícito*                     | *Analizar con usuario — ventanas de mantenimiento las define el cliente*                                                                                                                                     |
| *Confiabilidad (Reliability)*                  | *Sí — parcial*                                                                                        | *Implícito*                     | *Analizar con usuario — tolerancia a fallos depende del negocio*                                                                                                                                             |
| *Reusabilidad (Reusability)*                   | *Sí — parcial*                                                                                        | *Implícito*                     | *Asegurar — decisión interna del equipo*                                                                                                                                                                     |
| *Usabilidad (Usability)*                       | Sí — WhatsApp para pasajero/conductor y Panel Web para roles administrativos                          | *Implícito*                     | Analizar con usuarios — WhatsApp reduce fricción de adopción; el Panel Web debe soportar la operación administrativa y logística.                                                                            |
| *Portabilidad (Portability)*                   | *No —* La portabilidad entre entornos de despliegue (Docker, cloud) sí se cubre bajo Desplegabilidad. | *No aplica*                     | *—*                                                                                                                                                                                                          |
| *Compatibilidad (Compatibility)*               | *No de forma prominente — integración externa se cubre en Interoperabilidad*                          | *No aplica*                     | *—*                                                                                                                                                                                                          |
| *Idoneidad funcional (Functional Suitability)* | *No como atributo diferenciador — es transversal a todos los sistemas*                                | *No aplica*                     | *—*                                                                                                                                                                                                          |

# Traducción a lenguaje de usuario final

Cada atributo explícito hay que poder explicárselo al usuario final con recomendaciones, metáforas o ejemplos. La idea es que si mañana un directivo de la operador cliente pregunta "¿qué es escalabilidad?", uno tenga una frase lista que no suene a jerga técnica.

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 33%" />
<col style="width: 33%" />
</colgroup>
<thead>
<tr class="header">
<th><em>Atributo</em></th>
<th><em>En lenguaje técnico</em></th>
<th><em>En lenguaje de usuario final</em></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><em>Rendimiento</em></td>
<td><em>Tiempos de respuesta de la API bajo carga concurrente.</em></td>
<td><em>"Cuando un operador consulta las solicitudes del día o un pasajero pide un traslado por WhatsApp, la respuesta debe llegar en menos de lo que tarda en pestañear. Nada de pantallas cargando ni mensajes que llegan tarde." Metáfora: como pedir un café en la barra — se lo entregan de una, no lo hacen esperar diez minutos.</em></td>
</tr>
<tr class="even">
<td><em>Escalabilidad</em></td>
<td><em>Capacidad de crecer horizontalmente para atender más usuarios sin degradar el servicio.</em></td>
<td>"Cada operador funciona en su propia instancia lógica. Cuando FlowRide incorpora nuevos operadores, agrega nuevas instancias Silo en lugar de cargar miles de usuarios sobre una sola." Metáfora: abrir nuevas sucursales independientes en vez de meter a todos los clientes en una sola oficina.</td>
</tr>
<tr class="odd">
<td><em>Seguridad</em></td>
<td>Autenticación y autorización por roles mediante un mecanismo de identidad basado en tokens y un proveedor de identidad desacoplado.</td>
<td><em>"Solo la persona correcta ve lo que le corresponde. El conductor no ve la agenda del gerente, el pasajero no ve los ingresos de la empresa, y nadie de afuera puede entrar sin credenciales válidas." Metáfora: como el edificio de una empresa — cada quien tiene su tarjeta y solo abre los pisos donde tiene permiso.</em></td>
</tr>
<tr class="even">
<td><em>Disponibilidad</em></td>
<td>Disponibilidad diferenciada por componente: 99,9% canal WhatsApp; 99,5% motor de asignación y operaciones críticas; 99% Panel Web.</td>
<td><em>"La plataforma está disponible cuando la operación la necesita. Si un turno arranca a las 4 a.m., el sistema tiene que estar arriba a las 4 a.m. — no hay 'volvemos en cinco minutos'." Metáfora: como la luz eléctrica en un hospital — se da por hecho que va a estar, y cuando no está, es noticia.</em></td>
</tr>
<tr class="odd">
<td><em>Mantenibilidad</em></td>
<td><em>Código modular con separación de responsabilidades que permite corregir errores e incorporar cambios con bajo esfuerzo.</em></td>
<td><p><em>"Cuando la empresa necesita un ajuste — nueva regla de asignación, nuevo reporte, nuevo tipo de usuario — el cambio se hace en semanas, no en meses. Y no se rompe lo que ya funcionaba." Metáfora: como una casa modular donde uno puede cambiar el mueble</em></p>
<p><em>de la cocina sin tumbar la pared del baño.</em></p></td>
</tr>
<tr class="even">
<td><em>Interoperabilidad</em></td>
<td>Capacidad de comunicarse con sistemas externos mediante APIs REST y webhooks. En V1 incluye WhatsApp y la fuente de telemetría GPS; las integraciones HRMS/nómina no pertenecen al alcance funcional actual.</td>
<td>"La plataforma habla con WhatsApp y recibe ubicación de los vehículos desde una fuente de telemetría. Otras integraciones, como nómina, solo se incorporan cuando exista un cliente que realmente las necesite."</td>
</tr>
<tr class="odd">
<td><em>Aislamiento entre tenants</em></td>
<td>Cada operador de transporte (tenant) dispone de aislamiento físico de datos mediante modelo Silo. empresaId identifica al cliente empresarial atendido dentro de la BD del operador; no separa tenants.</td>
<td>"Cada operador tiene su propio espacio de datos físicamente separado. Dentro de ese espacio puede atender varias aerolíneas u otros clientes empresariales sin mezclar información con otros operadores." Metáfora: edificios separados para cada operador, con oficinas internas para sus clientes.</td>
</tr>
<tr class="even">
<td><em>Modificabilidad</em></td>
<td><em>Capacidad de introducir cambios funcionales con impacto localizado gracias a la arquitectura modular.</em></td>
<td>"Si un operador necesita ajustar una regla de asignación, el cambio debe quedar localizado y no obligar a rehacer el resto del sistema." Metáfora: cambiar una pieza de un LEGO sin desarmar todo el modelo.</td>
</tr>
<tr class="odd">
<td><em>Testabilidad</em></td>
<td><em>El código está estructurado en capas con interfaces que permiten pruebas unitarias y de integración automatizadas.</em></td>
<td><em>"Antes de que cualquier cambio llegue a los usuarios, el sistema se auto-prueba en decenas de escenarios. Si algo se rompe, se detecta antes de que un pasajero llame quejándose." Metáfora: como el chequeo pre-vuelo de un avión — pilotos y mecánicos revisan todo antes de despegar, no en pleno vuelo.</em></td>
</tr>
<tr class="even">
<td><em>Observabilidad</em></td>
<td><em>Registro estructurado de logs, métricas de operación y trazas para diagnóstico.</em></td>
<td><em>"Cuando algo no funciona bien, el equipo técnico puede ver exactamente qué pasó, cuándo pasó y por qué. No hay que adivinar. Y para lo que ya está funcionando, se ven métricas en tiempo real: viajes activos, conductores disponibles, solicitudes pendientes." Metáfora: como el tablero de un carro — velocímetro, medidor de gasolina, luz de aceite — todo visible sin abrir el capó.</em></td>
</tr>
<tr class="odd">
<td><em>Auditabilidad / Trazabilidad</em></td>
<td><em>Registro inmutable de acciones de usuarios administrativos y eventos críticos del sistema.</em></td>
<td><em>"Cada movimiento importante queda registrado: quién asignó qué viaje, cuándo se canceló una solicitud, quién modificó los datos de un conductor. Si alguien pregunta '¿qué pasó ayer con este viaje?', hay una respuesta clara y verificable." Metáfora: como las cámaras de un parqueadero — no están para vigilar, están para que si algo pasa, uno sepa qué pasó.</em></td>
</tr>
<tr class="even">
<td><em>Desplegabilidad</em></td>
<td><em>Capacidad de instalar y actualizar el sistema en distintos entornos con mínima intervención manual.</em></td>
<td><em>"Cuando sale una versión nueva, se pone en producción sin interrumpir la operación. El operador logístico no tiene que 'cerrar el sistema una hora' — la actualización es transparente." Metáfora: como cambiarle las llantas al carro sin parar el motor — la operación sigue corriendo.</em></td>
</tr>
<tr class="odd">
<td><em>Confiabilidad</em></td>
<td><em>El sistema se comporta de forma consistente y se recupera de fallos sin pérdida de información.</em></td>
<td><em>"Si algo falla — se cae la red, se apaga un servidor —, el sistema no pierde datos ni se queda en un estado raro. Retoma donde iba y sigue funcionando." Metáfora: como un cajero automático — si se corta la luz a mitad de una transacción, o el dinero sale o la plata sigue en la cuenta. Nunca se pierde a medias.</em></td>
</tr>
<tr class="even">
<td><em>Reusabilidad</em></td>
<td><em>Componentes del sistema (autenticación, módulo de asignación, capa de datos) diseñados para reutilizarse en nuevos módulos o productos.</em></td>
<td><em>"Lo que se construye una vez, sirve para muchas cosas. El módulo que autentica al pasajero es el mismo que autentica al conductor y al operador. No se hace tres veces la misma pieza." Metáfora: como los enchufes estándar de un edificio — todos los aparatos usan la misma toma, no toca fabricar un enchufe nuevo cada vez que se compra un electrodoméstico.</em></td>
</tr>
<tr class="odd">
<td>Usabilidad</td>
<td>Facilidad de uso de los dos canales funcionales: WhatsApp para pasajero/conductor y Panel Web para Administrador del operador y Operador logístico.</td>
<td>"El pasajero y el conductor usan WhatsApp, sin instalar una app nueva. El personal administrativo trabaja desde un panel diseñado para entender rápidamente el estado de la operación."</td>
</tr>
</tbody>
</table>

# Escenarios de calidad para validar con el usuario

- **Fuente** = quién o qué genera el estímulo (usuario, sistema externo, mantenimiento).

- **Estímulo** = el evento concreto que dispara la situación.

- **Artefacto** = la parte del sistema afectada.

- **Entorno** = en qué condiciones está el sistema cuando pasa (operación normal, pico, degradado).

- **Respuesta** = qué hace el sistema.

- **Medida de respuesta** = el número concreto, cuantitativo, que hace que se pueda validar.

**la medida de respuesta nunca puede ser cualitativa**. Nada de "rápido", "seguro", "estable". Siempre número, unidad y umbral.

Escenario 1 — Rendimiento

| Parte     | Contenido                                                            |
|-----------|----------------------------------------------------------------------|
| Fuente    | Operador logístico desde el Panel Web Administrativo                 |
| Estímulo  | Consulta el listado de solicitudes activas del día                   |
| Artefacto | API REST – endpoint GET /api/solicitudes?estado=activas              |
| Entorno   | Pico operativo de una instancia, con hasta 200 usuarios concurrentes |
| Respuesta | El sistema retorna el listado paginado con los datos completos       |
| Medida    | Para operaciones CRUD: P95 \< 500 ms y P99 \< 1,5 s                  |

Escenario 2 — Escalabilidad

| Parte     | Contenido                                                                                                                                                 |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Incorporación de nuevos operadores de transporte a la plataforma                                                                                          |
| Estímulo  | La plataforma crece de 5 tenants en V1 hacia 20 tenants en V2, manteniendo aislamiento Silo por operador                                                  |
| Artefacto | Plataforma FlowRide — aprovisionamiento de instancias por tenant y backend .NET                                                                           |
| Entorno   | Producción, múltiples tenants físicamente aislados                                                                                                        |
| Respuesta | El sistema incorpora nuevos tenants mediante nuevas instancias Silo sin aumentar la carga de una instancia existente por encima de su techo dimensionable |
| Medida    | Cada instancia mantiene hasta 200 usuarios concurrentes y los objetivos de rendimiento vigentes (CRUD P95 \< 500 ms, P99 \< 1,5 s)                        |

Escenario 3 - Seguridad

| Parte     | Contenido                                                                                                                                                               |
|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Atacante externo no autenticado                                                                                                                                         |
| Estímulo  | Intenta acceder a endpoints protegidos sin credenciales válidas, con token expirado, revocado o adulterado                                                              |
| Artefacto | Middleware de autenticación y autorización de la API REST                                                                                                               |
| Entorno   | Operación normal, sistema expuesto a internet                                                                                                                           |
| Respuesta | El sistema rechaza la petición, no expone información y registra el intento en el módulo de auditoría                                                                   |
| Medida    | 100% de peticiones sin credenciales válidas son rechazadas con HTTP 401/403 y no exponen datos protegidos; los intentos relevantes quedan registrados para trazabilidad |

Escenario 4 — Disponibilidad

| Parte     | Contenido                                                                                                                                                                       |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Falla no planeada de una instancia del backend o del motor de base de datos                                                                                                     |
| Estímulo  | Una instancia de la API deja de responder durante horario operativo                                                                                                             |
| Artefacto | Canal WhatsApp, motor de asignación/operaciones críticas y Panel Web, según el componente afectado                                                                              |
| Entorno   | Producción, día laboral en horario pico                                                                                                                                         |
| Respuesta | El sistema aplica la estrategia de continuidad correspondiente al componente y preserva la consistencia de las solicitudes y asignaciones en curso                              |
| Medida    | SLA objetivo: WhatsApp ≥ 99,9%; motor de asignación y operaciones críticas ≥ 99,5%; Panel Web ≥ 99%. Los mecanismos concretos de failover y despliegue se documentan en ADR-08. |

Escenario 5 — Mantenibilidad

| Parte     | Contenido                                                                                                                                 |
|-----------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Equipo de desarrollo interno                                                                                                              |
| Estímulo  | Se solicita agregar un nuevo módulo funcional (ejemplo: reportes ejecutivos por empresa)                                                  |
| Artefacto | Solución multi-proyecto (Domain, Application, Infrastructure, API)                                                                        |
| Entorno   | Ambiente de desarrollo                                                                                                                    |
| Respuesta | El módulo se agrega respetando la estructura por capas, sin modificar código existente en Domain y sin romper tests actuales              |
| Medida    | Módulo nuevo integrado en ≤ 5 días-desarrollador; 0 tests existentes rotos tras la integración; cobertura de tests del nuevo módulo ≥ 70% |

Escenario 6 — Aislamiento entre tenants

| Parte     | Contenido                                                                                                                                 |
|-----------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Usuario autenticado perteneciente al operador A                                                                                           |
| Estímulo  | Intenta acceder a información almacenada en el tenant del operador B                                                                      |
| Artefacto | Enrutamiento de tenant + capa de acceso a datos + bases físicas separadas por operador (modelo Silo)                                      |
| Entorno   | Operación normal, sistema multi-tenant en producción                                                                                      |
| Respuesta | El sistema impide que la sesión del operador A se enrute o autorice contra la persistencia del operador B y registra el intento relevante |
| Medida    | 0 fugas de datos entre tenants; 100% de pruebas automatizadas de aislamiento entre operadores deben bloquear accesos cruzados             |

Escenario 7 — Interoperabilidad

| Parte     | Contenido                                                                                                                                  |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Plataforma WhatsApp Business mediante el adaptador de integración definido para FlowRide                                                   |
| Estímulo  | Envía un webhook HTTP con un mensaje entrante de un pasajero                                                                               |
| Artefacto | Endpoint webhook de integración WhatsApp                                                                                                   |
| Entorno   | Operación normal                                                                                                                           |
| Respuesta | El sistema valida el evento, identifica al usuario, procesa o encola el mensaje y responde conforme al contrato del proveedor seleccionado |
| Medida    | 100% de webhooks válidos cumplen el contrato de integración y son reconocidos o encolados en ≤ 500 ms bajo operación normal                |

Escenario 8 — Modificabilidad

| Parte     | Contenido                                                                                                                            |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Administrador del operador                                                                                                           |
| Estímulo  | Solicita incorporar una nueva regla de asignación (ejemplo: balancear carga entre conductores o minimizar cambios de turno)          |
| Artefacto | Motor de Asignación (módulo interno)                                                                                                 |
| Entorno   | Ambiente de desarrollo                                                                                                               |
| Respuesta | Se agrega la nueva regla como implementación de una interfaz de estrategia, sin modificar reglas existentes                          |
| Medida    | Regla nueva agregada en ≤ 2 días-desarrollador; 0 modificaciones a las reglas ya activas; cobertura de tests de la nueva regla ≥ 80% |

Escenario 9 — Testabilidad

| Parte     | Contenido                                                                              |
|-----------|----------------------------------------------------------------------------------------|
| Fuente    | Pipeline de integración continua (CI)                                                  |
| Estímulo  | Se ejecuta la suite completa de tests unitarios y de integración al hacer merge a main |
| Artefacto | Proyectos FlowRide.Domain, FlowRide.Application, FlowRide.Infrastructure, FlowRide.API |
| Entorno   | Ambiente de CI                                                                         |
| Respuesta | La suite corre de forma automatizada, sin dependencias humanas, y reporta cobertura    |
| Medida    | Ejecución completa en ≤ 5 min; cobertura global ≥ 70%; cobertura de Domain ≥ 85%       |

Escenario 10 — Observabilidad

| Parte     | Contenido                                                                                                                                                             |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Operador logístico o equipo técnico                                                                                                                                   |
| Estímulo  | Se reporta que un viaje "no se asignó correctamente" hace 30 minutos                                                                                                  |
| Artefacto | Módulo de Logging + Módulo de Auditoría                                                                                                                               |
| Entorno   | Producción                                                                                                                                                            |
| Respuesta | El sistema permite trazar el flujo completo de la solicitud: recepción, evaluación de reglas, decisión y notificación                                                 |
| Medida    | 100% de eventos críticos con log estructurado que incluya traceId, identificador de tenant/operador y timestamp; objetivo operativo: reconstruir el flujo en ≤ 10 min |

Escenario 11 — Auditabilidad / Trazabilidad

| Parte     | Contenido                                                                                                                                            |
|-----------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Administrador del operador                                                                                                                           |
| Estímulo  | Solicita el registro histórico de quién modificó la asignación de un viaje específico hace 6 meses                                                   |
| Artefacto | BD flowride_audit                                                                                                                                    |
| Entorno   | Operación normal                                                                                                                                     |
| Respuesta | El sistema retorna el histórico completo con usuario, acción, fecha, valor anterior y valor nuevo                                                    |
| Medida    | Retención de auditoría legal: 5 años; consulta del histórico objetivo ≤ 5 s; 100% de acciones sensibles definidas como auditables quedan registradas |

Escenario 12 — Desplegabilidad

| Parte     | Contenido                                                                                                |
|-----------|----------------------------------------------------------------------------------------------------------|
| Fuente    | Equipo de operaciones                                                                                    |
| Estímulo  | Se libera una nueva versión del backend con corrección de un bug crítico                                 |
| Artefacto | Backend .NET desplegado en producción                                                                    |
| Entorno   | Producción, horario laboral                                                                              |
| Respuesta | La nueva versión se despliega mediante rolling update sin cortar el servicio                             |
| Medida    | Despliegue completo en ≤ 15 min; downtime percibido por el usuario = 0 s; rollback disponible en ≤ 5 min |

Escenario 13 — Confiabilidad

| Parte     | Contenido                                                                                                                 |
|-----------|---------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Fallo transitorio de red entre la API y PostgreSQL                                                                        |
| Estímulo  | Una transacción de asignación de conductor pierde la conexión a mitad de operación                                        |
| Artefacto | Capa de persistencia (FlowRide.Infrastructure) + módulo de asignación                                                     |
| Entorno   | Producción                                                                                                                |
| Respuesta | El sistema revierte la transacción, no deja al conductor en estado inconsistente y notifica al operador para reintentar   |
| Medida    | 0 asignaciones en estado inconsistente tras el fallo; 100% de transacciones ACID cumplidas; reintento automático en ≤ 3 s |

Escenario 14 — Reusabilidad

| Parte     | Contenido                                                                                                                                                    |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Fuente    | Equipo de desarrollo interno                                                                                                                                 |
| Estímulo  | Se debe implementar un nuevo tipo de usuario (ejemplo: Supervisor de Flota)                                                                                  |
| Artefacto | Módulo de Autenticación y Autorización + modelo de roles/usuarios                                                                                            |
| Entorno   | Ambiente de desarrollo                                                                                                                                       |
| Respuesta | El nuevo tipo de usuario reutiliza el flujo y las abstracciones de autenticación/autorización sin duplicar lógica ni acoplarse al proveedor concreto         |
| Medida    | ≥ 80% del flujo de autenticación/autorización reutilizado sin duplicación; incorporación de un nuevo rol sin modificar el mecanismo central de autenticación |

Escenario 15 — Usabilidad

| Parte     | Contenido                                                                                                        |
|-----------|------------------------------------------------------------------------------------------------------------------|
| Fuente    | Pasajero sin capacitación previa                                                                                 |
| Estímulo  | Envía por primera vez una solicitud de transporte por WhatsApp                                                   |
| Artefacto | Bot de WhatsApp + flujo conversacional                                                                           |
| Entorno   | Operación normal                                                                                                 |
| Respuesta | El bot guía al pasajero paso a paso y completa la solicitud sin intervención humana                              |
| Medida    | ≥ 90% de solicitudes completadas exitosamente por usuarios nuevos sin ayuda; tiempo promedio de solicitud ≤ 90 s |

## **Especificación de Requisitos Funcionales** 

# Especificación de Requisitos Funcionales

Esta sección consolida la especificación funcional vigente de FlowRide. El catálogo queda cerrado en 56 requisitos funcionales, alineados con el alcance actual del producto, el modelo B2B2B, el aislamiento Silo por operador, WhatsApp como canal de pasajero y conductor, el motor de asignación programado por reglas, la calificación del servicio y la telemetría GPS para monitoreo y detección de desviación de ETA.

Los requisitos arquitectónicamente significativos se identifican por su valor de negocio (VN) o por representar un reto tecnológico (RT). El catálogo evita duplicidades funcionales y excluye capacidades que no pertenecen al alcance actual del producto, de modo que la numeración RF-01 a RF-56 constituye la línea base funcional de este documento.

## **Módulo 1 — Autenticación y Autorización**

RF-01 — Autenticación mediante mecanismo de identidad basado en token

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                                                          |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Todos los usuarios del sistema (Pasajero, Conductor, Administrador del operador, Operador logístico)                                                                                                                                                                                                   |
| Descripción     | El sistema deberá autenticar y validar la identidad de los usuarios mediante un mecanismo basado en tokens y un proveedor de identidad desacoplado. El sistema usa un proveedor de identidad desacoplado y tokens de acceso estándar; la comparación y selección del proveedor se documenta en ADR-03. |
| Entradas        | Correo electrónico y contraseña del usuario                                                                                                                                                                                                                                                            |
| Proceso         | Valida credenciales y estado del usuario mediante el proveedor de identidad configurado y obtiene/emite un token de acceso con los atributos mínimos necesarios para autorización, sin usar empresaId como mecanismo de aislamiento entre tenants.                                                     |
| Salidas         | Token de acceso válido + información básica del usuario y rol                                                                                                                                                                                                                                          |
| Precondiciones  | El usuario debe estar registrado y activo en el sistema                                                                                                                                                                                                                                                |
| Postcondiciones | El usuario puede realizar peticiones autenticadas mientras su sesión/token sea válido según la política de sesión y expiración configurada                                                                                                                                                             |
| Prioridad       | Alta                                                                                                                                                                                                                                                                                                   |

**RF-02 — Autorización basada en roles**

| **Campo**       | **Contenido**                                                                                                                                                                        |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Todos los usuarios autenticados                                                                                                                                                      |
| Descripción     | El sistema deberá validar en cada petición a endpoints protegidos que el usuario posea el rol requerido para ejecutar la acción, rechazando la petición si no cumple.                |
| Entradas        | Token de acceso válido en el encabezado Authorization                                                                                                                                |
| Proceso         | Valida el rol y permisos requeridos para la operación mediante el mecanismo de autorización implementado por la aplicación, manteniendo desacople respecto al proveedor de identidad |
| Salidas         | Respuesta HTTP 200 (autorizado) o HTTP 403 (no autorizado)                                                                                                                           |
| Precondiciones  | El usuario debe estar autenticado con un token válido y no expirado                                                                                                                  |
| Postcondiciones | Solo los usuarios con el rol adecuado pueden ejecutar cada acción del sistema                                                                                                        |
| Prioridad       | Alta                                                                                                                                                                                 |

## Módulo 2 — Gestión de Operadores / Tenants

RF-03 — Registro de operador de transporte como tenant

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                             |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador técnico / proceso de aprovisionamiento SaaS                                                                                                                                                                                                                 |
| Descripción     | El sistema deberá permitir registrar un nuevo operador de transporte como tenant de FlowRide y aprovisionar su entorno de datos aislado físicamente mediante el modelo Silo.                                                                                              |
| Entradas        | Nombre o razón social del operador, NIT, correo de contacto y datos mínimos de aprovisionamiento                                                                                                                                                                          |
| Proceso         | Valida la identidad del operador, crea el identificador del tenant y dispara el aprovisionamiento de su persistencia aislada. El aprovisionamiento crea la persistencia aislada del tenant; migraciones y respaldos siguen la estrategia operativa documentada en ADR-02. |
| Salidas         | Tenant del operador registrado y entorno Silo preparado para configuración                                                                                                                                                                                                |
| Precondiciones  | El operador no debe existir previamente como tenant                                                                                                                                                                                                                       |
| Postcondiciones | El operador queda habilitado para configurar usuarios, conductores, vehículos, reglas y clientes empresariales dentro de su propia instancia                                                                                                                              |
| Prioridad       | Alta                                                                                                                                                                                                                                                                      |

RF-04 — Configuración inicial del tenant

| **Campo**       | **Contenido**                                                                                                                                                                       |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador del operador                                                                                                                                                          |
| Descripción     | El sistema deberá permitir configurar los parámetros base de operación del tenant: usuarios administrativos, recursos operativos y parámetros necesarios para iniciar la operación. |
| Entradas        | Datos de usuarios administrativos, recursos y parámetros operativos del tenant                                                                                                      |
| Proceso         | Persiste la configuración dentro de la instancia Silo del operador                                                                                                                  |
| Salidas         | Confirmación de configuración aplicada                                                                                                                                              |
| Precondiciones  | La empresa debe estar previamente registrada (RF-03)                                                                                                                                |
| Postcondiciones | El tenant puede operar con sus reglas y recursos sin afectar otros operadores                                                                                                       |
| Prioridad       | Alta                                                                                                                                                                                |

## **Módulo 3 — Gestión de Usuarios**

RF-05 — Creación de usuarios por tenant

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                                  |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador del operador                                                                                                                                                                                                                                                     |
| Descripción     | El sistema deberá permitir al administrador crear usuarios del tenant (pasajeros, conductores, operadores logísticos y administradores) con el rol y asociación operacional correspondiente.                                                                                   |
| Entradas        | Nombre, correo, teléfono, rol, código interno del usuario                                                                                                                                                                                                                      |
| Proceso         | Valida la unicidad aplicable dentro del tenant y persiste el usuario en la instancia del operador. La gestión de credenciales se delega al proveedor de identidad; la aplicación no almacena contraseñas en texto plano ni acopla el dominio a un emisor específico de tokens. |
| Salidas         | Usuario creado con su identificador único y credenciales para primer login                                                                                                                                                                                                     |
| Precondiciones  | El administrador debe estar autenticado y autorizado dentro del tenant                                                                                                                                                                                                         |
| Postcondiciones | El usuario puede autenticarse y operar según su rol dentro del tenant                                                                                                                                                                                                          |
| Prioridad       | Alta                                                                                                                                                                                                                                                                           |

**RF-06 — Asignación de roles a usuarios**

| **Campo**       | **Contenido**                                                                                                                                                |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador del operador                                                                                                                                   |
| Descripción     | El sistema deberá permitir asignar o modificar el rol de un usuario existente dentro de la empresa (Pasajero, Conductor, Operador Logístico, Administrador). |
| Entradas        | Identificador del usuario, nuevo rol                                                                                                                         |
| Proceso         | Valida que el rol exista en el catálogo, actualiza el rolId del usuario y registra la acción en auditoría                                                    |
| Salidas         | Usuario con rol actualizado                                                                                                                                  |
| Precondiciones  | El usuario y el rol deben existir dentro del mismo tenant; el administrador debe estar autorizado                                                            |
| Postcondiciones | Los permisos del usuario se actualizan de acuerdo con la política de sesión, roles y revocación configurada                                                  |
| Prioridad       | Alta                                                                                                                                                         |

## **Módulo 4 — Gestión de Conductores**

**RF-07 — Consulta de disponibilidad de conductores**

| **Campo**       | **Contenido**                                                                                                                                                                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Motor de Asignación (proceso interno) + Operador Logístico                                                                                                                                                                                      |
| Descripción     | El sistema deberá permitir consultar los conductores disponibles para una franja o servicio dentro del tenant, filtrando por estado y criterios operativos programados. La ubicación GPS no se utiliza para reasignación dinámica por cercanía. |
| Entradas        | Identificador del tenant y filtros operativos (estado, turno, vehículo asignado u otros definidos por reglas)                                                                                                                                   |
| Proceso         | Consulta conductores activos y disponibles dentro de la instancia del operador, respetando las reglas programadas de disponibilidad                                                                                                             |
| Salidas         | Listado de conductores disponibles con sus datos operativos                                                                                                                                                                                     |
| Precondiciones  | Debe existir al menos un conductor registrado y activo en el tenant                                                                                                                                                                             |
| Postcondiciones | La consulta no modifica el estado del sistema                                                                                                                                                                                                   |
| Prioridad       | Media                                                                                                                                                                                                                                           |

## **Módulo 5 — Gestión de Itinerarios y Solicitudes**

**RF-08 — Registro de itinerarios**

| **Campo**       | **Contenido**                                                                                                                                                                     |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Operador Logístico                                                                                                                                                                |
| Descripción     | El sistema deberá permitir registrar itinerarios de transporte que definen los servicios recurrentes o programados de la empresa (origen, destino, horarios, código de servicio). |
| Entradas        | Código de servicio, fecha, origen, destino, hora de salida, hora de llegada                                                                                                       |
| Proceso         | Valida los datos y persiste el itinerario dentro del tenant. Cuando corresponda, empresaId identifica al cliente empresarial atendido por el operador, no al tenant.              |
| Salidas         | Itinerario registrado con identificador único                                                                                                                                     |
| Precondiciones  | El operador debe estar autenticado y tener rol autorizado                                                                                                                         |
| Postcondiciones | El itinerario queda disponible como referencia para solicitudes de transporte                                                                                                     |
| Prioridad       | Alta                                                                                                                                                                              |

**RF-09 — Recepción de solicitudes de transporte**

| **Campo**       | **Contenido**                                                                                                                                                                            |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Pasajero (vía WhatsApp)                                                                                                                                                                  |
| Descripción     | El sistema deberá recibir solicitudes de transporte de los pasajeros, validarlas y registrarlas en la cola de asignación.                                                                |
| Entradas        | Identificador del pasajero, itinerario asociado (opcional), dirección origen, dirección destino, fecha y hora, indicador de urgencia y atributo esPersonal cuando aplique                |
| Proceso         | Valida datos mínimos, crea la solicitud con estado Pendiente dentro del tenant del operador y registra empresaId solo como referencia al cliente empresarial atendido cuando corresponda |
| Salidas         | Solicitud registrada con identificador único y confirmación al pasajero                                                                                                                  |
| Precondiciones  | El pasajero debe estar identificado dentro de un tenant activo                                                                                                                           |
| Postcondiciones | La solicitud queda encolada para asignación automática                                                                                                                                   |
| Prioridad       | Alta                                                                                                                                                                                     |

## **Módulo 6 — Motor de Asignación**

**RF-10 — Asignación automática de conductores**

| **Campo**       | **Contenido**                                                                                                                                                            |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Motor de Asignación (proceso interno disparado por nueva solicitud)                                                                                                      |
| Descripción     | El sistema deberá asignar automáticamente un conductor y un vehículo a cada solicitud pendiente, aplicando las reglas de negocio programadas y configurables del tenant. |
| Entradas        | Solicitud de transporte en estado Pendiente                                                                                                                              |
| Proceso         | Recupera conductores disponibles, aplica las reglas de asignación configuradas por el tenant, selecciona el mejor candidato y crea la asignación asociada                |
| Salidas         | Asignación creada con conductor y vehículo; solicitud actualizada a estado Asignada                                                                                      |
| Precondiciones  | Debe existir al menos un conductor disponible y las reglas del tenant deben estar configuradas                                                                           |
| Postcondiciones | Se notifica al conductor asignado (RF-17); la solicitud queda en flujo operativo                                                                                         |
| Prioridad       | Alta                                                                                                                                                                     |

**RF-11 — Aplicación de reglas de negocio para la asignación**

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                                                                                                |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Motor de Asignación                                                                                                                                                                                                                                                                                                                          |
| Descripción     | El sistema deberá aplicar reglas de negocio configurables por tenant, tales como disponibilidad programada, minimización de cambios de turno, balance de carga, distancia del servicio origen-destino y restricciones regulatorias. La telemetría GPS se usa para monitoreo y desviación de ETA, no para reasignación dinámica por cercanía. |
| Entradas        | Solicitud de transporte, configuración de reglas del tenant y conductores disponibles                                                                                                                                                                                                                                                        |
| Proceso         | Ejecuta las reglas activas, filtra y pondera candidatos según la configuración del operador y registra las razones de la decisión para trazabilidad                                                                                                                                                                                          |
| Salidas         | Conductor seleccionado o notificación de imposibilidad de asignación                                                                                                                                                                                                                                                                         |
| Precondiciones  | Las reglas deben estar configuradas por el Administrador del operador                                                                                                                                                                                                                                                                        |
| Postcondiciones | La decisión y las reglas aplicadas quedan disponibles para auditoría y trazabilidad                                                                                                                                                                                                                                                          |
| Prioridad       | Alta                                                                                                                                                                                                                                                                                                                                         |

**RF-12 — Reasignación manual de conductores**

| **Campo**       | **Contenido**                                                                                                                                                                                            |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Operador Logístico                                                                                                                                                                                       |
| Descripción     | El sistema deberá permitir al Operador logístico reasignar manualmente una asignación cuando exista una contingencia operativa, por ejemplo, cuando un conductor abandone su turno o quede indisponible. |
| Entradas        | Identificador de la asignación, nuevo conductor y vehículo                                                                                                                                               |
| Proceso         | Valida disponibilidad del nuevo conductor, actualiza la asignación, notifica a ambos conductores (el liberado y el nuevo) y registra la acción en auditoría                                              |
| Salidas         | Asignación actualizada; notificaciones enviadas                                                                                                                                                          |
| Precondiciones  | La asignación debe existir y el operador debe pertenecer al mismo tenant                                                                                                                                 |
| Postcondiciones | La nueva asignación queda vigente; la anterior queda registrada en histórico                                                                                                                             |
| Prioridad       | Alta                                                                                                                                                                                                     |

**RF-13 — Priorización de solicitudes urgentes**

| **Campo**       | **Contenido**                                                                                                                                                                                                       |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Motor de Asignación                                                                                                                                                                                                 |
| Descripción     | El sistema deberá priorizar solicitudes marcadas como urgentes o críticas de acuerdo con reglas configuradas por el tenant, sin convertir el motor en un esquema de competencia dinámica por proximidad.            |
| Entradas        | Solicitud con marca urgente = true                                                                                                                                                                                  |
| Proceso         | Prioriza la solicitud dentro de la lógica programada del motor y, cuando no exista capacidad suficiente, genera una situación de excepción para intervención del Operador logístico conforme a las reglas vigentes. |
| Salidas         | Asignación priorizada o alerta de imposibilidad de asignación                                                                                                                                                       |
| Precondiciones  | La solicitud debe estar marcada como urgente y el tenant debe tener reglas de priorización activas                                                                                                                  |
| Postcondiciones | La decisión queda registrada para trazabilidad y, si requiere intervención, se notifica al Operador logístico                                                                                                       |
| Prioridad       | Media                                                                                                                                                                                                               |

## **Módulo 7 — Integración con WhatsApp**

**RF-14 — Recepción de mensajes desde WhatsApp**

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                                     |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Plataforma WhatsApp Business / proveedor seleccionado en ADR-0008                                                                                                                                                                                                                   |
| Descripción     | El sistema deberá recibir mensajes entrantes de pasajero y conductor vía webhook HTTP desde la plataforma WhatsApp Business. La integración se implementa mediante un adaptador desacoplado del proveedor, preservando el contrato funcional del webhook y del envío de mensajes. |
| Entradas        | Payload HTTP POST del proveedor seleccionado con contenido del mensaje, número de origen y metadatos                                                                                                                                                                              |
| Proceso         | Valida autenticidad del webhook según el proveedor seleccionado, identifica al usuario y despacha el mensaje al flujo conversacional correspondiente                                                                                                                              |
| Salidas         | Respuesta HTTP conforme al contrato del proveedor; mensaje disponible para procesamiento                                                                                                                                                                                          |
| Precondiciones  | El usuario debe tener número de WhatsApp asociado a su cuenta                                                                                                                                                                                                                     |
| Postcondiciones | El mensaje queda disponible para procesamiento por el bot conversacional                                                                                                                                                                                                          |
| Prioridad       | Alta                                                                                                                                                                                                                                                                              |

**RF-15 — Procesamiento de solicitudes enviadas por WhatsApp**

| **Campo**       | **Contenido**                                                                                                                                                              |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Bot conversacional interno                                                                                                                                                 |
| Descripción     | El sistema deberá interpretar los mensajes conversacionales del pasajero, mantener el estado de la conversación y convertirlos en solicitudes de transporte estructuradas. |
| Entradas        | Mensaje entrante + estado de conversación previo                                                                                                                           |
| Proceso         | Aplica la máquina de estados conversacional, extrae datos (origen, destino, fecha, urgencia), y cuando la solicitud está completa la persiste vía RF-09                    |
| Salidas         | Solicitud de transporte creada; respuesta al pasajero solicitando el siguiente dato o confirmando                                                                          |
| Precondiciones  | El mensaje debe provenir de un usuario identificado                                                                                                                        |
| Postcondiciones | La solicitud queda registrada o la conversación continúa esperando más datos                                                                                               |
| Prioridad       | Alta                                                                                                                                                                       |

**RF-16 — Envío de confirmaciones al usuario por WhatsApp**

| **Campo**       | **Contenido**                                                                                                                                 |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Servicio de Notificaciones                                                                                                                    |
| Descripción     | El sistema deberá enviar al pasajero una confirmación por WhatsApp cuando su solicitud haya sido registrada y cuando haya sido asignada.      |
| Entradas        | Evento de cambio de estado de solicitud (Pendiente → Asignada)                                                                                |
| Proceso         | Genera el mensaje de confirmación con los datos operativos pertinentes y lo envía mediante el adaptador de WhatsApp configurado para FlowRide |
| Salidas         | Mensaje entregado al pasajero                                                                                                                 |
| Precondiciones  | La solicitud debe estar en estado Asignada                                                                                                    |
| Postcondiciones | El pasajero conoce el estado y detalles de su solicitud                                                                                       |
| Prioridad       | Alta                                                                                                                                          |

**RF-17 — Envío de notificaciones al conductor asignado**

| **Campo**       | **Contenido**                                                                                                                                           |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Servicio de Notificaciones                                                                                                                              |
| Descripción     | El sistema deberá notificar al conductor por WhatsApp cuando se le asigne un nuevo servicio, incluyendo los datos del pasajero, origen, destino y hora. |
| Entradas        | Evento de creación de asignación                                                                                                                        |
| Proceso         | Genera el mensaje con los detalles del servicio y lo envía al número WhatsApp del conductor                                                             |
| Salidas         | Notificación entregada al conductor                                                                                                                     |
| Precondiciones  | El conductor debe tener número WhatsApp registrado                                                                                                      |
| Postcondiciones | El conductor puede aceptar o rechazar la asignación desde WhatsApp                                                                                      |
| Prioridad       | Alta                                                                                                                                                    |

**RF-18 — Actualización de estado del viaje desde WhatsApp**

| **Campo**       | **Contenido**                                                                                                                                                                                 |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Conductor (vía WhatsApp)                                                                                                                                                                      |
| Descripción     | El sistema deberá permitir al conductor actualizar el estado del viaje (inicio, finalización, incidente) enviando mensajes por WhatsApp que se sincronicen con el estado interno del sistema. |
| Entradas        | Mensaje del conductor con palabra clave (INICIO, FIN, INCIDENTE) + viajeId implícito por conversación                                                                                         |
| Proceso         | Interpreta el comando, valida que el conductor esté asignado al viaje, actualiza el estado y notifica al pasajero y al operador si corresponde                                                |
| Salidas         | Estado del viaje actualizado; notificación de confirmación al conductor                                                                                                                       |
| Precondiciones  | El conductor debe tener una asignación activa                                                                                                                                                 |
| Postcondiciones | El estado del viaje refleja la realidad operativa; se disparan las notificaciones correspondientes                                                                                            |
| Prioridad       | Alta                                                                                                                                                                                          |

## Módulo 8 — Multi-tenancy (modelo Silo)

RF-19 — Aislamiento físico entre tenants (modelo Silo)

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Sistema (transversal a todas las operaciones)                                                                                                                                                                                                                |
| Descripción     | El sistema deberá garantizar aislamiento físico entre operadores de transporte (tenants) mediante el modelo Silo, evitando que una sesión o proceso de un operador acceda a la persistencia de otro.                                                         |
| Entradas        | Petición autenticada con contexto de tenant/operador resuelto por la plataforma                                                                                                                                                                              |
| Proceso         | Resuelve el tenant de la petición y enruta el acceso únicamente hacia la instancia y persistencia Silo del operador autorizado. empresaId no actúa como filtro de aislamiento entre tenants; identifica al cliente empresarial dentro de la BD del operador. |
| Salidas         | Respuesta obtenida exclusivamente desde el entorno de datos del tenant autorizado                                                                                                                                                                            |
| Precondiciones  | El usuario debe estar autenticado y asociado a un tenant válido                                                                                                                                                                                              |
| Postcondiciones | No existe acceso cruzado entre persistencias de tenants; el aislamiento se verifica con pruebas automatizadas y controles de aprovisionamiento                                                                                                               |
| Prioridad       | Alta                                                                                                                                                                                                                                                         |

RF-20 — Configuración separada de usuarios y recursos por tenant

| **Campo**       | **Contenido**                                                                                                                                                                           |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador del operador                                                                                                                                                              |
| Descripción     | El sistema deberá permitir que cada tenant tenga su propia configuración de usuarios, conductores, vehículos, reglas de asignación y parámetros operativos dentro de su instancia Silo. |
| Entradas        | Datos de configuración específicos del tenant                                                                                                                                           |
| Proceso         | Persiste la configuración dentro del entorno del tenant, garantizando que los cambios no afecten instancias de otros operadores                                                         |
| Salidas         | Configuración aplicada exclusivamente al tenant del operador                                                                                                                            |
| Precondiciones  | El administrador debe estar autenticado y autorizado en el tenant                                                                                                                       |
| Postcondiciones | Cada tenant opera con su configuración independiente                                                                                                                                    |
| Prioridad       | Alta                                                                                                                                                                                    |

## **Módulo 9 — Configuración Avanzada**

**RF-21 — Configuración avanzada de reglas de asignación**

| **Campo**       | **Contenido**                                                                                                                                                                                                                                                                                                     |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Administrador del operador                                                                                                                                                                                                                                                                                        |
| Descripción     | El sistema deberá permitir definir, activar, desactivar y ordenar reglas de asignación aplicables al tenant (por ejemplo: disponibilidad programada, minimización de cambios de turno, balance de carga, distancia del servicio y restricciones regulatorias). No se usa cercanía GPS para reasignación dinámica. |
| Entradas        | Definición de reglas con parámetros, orden de aplicación y condiciones de activación                                                                                                                                                                                                                              |
| Proceso         | Valida la coherencia de las reglas, las persiste dentro del tenant y las pone a disposición del motor para solicitudes posteriores                                                                                                                                                                                |
| Salidas         | Reglas configuradas y activas para el tenant                                                                                                                                                                                                                                                                      |
| Precondiciones  | El administrador debe pertenecer al tenant que configura                                                                                                                                                                                                                                                          |
| Postcondiciones | El motor de asignación aplica las nuevas reglas a partir de la siguiente solicitud                                                                                                                                                                                                                                |
| Prioridad       | Media                                                                                                                                                                                                                                                                                                             |

## Capacidades V1 complementarias

Las siguientes capacidades forman parte del alcance V1 y completan la línea base funcional: RF-54 Calificación pasajero → conductor, RF-55 Ingesta de telemetría GPS y RF-56 Detección de desviación de ETA.

### RF-54 — Calificación del servicio

| Campo           | Contenido                                                                                                                           |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Pasajero vía WhatsApp                                                                                                               |
| Descripción     | El sistema deberá permitir que el pasajero califique al conductor al finalizar un Viaje, mediante un puntaje y comentario opcional. |
| Entradas        | viajeId, puntaje, comentario opcional                                                                                               |
| Proceso         | Valida que el viaje haya finalizado y que el pasajero esté asociado al servicio; registra la Calificacion vinculada al Viaje.       |
| Salidas         | Confirmación de calificación registrada                                                                                             |
| Precondiciones  | Viaje finalizado y pasajero autorizado para calificarlo                                                                             |
| Postcondiciones | La calificación queda disponible para reportes operacionales agregados por conductor, cliente empresarial u operador                |
| Prioridad       | Alta                                                                                                                                |

### RF-55 — Ingesta de telemetría GPS

| Campo           | Contenido                                                                                                                                                    |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Fuente de telemetría GPS configurada para el tenant                                                                                                          |
| Descripción     | El sistema deberá recibir y almacenar eventos de ubicación de los vehículos/conductores para viajes en curso.                                                |
| Entradas        | vehiculoId, viajeId cuando aplique, timestamp, latitud, longitud y metadatos de la fuente                                                                    |
| Proceso         | Valida el evento, identifica el tenant y el viaje, y persiste RegistroUbicacion según la estrategia de persistencia de telemetría documentada para FlowRide. |
| Salidas         | Evento de ubicación aceptado o rechazado con trazabilidad                                                                                                    |
| Precondiciones  | Fuente de telemetría autenticada/autorizada y vehículo identificado                                                                                          |
| Postcondiciones | La ubicación queda disponible para monitoreo y análisis de desviación de ETA, no para reasignación dinámica por cercanía                                     |
| Prioridad       | Alta                                                                                                                                                         |

### RF-56 — Detección de desviación de ETA

| Campo           | Contenido                                                                                                                                       |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| Actor           | Módulo de Telemetría / sistema                                                                                                                  |
| Descripción     | El sistema deberá detectar desviaciones relevantes de ETA durante viajes en curso a partir de la telemetría GPS y generar una alerta operativa. |
| Entradas        | Secuencia de RegistroUbicacion + datos programados del Viaje                                                                                    |
| Proceso         | Evalúa la ubicación y el progreso del viaje contra la ETA programada y los umbrales configurados para el servicio.                              |
| Salidas         | Estado/alerta de desviación para Administrador del operador y Operador logístico; notificación proactiva cuando corresponda                     |
| Precondiciones  | Viaje en curso con telemetría disponible                                                                                                        |
| Postcondiciones | La desviación queda registrada para monitoreo y trazabilidad                                                                                    |
| Prioridad       | Alta                                                                                                                                            |

## **Catálogo abreviado — Requisitos no significativos para la arquitectura**

Los siguientes requisitos completan la línea base funcional de FlowRide. Corresponden a operaciones necesarias del producto que no condicionan por sí solas la arquitectura, porque se resuelven mediante patrones conocidos, reutilizan decisiones transversales o tienen menor impacto arquitectónico. Los reportes operacionales básicos y la observabilidad técnica permanecen dentro del alcance.

**Autenticación y acceso (2):**

- RF-22 Cierre de sesión — permite al usuario cerrar su sesión activa e invalidar el contexto local.

- RF-23 Recuperación o restablecimiento de credenciales — permite al usuario recuperar el acceso mediante correo electrónico.

**Gestión de usuarios (2):**

- RF-24 Actualización de información de usuarios — permite modificar datos personales de los usuarios existentes.

- RF-25 Activación e inactivación de usuarios — permite al Administrador del operador habilitar o deshabilitar el acceso de un usuario.

**Gestión de conductores (3):**

- RF-26 Registro de conductores — permite dar de alta un nuevo conductor con sus datos operativos y licencia.

- RF-27 Actualización de datos de conductores — permite modificar información del conductor.

- RF-28 Activación e inactivación de conductores — permite habilitar o deshabilitar operativamente a un conductor.

**Gestión de itinerarios y solicitudes (4):**

- RF-29 Validación de datos mínimos de solicitud — verifica que la solicitud contenga los campos obligatorios.

- RF-30 Consulta del estado de solicitudes — permite al pasajero u operador ver el estado actual de una solicitud.

- RF-31 Historial de solicitudes y viajes — permite consultar el histórico de solicitudes y viajes por usuario o cliente empresarial.

- RF-32 Modificación o cancelación de solicitudes — permite cancelar o ajustar una solicitud conforme a las reglas del flujo operativo.

**Asignación de transporte (2):**

- RF-33 Visualización de asignaciones generadas — permite al operador ver las asignaciones creadas por el motor.

- RF-34 Actualización del estado de asignación — permite marcar una asignación como aceptada, rechazada, en curso o finalizada.

**Integración con WhatsApp (1):**

- RF-35 Envío de recordatorios automáticos — envía recordatorios programados de viajes próximos al pasajero y al conductor.

**Panel web administrativo (7):**

- RF-36 Visualización centralizada de solicitudes y viajes — muestra una vista unificada de la operación.

- RF-37 Monitoreo operativo de servicios en curso — permite ver en tiempo real los servicios activos.

- RF-38 Consulta de usuarios, conductores y asignaciones — permite consultar el catálogo operativo del tenant.

- RF-39 Intervención manual en la operación — permite al Operador logístico tomar control manual de un servicio cuando exista una contingencia.

- RF-40 Consulta de trazabilidad de cada viaje — muestra el histórico completo de eventos de un viaje.

- RF-41 Búsqueda y filtrado de información — permite filtrar y buscar en los listados operativos.

- RF-42 Visualización de indicadores básicos — muestra métricas operacionales simples de la operación diaria.

**Notificaciones (3):**

- RF-43 Notificación automática de cambios de estado — notifica al usuario cuando cambia el estado de su servicio.

- RF-44 Notificación de reasignación o novedades — informa al pasajero y conductor sobre cambios operativos relevantes.

- RF-45 Alertas operativas para administradores u operadores — notifica eventos que requieren atención.

**Reportes (4):**

- RF-46 Reporte de viajes realizados — genera reporte de viajes completados en un período.

- RF-47 Reporte de solicitudes atendidas y pendientes — muestra el estado agregado de solicitudes.

- RF-48 Reporte de conductores asignados — muestra actividad y carga de trabajo por conductor.

- RF-49 Reporte histórico de operación por cliente empresarial — genera un reporte agregado histórico dentro del tenant.

**Logging y monitoreo (4):**

- RF-50 Registro de eventos del sistema — registra eventos operativos para trazabilidad técnica.

- RF-51 Registro de errores y excepciones — registra errores técnicos con contexto para diagnóstico.

- RF-52 Seguimiento de acciones de usuarios administrativos — registra en auditoría las acciones sensibles.

- RF-53 Monitoreo técnico de disponibilidad del sistema — expone métricas de disponibilidad y salud del sistema.

Este filtro se aplica sobre la línea base funcional RF-01 a RF-56 definida en este documento. Una funcionalidad se considera arquitectónicamente significativa cuando aporta valor de negocio diferencial o introduce un reto tecnológico que condiciona decisiones de diseño, integración, persistencia, seguridad, disponibilidad o despliegue.

# Filtro de significancia arquitectónica

(recordatorio):

Una funcionalidad es significativa arquitectónicamente si cumple al menos una condición:

Valor de negocio (VN): es lo que diferencia a FlowRide; el negocio no existe sin esto.

Reto tecnológico (RT): hay que investigar o probar cómo hacerlo; no es un patrón conocido de escritorio.

Todo lo demás son dependencias: cosas necesarias, pero que no marcan la arquitectura porque son patrones CRUD estándar o features secundarias.

## Tabla A — Requisitos funcionales SIGNIFICATIVOS para la arquitectura

| **\#** | **Requisito funcional**                                       | **Módulo**                | **Tipo** | **Justificación**                                                                                                                                                                                                                             |
|--------|---------------------------------------------------------------|---------------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RF-01  | Autenticación mediante mecanismo de identidad basado en token | Autenticación             | RT       | Reto técnico vigente por selección de proveedor, estrategia de token, revocación y desacople. La autenticación se mantiene desacoplada del proveedor concreto; ADR-03 documenta la decisión tecnológica sin alterar este requisito funcional. |
| RF-02  | Autorización basada en roles                                  | Autenticación             | VN       | Los cuatro roles operativos (Pasajero, Conductor, Administrador del operador, Operador logístico) son parte del modelo específico de transporte corporativo interno.                                                                          |
| RF-03  | Registro de operador de transporte como tenant                | Gestión de tenants        | VN + RT  | Es el punto de entrada del SaaS multi-tenant. Cada alta implica aprovisionar un tenant con aislamiento físico Silo, incluyendo su persistencia, configuración operativa y ciclo indesujeto a migraciones y respaldo.                          |
| RF-04  | Configuración inicial del operador                            | Gestión de tenants        | VN       | Habilita que cada tenant configure sus propios recursos y reglas dentro de su instancia aislada.                                                                                                                                              |
| RF-05  | Creación de usuarios por tenant                               | Gestion de usuarios       | VN       | Alta de usuarios dentro del tenant. La asociación al operador sustituye el uso de empresaId como mecanismo de aislamiento entre tenants.                                                                                                      |
| RF-06  | Asignación de roles a usuarios                                | Gestión de usuarios       | VN       | Habilita los cuatro roles operativos que son diferenciador (ver RF-02).                                                                                                                                                                       |
| RF-07  | Consulta de disponibilidad de conductores                     | Gestión de conductores    | VN       | Es el insumo crítico del motor de asignación. Sin esto, no hay asignación automática.                                                                                                                                                         |
| RF-08  | Registro de itinerarios                                       | Itinerarios y solicitudes | VN       | El itinerario es el objeto central del dominio. Sin itinerario no hay servicio que asignar.                                                                                                                                                   |
| RF-09  | Recepción de solicitudes de transporte                        | Itinerarios y solicitudes | VN       | Core del negocio. Es lo que dispara todo el flujo operativo.                                                                                                                                                                                  |
| RF-10  | Asignación automática de conductores                          | Motor de asignación       | VN + RT  | Core del negocio: externaliza la lógica de asignación que hoy vive en la cabeza del administrador y la vuelve configurable por tenant.                                                                                                        |
| RF-11  | Aplicación de reglas de negocio para la asignación            | Motor de asignación       | VN + RT  | Motor de reglas configurable por operador, orientado a turnos y programación. La telemetría no se usa para competencia dinámica por cercanía.                                                                                                 |
| RF-12  | Reasignación manual de conductores                            | Motor de asignación       | VN       | Es necesaria para contingencias reales del caso ancla, como un conductor que abandona su turno y obliga a redistribuir servicios.                                                                                                             |
| RF-13  | Priorización de solicitudes urgentes                          | Motor de asignación       | VN + RT  | Permite priorización operativa según reglas del tenant y gestión de excepciones sin convertir el motor en ride-hailing por proximidad.                                                                                                        |
| RF-14  | Recepción de mensajes desde WhatsApp                          | Integración WhatsApp      | VN + RT  | Canal principal para pasajero y conductor. Reto técnico: webhooks, contrato de integración, idempotencia, seguridad y desacople del proveedor mediante un adaptador específico.                                                               |
| RF-15  | Procesamiento de solicitudes enviadas por WhatsApp            | Integración WhatsApp      | RT       | Parseo conversacional + máquina de estados de conversación. No es CRUD; requiere investigar patrones de bots conversacionales.                                                                                                                |
| RF-16  | Envío de confirmaciones al usuario por WhatsApp               | Integración WhatsApp      | VN       | Cierra el flujo. Sin confirmación bidireccional, el pasajero no sabe si su solicitud fue procesada.                                                                                                                                           |
| RF-17  | Envío de notificaciones al conductor asignado                 | Integración WhatsApp      | VN       | Cierra el otro extremo del flujo. Sin esto, la asignación no llega al conductor.                                                                                                                                                              |
| RF-18  | Actualización de estado del viaje desde WhatsApp              | Integración WhatsApp      | RT       | Reto técnico: mantener sincronización bidireccional entre el estado del sistema y el estado reportado por conductor vía chat.                                                                                                                 |
| RF-19  | Aislamiento físico entre tenants (Silo)                       | Multi-tenancy             | VN + RT  | Diferenciador confirmado: un tenant = un operador de transporte con persistencia físicamente aislada. empresaId identifica al cliente empresarial dentro de la BD del operador y no funciona como filtro entre tenants.                       |
| RF-20  | Configuración separada de usuarios y recursos por cliente     | Multi-tenancy             | RT       | Configuración independiente por tenant dentro de su instancia Silo, sin compartir persistencia con otros operadores.                                                                                                                          |
| RF-21  | Configuración avanzada de reglas de asignación                | Configuración avanzada    | VN + RT  | Reglas parametrizables por tenant = diferenciador. Deben soportar turnos, balance de carga y restricciones operativas/regulatorias; no reasignación dinámica por cercanía GPS.                                                                |
| RF-54  | Calificación pasajero → conductor                             | Calificaciones            | VN       | Capacidad incluida explícitamente en V1. Agrega trazabilidad de calidad operacional sobre Viaje y alimenta reportes básicos.                                                                                                                  |
| RF-55  | Ingesta de telemetría GPS                                     | Telemetría                | VN + RT  | Capacidad incluida en V1. Introduce RegistroUbicacion, tratamiento de datos de ubicación y una estrategia de ingesta/persistencia adecuada al volumen operativo.                                                                              |
| RF-56  | Detección de desviación de ETA                                | Telemetría                | VN + RT  | Usa telemetría para monitoreo de viajes en curso y alertas de desviación. No cambia el motor a asignación dinámica por proximidad.                                                                                                            |

*VN = Valor de Negocio · RT = Reto Técnico*

## Tabla B — Requisitos funcionales NO significativos (dependencias)

| **Razón de descarte**                                                                                                       | **Requisitos funcionales**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| CRUD estándar — patrón conocido, no requiere decisiones arquitectónicas nuevas                                              | Cierre de sesión; Recuperación de credenciales; Actualización de información de usuarios; Activación/inactivación de usuarios; Registro de conductores; Actualización de datos de conductores; Activación/inactivación de conductores; Validación de datos mínimos de solicitud; Consulta del estado de solicitudes; Historial de solicitudes y viajes; Modificación o cancelación de solicitudes; Visualización de asignaciones generadas; Actualización del estado de asignación; Visualización centralizada de solicitudes y viajes; Consulta de usuarios/conductores/asignaciones; Búsqueda y filtrado de información; Visualización de indicadores básicos |
| Feature secundaria — aporta valor pero no moldea la arquitectura                                                            | Envío de recordatorios automáticos; Notificación automática de cambios de estado; Notificación de reasignación o novedades; Alertas operativas para administradores u operadores                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Reportes operacionales — necesarios para el producto, pero no introducen por sí solos una decisión arquitectónica adicional | Reporte de viajes realizados; Reporte de solicitudes atendidas y pendientes; Reporte de conductores asignados; Reporte histórico de operación por cliente empresarial                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Observabilidad transversal — cubierta por logging, auditoría y monitoreo técnico                                            | Registro de eventos del sistema; Registro de errores y excepciones; Seguimiento de acciones de usuarios administrativos; Monitoreo técnico de disponibilidad                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Operación estándar del Panel Web — necesaria, pero no toda es driver arquitectónico                                         | Intervención manual estándar; consulta de catálogos; búsqueda y filtrado. El monitoreo en tiempo real asociado a telemetría se cubre por RF-55 y RF-56.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
