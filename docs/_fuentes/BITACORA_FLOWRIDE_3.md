# BITÁCORA FLOWRIDE — Registro de decisiones por chat

> Documento complementario al `CONTEXTO_FLOWRIDE_v10.md`.
> **Objetivo:** mantener trazabilidad de las ideas y decisiones trabajadas en cada chat de trabajo, ordenadas cronológicamente por tema.
> **Última actualización:** Septiembre 2026
> **Estado:** Punto 4 cerrado — Capa 1 del proyecto (reorganización del contexto v10) completada.

---

## INSTRUCCIONES DE USO

1. Cada **Punto** corresponde a un tema trabajado en un chat.
2. Se numeran secuencialmente en el orden en que se abordan.
3. Estructura fija de cada Punto: **Contexto → Hallazgos → Decisiones → Pendientes**.
4. Al abrir un chat nuevo, adjuntar este documento junto con el contexto maestro y decir: *"Continúo trabajando en FlowRide, aquí están el contexto maestro y la bitácora de decisiones."*
5. Cuando se cierre un tema, actualizar este documento antes de pasar al siguiente.
6. El contexto maestro sigue siendo la fuente de verdad del proyecto. Esta bitácora es el diario de trabajo: registra el *por qué* de los cambios y las decisiones descartadas que no siempre quedan escritas en el contexto.

---

## ÍNDICE

| # | Tema | Fecha | Estado |
|---|---|---|---|
| 1 | Benchmark de competencia | Agosto 2026 | ✅ Cerrado |
| 2 | Clasificación de material del curso (Teams) | Agosto 2026 | 🔄 En curso — folders 1 y "Base curso" cubiertos |
| 3 | Revisión del banco de escenarios / Tradeoff QA (tradeoff.xlsx) | 2 sep 2026 | ✅ Cerrado — hallazgos con pendientes de corrección |
| 4 | Capa 1: reorganización del contexto v10 (base del proyecto) | Sep 2026 | ✅ Cerrado |

---

## PUNTO 1 — BENCHMARK DE COMPETENCIA

### 1.1 Contexto

Después del pitch, quedó claro que el argumento inicial de *"somos los únicos que usan WhatsApp para reservas"* no se sostiene frente a productos existentes en el mercado. Se abrió un chat de trabajo para hacer un benchmark ordenado de competidores, identificar diferenciadores reales, y decidir qué funcionalidades incorporar, cuáles descartar por diseño, y cuáles dejar en roadmap.

**Salida esperada:** una posición defendible frente al profesor sobre el nicho real de FlowRide y una lista concreta de cambios a la sección 11 del contexto maestro.

### 1.2 Categorías de competencia identificadas

Se distinguen dos categorías con implicaciones muy distintas:

**Categoría A — Telemetría de flota (competidor indirecto / potencial complemento):**
- **Satrack.** GPS + video-telemática + geocercas + hábitos de conducción. Vigila el vehículo desde afuera, no gestiona la operación de solicitudes ni la asignación por reglas. **No es competidor directo.** Posible integración futura como fuente de telemetría.

**Categoría B — Dispatch y gestión de transporte (competidor directo):**
- **AllRide** — suite modular de movilidad con IA.
- **ToolRides** — TMS/ERP en la nube con WhatsApp Bot y portal corporativo.
- **Sengerio** — reemplazo de Excel para chárter, limusinas y turismo.

### 1.3 Fichas técnicas resumidas

#### AllRide
- **Foco:** plataforma de movilidad con IA por vertical (Cab, Bus, Delivery, Logistics, eBike). Employee Transportation es una vertical más dentro del catálogo.
- **Canales:** apps móviles + web + integraciones HRMS/payroll. **No usa WhatsApp** en la vertical de empleados.
- **Modelo de precios:** SaaS con planes públicos ($49 / $99 / $149 al mes según fuentes de terceros) + licencia enterprise con cotización.
- **Diferenciadores propios:** forecasting de demanda con IA, mantenimiento predictivo, integración con HR.
- **Debilidad frente a FlowRide:** es plataforma genérica; el transporte corporativo interno es una vertical, no el foco.

#### ToolRides
- **Foco:** TMS/ERP flexible para taxis, remisse, transporte ejecutivo, mensajería, courier, corporativo y traslado de personal.
- **Canales:** Web, App pasajero, App conductor, **WhatsApp Bot**, Portal corporativo, API.
- **Modelo de precios:** costo único de implementación + suscripción mensual variable por viajes o conductores.
- **Diferenciadores propios:** WhatsApp Bot activo, marca blanca, chat integrado, calificación bidireccional, tarifas configurables, múltiples métodos de pago.
- **Es el competidor más peligroso.** Ya ofrece WhatsApp + reglas configurables + traslado de personal como caso de uso declarado.

#### Sengerio
- **Foco:** empresas de alquiler de buses, limusinas, chárter y turismo. Origen europeo.
- **Canales:** web + app conductor propia. **Sin WhatsApp** ni canal conversacional externo.
- **Modelo de precios:** cotización personalizada, facturación integrada con Stripe y cumplimiento PCI-DSS.
- **Diferenciadores propios:** cotización con tarifarios detallados, red de proveedores externos, facturación con pasarela integrada, agenda que detecta disponibilidad automáticamente.
- **Es competidor indirecto** — juega en un mercado distinto (turismo, no personal corporativo), pero su narrativa *"reemplaza el Excel"* valida el argumento del profesor.

### 1.4 Matriz comparativa (resumen)

| Área | FlowRide | AllRide | ToolRides | Sengerio |
|---|---|---|---|---|
| Multi-tenant real por empresa cliente | ✅ | ⚠️ | ⚠️ | ❌ |
| WhatsApp como canal | ✅ | ❌ | ✅ | ❌ |
| Motor de asignación por reglas | ✅ | ⚠️ | ✅ | ⚠️ |
| Roles internos (Pasajero/Conductor/Admin empresa/Operador) | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Calificación pasajero ↔ conductor | ❌ | ✅ | ✅ | ❌ |
| Chat integrado in-app | ❌ (por diseño) | ✅ | ✅ | ✅ |
| Tarifario / cobro por viaje | ❌ (por diseño) | ✅ | ✅ | ✅ |
| IA de forecasting | ❌ | ✅ | ❌ | ❌ |
| Facturación con pasarela integrada | ❌ | ⚠️ | ⚠️ | ✅ |
| Marca blanca | ❌ | ✅ | ✅ | ❌ |

Convenciones: ✅ tiene y es parte de su propuesta pública, ⚠️ tiene parcial o inferido, ❌ no aparece o no es su foco.

*(La matriz completa cruzada contra cada funcionalidad de la sección 11 del contexto maestro vive en el chat original de agosto 2026; este resumen conserva las filas de mayor impacto para la sustentación.)*

### 1.5 Decisiones

#### 1.5.1 Incorporar (alta prioridad)

| Funcionalidad | Justificación |
|---|---|
| Entidad `Calificacion` sobre `Viaje` | Cambio pequeño en el modelo, alto impacto en la narrativa de trazabilidad. Un puntaje numérico + comentario opcional por viaje, agregable por conductor y por sede/empresa. Refuerza el discurso de calidad operativa y da un indicador que hoy no está en la matriz. |

#### 1.5.2 Descartar por diseño (con argumento listo para el profe)

Cada uno con la fórmula **decisión → alternativa descartada → por qué**:

| Funcionalidad | Argumento |
|---|---|
| Tarifario / cobro por viaje | **Decisión:** no incluir. **Alternativa:** modelo tarifario tipo taxi como ToolRides. **Por qué:** FlowRide es gestión operativa intra-empresa, no marketplace. El pasajero es empleado de la empresa cliente, no cliente final que paga. Incluir un tarifario ensuciaría el modelo de dominio sin resolver un problema real del caso de uso. |
| Chat integrado in-app | **Decisión:** no incluir. **Alternativa:** chat propio dentro de la app como AllRide o ToolRides. **Por qué:** WhatsApp ya es el canal conversacional del sistema. Duplicar la conversación en un chat propio rompe el principio de canal único que sostiene la propuesta y agrega complejidad sin beneficio. |
| Marca blanca | **Decisión:** no incluir. **Alternativa:** white-label como ToolRides. **Por qué:** no es la fase de negocio del producto. Agregarlo hoy multiplica la complejidad de despliegue sin retorno para el semestre. |
| Red de proveedores externos / subcontratación | **Decisión:** no aplica. **Alternativa:** modelo de partners como Sengerio. **Por qué:** cada empresa cliente de FlowRide opera su propia flota interna; el modelo de negocio no contempla subcontratación de conductores externos. |

#### 1.5.3 Roadmap futuro

- Predicción de demanda con IA (estilo AllRide).
- Mantenimiento predictivo de vehículos.
- Integración con HRMS / payroll / sistemas de asistencia.
- Integración con Satrack u otro proveedor de telemetría como fuente externa de datos del vehículo.

Estas quedan mencionadas como visión, no como promesa de entrega.

> **Nota v10 (Punto 4):** la integración con proveedores de telemetría profesional (Satrack u otros) se mantiene como opción de V2 en adelante, pero **la telemetría GPS como capacidad del producto se incluye desde V1** con fuente propia (no dependiente de proveedor externo). Fuente concreta se decide en la Capa 3 de ADRs.

#### 1.5.4 Diferenciadores a defender frente al profesor

1. **Multi-tenant lógico con `empresaId` desde el dominio.** Sengerio es single-tenant; AllRide y ToolRides tienen portal corporativo, pero su multi-tenancy es del operador de transporte, no del cliente empresarial atendido. FlowRide es SaaS multi-empresa desde el modelo.
2. **Roles operativos específicos de transporte corporativo interno.** Pasajero interno + Conductor interno + Administrador de empresa + Operador logístico. Los competidores modelan roles de marketplace (cliente externo + conductor + operador de flota).
3. **Ausencia deliberada de tarifario y cobro por viaje.** Modelo de dominio limpio de conceptos de facturación al pasajero — decisión de diseño, no carencia.
4. **Reglas de asignación configurables por empresa cliente dentro del mismo SaaS.** ToolRides configura reglas por operador de transporte único; FlowRide las configura por cada tenant/empresa cliente.
5. **Motor de asignación pensado para turnos, recurrencia y disponibilidad programada** — no para competencia en tiempo real por proximidad geográfica como en un app de taxi. Este es el caso real de tripulaciones aéreas, cuadrillas mineras y personal médico entre sedes.

> **Nota v10 (Punto 4):** estos cinco diferenciadores se re-examinaron en el Punto 4 de esta bitácora contra el problema aterrizado. El diferenciador 1 se refuerza y reformula como *"multi-tenant SaaS con aislamiento físico por operador (Silo)"* — más fuerte que la formulación original con `empresaId` filtrado. Se agrega el diferenciador 6 (WhatsApp como canal único como decisión de diseño). Ver Punto 4.3 para lista final.

### 1.6 Pendientes derivados de este punto

- [x] Actualizar sección 11 del contexto con la entidad `Calificacion` y sus funcionalidades asociadas. — **Cerrado en v10, sección 15 (bloque "Calificación") y sección 17.3 (entidad pendiente de agregar al diagrama en Capa 4).**
- [ ] Reflejar en el diagrama de clases la nueva entidad `Calificacion` relacionada con `Viaje` — **Movido a Capa 4.**
- [x] Redactar el bloque de benchmark para el próximo entregable escrito. — **Insumo del contexto v10, secciones 5 y 6.**
- [ ] Evaluar en detalle la posible integración con Satrack — **Movido a V2 de roadmap; el reemplazo es telemetría con fuente propia en V1.**
- [x] Formular argumentos de sustentación consolidados para el profesor a partir de los cinco diferenciadores. — **Cerrado en v10, sección 7 con seis diferenciadores.**

---

## PUNTO 2 — CLASIFICACIÓN DE MATERIAL DEL CURSO (TEAMS)

> **Alcance del Punto:** revisar por folders todo el material que el profesor dejó en el Teams de Software 2, clasificar cada archivo por tipo y prioridad, y cruzarlo contra el contexto maestro. Cada folder revisado se registra como subsección (2.A, 2.B, 2.C…) para mantener el hilo bajo un solo tema.
>
> **Subsecciones cubiertas:**
> - 2.A — Folder 1 del Teams (primer encuentro): 6 archivos.
> - 2.B — Folder "Base curso": 4 archivos.
> - 2.C — *(pendiente — siguiente folder)*

### 2.1 Contexto (aplica a todas las subsecciones)

En el pitch anterior perdimos puntos porque preparamos material sin haber revisado los archivos que el profesor había dejado en el Teams de Software 2. Se abre un chat de trabajo para revisar por folders todo lo que hay allí, clasificar cada archivo por tipo y prioridad, y cruzarlo contra el contexto maestro para detectar coincidencias y choques antes de seguir produciendo entregables a ciegas.

**Salida esperada por folder:** un orden claro de qué se lee en casa, qué se llena ya, qué es entregable grande del semestre y qué es autoevaluación personal — con las banderas necesarias para no repetir el error del pitch.

> **Nota v10 (Punto 4):** las referencias a códigos PE-xxx que aparecen a lo largo de este Punto 2 quedan como **registro histórico**. A partir del Punto 4 (Capa 1) el equipo **no organiza el trabajo por códigos del planeador** — el profesor no los usa, el equipo tampoco. Ver Punto 4.3 (regla operativa).

---

## SUBSECCIÓN 2.A — FOLDER 1 DEL TEAMS (PRIMER ENCUENTRO)

### 2.A.2 Hallazgos

El folder 1 tiene 6 archivos que se agrupan en tres categorías bien diferenciadas:

**Grupo A — Material de estudio (leer en casa, no se entrega):**

| Archivo | Contenido | Relevancia para FlowRide |
|---|---|---|
| Encuentro 2.pptx | Diseño y arquitectura de software, patrones vs. enfoques, evaluación de arquitectura, recomendaciones (bajo acoplamiento / alta cohesión), ARID, referencia al libro *97 Things Every Software Architect Should Know* | Refuerza el discurso del monolito modular; slide 20 pide Elevator Pitch (ya lo tenemos) |
| Encuentro 3.pptx | Drivers arquitectónicos: motivadores de negocio, atributos de calidad, funcionalidad primaria, restricciones técnicas y de negocio; **escenarios de calidad (estímulo→respuesta)**; QAW; árbol de utilidad | **Alto** — la sección 12 del contexto (RNF) no está en formato de escenario y probablemente el profe va a exigirlo |

**Grupo B — Plantillas a llenar (esto sí es producto entregable):**

| Archivo | Tamaño | Se puede adelantar con el contexto actual |
|---|---|---|
| Plantilla Mapa de Impacto.xlsx | 2 pestañas (Mapa Why/Who/How/What + Visión estilo elevator pitch) | Sí — 30 min. **⚠️ Trae texto contaminado de otro proyecto en la celda Visión — hay que borrarlo antes.** |
| Resumen ejecutivo Proyecto.pptx | 9 slides con preguntas guiadas | Sí — 1 tarde. Todo se recicla de las secciones 3, 4, 7, 11 y 18 del contexto |
| Plantilla DAS.docx | Documento gigante — TOC llega a niveles tipo 9.1.9.2.3.2.3.11.2 | Parcial. El profe marca varias secciones como opcionales con "(NO)"; se pueden adelantar las secciones 2, 4, 6, 7 y 8 con el material actual |

**Grupo C — Autoevaluación personal (no se entrega):**

| Archivo | Contenido |
|---|---|
| Precalentamiento conceptual.xlsx | 155 preguntas S/N + Definición; el profe hace reflexión sobre esto en las slides 3-4 del Encuentro 2 |

Diez preguntas del precalentamiento anclan directo en decisiones ya tomadas de FlowRide y conviene tenerlas armadas de memoria: 21 (microservicios), 51 (token), 62 (REST), 63 (JSON), 77 (DDD), 80 (ORM), 85 (SaaS), 92 (computación en memoria), 130 (arquitectura **modulítica** — es el nombre técnico del monolito modular) y 133 (webhook).

**Secciones del DAS marcadas como opcionales por el profe (llevan "(NO)" en el TOC):**
- 1. Propósito del proyecto
- 9.1 Vista Funcional / Casos de Uso (sección completa)
- 9.2.1 Diagrama de clases
- 9.2.2 Diagrama de objetos
- 9.4.2 Diagrama de colaboración

**Choques detectados contra el contexto maestro:**

1. **RNF sin formato de escenario.** La sección 12 tiene enunciados sueltos ("10.000 usuarios concurrentes", "99% uptime"). El Encuentro 3 y la sección 2.3 del DAS piden estímulo→respuesta.
2. **Modelo de dominio anémico vs. enriquecido (sección 9.1.5 del DAS).** El diagrama de clases del contexto (sección 13) tiene métodos en las entidades (`solicitarTransporte`, `aceptarServicio`), pero la implementación actual pone la lógica en servicios de aplicación — hay desalineamiento entre diseño y código.
3. **Faltan diagramas de secuencia.** El DAS (9.4.1) los pide para flujos críticos y todavía no existen en el proyecto.
4. **Contaminación en la plantilla Mapa de Impacto** — trae texto sobre "recorrer plantas de fábrica y ensamblaje" de otro estudiante.

### 2.A.3 Decisiones

*[Sección de decisiones conservada íntegra del Punto 2.A original. Ver bitácora anterior si se requiere detalle.]*

### 2.A.4 Pendientes del folder 1

- [ ] Llenar la plantilla Mapa de Impacto con datos de FlowRide (borrar antes el texto contaminado).
- [ ] Llenar el Resumen ejecutivo Proyecto (9 slides).
- [ ] Leer Encuentro 2 y Encuentro 3.
- [x] Convertir la sección 12 del contexto al formato de escenarios de calidad. — **Reestructurada en v10, sección 8 (contexto de negocio) con cifras aterrizadas por endpoint y SLA por tier.**
- [ ] Redactar borrador de las secciones 2 (motivadores), 7.2 (estilos adoptados) y 8 (justificación alternativa) del DAS.
- [ ] Contestar el precalentamiento conceptual — mínimo las 10 preguntas ancla.
- [ ] Continuar el Punto 2 con las siguientes subsecciones cuando se adjunten más folders del Teams.
- [x] Decidir si el modelo de dominio se empuja hacia enriquecido o se documenta como anémico — **resuelto en subsección 2.B: producir ambos.**

---

## SUBSECCIÓN 2.B — FOLDER "BASE CURSO"

*[Contenido íntegro conservado del Punto 2.B original.]*

### 2.B.4 Pendientes del folder "Base curso"

- [ ] **Verificar en la plataforma** el estado de los entregables tempranos del semestre.
- [ ] Consolidar requisitos funcionales + no funcionales + reglas de negocio de FlowRide en un solo documento presentable.
- [ ] Leer QAW, capítulo ejemplo de *Designing Software Architecture*, GRASP, GoF, OOAD, ADD, principios SOLID.
- [x] **Corregir sección 1 del contexto:** cambiar "Alejandro Quintero" por "Andrés David Ortiz Mejía". — **Corregido en v9 y v10.**
- [x] **Reescribir secciones 19 y 20 del contexto** — **Cerrado en v10: el calendario del profesor queda en la sección 23 simplificada; los códigos PE-xxx dejan de ser estructura de trabajo.**
- [x] Actualizar la pendiente sobre modelo de dominio (anémico + enriquecido). — **Contexto v10 sección 17 mantiene el modelo con métodos en entidades; anémico vs. enriquecido pasa a Capa 4.**
- [ ] Buscar en el Teams la carpeta "Bibliografía y Documentos de Apoyo" (Subsección 2.C candidata).
- [x] Añadir a la ficha del proyecto la trazabilidad "código → sección del DAS → sección del contexto maestro" — **Reformulado en v10 como trazabilidad interna entre secciones del contexto, sin usar códigos del planeador.**

---

## PUNTO 3 — REVISIÓN DEL BANCO DE ESCENARIOS / TRADEOFF QA (tradeoff.xlsx)

### 3.1 Contexto

Se adjuntó `tradeoff.xlsx`, el libro de trabajo del Mini QAW / árbol de utilidad: 14 hojas que van desde stakeholders y lluvia de escenarios hasta un banco de 164 escenarios propuestos, 33 seleccionados, tácticas de Bass, ADRs y PoC/Spikes. Se pidió una revisión completa contra el contexto maestro (`CONTEXTO_FLOWRIDE_v8.md`) y la bitácora, para detectar qué corregir antes de usarlo como insumo de entrega.

Se verificó el archivo celda por celda (no solo por lectura), incluyendo el cuadre aritmético de pesos y cuotas, la ausencia de códigos duplicados o huérfanos entre hojas, y la cobertura cruzada de los 33 escenarios seleccionados contra el catálogo de tácticas.

### 3.2 Hallazgos

*[Contenido íntegro conservado del Punto 3 original — 13 hallazgos entre estructura, formato y contenido.]*

### 3.3 Decisiones

- Se registra esta revisión como línea base de correcciones antes de usar el tradeoff como insumo directo del trabajo de drivers arquitectónicos. No se modificó el archivo original — las correcciones quedan como pendientes explícitos para que el equipo decida y ejecute.
- Se prioriza corregir primero los hallazgos con impacto en nota o en coherencia del argumento arquitectónico (1 a 4 y 9 a 13) antes que los cosméticos (5 a 8).
- Los hallazgos 10 y 12 en particular tocan el diagrama de clases ya validado por el profesor del semestre anterior — cualquier cambio ahí debe declararse explícitamente como tal en la próxima entrega, no aplicarse en silencio.

### 3.4 Pendientes

**De estructura/formato:** *[Ver bitácora anterior.]*

**De contenido:**

- [x] Reconciliar las metas de disponibilidad: declarar esquema de SLA por niveles o homogeneizar. — **Cerrado en v10 sección 8.3 con SLA por tier: WhatsApp 99.9%, motor 99.5%, panel 99%.**
- [x] Acotar `PAS-CONF-01` para que no dependa de detección de desviación de ETA en tiempo real, o reabrir formalmente la decisión de excluir telemetría. — **Resuelto en Punto 4: se incluye telemetría GPS en el producto. `PAS-CONF-01` se mantiene como está y adquiere fuente técnica real. Aplica en Capa 2 al revisar el tradeoff.**
- [ ] Promover a un ADR propio la caché de estado "activo" con TTL corto que exige `ADM-SEG-01`. — **Movido a Capa 3, ADR de autenticación.**
- [ ] Añadir al diagrama de clases las entidades exigidas por escenarios seleccionados. — **Movido a Capa 4; declarado en v10 sección 17.3 como pendiente explícito.**
- [x] Decidir formalmente si el importador CSV/Excel de RRHH (`ADM-INTEROP-02`) entra al alcance. — **Cerrado en Punto 4: se saca del alcance. Aplica en Capa 2 al corregir el tradeoff.**

---

## PUNTO 4 — CAPA 1: REORGANIZACIÓN DEL CONTEXTO v10 (BASE DEL PROYECTO)

### 4.1 Contexto

Después del ADR-05 de WhatsApp, quedó demostrado que hay un método de trabajo mejor que el que veníamos usando: **contexto → investigación honesta → alternativas reales → decisión con trade-offs conscientes**. El ADR-05 se produjo con ese método y quedó defendible. La decisión estratégica fue re-plantar todo el proyecto con el mismo método antes de comprometernos con más entregables — no queremos llegar al DAS (28 oct) ni a la línea base (4 nov) con decisiones heredadas del semestre de DOO que no sobreviven el estándar del profesor Wider.

Se acordó estructurar el trabajo por capas:

- **Capa 1** — Base del proyecto (este Punto 4): problema, visión, cliente, segmentos, alcance explícito, diferenciadores, contexto de negocio realista.
- **Capa 2** — Drivers arquitectónicos: limpieza del tradeoff, aplicando los 13 hallazgos del Punto 3.
- **Capa 3** — Diez ADRs con el método del ADR-05, uno por chat.
- **Capa 4** — Diseño detallado (diagramas UML de las 4+1 vistas, ajustes al diagrama de clases).
- **Capa 5** — Implementación y línea base.

Este chat abordó exclusivamente la Capa 1, con cinco puntos: (1) qué problema resolvemos, (2) para quién, (3) qué NO hacemos, (4) diferenciadores reales, (5) contexto de negocio realista.

### 4.2 Hallazgos

#### 4.2.1 Sobre el problema (Punto 1 del chat)

- El enunciado del v9 estaba escrito desde el ángulo de la solución ("qué automatizamos") y no desde el ángulo del dolor del cliente. Todos los sustantivos eran abstractos: "ineficiencias", "trazabilidad", "escala".
- La entrevista con el cliente ancla (dueño de empresa que atiende tripulaciones aéreas) confirmó cifras concretas: ~40 servicios/día, 4-5 conductores, un chat de WhatsApp por servicio. La lógica de asignación, la trazabilidad y la contingencia viven en la cabeza del administrador.
- Un incidente reportado (conductor que abandona turno en pleno día) muestra las tres capas del dolor: dependencia total del administrador (dobló turno él mismo como conductor), pérdida de confianza del cliente empresarial (pasajero llegó tarde), riesgo de pérdida del contrato mensual.
- El problema no es que "no exista un sistema": el sistema existe y es una persona con un celular, WhatsApp y Excel. Lo que falta es externalizar de esa persona lo que hoy es cuello de botella humano frágil.

#### 4.2.2 Sobre el cliente ideal (Punto 2 del chat)

- Solo hay validación real en una vertical (tripulaciones aéreas). Prometer "para cualquier empresa" repite el error del pitch inicial ("somos los únicos con WhatsApp").
- El modelo de dominio ya es neutral (Pasajero, empresaTransporte, codigoServicio) — la generalización queda demostrada por el diseño, no hay que prometerla.
- Se identificaron tres condiciones necesarias para ser cliente target: (a) B2B2B, (b) horarios estrictos, (c) flota propia o bajo contrato exclusivo. Un cliente que no cumpla las tres no es target de V1.
- Segmentos descartados con argumento: ride-hailing (marketplace, otro comprador), turismo (evento único vs. recurrencia), transporte escolar (regulación de menores), delivery (no hay pasajero), transporte público municipal (no hay contrato B2B).

#### 4.2.3 Sobre el alcance (Punto 3 del chat)

- Se consolidaron las exclusiones ya conocidas del benchmark y se agregaron nuevas con argumento objetivo, sin invocar "el semestre" como razón.
- Se decidió incluir tres cosas dentro del producto: (a) calificación pasajero → conductor, (b) reportes operacionales básicos, (c) telemetría GPS. Los dashboards analíticos avanzados quedan fuera de V1.
- **Cambio grande frente al v9:** se **incluye telemetría GPS del vehículo con detección de desviación de ETA** — antes estaba como "roadmap futuro fuera del alcance del curso". El equipo pide explícitamente quitar el argumento del semestre; la telemetría se piensa como capacidad de producto real, con horizonte realista.
- Se saca del alcance el importador CSV/Excel de RRHH (`ADM-INTEROP-02`) que estaba en los 33 escenarios del tradeoff sin decisión de negocio explícita — cerrando el hallazgo 13 del Punto 3.

#### 4.2.4 Sobre los diferenciadores (Punto 4 del chat)

- Los cinco diferenciadores del Punto 1.5.4 sobreviven la re-evaluación con reformulaciones puntuales.
- Se identificó un sexto diferenciador que estaba implícito y merece nombre propio: **WhatsApp como canal único como decisión de diseño** (no como parche o feature). ToolRides tiene WhatsApp pero también app, portal y chat; FlowRide toma la decisión opuesta.
- Se descartó el candidato "contingencia asistida al administrador" — es consecuencia de los diferenciadores 4 y 5, no un diferenciador independiente.
- Se descartó el meta-diferenciador "producto especializado, no plataforma genérica" propuesto por Claude — el equipo prefiere quedarse con los seis directos.

#### 4.2.5 Sobre multi-tenant y JWT (Punto 4 del chat, sub-discusión)

- El equipo detectó dos zonas incómodas del contexto v9 que Claude tuvo que aclarar:
  - **"Multi-tenant" como propuesta** vs. **"modelo Pool" como implementación** son cosas distintas. El equipo entendió el matiz y descartó Pool por sus dos preocupaciones legítimas (fallo cruzado y consumo cruzado entre tenants). Pero no descartó multi-tenancy — descartó Pool.
  - **"JWT" como formato de token** vs. **"JWT propio HMAC-SHA256" como implementación** también son cosas distintas. Cualquier proveedor externo (Auth0, Cognito, Keycloak, IdentityServer) emite JWTs; la pregunta real es quién emite y quién administra.
- El equipo pidió reabrir ambas decisiones sin comprometer proveedor en Capa 1, para llegar a la Capa 3 con las opciones abiertas.

#### 4.2.6 Sobre el contexto de negocio (Punto 5 del chat)

- Las cifras del v9 (10.000 concurrentes, 2 segundos, 99% uptime) no eran defendibles. Se aterrizaron con base en el caso ancla real:
  - **Volumen:** ~200 concurrentes por instancia en pico, 5 tenants V1 → 20 tenants V2 → 50-100 tenants V3.
  - **Rendimiento:** tabla de umbrales por endpoint (CRUD P95 500ms, consultas P95 300ms, reportes P95 2s).
  - **Disponibilidad:** SLA por tier (WhatsApp 99.9%, motor 99.5%, panel 99%) — declarado como decisión consciente, no inconsistencia.
- Se declararon presupuestos operacionales como marco para las decisiones de la Capa 3 (20-40 USD/mes infra por tenant, pricing 150-500 USD/mes al cliente, referencia de mercado ToolRides ~100-300).
- Se declaró marco legal (Habeas Data, contrato SaaS, propiedad de datos, retención, regulación sector transporte especial).
- Se declaró horizonte realista V1/V2/V3 sin fechas académicas: los ADRs de Capa 3 se toman pensando en V1 con revisiones planeadas para no cerrar puertas de V2/V3.

### 4.3 Decisiones

#### 4.3.1 Reglas de argumentación permanentes (van al contexto v10, sección 0)

- **Regla de framing:** ninguna decisión o exclusión se justifica con "no hay tiempo en el semestre" o variantes académicas. Toda exclusión con criterio objetivo: dolor del cliente, dominio, arquitectura, contrato de negocio, presupuesto.
- **Regla operativa:** no se usan códigos del planeador del profesor (PE-DAN-X, PE-DEP-X, PE-DD-X, PE-LB-X) como estructura de trabajo ni de entregables. El trabajo se organiza por capas.

Estas dos reglas se registran en la memoria del proyecto para futuros chats.

#### 4.3.2 Estrategia por capas (va al contexto v10, sección 0.4)

Se documenta la estructura por capas como estrategia declarada. Este chat cierra Capa 1; los siguientes chats trabajan Capa 2 y luego Capa 3 (un ADR por chat).

#### 4.3.3 Problema aterrizado con caso real

Reescritura completa de la sección 4 del contexto. Formato: actor principal (administrador) + situación concreta (40 servicios/día en 40 chats + Excel manual) + consecuencia medible (dependencia total; incidente real con pérdida de confianza del cliente empresarial).

#### 4.3.4 Cliente ideal V1: operadores de transporte de tripulaciones aéreas y verticales análogas

Tres condiciones: B2B2B, horarios estrictos, flota propia. Segmentos NO con argumento defendible listado explícitamente.

#### 4.3.5 Alcance con inclusiones y exclusiones explícitas

- **Se saca del alcance:** tarifario, chat in-app, marca blanca, subcontratación, facturación al cliente, app móvil nativa, IA de forecasting, mantenimiento predictivo, ruteo dinámico, integración con RRHH, multi-idioma, notificaciones fuera de WhatsApp (excepto correo para reset de contraseña del panel).
- **Se incluye en el alcance:** calificación pasajero → conductor, reportes operacionales básicos, **telemetría GPS con detección de desviación de ETA** (nueva inclusión).

#### 4.3.6 Modelo de aislamiento: Silo, no Pool

**Decisión:** base de datos física por operador cliente (Silo).

**Alternativa descartada 1 — Pool** (una BD, columna `empresaId` filtrada). *Por qué no:* riesgo de fuga cruzada si algún query olvida el filtro; consumo cruzado entre tenants; fallo del sistema tumba a todos. Estándar de Shopify/Notion pero exige disciplina fuerte que un equipo pequeño no siempre sostiene sin errores.

**Alternativa descartada 2 — Single-tenant real / instalable por cliente.** *Por qué no:* cambia el producto (deja de ser SaaS, licencia por instalación, pierde el diferenciador 1, cada cliente requiere despliegue asistido).

**Consecuencia sobre `empresaId`:** ya no es filtro de aislamiento; se queda como identificador del cliente empresarial dentro de la BD del operador (útil para que un operador atienda a varias aerolíneas/mineras/hospitales).

#### 4.3.7 Diferenciadores finales — seis

1. Multi-tenant SaaS con aislamiento físico por operador (Silo).
2. Roles operativos específicos de transporte corporativo interno.
3. Ausencia deliberada de tarifario y cobro al pasajero.
4. Reglas de asignación configurables por operador.
5. Motor programado por turnos con telemetría GPS para monitoreo y detección de desviación (no reasignación por cercanía).
6. WhatsApp como canal único para pasajero y conductor — decisión de diseño.

#### 4.3.8 JWT y WhatsApp abiertos como decisiones de Capa 3

- **Autenticación:** el contexto v10 dice "token con roles y trazabilidad al operador". Proveedor concreto (JWT propio, Auth0, Cognito, Keycloak, IdentityServer) se decide en ADR dedicado en Capa 3.
- **WhatsApp:** el contexto v10 dice "canal WhatsApp Business API". Proveedor concreto (Meta Cloud API directo, Twilio, otro intermediario) se decide en ADR dedicado en Capa 3. El equipo declara preferencia por Meta Cloud API directo pero no se compromete todavía.

Estas dos decisiones reabren parcialmente el ADR-05 ya escrito (Twilio) cuando llegue el momento en Capa 3.

#### 4.3.9 Cifras aterrizadas de RNF

Reemplazo completo de la sección 12 del v9 por la sección 8 del v10:
- Volumen: 200 concurrentes por instancia; 5/20/50-100 tenants V1/V2/V3.
- Rendimiento: tabla por endpoint con P95 y P99.
- Disponibilidad: SLA por tier.
- Presupuesto: 20-40 USD/mes infra por tenant, pricing 150-500 al cliente.
- Legal: Habeas Data, contrato SaaS, propiedad, retención, sector transporte.
- Horizonte: V1/V2/V3 sin fechas académicas.

### 4.4 Pendientes derivados de este punto

**De aplicación inmediata al contexto v10:** cerrados con la generación del propio v10.

**Que se arrastran a Capa 2 (limpieza del tradeoff):**

- [ ] Reflejar en el tradeoff los seis diferenciadores finales, no los cinco originales.
- [ ] Renombrar "Administrador" a "Administrador de empresa" en las hojas del workbook — hallazgo 7 del Punto 3.
- [ ] Aplicar los 13 hallazgos del Punto 3 pendientes de corrección.
- [ ] Corregir la implicación del hallazgo 10: telemetría se incluye, por lo tanto `PAS-CONF-01` se mantiene como está y adquiere fuente técnica real (definida en Capa 3).
- [ ] Corregir la implicación del hallazgo 13: `ADM-INTEROP-02` sale del banco de 33 seleccionados y queda en el pool de 164 como candidato de V3.
- [ ] Producir el catálogo RF-01 a RF-56 como documento propio y enlazarlo desde el contexto — hallazgo 3 del Punto 3.
- [ ] Llenar la hoja "Tradeoff QA" con las tensiones reales entre atributos, o eliminarla — hallazgo 1 del Punto 3.

**Que se arrastran a Capa 3 (los 10 ADRs, uno por chat):**

- [ ] ADR de arquitectura de aplicación (Monolito Modular vs. Microservicios vs. Capas simples, con ponderación numérica reproducible).
- [ ] ADR de modelo de aislamiento multi-tenant (formalizar Silo con las consecuencias operacionales).
- [ ] ADR de autenticación (JWT propio vs. Auth0 vs. Cognito vs. Keycloak vs. IdentityServer).
- [ ] ADR de base de datos transaccional (Postgres vs. otro).
- [ ] ADR de motor de base de datos para telemetría (Postgres particionado vs. TimescaleDB vs. otro).
- [ ] ADR de caché (política, TTL, distribuida por instancia si aplica).
- [ ] ADR de escalabilidad horizontal (aprovisionamiento por instancia Silo, control plane).
- [ ] Reapertura del ADR-05 de WhatsApp con Meta Cloud API directo como candidato preferente.
- [ ] ADR de telemetría GPS (fuente: hardware, Live Location por WhatsApp, o app propia — con las cinco preguntas declaradas en v10 sección 6.3).
- [ ] ADR de notificaciones y reportes asíncronos (Outbox, ETL a `flowride_reporting`).

**Que se arrastran a Capa 4 (diseño detallado):**

- [ ] Agregar al diagrama de clases: `Calificacion`, `CancelacionSolicitud`, `EstadoSolicitudHistorial`, `RegistroUbicacion`, atributo `esPersonal` en `SolicitudTransporte`. Declarar explícitamente el cambio frente al profesor por tratarse de una sección validada en el semestre anterior.
- [ ] Producir las 4+1 vistas UML alineadas al contexto v10.
- [ ] Resolver el desalineamiento anémico vs. enriquecido detectado en el Punto 2.A.2 (métodos en entidades del diagrama vs. lógica en servicios en el código).

---

## PLANTILLA PARA PRÓXIMOS PUNTOS

> Copiar la estructura de abajo, numerar como Punto 5, etc., y actualizar el índice al inicio del documento.
> Si un mismo Punto abarca varios lotes de material relacionado, abrir subsecciones N.A, N.B, N.C dentro del mismo Punto en vez de crear puntos separados.

### Punto N — [Tema]

#### N.1 Contexto
*(por qué se abre este chat, qué feedback o necesidad lo motiva)*

#### N.2 Hallazgos
*(qué se descubrió, qué se investigó, qué opciones se compararon)*

#### N.3 Decisiones
*(qué se acordó, con la fórmula "decisión → alternativa descartada → por qué" cuando aplique)*

#### N.4 Pendientes
*(tareas concretas derivadas de este chat, con checkboxes)*

---

*Fin del documento BITACORA_FLOWRIDE_3.md*
