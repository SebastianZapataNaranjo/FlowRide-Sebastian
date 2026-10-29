# ADR-0002: aislar cada operador con un stack Silo propio (API + Worker + BD) y un control plane fuera del camino crítico

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0002 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando la prueba de aislamiento `PLAT-AIS-01` pase en el compose con dos stacks (tenant A y tenant B). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0002 del xlsx (que resolvía el tenant con un claim `tenantid` en el JWT), ADR-02 del tradeoff, `adr-previos/serie-11-adr/adr-0002` |
| Estado de implementación | Diseñado |

## Contexto

En FlowRide un **tenant es un operador de transporte**, no la empresa cliente que ese operador atiende (aerolínea, minera, hospital). El aislamiento entre operadores es una cláusula del contrato SaaS (contexto v10 §8.5) y un diferenciador (diferenciador 1). Dentro de cada operador, `empresaId` identifica al cliente empresarial y **nunca** funciona como frontera de seguridad.

Las fuentes ya habían elegido Silo, pero dejaban dos problemas:

1. **Una resolución de tenant propia de Pool dentro de un Silo.** El ADR-0002 del xlsx resolvía el tenant leyendo un claim `tenantid` del JWT. Si cada operador tiene su propio stack, ese claim sobra y además abre una superficie: un token con el claim equivocado podría apuntar a otra base.
2. **El canal WhatsApp no trae token.** El pasajero y el conductor escriben a un número. La pregunta real es *cómo llega cada mensaje al stack correcto*.

La investigación cerró el segundo punto: Meta permite **sobrescribir la URL del webhook por número de teléfono** (`POST /<PHONE_NUMBER_ID>` con `webhook_configuration.override_callback_uri`) para el campo `messages` ([Webhook overrides](https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override), consulta 2026-10-28). Así, cada número de operador entrega sus mensajes directamente al stack de ese operador.

## Decisión

1. **Unidad de aislamiento = stack por operador**: un proceso API, un proceso Worker y una base de datos PostgreSQL propia, con credenciales propias. Ningún proceso de un operador tiene la cadena de conexión de otro.
2. **El tenant lo fija el despliegue, no la petición.** La configuración del stack (cadena de conexión, `phone_number_id`, secretos del canal) se inyecta al arrancar. No existe en el código una función del tipo "resolver tenant a partir de la petición": el stack no puede apuntar a otra base aunque quisiera.
3. **Enrutamiento del canal**: cada número de WhatsApp del operador tiene su `override_callback_uri` apuntando a su stack. En el compose local, el número de prueba de Meta apunta al stack A (por túnel HTTPS) y el simulador `whatsapp-sim` solo conoce la URL del stack B (ADR-0005).
4. **Control plane (`flowride_control_plane`) fuera del camino crítico.** Solo guarda el catálogo de operadores, el estado del aprovisionamiento y las *referencias* a secretos (nunca los secretos). Lo usa la herramienta de aprovisionamiento: crear la base, correr migraciones, registrar el número y configurar el override. **Ningún mensaje ni asignación consulta el control plane en tiempo de ejecución.** Si el control plane cae, los operadores siguen operando; solo se detiene el alta de operadores nuevos.
5. **Webhooks que Meta no permite sobrescribir** (estado de plantillas y de cuenta) llegan a la URL por defecto de la app. Ahí los recibe un endpoint del control plane que actualiza el estado de las plantillas por operador. Queda **Diseñado**: la ruta E2E no depende de plantillas (ADR-0005).

## Alternativas consideradas

Pesos tomados de la prioridad global de los Drivers. El costo es restricción eliminatoria.

| Criterio (peso) | Pool (BD compartida, `tenant_id`) | Bridge (esquema por tenant) | Silo de datos (BD por tenant, cómputo compartido) | **Silo de stack (elegida)** |
|---|---|---|---|---|
| Disponibilidad / blast radius (11) | 2 — una falla tumba a todos | 2 | 3 — cómputo y control plane compartidos en el camino crítico | 5 |
| Seguridad (10) | 2 — un filtro olvidado expone datos | 3 | 4 | 5 — no hay credencial de otro operador en el proceso |
| Confiabilidad (8) | 4 | 4 | 4 | 4 |
| Aislamiento (7) | 1 | 3 — mismo motor y credencial | 4 | 5 |
| Escalabilidad (2) | 5 | 4 | 4 | 3 |
| Mantenibilidad / operación (1) | 5 | 3 — migraciones por esquema | 3 | 2 — N stacks que operar |
| **Promedio ponderado** | 2,46 | 2,97 | 3,69 | **4,62** |
| Costo 20–40 USD/operador | Cumple (el más barato) | Cumple | Cumple (~10–20) | Cumple con 1 réplica (~28–31); **no** con 2 réplicas fijas (~40–46) — ADR-0009 |

Cálculo: Σ(peso × puntaje) / 39. Para la opción elegida: (11·5 + 10·5 + 8·4 + 7·5 + 2·3 + 1·2) / 39 = 180 / 39 = 4,62.

**El Silo de datos** era la alternativa más fuerte. Era, además, lo que asumía la versión anterior de CLAUDE.md (una API que atiende dos bases). Se descarta porque obliga a resolver el tenant en cada mensaje (por `phone_number_id` contra el control plane). Eso pone el control plane en el camino crítico de **todos** los operadores y deja las credenciales de varias bases dentro del mismo proceso.

## Consecuencias

**Positivas**
- La cláusula contractual "sus datos viven en su propia base física y ningún proceso de otro operador puede alcanzarla" se demuestra con la configuración, no con disciplina en las consultas.
- Un pico, un bug o un ataque al número de un operador no degrada a los demás.
- Backup, restauración y migración por operador son independientes.
- No existe código de resolución de tenant que se pueda equivocar.

**Negativas**
- El costo crece linealmente con los operadores, sin economías de escala, y la segunda réplica no cabe en el techo (ADR-0009).
- Las migraciones de EF Core hay que correrlas N veces. La herramienta de aprovisionamiento las corre por operador, nunca dentro del arranque del contenedor.
- **Queda un secreto compartido**: el *app secret* de la app de Meta firma los webhooks de todos los números. Si se compromete, afecta a todos los operadores (ADR-0005 y ADR-0014).
- La tabla de números y overrides del control plane tiene que mantenerse sincronizada con Meta. Un override mal configurado entregaría los mensajes del operador A al stack B. Se mitiga validando en el stack que el `metadata.phone_number_id` del webhook coincida con el `phone_number_id` configurado; si no coincide, se rechaza y se audita.

## Tácticas de Bass

| Táctica | Dónde |
|---|---|
| Separar entidades (aislamiento de recursos) | Una base, una credencial y un proceso por operador |
| Limitar el acceso | El proceso solo tiene su propia cadena de conexión |
| Limitar el impacto / contener el fallo | Una falla queda dentro del stack del operador |
| Detectar intrusiones (validar la entrada) | Comparación del `phone_number_id` del webhook con el configurado |

## Escenarios y requisitos relacionados

- `PLAT-AIS-01` — un mensaje destinado al operador B nunca produce lecturas ni escrituras en la base de A (escenario principal).
- `OPLOG-SEG-02`, `ADM-SEG-02` (banco de 33) — aislamiento desde el panel; quedan Diseñados porque el panel es V1 (ADR-0011).
- RF-19 (aislamiento físico), RF-20 (configuración por tenant), RF-03 (registro de operador: Diseñado; en la línea base el aprovisionamiento es un script).

## Condiciones de revisión

- Si la plataforma supera unos 40 operadores y el costo lineal deja de ser sostenible, evaluar el Silo de datos con varios stacks agrupados por segmento, manteniendo los operadores grandes en Silo completo.
- Si Meta elimina el override por número, la entrada pasa a un router de webhooks compartido (ADR-0015, borde común) y este ADR se revisa por el nuevo punto de falla común.
- Si un operador exige alta disponibilidad, se aplica la segunda réplica como opción contractual (ADR-0009), sin cambiar este ADR.

## Referencias

- Meta — Webhook overrides: https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override (consulta 2026-10-28).
- `docs/_fuentes/CONTEXTO_FLOWRIDE_v10.md` §7 (diferenciador 1), §8.5 y §16.
- `docs/_fuentes/ADR_Decisiones_Arquitectonicas_FlowRide_LISTA.xlsx` — ADR-0002 (versión previa).
- ADR-0005 (canal y simulador), ADR-0009 (costo y despliegue), ADR-0014 (secretos).
