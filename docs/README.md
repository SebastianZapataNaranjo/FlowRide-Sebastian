# FlowRide — Documentación de arquitectura (F1–F3)

FlowRide es una plataforma SaaS para operadores de transporte corporativo interno (caso ancla: tripulaciones aéreas, ~40 servicios/día, 4–5 conductores coordinados hoy por WhatsApp y Excel). Esta documentación es el **diseño objetivo** de las tres funcionalidades que se entregan de extremo a extremo con HTTPS en la línea base del 4-nov-2026:

| | Funcionalidad | Requisitos |
|---|---|---|
| **F1** | El pasajero registra su itinerario por WhatsApp | RF-08 (corregido), RF-09, RF-14, RF-15, RF-16 |
| **F2** | El motor asigna conductor y vehículo con las reglas del operador | RF-10, RF-11, RF-45 |
| **F3** | El conductor recibe la propuesta y acepta o rechaza por WhatsApp | RF-17, RF-57, RF-58 |

> **Estado: Diseñado.** No hay código todavía. Ningún componente se presenta como ejecutado o probado; cada escenario dice con qué prueba se medirá.

## Ruta de lectura para la sustentación

1. **Decisiones** — [`adr/README.md`](adr/README.md): ADR-0001 a 0010 sostienen F1–F3; 0011 a 0016 quedan Diseñados.
2. **Vista de contexto y contenedores** — [`c4-contexto.md`](architecture/diagramas/c4-contexto.md), [`c4-contenedores.md`](architecture/diagramas/c4-contenedores.md).
3. **Cómo funciona** — [`secuencias-f1-f3.md`](architecture/diagramas/secuencias-f1-f3.md) y [`diseno-detallado.md`](architecture/diseno-detallado.md) (conversaciones, plantillas, máquinas de estado, DDL, endpoints, parámetros).
4. **Estructura interna** — [`componentes.md`](architecture/diagramas/componentes.md) (incluye los building blocks instanciados y no instanciados) y [`paquetes.md`](architecture/diagramas/paquetes.md) (capas × módulos y reglas de NetArchTest).
5. **Despliegue** — [`despliegue.md`](architecture/diagramas/despliegue.md): compose local (se construye) y Azure (Diseñado).
6. **Calidad** — [`escenarios-de-calidad.md`](quality/escenarios-de-calidad.md) (SEI con la prueba de cada uno) y [`tacticas-por-escenario.md`](quality/tacticas-por-escenario.md).
7. **Trazabilidad** — [`trazabilidad.md`](architecture/trazabilidad.md): RF → escenario → táctica → ADR → componente → prueba.
8. **Riesgos y pendientes** — [`riesgos.md`](architecture/riesgos.md), [`preguntas-abiertas.md`](architecture/preguntas-abiertas.md).
9. **Cambios frente al trabajo previo** — [`registro-de-cambios-vs-fuentes.md`](architecture/registro-de-cambios-vs-fuentes.md), con la equivalencia de numeraciones de ADR.
10. **Construcción** — [`ejecucion-local.md`](delivery/ejecucion-local.md), [`ramas-ci-y-calidad.md`](delivery/ramas-ci-y-calidad.md), [`checklist-seguridad.md`](delivery/checklist-seguridad.md).

## Convenciones

- Diagramas en Mermaid dentro de cada `.md`; todos compilan con `@mermaid-js/mermaid-cli` 11.4.2. Cada diagrama declara propósito, alcance, leyenda, protocolo en cada flecha y estado.
- Los datos del mundo actual (precios, SLA, límites de Meta, normas) citan la URL y la fecha de consulta (2026-10-28).
- `docs/_fuentes/` guarda el trabajo previo tal como estaba; no se edita. Los cambios se registran en `registro-de-cambios-vs-fuentes.md`.
