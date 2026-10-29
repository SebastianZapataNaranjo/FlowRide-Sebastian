# C4 nivel 2 — Contenedores

**Propósito**: mostrar los contenedores ejecutables y los almacenes de datos de FlowRide y cómo se comunican.
**Alcance**: el stack de **un** operador, que se repite N veces (ADR-0002), más los elementos de plataforma. La instancia concreta con dos operadores (A y B) está en `despliegue.md`.
**Estado**: todos *Diseñados*. Los que tienen la etiqueta `[E2E]` se construyen para la línea base del 4-nov.

```mermaid
flowchart LR
    subgraph usuarios["Personas (por WhatsApp)"]
        pas["👤 Pasajero"]
        con["👤 Conductor"]
        ope["👤 Operador logístico"]
    end
    adm["👤 Administrador técnico<br/>(Swagger / curl)"]

    meta["⬜ Meta Cloud API<br/>WhatsApp Business Platform"]
    sim["🧪 whatsapp-sim<br/>DOBLE DE PRUEBA — solo compose<br/>Imita Graph API y webhooks firmados"]

    subgraph stack["Stack Silo del operador ×N  (límite de aislamiento — ADR-0002)"]
        api["🟦 API<br/>ASP.NET Core .NET 10<br/>Webhook WhatsApp + API /admin<br/>[E2E]"]
        wrk["🟦 Worker<br/>.NET 10 BackgroundService<br/>Inbox, outbox, motor, notificaciones, barridos<br/>[E2E]"]
        db[("🛢 BD del operador<br/>PostgreSQL 17<br/>solicitudes, asignaciones, inbox,<br/>outbox, catálogos, auditoría<br/>[E2E]")]
    end

    subgraph plataforma["Plataforma FlowRide (sin datos operativos)"]
        cp[("🛢 flowride_control_plane<br/>catálogo de operadores, canal y despliegue<br/>[E2E: solo datos semilla]")]
        prov["🟦 Herramienta de aprovisionamiento<br/>crea BD, migra, registra número<br/>[línea base: script; resto Diseñado]"]
    end

    panel["🟦 Panel web<br/>[Diseñado V1 — ADR-0011]"]

    pas & con & ope -- "[app WhatsApp]" --> meta
    meta -- "Webhook mensajes/estados<br/>[HTTPS POST JSON, X-Hub-Signature-256]" --> api
    sim -- "Webhook simulado (solo stack B)<br/>[HTTPS POST JSON, firma con secreto del stack B]" --> api
    wrk -- "Envía mensajes y plantillas<br/>[HTTPS REST, Bearer]" --> meta
    wrk -- "Envía (solo stack B)<br/>[HTTPS REST, Bearer de prueba]" --> sim
    adm -- "Datos maestros, consultas<br/>[HTTPS REST JSON, X-Api-Key]" --> api
    api -- "Inserta inbox; lee y escribe /admin<br/>[TCP 5432, Npgsql]" --> db
    wrk -- "Inbox y outbox con SKIP LOCKED; transacciones<br/>[TCP 5432, Npgsql]" --> db
    prov -- "Catálogo y estado<br/>[TCP 5432]" --> cp
    prov -- "Crea BD y migra<br/>[TCP 5432]" --> db
    panel -. "API del panel<br/>[HTTPS REST, OIDC] + alertas [SSE]" .-> api

    classDef persona fill:#08427b,color:#fff,stroke:#052e56
    classDef cont fill:#1168bd,color:#fff,stroke:#0b4884
    classDef datos fill:#1168bd,color:#fff,stroke:#0b4884
    classDef externo fill:#8a8a8a,color:#fff,stroke:#5e5e5e
    classDef doble fill:#fff4e5,color:#7a4a00,stroke:#d48a00,stroke-dasharray: 6 3
    classDef disenado fill:#f4f4f4,color:#555,stroke:#999,stroke-dasharray: 4 3
    class pas,con,ope,adm persona
    class api,wrk,prov cont
    class db,cp datos
    class meta externo
    class sim doble
    class panel disenado
```

**Leyenda**: azul = contenedor de FlowRide; cilindro = almacén de datos; gris = sistema externo; naranja discontinuo = **doble de prueba** (existe solo en el `docker compose` local, nunca en Azure); gris claro discontinuo = Diseñado V1. Cada flecha indica el protocolo entre corchetes.

**Por qué estas piezas y no otras**

| Decisión visible en el diagrama | ADR |
|---|---|
| API y Worker separados, sin llamadas directas entre ellos: se integran a través de las tablas de inbox y outbox | ADR-0001, ADR-0003, ADR-0006 |
| Una base por operador; el control plane no aparece en ninguna flecha de la ruta de mensajes | ADR-0002 |
| Ni broker, ni caché, ni gateway compartido entre Meta y el stack (override de webhook por número) | ADR-0006, ADR-0012, ADR-0015 |
| El simulador habla el mismo contrato que Meta y solo conoce la URL del stack B | ADR-0005 |
| Las alertas al operador salen por WhatsApp; el panel con SSE es V1 | ADR-0003, ADR-0008, ADR-0011 |
