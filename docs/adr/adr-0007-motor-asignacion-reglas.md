# ADR-0007: motor de asignación por evento con tres niveles de reglas, desempate determinista y expiración relativa

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0007 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando la PoC 1 de las fuentes ("Motor de reglas con conflictos") pase: 100 % de los casos de prueba con resultado idéntico al repetirse, ninguna eliminatoria ignorada y una explicación por decisión (`EQ-MANT-01`, `OPLOG-CONF-04`). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0009 del xlsx (Strategy/Policy, Accepted sin PoC), `adr-previos/serie-11-adr/adr-0004` |
| Estado de implementación | Diseñado |

## Contexto

Hoy la asignación vive en la cabeza del administrador del caso ancla. El incidente real (un conductor abandona su turno y el pasajero llega tarde) muestra que el problema no es solo elegir, sino **reaccionar cuando la elección falla**. Lo que este ADR tiene que resolver:

- **Cuándo** se asigna: por evento, en cuanto se registra la solicitud (decisión de la ronda 1, P2).
- **Con qué reglas**, separando lo que la ley exige de lo que prefiere cada operador (diferenciador 4).
- **Qué pasa si el conductor rechaza o no responde**, incluido el caso de un servicio a las 04:30 asignado a las 22:00 con todos los conductores dormidos.
- **Cómo se explica** cada decisión, para auditoría y para disputas (`OPLOG-AUD-01`).

Datos de entrada que llegan de F1: la **hora de presentación** en destino, que da el pasajero (decisión de la ronda 3, P13), el origen y el destino (lugares frecuentes), y la fecha.

## Decisión

### 1. Cálculo de la franja del servicio

- `duracion` = `duracion_entre_lugares(origen, destino)`, una matriz estática por operador, o `duracion.por_defecto_min` si el par no está cargado.
- `recogida = presentacion − duracion − recogida.margen_min`.
- `franja = [recogida − franja.alistamiento_min, presentacion + franja.retorno_min)`, el rango que bloquea al conductor y al vehículo en la restricción de exclusión (ADR-0006).

**Riesgo y sensibilidad declarados**: la matriz es estática y **no considera el tráfico**. El margen compensa solo en parte; un trayecto de madrugada y uno a las 18:00 tienen la misma duración en la matriz. La promesa queda **registrada** (`solicitud_transporte.presentacion_comprometida`). **No** es verificable en la entrega E2E: compararla con la llegada real exige el inicio y fin de viaje (RF-18), que queda Diseñado. La telemetría y el ETA dinámico son V1 (ADR-0013).

### 2. Tres niveles de reglas (P8)

| Nivel | Reglas | ¿Configurable por operador? | Fundamento |
|---|---|---|---|
| **1. Eliminatorias legales fijas** | Licencia de conducción vigente a la fecha del servicio y de categoría de servicio público (C); SOAT vigente; revisión técnico-mecánica vigente | **No.** Están en código de `Dominio`, no en el catálogo de parámetros | SOAT: Ley 769 de 2002, art. 42. RTM: Ley 769, art. 51 (anual). Categoría C para servicio público: boletín del Consejo de Estado. La redacción vigente del art. 22 (vigencia de la licencia) está pendiente (`preguntas-abiertas.md`); se valida contra la fecha de vencimiento registrada |
| **2. Filtros operativos** | Conductor activo; turno que cubre la franja (`turno_conductor`); **presente**: si el turno ya empezó, el conductor hizo check-in (RF-58); sin solape (restricción de exclusión); capacidad del vehículo ≥ número de pasajeros; no rechazó ni dejó expirar esta misma solicitud | Los umbrales (alistamiento, retorno, tolerancia del check-in) sí; la existencia del filtro no | Dominio del operador |
| **3. Puntaje** | `balance_carga_dia`: prefiere al conductor con menos servicios propuestos o aceptados ese día local | **Sí**: activación y peso en `regla_asignacion` | Diferenciador 4 |
| Desempate | Puntaje mayor → `ultima_asignacion_en` más antigua → `conductor_id` menor | No | Determinismo |

Implementación: cada regla implementa `IReglaEliminatoria` o `IReglaPuntaje` en `Dominio.Asignacion.Reglas`. El `MotorAsignacion` las compone en el orden fijo nivel 1 → nivel 2 → nivel 3. Agregar una regla de puntaje es una clase nueva más una fila en `regla_asignacion`, sin modificar el motor (`EQ-MANT-01`).

**Explicación obligatoria**: cada ejecución guarda en `asignacion.razones` (jsonb) los candidatos evaluados, la regla que descartó a cada uno, los puntajes y el criterio de desempate usado. Se audita también cuando no hay candidato.

### 3. Ciclo de vida de la asignación y expiración relativa (P7)

Estados de la asignación: `Propuesta → Aceptada | Rechazada | Expirada | Cancelada`.

Hora de expiración, con `t0` = momento de la propuesta y `L = recogida − asignacion.margen_confirmacion_min` (hora límite para que alguien confirme):

```
si L <= t0                                  → no se propone; escalar de inmediato (AlertaOperador)
si (recogida − t0) > asignacion.horizonte_h → expira_en = L          (servicio lejano: queda abierta hasta la hora límite)
si no                                       → expira_en = min(t0 + asignacion.expiracion_min, L)
```

- **Rechazo** (botón "Rechazar") o **expiración** (barrido del Worker cada minuto sobre `expira_en < now()`): la asignación pasa a `Rechazada` o `Expirada`, la solicitud vuelve a `Pendiente` y se encola de nuevo `SolicitudRegistrada`. El conductor queda excluido para esa solicitud.
- Tras `asignacion.max_intentos` (N) propuestas fallidas, o si no hay candidatos: la solicitud pasa a `SinCandidato` y se envía **`AlertaOperador` por WhatsApp** al número del operador (RF-45), además de registrarse.
- Parámetros (`ParameterCatalog`, por operador): `expiracion_min` = T (por defecto 15), `horizonte_h` = X (por defecto 6), `margen_confirmacion_min` (por defecto 90) y `max_intentos` = N (por defecto 3). Rangos validados en código (ADR-0008).

### 4. Qué detecta el check-in y qué no

- **Detecta que el conductor no se presentó al turno.** Si el turno empezó hace más de `turno.tolerancia_checkin_min` y no hay check-in, el conductor deja de ser candidato (filtro de nivel 2) y se alerta al operador. Liberar automáticamente sus asignaciones ya aceptadas para ese turno queda **Diseñado**.
- **No detecta el abandono a mitad del turno.** Ese caso aparece cuando el conductor **rechaza o deja expirar** su siguiente asignación, y lo resuelve el reencolado. La reasignación manual de servicios ya aceptados es RF-12 (Diseñado con el panel, ADR-0011).

### 5. Revalidación previa (Diseñada)

Para asignaciones hechas con días de anticipación: X horas antes de la recogida, un barrido revalida las eliminatorias y el turno del conductor aceptado (licencia o SOAT que vencieron, turno modificado). Si falla, libera la asignación y reencola. **Condición de revisión** asociada (P2): se activa cuando los operadores registren itinerarios con más de 24 h de anticipación de forma habitual.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| Condicionales en el servicio de aplicación | Cada operador nuevo modifica el motor; no hay explicación estructurada. |
| Motor de reglas externo (Microsoft RulesEngine, NRules) o DSL | Expresividad que no se necesita para unas 8 reglas. Las eliminatorias legales **no deben** ser editables por el operador, y un DSL tendería a exponerlas. Agrega una dependencia y un lenguaje que aprender. |
| Tabla de decisión configurable | Encaja con el puntaje pero no con los filtros, que consultan la base (turnos, solape). |
| **Optimización global** (asignar todas las solicitudes pendientes a la vez, por ejemplo con el algoritmo húngaro o OR-Tools) | Da mejores asignaciones de conjunto, pero choca con la decisión por evento y con la confirmación inmediata al pasajero. Queda como evolución del lote de replanificación (condición de revisión). |
| Asignar por cercanía GPS | Excluido por el diferenciador 5: servicios programados, no proximidad. |
| Expiración fija (P7 opción a) | Fallaba en el caso de la madrugada (todos dormidos → escalamiento a las 22:45 a nadie). |

## Consecuencias

**Positivas**
- Las obligaciones legales no se pueden desactivar por configuración.
- El operador cambia su política de puntaje sin un despliegue.
- Cada decisión es explicable y reproducible con el mismo estado de la base.
- El incidente del caso ancla tiene un camino definido: rechazo o expiración → reencolado → escalamiento por WhatsApp.

**Negativas**
- La matriz de duración estática puede comprometer recogidas tardías en hora pico (riesgo y sensibilidad en `riesgos.md`).
- Asignar por evento optimiza solicitud por solicitud, no el día completo.
- El barrido de expiración agrega un ciclo periódico al Worker.
- Con un solo conductor disponible, N reintentos no sirven y se escala antes (comportamiento esperado, se documenta en la secuencia).

## Tácticas de Bass

| Táctica | Dónde |
|---|---|
| Diferir el binding | Reglas de puntaje y parámetros en tiempo de ejecución |
| Abstraer servicios comunes / interfaz común | `IReglaEliminatoria`, `IReglaPuntaje` |
| Detectar fallos — tiempo de espera (timeout) | Expiración relativa de la propuesta |
| Recuperarse — reintentar y escalar | Reencolado con N intentos y `AlertaOperador` |
| Auditar | `asignacion.razones` y auditoría |
| Prevenir fallos — eliminar la excepción por diseño | Eliminatorias legales en código, no configurables |

## Escenarios y requisitos relacionados

- `OPLOG-CONF-04` (documentos vencidos → 0 asignaciones), `PAS-CONF-03` (ninguna solicitud queda huérfana), `OPLOG-CONF-03`, `COND-CONF-02` (reescrito), `EQ-MANT-01`.
- RF-10, RF-11, RF-13 (urgentes: Diseñado), RF-21 (configuración de reglas: vía API de administración), RF-45, RF-57 y RF-58.

## Condiciones de revisión

- Si más del 20 % de las solicitudes de un operador termina en `SinCandidato` con conductores disponibles, revisar la expiración o el horizonte.
- Si los operadores registran habitualmente con más de 24 h de anticipación, activar la revalidación previa y evaluar un lote nocturno con optimización global.
- Si la diferencia entre la presentación comprometida y la llegada real (cuando exista RF-18) supera el margen en más del 10 % de los servicios, reemplazar la matriz estática por una fuente con tráfico.

## Referencias

- Ley 769 de 2002, art. 42 (SOAT) y art. 51 (RTM), citadas en el [Decreto 2497 de 2022](https://www.dmsjuridica.com/buscador_20179478954/legislacion/decretos/2023/08/23/decreto-2497-de-2022/?pdf=56935) y el [Concepto de MinTransporte de 2024](https://www.ambitojuridico.com/sites/default/files/2024-05/Cpto-20241340325131-24.pdf); categorías de licencia: [boletín del Consejo de Estado](https://www.consejodeestado.gov.co/documentos/boletines/226/11001-03-24-000-2014-00069-00.pdf). Consulta 2026-10-28.
- `docs/_fuentes/PoC_y_Spikes_FlowRide_CORREGIDO.xlsx` — PoC 1.
- ADR-0006 (restricción de exclusión), ADR-0008 (parámetros), ADR-0011 (panel y RF-12).
