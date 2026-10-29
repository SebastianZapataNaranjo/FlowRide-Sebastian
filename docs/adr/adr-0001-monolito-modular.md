# ADR-0001: construir FlowRide como monolito modular con límites verificados por pruebas arquitectónicas

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0001 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando las reglas de NetArchTest de `docs/architecture/diagramas/paquetes.md` corran en CI y pasen sobre el código de la línea base (escenario `EQ-MANT-01`). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0001 del `ADR_Decisiones_Arquitectonicas_FlowRide_LISTA.xlsx` (Accepted sin evidencia), ADR-01 del tradeoff, `adr-previos/serie-11-adr/adr-0001` |
| Estado de implementación | Diseñado |

## Contexto

FlowRide reemplaza la coordinación manual de un operador de transporte corporativo (caso ancla: ~40 servicios/día, 4–5 conductores, un chat de WhatsApp por servicio). Las tres funcionalidades que se entregan de extremo a extremo forman una cadena dentro de un mismo operador:

- **F1** — el pasajero registra su itinerario por WhatsApp y se crea una solicitud.
- **F2** — el motor asigna conductor y vehículo con las reglas del operador.
- **F3** — se notifica al conductor, que acepta o rechaza por WhatsApp.

Fuerzas que condicionan el estilo:

- **Consistencia**: F2 cambia en una misma operación la solicitud, la asignación, el historial y la auditoría, y encola la notificación. Partir eso entre procesos obliga a una consistencia distribuida (sagas o compensaciones) para ~400 eventos/día por operador.
- **Volumen**: techo dimensionado de 200 usuarios simultáneos y 50 conductores por instancia (contexto v10 §8.1). Ningún módulo necesita escalar por separado.
- **Costo**: 20–40 USD/mes de infraestructura por operador (restricción de negocio). Con Silo (ADR-0002), cada operador paga su propio cómputo; cada proceso adicional por operador cuesta unos 12–15 USD/mes en Azure Container Apps (cálculo en ADR-0009).
- **Diferenciador 4**: las reglas de asignación cambian por operador. Ese cambio tiene que quedar localizado (QA-11 de Drivers, escenario `EQ-MANT-01`).
- **Estándares del curso**: separación dominio / servicios (casos de uso y puertos) / infraestructura (adaptadores de entrada y salida), con dependencias hacia el núcleo (lineamientos del curso Ingeniería del Software 2).

## Decisión

1. Un **único artefacto desplegable** por operador, con **dos procesos** que comparten código: `FlowRide.Inicializador.Api` (recibe webhooks y la API de administración) y `FlowRide.Inicializador.Worker` (procesa inbox y outbox). Los dos salen de la misma imagen con distinto punto de entrada (ADR-0003, ADR-0009).
2. **Eje horizontal — capas como en la estructura del profesor**: `FlowRide.Dominio`, `FlowRide.Servicios`, `FlowRide.Infraestructura`, `FlowRide.Transversal` y los dos `Inicializador`. Las dependencias apuntan hacia `Dominio`.
3. **Eje vertical — módulos como cortes dentro de cada capa**: `Itinerarios`, `Asignacion`, `Flota` (conductores, vehículos, turnos), `Notificaciones` y `Canal` (WhatsApp). Un módulo no referencia tipos internos de otro módulo: se comunican por **puertos de `Servicios`** o por **eventos de la outbox** (ADR-0006).
4. Los límites **se verifican con pruebas arquitectónicas** (NetArchTest) que corren en CI. Las reglas exactas están en `docs/architecture/diagramas/paquetes.md`; si una regla no está escrita como prueba, no cuenta como límite.

## Alternativas consideradas

Criterios ponderados con la prioridad arquitectónica global de los Drivers (escala 11 a 1). Puntaje de 1 a 5. El costo se trata como **restricción eliminatoria** y no como peso, porque es una restricción de negocio.

| Criterio (peso) | Monolito modular (elegida) | Microservicios | Capas sin módulos | Serverless por función |
|---|---|---|---|---|
| Disponibilidad (11) | 4 — un proceso; una falla tumba los módulos de un operador, mitigado por Silo | 3 — más piezas que pueden fallar con una réplica por servicio | 4 | 2 — arranque en frío en el webhook |
| Seguridad (10) | 4 | 3 — autenticación entre servicios, más superficie | 4 | 3 |
| Rendimiento (9) | 5 — llamadas en proceso | 3 — saltos de red | 5 | 2 |
| Confiabilidad (8) | 5 — una transacción PostgreSQL más outbox | 2 — consistencia distribuida | 5 | 2 |
| Aislamiento (7) | 4 — lo resuelve el Silo en los cuatro casos | 4 | 4 | 4 |
| Mantenibilidad (1) | 4 — módulos con límites probados | 4 | 2 — todo puede depender de todo | 2 |
| **Promedio ponderado** | **4,37** | 3,00 | 4,33 | 2,52 |
| Costo 20–40 USD/operador | Cumple (~28–31) | **No cumple**: 4–5 procesos × 12–15 USD | Cumple | Cumple |

Cálculo: Σ(peso × puntaje) / 46. Ejemplo para la opción elegida: (11·4 + 10·4 + 9·5 + 8·5 + 7·4 + 1·4) / 46 = 201 / 46 = 4,37.

**Lectura honesta de la tabla.** La ponderación no separa el monolito modular de las capas sin módulos (4,37 frente a 4,33): la única diferencia es Mantenibilidad, que tiene el peso más bajo. Si el profesor pregunta "¿y si da casi lo mismo, por qué no capas simples?", la respuesta es:

- El diferenciador 4 obliga a cambiar reglas del motor sin tocar Itinerarios ni el Canal. Con capas sin módulos, nada impide que un caso de uso de Itinerarios consulte directamente las tablas de Asignación, y el acoplamiento aparece sin que nadie lo decida.
- El costo de la modularidad es bajo: namespaces más pruebas de NetArchTest. No agrega procesos ni infraestructura.
- El desempate se toma por el escenario `EQ-MANT-01`, no por el número.

**Microservicios**: descartado por la restricción de costo y por la consistencia. Ningún módulo tiene una carga o un ritmo de despliegue que justifique separarlo.
**Serverless**: descartado. El webhook es sensible al arranque en frío, y el Worker de outbox es un proceso que necesita estar corriendo todo el tiempo.

## Consecuencias

**Positivas**
- La cadena F1→F2→F3 se prueba de extremo a extremo con un `docker compose` por operador.
- Una transacción local protege cada cambio de estado (ADR-0006), sin coordinación distribuida.
- Un cambio de regla del motor toca solo `Asignacion` en Dominio y Servicios, y lo demuestran las pruebas.

**Negativas**
- Una falla del proceso API afecta a todos los módulos de ese operador. Lo contienen el Silo (no afecta a otros operadores) y la separación API/Worker (si cae el Worker, los webhooks se siguen recibiendo).
- El despliegue es compartido entre módulos: una corrección en Notificaciones redespliega todo el stack del operador.
- Las reglas de NetArchTest hay que mantenerlas: un módulo nuevo exige reglas nuevas.
- La estructura por capas del profesor hace que un módulo quede repartido en cuatro proyectos. Cuesta más navegar un corte vertical que con proyectos por módulo.

## Tácticas de Bass

| Táctica | Dónde se aplica |
|---|---|
| Aumentar cohesión (dividir módulo) | Cortes verticales `Itinerarios`, `Asignacion`, `Flota`, `Notificaciones`, `Canal` |
| Reducir acoplamiento — usar un intermediario | Los módulos se comunican por puertos o eventos de la outbox, no por referencias directas |
| Reducir acoplamiento — restringir dependencias | Reglas de NetArchTest sobre namespaces |
| Diferir el binding | Reglas del motor y parámetros cargados en tiempo de ejecución (ADR-0007, ADR-0008) |

## Escenarios y requisitos relacionados

- `EQ-MANT-01` — agregar o modificar una regla del motor sin modificar otros módulos (escenario principal de este ADR).
- `OPLOG-CONF-03`, `CANAL-CONF-01` — la transacción local de ADR-0006 depende de que todo viva en un proceso con una base de datos.
- RF-08, RF-09, RF-10, RF-11, RF-14 a RF-17 — la cadena F1→F3.

## Condiciones de revisión

- **Pasar a proyectos por módulo** (alternativa considerada en la ronda de diseño): cuando haya dos o más personas trabajando en paralelo en módulos distintos y los conflictos de merge en los proyectos por capa sean frecuentes, o cuando un módulo necesite un ciclo de versión propio. La migración es mecánica porque los namespaces ya siguen el eje de módulos.
- **Extraer un servicio**: solo si un módulo concreto muestra una carga o una disponibilidad que el resto no necesita (por ejemplo, la ingesta de telemetría GPS de V1, ADR-0013) y el costo por operador lo permite.
- Si las pruebas de NetArchTest se desactivan o se ignoran en CI, este ADR deja de sostenerse y vuelve a `Proposed`.

## Referencias

- Bass, Clements, Kazman. *Software Architecture in Practice*, 4.ª ed., cap. 8 (Modificabilidad).
- Lineamientos del curso ingeniería del software 2 (estructura de paquetes del profesor y pruebas arquitectónicas).
- `docs/_fuentes/DriversArquitectonicosFlowRide_ACTUALIZADO.md` — prioridades 11→1 y QA-11.
- ADR-0002 (Silo), ADR-0003 (stack y protocolos), ADR-0006 (consistencia), ADR-0009 (costo por proceso).
