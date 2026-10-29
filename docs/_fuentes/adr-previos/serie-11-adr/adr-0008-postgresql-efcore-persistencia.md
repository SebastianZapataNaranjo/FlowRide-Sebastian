# ADR-0008: usar PostgreSQL con EF Core/Npgsql como persistencia transaccional

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0008 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-08 histórico de PostgreSQL, actualizado para modelo Silo y telemetría. |

## Contexto

El núcleo de FlowRide es relacional y transaccional: usuarios, conductores, vehículos, itinerarios, solicitudes, asignaciones y viajes mantienen relaciones e invariantes fuertes. La implementación actual ya usa PostgreSQL y EF Core/Npgsql y no existe un driver que justifique migrar el núcleo a otro modelo.

El aislamiento Silo significa que cada operador mantiene su propia persistencia, por lo que la operación de migraciones y conexiones debe ser repetible.

## Decisión

Usar **PostgreSQL** como motor relacional de cada Silo y **Entity Framework Core + Npgsql** como mecanismo principal de mapeo y migraciones.

Permitir SQL optimizado o micro-ORM solo para consultas específicas si existe evidencia de rendimiento y sin saltarse los límites de repositorio/aplicación.

Organizar datos operativos y auditoría en estructuras claramente separadas dentro del Silo. La telemetría se resuelve en ADR-0009 y puede utilizar particionamiento dentro del mismo motor.

## Alternativas consideradas

### Alternativa A — SQL Server
Descartada como base principal por costo/licenciamiento y ausencia de ventaja necesaria para los drivers vigentes.

### Alternativa B — MySQL/MariaDB
Viable, pero cambiar implicaría migración sin beneficio arquitectónico demostrado.

### Alternativa C — PostgreSQL + EF Core/Npgsql
Elegida por capacidades transaccionales, madurez y continuidad con la implementación.

### Alternativa D — NoSQL como base principal
Descartada porque las relaciones e invariantes del núcleo son naturalmente relacionales.

## Consecuencias positivas

- Transacciones ACID para el núcleo.
- Migraciones versionadas y modelo fuerte en C#.
- Continuidad con código y experiencia ya existentes.
- Particionamiento y SQL avanzado disponibles si el volumen lo exige.

## Consecuencias negativas

- EF Core puede generar consultas subóptimas si no se revisan.
- Las migraciones deben orquestarse por múltiples Silos.
- Dependencia del dialecto/capacidades de PostgreSQL en Infrastructure.

## Tácticas de Bass aplicadas

- **Mantener consistencia** — transacciones relacionales.
- **Reducir overhead** — índices y consultas optimizadas.
- **Gestionar recursos** — pooling y paginación.

## Escenarios y requisitos relacionados

- `COND-CONF-02` — estado consistente de conductor.
- `PAS-CONF-02` — estado confirmado de solicitud.
- `ADM-CAP-02` — consultas paginadas e indexadas.
- `RF transaccionales` — persistencia del dominio operativo.

## ADRs relacionados

- **ADR-0002** — una persistencia PostgreSQL por Silo.
- **ADR-0006** — Outbox y concurrencia usan la transacción del motor.
- **ADR-0009** — telemetría usa particiones en PostgreSQL para V1.

## Notas de implementación

- Versionar migraciones y registrar versión de esquema por tenant.
- Definir índices según consultas reales; evitar índices basados en `empresaId` como separación de tenant.
- Usar transacciones explícitas en casos que afecten varios agregados/estados.
- Revisar consultas críticas con `EXPLAIN` cuando aparezcan problemas medidos.

## Referencias

- PostgreSQL Documentation — transactions and declarative partitioning.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx`.
