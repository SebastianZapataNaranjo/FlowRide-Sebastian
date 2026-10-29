# ADR-0010: desplegar un stack aislado por tenant con contenedores y escalado horizontal controlado

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0010 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | — |

## Contexto

El modelo Silo exige separar operadores y el sistema debe soportar hasta 200 usuarios simultáneos por instancia. Los SLA vigentes son 99,9% para el canal WhatsApp, 99,5% para motor/operaciones críticas y 99% para Panel Web. Al mismo tiempo, el costo básico de infraestructura objetivo es aproximadamente 20–40 USD/mes por tenant, sin contar costos variables del canal.

Un clúster complejo por tenant sería incompatible con ese sobre de costo. El despliegue necesita automatización, recuperación y posibilidad de escala sin adoptar orquestación pesada como requisito base.

## Decisión

Empaquetar el backend como **contenedor Docker** y desplegar un **stack lógico independiente por tenant** con configuración/secretos propios y conexión a su Silo PostgreSQL.

La línea base puede operar con una instancia de aplicación por tenant cuando la disponibilidad/carga lo permita. Para tenants o periodos que requieran mayor continuidad/capacidad, permitir múltiples réplicas stateless detrás de un balanceador, manteniendo estado crítico fuera del proceso.

Automatizar health checks, restart, despliegue rolling cuando existan varias réplicas y migraciones por tenant. No fijar Kubernetes como requisito; usarlo solo si el número de tenants o réplicas demuestra la necesidad.

## Alternativas consideradas

### Alternativa A — VM/proceso manual sin contenedores
Descartada por reproducibilidad y automatización inferiores.

### Alternativa B — Kubernetes desde V1
Descartada como obligación base por costo/operación desproporcionados al volumen actual.

### Alternativa C — Serverless completo
Descartada como arquitectura general por complejidad en procesos persistentes, webhooks y consistencia.

### Alternativa D — Contenedores por tenant + escalado horizontal cuando aplique
Elegida por balance entre aislamiento, portabilidad y costo.

## Consecuencias positivas

- Despliegue reproducible por tenant.
- Permite crecer de una a varias réplicas sin rediseñar la aplicación.
- Encaja con Silo y facilita rollback.
- No obliga a mantener un orquestador complejo desde el inicio.

## Consecuencias negativas

- Una sola réplica puede no satisfacer todos los objetivos de disponibilidad ante fallos de host.
- Varias réplicas complican caché local y tareas en segundo plano.
- El costo crece linealmente si cada tenant exige alta redundancia permanente.

## Tácticas de Bass aplicadas

- **Incrementar recursos** — añadir réplicas cuando la carga lo justifique.
- **Detección de fallos** — health checks.
- **Recuperación** — restart/reemplazo de instancia.
- **Reconfiguración** — balanceador retira instancias no sanas.

## Escenarios y requisitos relacionados

- `COND-ESC-01` — hasta 200 usuarios simultáneos por instancia.
- `ADM-DISP-01` — recuperación de operaciones críticas.
- `OPLOG-DISP-02` — Panel Web con SLA 99%.
- `RF-53` — monitoreo técnico de disponibilidad.

## ADRs relacionados

- **ADR-0001** — despliega el monolito modular.
- **ADR-0002** — cada stack resuelve un Silo.
- **ADR-0007** — múltiples réplicas pueden disparar revisión de caché.
- **ADR-0011** — health checks y métricas verifican SLA.

## Notas de implementación

- Construir imagen inmutable y configurar por variables/secretos externos.
- No ejecutar jobs de fondo duplicados sin coordinación cuando existan varias réplicas.
- Medir CPU, memoria, cola de mensajes y conexiones a BD antes de escalar.
- El mecanismo concreto de hosting cloud puede variar si conserva aislamiento, costo y automatización.

## Referencias

- `RestriccionesNegocio_FlowRide_ACTUALIZADO.xlsx` — costo y SLA.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx` — 200 usuarios simultáneos.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — T-03.
