# ADR-0004: PostgreSQL con EF Core y Npgsql como persistencia transaccional de cada Silo

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0004 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando las migraciones de EF Core creen el esquema de `diseno-detallado.md` (incluida la restricción de exclusión) en las dos bases de tenant del compose, y pasen las pruebas de integración con Testcontainers. |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0004 del xlsx, ADR-04 del tradeoff, `adr-previos/serie-11-adr/adr-0008` |
| Estado de implementación | Diseñado |

## Contexto

El núcleo de F1–F3 es relacional y está lleno de invariantes: un itinerario por pasajero, código de servicio y fecha; una asignación viva por solicitud; ningún conductor con dos franjas solapadas. ADR-0006 pone la última defensa de esas invariantes en la base de datos, así que el motor tiene que soportar **restricciones de exclusión sobre rangos**, transacciones ACID y `FOR UPDATE SKIP LOCKED` para la outbox. La restricción técnica vigente ya fija PostgreSQL como motor principal.

## Decisión

- **PostgreSQL 17** por operador (una base por Silo, ADR-0002), con las extensiones `btree_gist` (restricción de exclusión) y `citext` (claves de catálogo sin distinguir mayúsculas). Sin TimescaleDB ni otros motores en F1–F3.
- **EF Core 10 + Npgsql** para el modelo transaccional, con migraciones versionadas en `Infraestructura.Output.Repositorios.Estructurados.Migraciones`. Las restricciones que EF no expresa (exclusión, índices parciales) se escriben en SQL dentro de la migración.
- Las entidades de `Dominio` **no** conocen EF. El mapeo vive en `Infraestructura` con configuraciones Fluent (`IEntityTypeConfiguration<T>`). Hay entidades de persistencia separadas solo donde el modelo de dominio y el relacional divergen (por ejemplo, `razones` en jsonb).
- La outbox, el inbox y el sondeo con `SKIP LOCKED` usan SQL parametrizado con Npgsql en lugar de LINQ, por precisión y legibilidad.
- Concurrencia optimista con `xmin` (ADR-0006).
- **Paginación del lado del servidor** en la API de administración: `limit` por defecto 20 y máximo 100, orden estable por `(creado_en, id)` y filtros explícitos por endpoint, con consultas parametrizadas (lineamientos del curso Ingeniería del Software 2).
- **Migraciones fuera del arranque del contenedor**: las corre la herramienta de aprovisionamiento (en compose, un servicio `migrador` de una sola ejecución por base) antes de levantar API y Worker.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| SQL Server | Sin restricción de exclusión sobre rangos equivalente a `EXCLUDE USING gist`; licencia; ninguna ventaja para estas invariantes. |
| MySQL / MariaDB | Sin restricciones de exclusión; `SKIP LOCKED` existe, pero la invariante central de ADR-0006 tendría que vivir en código. |
| MongoDB / NoSQL | El dominio es relacional y transaccional entre agregados; se perderían la integridad referencial y las restricciones. |
| Dapper en todo | Más control, pero sin migraciones ni seguimiento de cambios. Se usa SQL directo solo donde aporta (outbox e inbox). |
| Base compartida | Descartada por ADR-0002. |

## Consecuencias

**Positivas**: invariantes garantizadas por el motor; un solo motor en el stack; migraciones versionadas junto al código.
**Negativas**: las migraciones se corren N veces (una por operador); la dependencia del dialecto de PostgreSQL queda en `Infraestructura` (exclusión, `xmin`, `SKIP LOCKED`), lo que hace costoso cambiar de motor. Se acepta conscientemente.

## Tácticas de Bass

Mantener la consistencia de los datos (transacciones y restricciones); gestionar recursos (pool de conexiones de Npgsql e índices); restringir dependencias (EF solo en `Infraestructura`).

## Escenarios y requisitos relacionados

`OPLOG-CONF-03`, `CANAL-CONF-01`, `PLAT-AIS-01`; restricción técnica de persistencia; RF-08, RF-09, RF-10 y RF-19.

## Condiciones de revisión

- Si la telemetría (V1) supera lo que maneja una tabla particionada, ADR-0013 evalúa un almacén separado sin tocar este ADR.
- Si Azure Database for PostgreSQL Flexible Server dejara de ofrecer `btree_gist`, la invariante de solape tendría que pasar a un bloqueo pesimista (ADR-0006, alternativa C).

## Referencias

- PostgreSQL — Exclusion constraints, `btree_gist`, `SELECT … FOR UPDATE SKIP LOCKED`.
- Npgsql EF Core — Concurrency tokens (`xmin`): https://www.npgsql.org/efcore/modeling/concurrency.html (consulta 2026-10-28).
- Lineamientos del curso ingeniería del software 2 (paginación del lado del servidor).
