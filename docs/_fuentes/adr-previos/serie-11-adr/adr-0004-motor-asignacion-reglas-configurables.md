# ADR-0004: implementar el motor de asignación con reglas configurables y composición determinista

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0004 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-04 histórico del tradeoff anterior, conservando el tema del motor de asignación pero actualizando las reglas. |

## Contexto

El principal valor diferencial de FlowRide es reemplazar la asignación manual que hoy depende del conocimiento del administrador por un motor que aplique reglas programadas y explicables. La operación no sigue un modelo ride-hailing: los servicios son programados y la ubicación GPS se utiliza para monitoreo/ETA, no para competir dinámicamente por cercanía.

Las reglas vigentes consideran disponibilidad programada, cambios de turno, balance de carga, restricciones regulatorias y características del servicio.

## Decisión

Implementar el motor como un componente interno del monolito modular con **reglas encapsuladas detrás de una interfaz común** y una composición determinista.

Cada regla puede filtrar, validar o puntuar candidatos, pero el orden y la precedencia deben estar explícitamente configurados y ser auditables. Una regla nueva no puede requerir modificar las reglas existentes. La configuración pertenece al tenant Silo.

La telemetría GPS queda fuera del conjunto de criterios de reasignación dinámica. Solo puede aportar señales de monitoreo y desviación de ETA.

## Alternativas consideradas

### Alternativa A — Condicionales hardcodeados
Descartada. Hace que cada nueva regla incremente complejidad ciclomática y riesgo de regresión.

### Alternativa B — Motor de reglas externo/DSL
Descartada para V1. Aporta expresividad, pero introduce una plataforma adicional y una curva de operación no justificada.

### Alternativa C — Strategy/Policy internas configurables
Elegida por permitir extensibilidad y trazabilidad con complejidad contenida.

## Consecuencias positivas

- Nuevas reglas se agregan con impacto localizado.
- El orden de decisión puede explicarse y auditarse.
- Se mantiene el motor dentro del mismo proceso y transacción del dominio.
- Evita convertir la telemetría en lógica tipo Uber.

## Consecuencias negativas

- Una composición mal diseñada puede producir reglas contradictorias.
- Requiere validación explícita de configuración para evitar ciclos o precedencias inválidas.
- El patrón Strategy por sí solo no resuelve persistencia ni edición dinámica de reglas.

## Tácticas de Bass aplicadas

- **Diferir binding** — la configuración selecciona/ordena reglas en tiempo de ejecución.
- **Reducir acoplamiento** — cada regla implementa un contrato común.
- **Aumentar cohesión** — la lógica de selección permanece en el motor y no en controladores.

## Escenarios y requisitos relacionados

- `OPLOG-CAP-01` — el motor sostiene picos operativos del tenant.
- `RF-10` — asignación automática.
- `RF-11` — aplicación de reglas de negocio.
- `RF-13` — priorización de solicitudes críticas.
- `RF-21` — configuración de reglas.

## ADRs relacionados

- **ADR-0001** — el motor es un módulo interno del monolito.
- **ADR-0006** — protege consistencia cuando varias asignaciones compiten.
- **ADR-0009** — telemetría se usa solo para ETA/monitoreo.

## Notas de implementación

- Separar reglas eliminatorias de reglas de puntuación ayuda a mantener determinismo.
- Registrar versión de configuración usada en cada decisión para auditoría.
- Definir un desempate estable y reproducible.
- Incluir PoC con combinaciones de reglas en conflicto antes de ampliar el catálogo.

## Referencias

- `FuncionalidadesSignificativas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — PoC Motor de reglas.
- Software Architecture in Practice — tácticas de modificabilidad.
