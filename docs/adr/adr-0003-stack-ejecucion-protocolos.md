# ADR-0003: stack .NET 10, modelo de ejecución no reactivo con E/S asíncrona y protocolo explícito por cada par de componentes

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0003 |
| Fecha | 2026-10-28 |
| Estado | **Proposed** — pasa a `Accepted` cuando el compose de la línea base levante API y Worker con HTTPS y la prueba de contrato del webhook pase (escenarios `CANAL-SEG-01` y `PAS-DISP-01`). |
| Autor y defensor | Sebastián Zapata Naranjo |
| Reemplaza | ADR-0017 del xlsx (lenguaje), que pasa a ser restricción técnica justificada aquí; "HTTPS local desactivado" del contexto v10 §22 |
| Estado de implementación | Diseñado |

## Contexto

El profesor pidió explícitamente un ADR que responda: stack backend, modelo de ejecución (reactivo, no reactivo o mixto) y protocolo entre cada par de componentes, con REST, GraphQL, gRPC, WebSocket y SSE como opciones a considerar (lineamientos del curso Ingeniería del Software 2).

Hechos que acotan la respuesta:

- **No hay frontend en la entrega E2E.** Los usuarios operativos (pasajero, conductor, operador) usan WhatsApp. El panel web es V1 (ADR-0011).
- **Las interfaces externas las fija Meta**: los mensajes entran como webhook HTTPS con JSON firmado y salen por la Graph API (HTTPS/REST). FlowRide no puede elegir otro protocolo con Meta.
- **La carga es de E/S, no de CPU**: esperar a la base, esperar a Meta. El motor evalúa como mucho 50 conductores por solicitud.
- **Volumen bajo y en ráfagas**: ~40 servicios al día, picos en el arranque de turno (03:00–05:00).
- **El equipo** conoce C# y .NET desde el semestre de DOO, y la restricción técnica vigente fija C# sobre .NET 10.

## Decisión

### 1. Stack (restricción técnica justificada)

| Elemento | Elección | Justificación |
|---|---|---|
| Lenguaje y runtime | C# sobre **.NET 10 (LTS)** | Experiencia previa del equipo; versión con soporte extendido; tipado estático que refuerza los límites de ADR-0001; un solo runtime para API, Worker y pruebas. |
| Web | ASP.NET Core con **controladores** (`Infraestructura.Input.Controladores`) | Coincide con la estructura del profesor (`infraestructura.input.controladores`). Minimal APIs serían igual de válidas, pero romperían esa correspondencia. |
| Procesos de fondo | `BackgroundService` en un host separado (`Inicializador.Worker`) | Procesa inbox y outbox (ADR-0006) sin competir con los webhooks. |
| Acceso a datos | EF Core + Npgsql (ADR-0004) | — |
| Serialización | `System.Text.Json` | Viene en el framework; no hay requisito de otro formato. |
| Resiliencia HTTP | `Microsoft.Extensions.Http.Resilience` sobre `HttpClient` tipado | Reintentos con backoff y *circuit breaker* hacia la Graph API (ADR-0005). |
| Contrato de la API | OpenAPI con Swagger UI, solo para el rol Administrador técnico | Contexto v10 §13: el Administrador técnico usa consola o Swagger. |

Lo que **no** es restricción sino preferencia queda en sus ADR (PostgreSQL en ADR-0004, Azure en ADR-0009).

### 2. Modelo de ejecución: no reactivo, imperativo, con E/S asíncrona

- Cada petición y cada evento de la outbox se procesa con código imperativo `async/await`. Los hilos no se bloquean mientras se espera a PostgreSQL o a Meta.
- **No** se usan streams reactivos (Rx.NET, System.Reactive, backpressure con `IObservable`). FlowRide no tiene flujos continuos de eventos que componer o transformar en V1: tiene transacciones discretas (un mensaje → un cambio de estado).
- La concurrencia entre peticiones la da el pool de hilos de ASP.NET Core. La concurrencia sobre los datos la controla ADR-0006, no el modelo de ejecución.
- **Stateless**: ningún proceso guarda estado de negocio ni conversacional en memoria. El estado de la conversación vive en `conversacion` (base de datos). Una réplica nueva o un reinicio no pierden nada.
- **Tiempo**: todas las marcas se guardan como `timestamptz` en UTC. Las reglas de calendario (turnos, "hoy/mañana", franjas) se evalúan en la zona del operador (`operador.zona_horaria`, por defecto `America/Bogota`), con un `IReloj` abstraído para las pruebas.

**Por qué no reactivo**: ataca un problema (flujos de alto volumen con contrapresión) que FlowRide no tiene en F1–F3. Agrega una curva de aprendizaje y hace más difícil depurar y explicar las transacciones. **Cuándo se reconsidera**: con la ingesta de telemetría GPS de V1 (~100 eventos/min por instancia, ADR-0013); aun ahí, un `Channel<T>` con E/S asíncrona probablemente alcanza.

### 3. Protocolo por cada par de componentes

| Origen → Destino | Protocolo | Autenticación / integridad | Estado |
|---|---|---|---|
| Pasajero, Conductor, Operador → WhatsApp (app de Meta) | Protocolo cliente de WhatsApp (fuera del control de FlowRide) | Cifrado del cliente de WhatsApp | Externo |
| Meta Cloud API → `api-{operador}` (webhook) | **HTTPS** POST, JSON | Firma `X-Hub-Signature-256` (HMAC-SHA256 del cuerpo crudo con el app secret). Verificación inicial `GET` con `hub.verify_token` | E2E (tenant A) |
| `whatsapp-sim` → `api-b` (webhook, doble de prueba) | **HTTPS** POST, JSON con el mismo contrato que Meta | Misma firma, con un **secreto propio del stack B** (ADR-0005) | E2E (tenant B, solo en compose) |
| `api-*` / `worker-*` → Meta Graph API | **HTTPS** REST, JSON (`POST /{phone_number_id}/messages`) | `Authorization: Bearer` (token del sistema del operador) | E2E (tenant A) |
| `worker-b` → `whatsapp-sim` (Graph API simulada) | **HTTPS** REST, el mismo contrato | Bearer de prueba | E2E (tenant B) |
| Administrador técnico (Swagger, curl) → `api-*` `/admin/*` | **HTTPS** REST, JSON | Encabezado `X-Api-Key` con una clave **por stack** (ver §4) | E2E |
| `api-*` y `worker-*` → PostgreSQL del operador | **TCP 5432**, protocolo de PostgreSQL (Npgsql) | Usuario y contraseña propios del operador. **Sin TLS dentro de la red interna del compose**; `sslmode=require` obligatorio en Azure | E2E (compose) / Diseñado (Azure) |
| `api-*` ↔ `worker-*` | **Sin llamadas directas.** Se integran a través de las tablas `mensaje_entrante` y `outbox_evento` de su propia base | Mismo usuario de base del stack | E2E |
| Orquestador (compose healthcheck / ACA probes) → `api-*` | HTTPS `GET /health/live`, `/health/ready` | Sin autenticación; no exponen datos | E2E |
| Herramienta de aprovisionamiento → `flowride_control_plane` y bases de operador | TCP 5432 | Credencial de plataforma, fuera de los stacks | Diseñado (en la línea base es un script SQL) |
| `worker-*` → Graph API (alerta al operador) | Igual que la notificación al conductor | — | E2E dentro de la ventana de 24 h |

**HTTPS**: en el compose, Kestrel publica **solo** el puerto HTTPS con el certificado de desarrollo de .NET (`dotnet dev-certs https`) montado en el contenedor. El puerto HTTP no se publica, así que una conexión HTTP es rechazada ([Docker over HTTPS, .NET 10](https://learn.microsoft.com/en-us/aspnet/core/security/docker-https?view=aspnetcore-10.0), consulta 2026-10-28). En Azure Container Apps el TLS termina en el ingress y `allowInsecure: false` redirige HTTP a HTTPS ([ACA ingress](https://learn.microsoft.com/en-us/azure/container-apps/ingress-how-to), consulta 2026-10-28). El contenedor no necesita certificado allí. Para que Meta llegue al stack A local se usa un túnel HTTPS público (dev tunnels o ngrok), documentado en `delivery/ejecucion-local.md`.

### 4. Credencial del Administrador técnico (debilidad declarada)

La API de administración (`/admin/*`: carga de datos maestros, consulta de solicitudes, asignaciones, alertas y auditoría) se protege con una **API key por stack**, leída de user-secrets o de una variable de entorno, nunca del repositorio.

- **Debilidad**: identifica al *stack*, no a la *persona*. La auditoría registra "Administrador técnico" como actor genérico.
- **Consecuencia**: la reasignación manual del Operador logístico con auditoría por persona (`OPLOG-REND-01`, `OPLOG-AUD-02`) y la revocación en ≤ 60 s (`ADM-SEG-01`) quedan **Diseñadas** con OIDC (ADR-0011). La API key no se presenta como la solución de identidad del panel.

## Alternativas consideradas

| Alternativa | Para qué serviría en FlowRide | Por qué no |
|---|---|---|
| **gRPC** | Llamadas internas tipadas entre servicios | No hay llamadas entre servicios (ADR-0001). Meta no habla gRPC. El administrador usa Swagger. Agregaría HTTP/2 y contratos `.proto` sin consumidor. |
| **GraphQL** | Consultas flexibles desde un frontend | No hay frontend en la entrega. La API de administración es chica y estable. Agrega superficie de ataque (profundidad de consultas, introspección) y caché más complejo. Se reevalúa con el panel de V1 si las vistas exigen combinar muchas entidades. |
| **WebSocket** | Alertas en vivo al panel | No hay panel en la entrega. La alerta al operador va por WhatsApp (decisión de la ronda 2). |
| **SSE** | Empujar alertas del servidor al panel | Es la candidata para el canal de alertas del panel en V1, porque el flujo es unidireccional (servidor → navegador) y funciona sobre HTTP normal. Queda **Diseñada** en ADR-0011. |
| **Broker de mensajes** entre API y Worker | Desacople asíncrono | Lo resuelve la outbox en la base (ADR-0006). |
| **Modelo reactivo** (Rx.NET) | Flujos continuos con contrapresión | Ver §2. |
| **Java/Spring** (stack del ejercicio académico) | Alinear con `uco-test-api` | FlowRide ya tiene la restricción .NET por experiencia del equipo. La equivalencia de paquetes con el ejercicio del profesor se mantiene (ADR-0001), aunque el lenguaje cambie. |

## Consecuencias

**Positivas**
- Cada flecha de los diagramas tiene un protocolo, una autenticación y un estado de implementación, y se puede probar.
- Un solo runtime para todo el stack: menos herramientas que aprender y operar.
- La integración API ↔ Worker por base de datos no agrega piezas de infraestructura.

**Negativas**
- Integrar API y Worker por base de datos acopla ambos procesos al esquema de `mensaje_entrante` y `outbox_evento`. Es aceptable porque viven en la misma solución y se despliegan juntos.
- La red interna del compose va sin TLS hacia PostgreSQL. Está declarado y solo vale para el entorno local.
- La API key no identifica personas (ver §4).
- El túnel HTTPS para la demo con Meta es una dependencia externa de la demo, no del producto.

## Tácticas de Bass

| Táctica | Dónde |
|---|---|
| Autenticar actores | Firma HMAC del webhook; API key del administrador; Bearer hacia Meta |
| Cifrar los datos (en tránsito) | HTTPS en toda flecha que cruza la red pública |
| Introducir concurrencia (rendimiento) | E/S asíncrona con `async/await` |
| Mantener múltiples copias (stateless) | Ningún estado en memoria; las réplicas son intercambiables |
| Adaptar la interfaz | El contrato de Meta se traduce en `Infraestructura` (ADR-0005) |

## Escenarios y requisitos relacionados

- `CANAL-SEG-01` (firma inválida rechazada), `PAS-DISP-01` (confirmación P95 < 10 s), `COND-DISP-01` (respuesta al conductor P95 < 5 s), `PLAT-AIS-01`.
- Restricciones técnicas: plataforma .NET 10; interoperabilidad HTTP/HTTPS + JSON y webhooks autenticados; seguridad de datos (cifrado en tránsito).

## Condiciones de revisión

- Cuando entre el panel web (V1): decidir entre SSE y WebSocket para las alertas, y reevaluar GraphQL si las vistas lo exigen.
- Cuando entre la telemetría GPS: reevaluar el modelo de ejecución para la ingesta.
- Si deja de haber soporte de .NET 10 antes de migrar a la siguiente LTS.

## Referencias

- Lineamientos del curso ingeniería del software 2 (ADR de plataforma y comunicación; UTC-0; stateless).
- Microsoft Learn — Docker over HTTPS (.NET 10) y ACA ingress, consulta 2026-10-28.
- Meta — Webhooks y Messages API, consulta 2026-10-28.
