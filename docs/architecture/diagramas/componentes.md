# Diagrama de componentes — dentro del stack de un operador

**Propósito**: mostrar los componentes internos de la API y del Worker, sus puertos y qué building block instancia cada uno en F1–F3.
**Alcance**: un stack (API + Worker + BD). Los nombres coinciden con `paquetes.md`.
**Estado**: Diseñado. `[Diseñado V1]` = no se construye para el 4-nov.

```mermaid
flowchart LR
    meta["⬜ Meta Cloud API / 🧪 whatsapp-sim"]
    adm["👤 Administrador técnico"]

    subgraph API["Contenedor API (ASP.NET Core)"]
        direction TB
        corr["CorrelationMiddleware<br/>(genera correlation_id)"]
        firma["FiltroFirmaWebhook<br/>HMAC-SHA256 + phone_number_id"]
        whc["WhatsAppWebhookController"]
        key["FiltroApiKey"]
        admc["Admin*Controller<br/>(datos maestros, consultas paginadas)"]
        ein["EscritorInbox"]
    end

    subgraph WRK["Contenedor Worker (.NET BackgroundService)"]
        direction TB
        pin["ProcesadorInbox"]
        pout["ProcesadorOutbox<br/>SKIP LOCKED, backoff"]
        bar["BarridoExpiraciones<br/>(cada minuto)"]
        subgraph CU["Servicios.CasoUso"]
            enr["EnrutarMensajeEntrante<br/>(por rol y estado de la conversación)"]
            conv["AvanzarConversacion / RegistrarItinerario (F1)"]
            chk["RegistrarCheckin"]
            asg["AsignarConductor (F2)"]
            resp["ResponderAsignacion (F3)"]
            fall["GestionarFallaAsignacion"]
            notif["Notificar"]
        end
        motor["MotorAsignacion + Reglas<br/>(Dominio)"]
        ng["NotificationGateway<br/>sesión vs. plantilla"]
        cli["ClienteGraphApi<br/>HttpClient + Resilience"]
    end

    subgraph CAT["Catálogos (Infraestructura)"]
        mc["MessageCatalog"]
        nc["NotificationCatalog"]
        pc["ParameterCatalog"]
    end

    subgraph REP["Repositorios.Estructurados (EF Core + Npgsql)"]
        rusr["UsuarioPorTelefono"]
        rsol["Solicitudes"]
        rasg["Asignaciones"]
        rflo["Flota"]
        rconv["Conversaciones"]
        raud["Auditoria"]
        eout["EscritorOutbox"]
    end
    db[("BD del operador")]
    idp["IdentityProvider OIDC<br/>[Diseñado V1 — ADR-0011]"]

    meta -- "HTTPS webhook" --> corr --> firma --> whc
    whc -- "puerto RecibirMensajeEntrante (Input.Ports)" --> ein
    adm -- "HTTPS REST, X-Api-Key" --> key --> admc
    ein -- "TCP" --> db
    pin --> enr
    enr --> conv & chk & resp
    pout --> asg & notif
    bar --> fall
    resp --> fall
    asg --> motor
    conv & chk & asg & resp & fall --> eout
    notif --> ng --> cli -- "HTTPS REST" --> meta
    ng --> mc & nc
    asg & fall & conv --> pc
    enr --> rusr
    conv --> rsol
    asg & resp & fall --> rasg
    asg --> rflo
    conv --> rconv
    conv & asg & resp & fall & chk --> raud
    admc --> rsol & rflo & raud & pc
    REP -- "TCP 5432" --> db
    admc -. "[V1]" .-> idp

    classDef dis fill:#f4f4f4,color:#555,stroke:#999,stroke-dasharray: 4 3
    class idp dis
```

## Building blocks por funcionalidad

| Building block (taxonomía del profesor) | Componente FlowRide | F1 | F2 | F3 | Nota |
|---|---|---|---|---|---|
| Adaptador de entrada (webhook) | `WhatsAppWebhookController` + `FiltroFirmaWebhook` | ✔ | | ✔ | HTTPS + firma (ADR-0005) |
| Notification gateway | `NotificationGateway` + `ClienteGraphApi` | ✔ | ✔ (alerta) | ✔ | Decide sesión o plantilla (ADR-0008) |
| Identidad del canal | Repositorio `UsuarioPorTelefono` | ✔ | | ✔ | No es un proveedor externo (ADR-0005) |
| Message catalog | `MessageCatalog` | ✔ | ✔ | ✔ | |
| Notification catalog | `NotificationCatalog` (plantillas) | | ✔ | ✔ | Camino fuera de la ventana: Diseñado |
| Parameter catalog | `ParameterCatalog` | ✔ | ✔ | ✔ | Rangos en código |
| Repositorio estructurado | `Repositorios.Estructurados` | ✔ | ✔ | ✔ | PostgreSQL por operador |
| Mensajería (productores y consumidores) | Inbox y outbox en PostgreSQL | ✔ | ✔ | ✔ | Sin broker (ADR-0006) |
| Concurrencia | `xmin` + restricción de exclusión | | ✔ | ✔ | |
| Reintento, backoff, *circuit breaker* | `ProcesadorOutbox` + Resilience | | | ✔ | `Fallido` = dead letter |
| Auditoría | `Auditoria` + historial | ✔ | ✔ | ✔ | Solo inserción |
| Logging + correlation ID | `CorrelationMiddleware`, ámbitos de log | ✔ | ✔ | ✔ | Propagado por la outbox |
| HTTPS | Kestrel (compose) / ingress de ACA (Azure) | ✔ | | ✔ | ADR-0003 |

**Building blocks que NO se instancian, con el criterio**

| Building block | Por qué no en F1–F3 |
|---|---|
| Caché | La disponibilidad del conductor es justo lo que se decide; servirla obsoleta rompe `COND-CONF-02`. Con ~40 servicios/día no hay driver de rendimiento (ADR-0012). |
| Message broker | ~400 eventos/día; la outbox en PostgreSQL alcanza y es atómica con el cambio de estado (ADR-0006). |
| Identity provider (OIDC) | Ningún usuario de F1–F3 usa navegador; entra con el panel (ADR-0011). |
| API Gateway / WAF compartido | El override de webhook por número entrega directo a cada stack; un borde común sería un punto de falla común (ADR-0015). |
| Telemetría | V1 (ADR-0013); no participa en la asignación (diferenciador 5). |
| Payment gateway | Sin cobro ni tarifario (exclusión de contexto v10 §6.1). |
| CDN | No hay contenido estático: no hay frontend en la entrega. |
| Generador de QR, acortador de URL, CAPTCHA | Ningún flujo los necesita; el CAPTCHA no aplica a un canal autenticado por la firma de Meta. |
| Service mesh, data mesh | Un solo proceso de aplicación por stack; no hay malla que gestionar. |
| Agentes de IA | Excluidos (contexto v10 §6.1); la conversación es determinista. |
| Storage account | Backups de la BD gestionados por Azure; archivo de auditoría Diseñado (ADR-0016). |
