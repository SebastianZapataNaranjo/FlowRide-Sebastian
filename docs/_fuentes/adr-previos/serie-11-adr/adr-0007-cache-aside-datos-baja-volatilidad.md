# ADR-0007: usar Cache-Aside local solo para datos de baja volatilidad

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0007 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-07 histórico que fijaba IMemoryCache + clave por empresaId para multi-tenancy lógico. |

## Contexto

FlowRide tiene lecturas repetidas de catálogos, configuración y reportes agregados, pero también datos que cambian continuamente: solicitudes, asignaciones, viajes y telemetría. El modelo Silo reduce la necesidad de una caché global compartida entre tenants.

Una caché distribuida por defecto elevaría costo y operación. Una caché local puede mejorar lecturas sin convertirse en fuente de verdad siempre que su alcance esté restringido.

## Decisión

Aplicar **Cache-Aside** con caché en proceso (`IMemoryCache`) como optimización inicial de cada instancia Silo, exclusivamente para datos de baja volatilidad y lecturas tolerantes a obsolescencia acotada.

No cachear como fuente de verdad solicitudes, asignaciones, viajes en curso, disponibilidad crítica ni telemetría. Usar TTL corto e invalidación explícita en operaciones que modifiquen el dato cacheado.

Si un tenant requiere múltiples instancias activas y la inconsistencia temporal de caché local deja de ser aceptable, evaluar Redis/servicio distribuido en una revisión futura del ADR.

## Alternativas consideradas

### Alternativa A — Sin caché
Más simple y siempre consistente; descartada como política absoluta porque algunas lecturas agregadas pueden beneficiarse de copias temporales.

### Alternativa B — Redis distribuido desde V1
Descartada como requisito base por costo y operación adicionales; puede reabrirse con evidencia de necesidad.

### Alternativa C — IMemoryCache + Cache-Aside restringido
Elegida como optimización simple y sustituible.

## Consecuencias positivas

- Bajo costo operacional.
- Menos lecturas repetidas a PostgreSQL.
- No introduce infraestructura compartida que complique el Silo.
- La caché puede eliminarse sin cambiar reglas de negocio.

## Consecuencias negativas

- Cada instancia mantiene su propia copia.
- Puede existir obsolescencia temporal hasta invalidación/TTL.
- No sirve para coordinación, locks ni estado crítico.

## Tácticas de Bass aplicadas

- **Mantener múltiples copias** — caché para lectura.
- **Gestionar recursos** — reducir consultas repetidas.
- **Acotar tiempo de vida** — TTL limita obsolescencia.

## Escenarios y requisitos relacionados

- `ADM-REND-01` — dashboard/reportes con caché opcional.
- `ADM-CAP-02` — listados paginados e indexados.
- `RF-42 / RF-46 a RF-49` — indicadores y reportes.

## ADRs relacionados

- **ADR-0002** — cada caché pertenece al Silo/instancia del operador.
- **ADR-0008** — PostgreSQL sigue siendo fuente de verdad.
- **ADR-0010** — múltiples instancias pueden obligar a revisar la estrategia.

## Notas de implementación

- Las claves deben incorporar el recurso y parámetros funcionales; no usar `empresaId` como frontera de tenant.
- Invalidar después de commits exitosos, no antes.
- Instrumentar hit-rate y latencia para demostrar beneficio.
- Eliminar cachés cuyo hit-rate o mejora no justifique su complejidad.

## Referencias

- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — T-06.
