# ADR-0009: ingerir telemetría GPS mediante adapter y almacenarla particionada en PostgreSQL

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0009 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | — |

## Contexto

La V1 incluye telemetría GPS para monitorear viajes en curso y detectar desviaciones de ETA. El dimensionamiento vigente es de al menos 100 eventos de ubicación por minuto por instancia, con hasta 50 vehículos/conductores reportando aproximadamente cada 30 segundos y retención de 6 meses.

El proveedor físico de GPS puede variar entre operadores; por ello no conviene acoplar el dominio a Satrack, WhatsApp Live Location u otra fuente específica. Tampoco existe evidencia de volumen que justifique introducir desde V1 una base de series temporales adicional.

## Decisión

Definir una interfaz `ITelemetryProviderAdapter` que normalice eventos externos al modelo interno `RegistroUbicacion`.

La entrada inicial soportará integración HTTP/webhook o polling encapsulado en Infrastructure según el proveedor del operador. Cada evento normalizado contiene al menos vehículo/conductor, viaje cuando aplique, timestamp de origen, coordenadas y calidad/fuente.

Almacenar la telemetría de V1 en **PostgreSQL con particionamiento declarativo por tiempo** —preferentemente mensual— dentro del Silo del operador, con índices por vehículo/viaje y timestamp. Ejecutar una política automatizada de eliminación de particiones/datos superiores a 6 meses.

La telemetría alimenta monitoreo y `RF-56` detección de desviación de ETA. No participa como señal de reasignación dinámica por cercanía.

## Alternativas consideradas

### Alternativa A — Proveedor GPS fijo en el dominio
Descartada por lock-in y porque distintos operadores pueden tener proveedores diferentes.

### Alternativa B — TimescaleDB desde V1
Descartada como requisito inicial; añade extensión/operación sin evidencia de que PostgreSQL particionado no alcance el volumen previsto.

### Alternativa C — Base NoSQL/series temporales separada
Descartada para V1 por costo y complejidad.

### Alternativa D — Adapter + PostgreSQL particionado
Elegida por simplicidad, continuidad tecnológica y capacidad suficiente para el volumen estimado.

## Consecuencias positivas

- Proveedor GPS sustituible por tenant.
- No añade un motor adicional en V1.
- La retención de 6 meses se implementa eficientemente por particiones.
- Permite correlacionar telemetría con viajes sin usarla para asignación dinámica.

## Consecuencias negativas

- PostgreSQL puede requerir reevaluación si el volumen real crece varios órdenes de magnitud.
- Los proveedores pueden entregar formatos, frecuencias o calidades diferentes.
- El cálculo de ETA puede sufrir por datos GPS ruidosos o retrasados.

## Tácticas de Bass aplicadas

- **Adaptar interfaz** — normalizar proveedores GPS.
- **Gestionar recursos** — particionamiento y retención.
- **Detectar condiciones** — identificar desviación de ETA.
- **Aislar fallos** — eventos inválidos no detienen la ingesta.

## Escenarios y requisitos relacionados

- `TEL-INTEROP-01` — ≥100 eventos/min y ≥99% de eventos válidos aceptados.
- `PAS-CONF-01` — detectar desviación >5 min y alertar en <60 s.
- `RF-55` — ingesta de telemetría.
- `RF-56` — detección de desviación de ETA.

## ADRs relacionados

- **ADR-0002** — telemetría pertenece al Silo del operador.
- **ADR-0008** — PostgreSQL es el motor de almacenamiento de V1.
- **ADR-0011** — métricas y alertas de ingesta/ETA.

## Notas de implementación

- Particionar por `timestamp` de evento; crear particiones de forma anticipada.
- Indexar por `(VehiculoId, Timestamp)` y/o `(ViajeId, Timestamp)` según consultas medidas.
- Registrar timestamp del proveedor y timestamp de recepción para medir retraso.
- Descartar o aislar coordenadas imposibles, duplicados e información fuera de ventana.
- Automatizar purga de datos >6 meses.

## Referencias

- PostgreSQL Documentation — Declarative Partitioning.
- `FuncionalidadesSignificativas_FlowRide_ACTUALIZADO.xlsx` — RF-55 y RF-56.
- `RestriccionesNegocio_FlowRide_ACTUALIZADO.xlsx` — retención GPS.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — TEL-INTEROP-01 y PAS-CONF-01.
