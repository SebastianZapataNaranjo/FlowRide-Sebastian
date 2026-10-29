# ADR-0001: adoptar monolito modular con límites internos explícitos

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0001 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-01 histórico del tradeoff anterior, que trataba la decisión como heredada de DOO. |

## Contexto

FlowRide es un SaaS B2B2B para operadores de transporte corporativo con servicios programados, horarios estrictos y flota propia o de uso exclusivo. El sistema debe soportar hasta 200 usuarios simultáneos por instancia, mantener una operación transaccional consistente y evolucionar sin introducir complejidad de distribución que no esté respaldada por un driver real.

El dominio contiene módulos diferenciables —identidad, usuarios, conductores, solicitudes, asignación, notificaciones, telemetría, auditoría y reportes—, pero no existe una necesidad de despliegue independiente ni de escalado radicalmente distinto por módulo en la V1. Al mismo tiempo, el aislamiento Silo por operador ya proporciona una frontera fuerte de despliegue por tenant.

## Decisión

Construir FlowRide como un **monolito modular** en .NET 10, con límites de módulo explícitos y dependencias dirigidas hacia el núcleo del negocio.

Mantener la solución organizada en proyectos `FlowRide.Domain`, `FlowRide.Application`, `FlowRide.Infrastructure` y `FlowRide.API`, sin interpretar esos proyectos como microservicios. Los módulos internos comparten proceso de despliegue, pero no deben acoplar su lógica de negocio mediante acceso directo a implementaciones internas de otros módulos.

El Panel Web consume la API directamente por HTTP/JSON. WhatsApp, telemetría y proveedores externos se integran mediante adapters.

## Alternativas consideradas

### Alternativa A — Microservicios
Descartada para V1. Aporta independencia de despliegue y escalado por servicio, pero añade descubrimiento, comunicación distribuida, observabilidad distribuida, coordinación de datos y operación de múltiples despliegues. Ningún driver vigente exige esas propiedades por módulo.

### Alternativa B — Monolito sin modularización
Descartada. Reduce estructura inicial, pero favorece dependencias transversales y dificulta la evolución del motor de asignación, telemetría e integraciones.

### Alternativa C — Serverless por función
Descartada como arquitectura general. Puede ser útil para tareas puntuales, pero dispersaría el dominio en funciones y complicaría consistencia, depuración y portabilidad.

## Consecuencias positivas

- Un único artefacto de aplicación simplifica despliegue, diagnóstico y pruebas end-to-end.
- Los módulos permiten localizar cambios sin introducir comunicación distribuida.
- Encaja con el aislamiento Silo: cada operador ejecuta una instancia lógica independiente del mismo producto.
- Permite evolucionar hacia extracción de servicios solo si un driver futuro lo justifica.

## Consecuencias negativas

- El ciclo de despliegue es compartido entre módulos.
- Un fallo de proceso puede afectar varios módulos de un mismo tenant si no se aplican límites y aislamiento internos.
- Requiere disciplina para impedir dependencias cruzadas que degraden el monolito modular en un monolito acoplado.

## Tácticas de Bass aplicadas

- **Reducir acoplamiento** — interfaces y adapters separan dominio de infraestructura y proveedores.
- **Aumentar cohesión** — cada módulo concentra responsabilidades de negocio relacionadas.
- **Localizar cambios** — las reglas y dependencias externas se encapsulan detrás de límites explícitos.

## Escenarios y requisitos relacionados

- `OPLOG-CAP-01` — el motor debe sostener picos operativos del tenant sin requerir distribución por microservicios.
- `PAS-REND-02` — las consultas simples no deben invocar lógica innecesaria del motor.
- `Escenario de mantenibilidad` — los cambios funcionales deben permanecer localizados.

## ADRs relacionados

- **ADR-0002** — define la frontera Silo por operador.
- **ADR-0004** — define la estructura extensible del motor de asignación.
- **ADR-0010** — define despliegue y escalado de cada instancia.

## Notas de implementación

- Las referencias entre proyectos deben mantener `Domain` sin dependencias hacia infraestructura.
- Los módulos no comparten detalles de proveedor; consumen interfaces de Application o contratos de dominio.
- Un módulo solo se extraerá como servicio independiente cuando exista evidencia de escalado, disponibilidad o despliegue diferenciado.

## Referencias

- `Drivers_Arquitectonicos_FlowRide_FINAL.docx`.
- `FuncionalidadesSignificativas_FlowRide_ACTUALIZADO.xlsx`.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- Software Architecture in Practice — Bass, Clements y Kazman.
