# ADR-0002: aislar cada operador mediante modelo multi-tenant Silo

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0002 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-02 histórico basado en filtro global EF Core por `empresaId`. |

## Contexto

En FlowRide un **tenant es un operador de transporte**, no una empresa empresarial atendida por ese operador. Cada operador puede prestar servicio a aerolíneas, mineras, hospitales u otros clientes empresariales dentro de su propia operación.

El aislamiento de datos entre operadores es una condición contractual y de seguridad. El identificador `empresaId` permanece en el dominio para identificar al cliente empresarial atendido dentro del tenant, pero no puede utilizarse como frontera de seguridad entre tenants.

## Decisión

Adoptar un modelo multi-tenant **Silo**, con persistencia y credenciales de acceso separadas por operador. La sesión de aplicación debe resolver primero el tenant y, a partir de esa resolución, utilizar únicamente la conexión y recursos asignados a ese Silo.

Automatizar aprovisionamiento, migraciones, backups, restauración y configuración por tenant. Ninguna consulta funcional debe depender de un filtro global por `empresaId` para impedir acceso a otro operador.

## Alternativas consideradas

### Alternativa A — Pool: una única base compartida con tenantId
Descartada. Minimiza costo operativo, pero un error de filtro o autorización puede exponer datos de otro operador y aumenta el blast radius.

### Alternativa B — Bridge: esquema por tenant en la misma base
Descartada para la línea base. Mejora separación lógica, pero comparte motor, credenciales y recursos, y conserva mayor superficie de error operacional que Silo.

### Alternativa C — Silo
Elegida por alinearse con la confidencialidad B2B2B, contención de fallos y simplicidad conceptual de la frontera de tenant.

## Consecuencias positivas

- Aislamiento físico verificable entre operadores.
- Reduce el impacto de errores de consulta y facilita backup/restauración por cliente.
- Permite adaptar capacidad y mantenimiento por operador.
- El significado de `empresaId` queda correctamente limitado al cliente empresarial interno.

## Consecuencias negativas

- Aumenta la cantidad de recursos y operaciones de aprovisionamiento.
- Las migraciones deben ejecutarse de forma controlada sobre múltiples Silos.
- Puede elevar el costo si el despliegue por tenant no se automatiza.

## Tácticas de Bass aplicadas

- **Separar entidades/recursos** — cada operador obtiene persistencia y credenciales independientes.
- **Limitar acceso** — la resolución de tenant impide seleccionar recursos de otro Silo.
- **Detección de fallos** — la salud se supervisa por instancia/tenant para limitar el blast radius.

## Escenarios y requisitos relacionados

- `OPLOG-SEG-02` — un operador no puede consultar servicios de otro tenant.
- `ADM-SEG-02` — un administrador no puede modificar usuarios o recursos de otro tenant.
- `RF-19` — aislamiento físico entre tenants.
- `RF-20` — configuración independiente por tenant.

## ADRs relacionados

- **ADR-0001** — el Silo es una frontera de despliegue del monolito modular.
- **ADR-0008** — PostgreSQL es la persistencia transaccional de cada Silo.
- **ADR-0010** — automatiza el despliegue y la operación por tenant.

## Notas de implementación

- El tenant debe resolverse a partir de una identidad confiable y no de un parámetro libre enviado por el cliente.
- Las credenciales de conexión nunca se comparten entre operadores.
- `empresaId` puede repetirse en Silos distintos sin implicar colisión global.
- Las migraciones deben registrar versión de esquema por tenant y ser idempotentes/reintentables.

## Referencias

- `RestriccionesNegocio_FlowRide_ACTUALIZADO.xlsx`.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — T-02 Aislamiento vs. costo operacional.
