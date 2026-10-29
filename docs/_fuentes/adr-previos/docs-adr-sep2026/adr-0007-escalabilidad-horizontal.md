# ADR-0007: escalabilidad horizontal sobre contenedores gestionados en Azure Container Apps

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0007 |
| Fecha | 2026-09-16 |
| Estado | **Proposed** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO) |
| Profesor evaluador | Wider Farid Sánchez Garzón |

## Contexto

El modelo de aislamiento multi-tenant elegido para FlowRide es Silo: cada operador cliente vive en su propia base de datos física y, opcionalmente, en su propio cómputo. Ese modelo obliga a resolver tres preguntas de infraestructura que este ADR aborda: cómo se empaqueta y ejecuta la aplicación por operador, cómo se aprovisiona una instancia nueva cuando entra un tenant, y cómo se escala una instancia existente cuando la carga o el SLA lo exijan.

Los parámetros que acotan la decisión son:

- **Volumen por instancia:** hasta 200 usuarios concurrentes en pico, 500 usuarios activos, 50 conductores reportando telemetría GPS cada 30 segundos (~144.000 eventos/día).
- **Volumen de plataforma:** 5 operadores activos al lanzamiento, con proyección de crecimiento a 20 en 12–24 meses y 50–100 en horizonte de 36 meses. El crecimiento es lineal en instancias.
- **SLA por tier de servicio:** canal WhatsApp 99,9% (≈8h de downtime/año), motor de asignación y operaciones críticas 99,5% (≈43h/año), panel web administrativo 99% (≈87h/año). Los tiers no comparten disponibilidad y se pagan por separado.
- **Presupuesto de infraestructura:** techo de 20–40 USD/mes por operador, con pricing al cliente de 150–500 USD/mes.
- **Equipo:** tres personas sin experiencia previa operando orquestadores de contenedores en producción.
- **Restricciones operacionales del modelo Silo:** aprovisionamiento automatizado al captar un operador nuevo, ejecución de migraciones EF Core por instancia, monitoreo por instancia con visibilidad agregada al equipo FlowRide, backup y restauración por instancia.

Sobre este contexto, dos caminos quedan estructuralmente descartados antes de comparar alternativas: cualquier decisión que dependa de operación manual no soporta la promesa de aprovisionamiento automatizado que un SaaS multi-tenant necesita para no romperse al décimo operador, y cualquier decisión que privilegie un orquestador pesado desde el primer tenant queda fuera del sobre de costo.

La pregunta operacional relevante entonces no es "cuánto podemos construir", sino **qué compramos como plataforma y qué construimos como producto**. El core del producto FlowRide es el motor de asignación, la trazabilidad y la integración con WhatsApp — no un plano de control multi-tenant hecho a mano. Todo lo que no sea core del producto debería delegarse a plataforma probada siempre que el costo y el aislamiento lo permitan.

## Decisión

Empaquetar el backend .NET 10 como **imagen Docker inmutable** almacenada en **Azure Container Registry**, y desplegar cada operador como un stack lógico independiente en **Azure Container Apps**, con la base de datos Silo en **Azure Database for PostgreSQL Flexible Server** dedicada por operador.

El aprovisionamiento de un operador nuevo se automatiza con **Bicep** como IaC declarativo, invocado por un plano de control interno mínimo (catálogo de operadores, credenciales, orquestación de migraciones EF Core, salud agregada). Ese plano de control no construye orquestación, autoescalado ni balanceadores; solo coordina las APIs de Azure.

El escalado horizontal se aplica **según el SLA del tier**, no por defecto:

- **Canal WhatsApp (99,9%)** — Container App configurada con `minReplicas=2` para mantener dos réplicas siempre calientes y eliminar cold start. Autoescalado a más réplicas vía KEDA cuando la profundidad de la cola HTTP o el consumo de CPU lo exijan.
- **Motor de asignación y operaciones críticas (99,5%)** — Container App con `minReplicas=1` caliente y autoescalado bajo demanda. Restart automático y rolling deploy nativos.
- **Panel web administrativo (99%)** — Container App con `minReplicas=0` (escala a cero cuando no hay tráfico) y `maxReplicas=3`. El cold start ocasional al primer acceso administrativo del día es aceptable dentro del SLA 99% y reduce el costo de operadores con panel poco usado.

El estado crítico (sesiones, colas, caché compartido cuando aparezca) se mantiene fuera del proceso del backend para que las réplicas sean intercambiables sin coordinación. Los jobs de fondo (Outbox Processor, ETL a la base de reportes) se ejecutan con leader election basado en advisory locks de PostgreSQL para evitar ejecuciones duplicadas entre réplicas.

La decisión se revisa formalmente cuando el número de operadores activos supere 40 o cuando el equipo detecte que las limitaciones de Container Apps (por ejemplo, control fino sobre networking, sidecars complejos o requisitos avanzados de service mesh) empiecen a bloquear features del producto. En ese punto, la imagen Docker inmutable permite migrar a **Azure Kubernetes Service** sin rearquitectura.

## Alternativas consideradas

### Alternativa A — Máquinas virtuales manuales, sin contenedores

Provisionar una VM (Azure VM, Amazon EC2 o Google Compute Engine) por operador y desplegar el binario .NET directamente sobre el sistema operativo host. Actualizaciones y parches por SSH.

**Descartada** por dos razones. La operación manual no escala a 20 operadores sin duplicar personal: cada onboarding, cada migración, cada parche del sistema operativo multiplica trabajo. Y la ausencia de imagen inmutable rompe reproducibilidad — dos VMs supuestamente idénticas divergen en semanas por drift de dependencias y de versiones del sistema operativo.

### Alternativa B — Kubernetes gestionado desde el primer operador (AKS, EKS o GKE)

Levantar un clúster Kubernetes gestionado (Azure Kubernetes Service, Amazon EKS o Google Kubernetes Engine) desde el primer operador y modelar cada uno como un namespace con sus propios deployments, secrets y persistent volumes.

**Descartada** por costo y complejidad desproporcionados al volumen inicial. Un clúster mínimo con plano de control pagado y dos nodos worker parte de ~150 USD/mes fijos antes del primer operador, lo que rompe el sobre de 20–40 USD/operador con solo cinco activos. Además, la superficie operacional real de Kubernetes (RBAC, Ingress, cert-manager, HPA, NetworkPolicies, upgrades del clúster, gestión de CRDs) es sustancial para un equipo de tres personas sin experiencia previa. El riesgo de outage por mala configuración excede el que se está tratando de mitigar. Se mantiene como camino de migración futura cuando el volumen justifique el overhead.

### Alternativa C — Funciones serverless (Azure Functions o AWS Lambda)

Descomponer el backend en funciones serverless que se aprovisionan por operador y escalan a cero cuando no hay tráfico. Aislamiento por suscripción o por prefijo de recurso.

**Descartada** por incompatibilidad estructural con el sistema. El webhook de WhatsApp es un endpoint persistente y sensible a latencia; el cold start típico de una función fría (500ms–3s en la primera invocación) impacta directamente el SLA 99,9% y degrada la experiencia conversacional. El motor de asignación con reglas configurables y el ETL asíncrono no encajan en el modelo de función corta y aislada. Mantener el equivalente de un monolito modular como veinte funciones acopladas por operador multiplica la superficie operacional en lugar de reducirla.

### Alternativa D — PaaS de aplicaciones clásico (Azure App Service o AWS Elastic Beanstalk)

Desplegar la aplicación como servicio gestionado con una instancia por operador, donde la plataforma administra el sistema operativo, el runtime y el balanceador.

**Descartada** por costo. Una instancia con SLA razonable en Azure App Service (plan P0v3 o P1v3) parte de ~85 USD/mes en tier Premium, sin contar la base de datos; con la BD gestionada aparte el costo por operador sube a 120–180 USD/mes. Eso multiplica por 3–5 el sobre presupuestado y no deja margen para el canal WhatsApp variable ni para el motor de telemetría. El modelo de facturación por instancia siempre encendida penaliza especialmente a operadores pequeños con paneles administrativos poco usados.

### Alternativa E — Contenedores Docker con plano de control propio (self-hosted)

Empaquetar el backend como imagen Docker inmutable y desplegarlo sobre VMs propias, construyendo internamente el plano de control: aprovisionamiento automatizado por scripts, health checks, restart, balanceador con retiro de instancias no sanas, rolling deploy y catálogo de operadores.

**Descartada** por costo oculto de construcción y mantenimiento. Aunque el costo directo de infraestructura sería competitivo (~20–35 USD/operador), el equipo tendría que construir y operar durante meses un plano de control cuyas capacidades ya vienen resueltas en las plataformas gestionadas. Cada bug del plano de control es un incidente que afecta a todos los operadores simultáneamente, y cada hora invertida en mantenerlo es una hora que no se invierte en el motor de asignación, en la trazabilidad o en la integración con WhatsApp — que sí son core del producto. La portabilidad total entre proveedores que ofrece este camino no es un objetivo real: FlowRide ya depende de Meta para WhatsApp y va a depender de proveedores externos para identidad y telemetría.

### Alternativa F — AWS App Runner

Desplegar la imagen Docker en AWS App Runner, servicio gestionado que ejecuta contenedores con autoescalado, health checks y balanceador incluidos.

**Descartada** frente a la elegida por dos razones concretas. Su control sobre concurrencia mínima y réplicas mínimas es menos granular que el de Container Apps y Cloud Run; hasta muy recientemente no permitía forzar réplicas siempre calientes con la granularidad necesaria para sostener el SLA 99,9% del canal WhatsApp con costo predecible. Y su ecosistema alrededor (integración con RDS, tooling de observabilidad, IaC) es menos maduro para stacks .NET que las opciones de Azure y GCP.

### Alternativa G — Google Cloud Run + Cloud SQL para PostgreSQL

Desplegar la imagen Docker en Google Cloud Run, con la BD Silo por operador en Cloud SQL para PostgreSQL, aprovisionamiento vía Terraform y observabilidad en Cloud Logging.

**Considerada seriamente y descartada** por afinidad de ecosistema. Cloud Run es técnicamente comparable a Container Apps para este caso de uso: soporta `min-instances` para eliminar cold start, autoescala nativamente, tiene facturación por request muy competitiva, e integra con Cloud SQL para PostgreSQL. La diferencia decisiva es que el stack es .NET 10 y el equipo va a operar la plataforma sin haber trabajado antes con ninguno de los tres proveedores; en igualdad técnica, la afinidad natural entre .NET y Azure (tooling, documentación, integración con Application Insights, Azure Container Registry, Bicep) reduce fricción de aprendizaje. Se mantiene como plan B viable si Azure Container Apps impone restricciones bloqueantes durante la construcción.

### Alternativa H (elegida) — Azure Container Apps + Azure Database for PostgreSQL Flexible Server

Imagen Docker inmutable en Azure Container Registry, una Container App por servicio y por operador con réplicas mínimas ajustadas al SLA del tier, base de datos Silo en Azure Database for PostgreSQL Flexible Server dedicada por operador, aprovisionamiento con Bicep, observabilidad en Application Insights.

**Elegida** porque es la única alternativa que satisface simultáneamente cuatro condiciones que las otras no reúnen a la vez: el sobre de costo por operador (~20–40 USD/mes es alcanzable con Container Apps consumo + Flexible Server Burstable B1ms), el SLA diferenciado por tier (control granular sobre `minReplicas` por servicio permite eliminar cold start solo donde importa), la capacidad operativa realista de un equipo de tres personas sin experiencia previa en orquestación (KEDA, autoescalado, rolling deploy, health probes y traffic splitting vienen resueltos por la plataforma), y la afinidad natural con el stack .NET (Container Apps, Application Insights, Azure SDK para .NET, Bicep tienen la documentación y el tooling más maduros del ecosistema Microsoft).

## Consecuencias positivas

- Despliegue reproducible por operador desde una imagen inmutable versionada.
- Aislamiento consistente con el modelo Silo: cómputo dedicado por operador con contención de recursos, BD física dedicada, red aislada.
- Costo predecible y compatible con el sobre de 20–40 USD/mes por operador en el rango de 5 a 40 operadores activos.
- Autoescalado, health checks, rolling deploy, traffic splitting y balanceador delegados a plataforma probada, no construidos internamente.
- El equipo invierte tiempo en el motor de asignación, la trazabilidad y la integración con WhatsApp — no en operar infraestructura.
- Escalado diferencial por tier: se paga replicación permanente solo donde el SLA lo exige.
- La imagen Docker es portable a Azure Kubernetes Service sin rearquitectura cuando el volumen justifique migrar.
- Bicep como IaC permite versionar la infraestructura junto con el código, con revisiones por pull request.

## Consecuencias negativas

- Dependencia de Azure como proveedor. La imagen es portable, pero el plano de control (Bicep, Application Insights, KEDA sobre Container Apps) es específico. Cambiar de proveedor implica reescribir la infraestructura declarativa.
- Container Apps tiene límites concretos que hay que respetar: máximo de réplicas por revisión, restricciones sobre networking avanzado y sobre sidecars complejos. Si el producto necesita features que caen fuera de esos límites, el camino natural es migrar a AKS.
- El cold start del panel administrativo con `minReplicas=0` produce latencia de ~2–4s en el primer acceso del día. Es aceptable dentro del SLA 99% pero hay que documentarlo para operadores nuevos.
- El costo crece linealmente con el número de operadores sin economías de escala relevantes hasta llegar al umbral de migración a AKS.
- El leader election de jobs de fondo mediante advisory locks de PostgreSQL agrega complejidad al código de la aplicación que no existiría con una réplica única. Está resuelto por la arquitectura del Outbox, pero es carga cognitiva permanente.
- Bicep es específico de Azure. Alternativas más portables (Terraform, Pulumi) existen y son perfectamente utilizables sobre Container Apps; se prefirió Bicep por integración nativa y por menor curva de aprendizaje, asumiendo el trade-off.

## Tácticas de Bass aplicadas

- **Incrementar recursos disponibles** — autoescalado por KEDA en respuesta a métricas de carga y de cola HTTP.
- **Mantener múltiples copias del cómputo** — `minReplicas=2` en el canal WhatsApp para redundancia activa.
- **Detección de fallos** — health probes de liveness y readiness por Container App.
- **Recuperación tras fallo** — restart automático de réplicas no sanas y retiro del balanceador nativo durante rolling deploy.
- **Reconfiguración dinámica** — traffic splitting entre revisiones para despliegues progresivos y rollback instantáneo.
- **Aislamiento del fallo** — Silo garantiza que un operador caído no arrastra al resto; Container Apps refuerza técnicamente ese aislamiento con cómputo y BD dedicados.

## Escenarios y requisitos relacionados

- `COND-ESC-01` — hasta 200 usuarios simultáneos por instancia; validado por el dimensionamiento de recursos por réplica y el autoescalado KEDA.
- `ADM-DISP-01` — recuperación de operaciones críticas dentro del SLA 99,5%.
- `OPLOG-DISP-02` — panel web con SLA 99%.
- Escenarios del canal WhatsApp con SLA 99,9% — sostenidos por réplicas siempre calientes.

## ADRs relacionados

- **ADR de arquitectura de aplicación (monolito modular)** — este ADR define cómo se ejecuta esa unidad por operador y cómo se replica cuando el SLA lo exige.
- **ADR de modelo de aislamiento multi-tenant (Silo)** — este ADR resuelve las consecuencias operacionales que aquel deja abiertas: aprovisionamiento, migraciones por instancia, monitoreo agregado.
- **ADR de caché** — la existencia de múltiples réplicas en el canal WhatsApp exige externalizar el caché o aceptar inconsistencia temporal por réplica; la política concreta se define en ese ADR.
- **ADR de WhatsApp** — el SLA 99,9% del canal es el driver principal de la replicación permanente elegida aquí.
- **ADR de notificaciones y reportes asíncronos** — el patrón Outbox con leader election vía advisory locks de PostgreSQL se define allá y este ADR lo asume como resuelto.
- **ADR de telemetría GPS** — el volumen sostenido de ingesta (~144.000 eventos/día por instancia) es entrada para el dimensionamiento definido aquí.

## Notas de implementación

- Construir imagen Docker inmutable versionada por tag semántico; configuración exclusivamente por variables de entorno y secretos externos (Azure Key Vault referenciado desde Container Apps).
- Ejecutar migraciones EF Core como job independiente previo al arranque de nuevas revisiones, coordinado desde el plano de control; nunca como parte del entrypoint del contenedor bajo múltiples réplicas.
- Jobs de fondo (Outbox Processor, ETL a la base de reportes) deben usar leader election vía `pg_advisory_lock` cuando existan varias réplicas; nunca en paralelo sin coordinación.
- Métricas a monitorear en Application Insights antes de ajustar `maxReplicas`: CPU sostenida por réplica, memoria residente, latencia P95 por endpoint, profundidad de cola HTTP, número de conexiones abiertas a Flexible Server, throughput de ingesta de telemetría.
- Backup automatizado por instancia de Flexible Server con retención alineada a la política de retención de datos (24 meses operativos, 6 meses de telemetría, 5 años de auditoría).
- El plano de control expone endpoint interno de salud agregada y catálogo de operadores; su seguridad no es negociable y no se expone públicamente.
- Reevaluar formalmente la adopción de AKS cuando el número de operadores activos supere 40, o cuando aparezca un requisito de producto (networking avanzado, sidecars complejos, service mesh) que Container Apps no cubra sin workarounds.

## Referencias

- Bass, Clements, Kazman — *Software Architecture in Practice*, 4ª edición, capítulos sobre disponibilidad y escalabilidad.
- Microsoft Learn — Azure Container Apps: escenarios de aplicaciones multi-tenant, autoescalado con KEDA, revisiones y traffic splitting.
- Microsoft Learn — Azure Database for PostgreSQL Flexible Server: dimensionamiento, backup y alta disponibilidad.
- Microsoft Learn — Bicep como lenguaje de infraestructura declarativa para Azure.
