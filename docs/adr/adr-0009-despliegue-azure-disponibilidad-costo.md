# ADR-0009: desplegar cada stack en Azure Container Apps + PostgreSQL Flexible Server, con disponibilidad compuesta calculada y alta disponibilidad como opción contractual

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0009 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — el entorno de la línea base es `docker compose` (local). Azure está **Diseñado**; este ADR pasa a `Accepted` con un despliegue piloto que mida el costo real de un mes contra la tabla de §3. |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | `adr-previos/docs-adr-sep2026/adr-0007` (ACA con `minReplicas=2` para el canal: rompe el techo de costo) y `…/adr-0010` (stack por tenant sin plataforma concreta); ADR-0010 del xlsx; ADR-08 del tradeoff |
| Estado de implementación | Diseñado (Azure). El compose local está especificado en `delivery/ejecucion-local.md` |

## Contexto

- ADR-0002 fija un stack por operador (API + Worker + BD). ADR-0001 y ADR-0003 fijan dos procesos sin estado salidos de la misma imagen.
- **Techo**: 20–40 USD/mes de infraestructura básica por operador, sin contar WhatsApp (restricción de negocio).
- **SLA declarado en las fuentes**: canal WhatsApp 99,9 %, motor 99,5 %, panel 99 %. Decisión de la ronda 1 (P4): la base pasa a **99,5 % de disponibilidad del stack + "0 mensajes perdidos"**, y la redundancia se ofrece como opción contractual.
- El profesor mencionó Azure. Los lineamientos del curso Ingeniería del Software 2 piden verificar precios y niveles vigentes antes de elegir.
- La franja crítica del caso ancla es 03:00–05:00 (arranque de tripulaciones).

## Decisión

### 1. Topología por operador

| Recurso | Configuración | Por qué |
|---|---|---|
| **Una Container App por operador con dos contenedores** (`api` y `worker`, misma imagen y distinto comando) | 0,25 vCPU / 0,5 GiB cada uno (0,5 vCPU / 1 GiB por réplica); `minReplicas=1`, `maxReplicas=1` en la base | Mantiene la separación de procesos (si el Worker se reinicia, la API sigue recibiendo webhooks) **sin pagar dos apps**. Una sola réplica, porque la outbox con `SKIP LOCKED` admite más réplicas sin cambiar el código (ADR-0006). |
| Ingress | Externo, `allowInsecure: false` (HTTP → HTTPS), `targetPort` HTTP del contenedor | TLS termina en Envoy ([ACA ingress](https://learn.microsoft.com/en-us/azure/container-apps/ingress-how-to)) |
| **Azure Database for PostgreSQL Flexible Server** por operador | Burstable **B1ms**, 32 GB, backup automático; `sslmode=require` | Silo físico (ADR-0002); `btree_gist` disponible (ADR-0004) |
| Key Vault por operador | Credencial de la BD, token de Meta del operador y API key de administración; acceso por identidad administrada del Container App | ADR-0014 |
| Compartidos de plataforma (sin datos operativos) | Azure Container Registry Basic, Log Analytics, Key Vault de plataforma (app secret de Meta) y control plane | Prorrateados |

Migraciones: un *job* de Container Apps por operador, antes de cada revisión nueva, nunca en el arranque del contenedor. Despliegue en revisión única fuera de la franja 03:00–05:00.

### 2. Disponibilidad compuesta (lo que se puede prometer y lo que no)

SLA publicados (consulta 2026-10-28): Container Apps **99,95 %** (página de producto, [Azure Container Apps](https://azure.microsoft.com/en-us/products/container-apps); el documento formal de SLA queda por verificar); PostgreSQL Flexible Server **sin HA: 99,9 %** ([Reliability](https://learn.microsoft.com/en-us/azure/reliability/reliability-postgresql-flexible-server)); Meta Cloud API **sin SLA comercial**, 99,9 % observado ([Support FAQ](https://developers.facebook.com/docs/whatsapp/cloud-api/support/)). Queda por confirmar si el 99,9 % de Flexible Server aplica también al nivel Burstable (`preguntas-abiertas.md`).

Componentes en serie (todos tienen que estar arriba):

| Cadena | Cálculo | Disponibilidad | Caída permitida (mes de ~730 h) |
|---|---|---|---|
| Techo de la plataforma del stack (ACA × PostgreSQL) | 0,9995 × 0,999 | **99,85 %** | ~66 min |
| Lo mismo, incluyendo Meta (lo que percibe el usuario) | 0,9995 × 0,999 × 0,999 | **99,75 %** | ~110 min |
| **Objetivo comprometido del stack** | — | **99,5 %** | ~219 min |
| Margen para fallas propias, despliegues y migraciones | 219 − 66 | — | ~153 min |
| Lo que percibe el usuario si el stack cumple justo el 99,5 % | 0,995 × 0,999 | **99,40 %** | ~263 min |

**Conclusión para la sustentación**:
- **99,9 % no se puede prometer** en el canal. Solo la plataforma (ACA × PostgreSQL sin HA) ya deja un techo de 99,85 %, y Meta, que está en la cadena, no garantiza nada.
- Por eso la medida cambia de forma: **"0 mensajes perdidos"**. Mientras el stack está caído, Meta reintenta durante hasta 7 días ([Set up webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks)) y el inbox absorbe los duplicados (ADR-0006). Una caída **retrasa** el procesamiento, no lo pierde. Esto se mide aparte de la disponibilidad (`PAS-DISP-01`).

### 3. Costo por operador (precios minoristas oficiales, East US 2, 730 h/mes)

Fuente: [Azure Retail Prices API](https://prices.azure.com/api/retail/prices) y [ACA pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/), consulta 2026-10-28.

| Componente | Precio unitario | USD/mes |
|---|---|---|
| ACA — 0,5 vCPU + 1 GiB por réplica, en reposo todo el mes | vCPU en reposo 3e-6 USD/s; memoria 3e-6 USD/GiB-s | 11,8 |
| ACA — la misma réplica activa ~10 % del tiempo | vCPU activa 2,4e-5 USD/s | **~14,6** |
| PostgreSQL B1ms | 0,017 USD/h | 12,4 |
| Almacenamiento 32 GB | 0,115 USD/GB-mes | 3,7 |
| Backup | Gratis hasta el 100 % del almacenamiento provisionado (conocimiento previo, por verificar) | ~0 |
| Log Analytics | Primeros 5 GB gratis por cuenta; luego 2,76 USD/GB | ~0–1,4 (estimado de 0,5 GB/operador) |
| ACR Basic | 0,1666 USD/día ≈ 5,07/mes, prorrateado entre 5 operadores | ~1,0 |
| Key Vault | Por operación; uso bajo | < 1 |
| **Total base** | | **~33–35** |
| Bolsa gratis de ACA (180.000 vCPU-s y 360.000 GiB-s por suscripción) | No se descuenta por operador | — |

**Cabe en el techo de 20–40 USD**, con poco margen. El tráfico de Meta (9–15 USD, ADR-0005) va aparte, como dice la restricción.

### 4. Opciones contractuales de disponibilidad (P4c)

| Nivel | Qué cambia | USD/mes adicional | Efecto |
|---|---|---|---|
| Base | 1 réplica, B1ms | 0 | Objetivo de 99,5 % del stack |
| **Plus** | `minReplicas=2` en la Container App | ~+12–15 | Elimina la caída por reinicio de réplica y por despliegue. La BD (99,9 %) sigue siendo el techo |
| **Crítico** | Plus + PostgreSQL General Purpose 2 vCore con HA de zona | ~+260 (2 vCore × 0,178 USD/h × 730 h ≈ 130 por nodo × 2 nodos) | La BD pasa a 99,99 % con HA de zona. **El nivel Burstable no soporta HA** ([HA concepts](https://learn.microsoft.com/en-us/azure/postgresql/high-availability/concepts-high-availability)). Solo es viable con un precio cercano al techo comercial de 500 USD/mes |

El escenario `ADM-DISP-01` del banco (failover ≤ 30 s) **solo se cumple con el nivel Crítico**. Por eso pasa a ser una cláusula contractual, no una promesa de la base.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| `minReplicas=2` para todos (fuente anterior) | ~40–46 USD/mes: rompe el techo. Y la BD sin HA sigue siendo el límite. |
| App Service | Plan siempre encendido; Premium v3 parte de unos 85 USD/mes (dato del ADR previo, no verificado aquí). |
| AKS | Plano de control y nodos fijos antes del primer operador; operación desproporcionada para 5 operadores. |
| Azure Functions | Arranque en frío en el webhook; Worker de outbox persistente. |
| Google Cloud Run + Cloud SQL | Técnicamente comparable; se mantiene como plan B. Azure se elige por afinidad con .NET y porque el profesor lo mencionó. |
| BD compartida por varios operadores en un servidor más grande | Es Silo de datos o Bridge: lo descarta ADR-0002. |

## Consecuencias

**Positivas**: el costo por operador está explicado línea por línea; el SLA prometido es alcanzable y se dice qué no se puede prometer; la redundancia se paga solo donde un contrato la exige.
**Negativas**: con una réplica, cada despliegue causa unos segundos de indisponibilidad (se hacen fuera de la franja crítica y Meta reintenta); el margen frente al techo es corto (~5 USD); el costo crece linealmente con los operadores; la plataforma compartida (ACR, Key Vault de plataforma) es un punto de falla común fuera del camino de los mensajes.

## Tácticas de Bass

Detectar fallos (sondas de liveness y readiness); recuperarse (reinicio automático del contenedor); redundancia activa (nivel Plus); incrementar recursos (más vCPU o réplicas); mantener copias de los datos (backup automático, HA en el nivel Crítico).

## Escenarios y requisitos relacionados

`PAS-DISP-01` y `COND-DISP-01` (reformulados), `ADM-DISP-01` (pasa a opción contractual), `PLAT-AIS-01`; restricciones de costo y disponibilidad; RF-53 (monitoreo técnico).

## Condiciones de revisión

- Si el costo real medido supera 40 USD por operador, revisar el tamaño de la réplica o mover los logs a un nivel de retención más barato.
- Si más de 40 operadores, evaluar AKS o agrupar operadores pequeños (ADR-0002).
- Si Meta publica un SLA, recalcular §2.

## Referencias

Todas las URL citadas, consultadas el 2026-10-28.
