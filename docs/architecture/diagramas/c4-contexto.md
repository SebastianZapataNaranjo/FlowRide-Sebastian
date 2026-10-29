# C4 nivel 1 — Contexto del sistema

**Propósito**: mostrar quién usa FlowRide y de qué sistemas externos depende para F1–F3.
**Alcance**: un operador de transporte (tenant). Cada operador ve exactamente este mismo contexto, aislado de los demás (ADR-0002).
**Estado**: FlowRide está *Diseñado*. Los elementos marcados `[Diseñado V1]` no forman parte de la entrega E2E.

```mermaid
flowchart TB
    pas["👤 Pasajero (tripulante)<br/>Registra su itinerario y recibe la confirmación"]
    con["👤 Conductor<br/>Hace check-in de turno, acepta o rechaza servicios"]
    ope["👤 Operador logístico<br/>Recibe alertas operativas por WhatsApp"]
    adm["👤 Administrador técnico (FlowRide)<br/>Carga datos maestros y consulta auditoría"]
    aem["👤 Administrador de empresa<br/>[Diseñado V1: panel web]"]

    fr["🟦 FlowRide<br/>Registro de itinerarios, asignación por reglas del operador,<br/>confirmación del conductor — un stack por operador"]

    meta["⬜ WhatsApp Business Platform<br/>(Meta Cloud API)"]
    idp["⬜ Proveedor de identidad OIDC<br/>[Diseñado V1 — ADR-0011]"]
    gps["⬜ Fuente de telemetría GPS<br/>[Diseñado V1 — ADR-0013]"]

    pas -- "Mensajes y botones<br/>[app WhatsApp]" --> meta
    con -- "Mensajes y botones<br/>[app WhatsApp]" --> meta
    ope -- "Lee alertas<br/>[app WhatsApp]" --> meta
    meta -- "Webhook de mensajes y estados<br/>[HTTPS POST, JSON, firma HMAC-SHA256]" --> fr
    fr -- "Envía mensajes, interactivos y plantillas<br/>[HTTPS REST, Graph API, Bearer]" --> meta
    adm -- "Administración<br/>[HTTPS REST, X-Api-Key]" --> fr
    aem -. "Panel web<br/>[HTTPS, OIDC]" .-> fr
    fr -. "Valida tokens<br/>[HTTPS, OIDC/JWKS]" .-> idp
    gps -. "Eventos de ubicación<br/>[HTTPS, JSON]" .-> fr

    classDef persona fill:#08427b,color:#fff,stroke:#052e56
    classDef sistema fill:#1168bd,color:#fff,stroke:#0b4884
    classDef externo fill:#8a8a8a,color:#fff,stroke:#5e5e5e
    classDef disenado fill:#f4f4f4,color:#555,stroke:#999,stroke-dasharray: 4 3
    class pas,con,ope,adm persona
    class fr sistema
    class meta externo
    class aem,idp,gps disenado
```

**Leyenda**: azul oscuro = persona; azul = el sistema; gris = sistema externo; borde discontinuo = Diseñado V1 (no forma parte de la entrega E2E). Línea continua = relación E2E; línea punteada = relación Diseñada.

**Notas para la sustentación**
- Pasajero, conductor y operador **no tienen cuenta ni token**: se identifican por su número registrado en la base de su operador (ADR-0005).
- La única dependencia externa de la ruta E2E es Meta, que no ofrece SLA comercial. Su efecto en la disponibilidad compuesta está en ADR-0009.
- Ya no aparecen el proveedor de rutas/ETA ni el correo SMTP de las alertas que tenía el C4 anterior (`docs/_fuentes/Modelo C4 actualizado.drawio.xml`). Los reemplazan la matriz de duración (ADR-0007) y la alerta por WhatsApp (RF-45). Ver el registro de cambios.
