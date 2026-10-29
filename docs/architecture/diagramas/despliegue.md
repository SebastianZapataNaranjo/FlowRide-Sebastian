# Diagrama de despliegue

Los lineamientos del curso Ingeniería del Software 2 advierten que el C4 nivel 2 y el despliegue no son equivalentes. Aquí se muestran nodos, redes, zonas de confianza e instancias. Hay dos vistas: la **local** (`docker compose`), que es la que se construye para el 4-nov, y la de **Azure**, que queda Diseñada (ADR-0009).

## D1 — Local: `docker compose` (línea base, se construye para el 4-nov)

```mermaid
flowchart TB
    subgraph internet["Internet (zona no confiable)"]
        meta["⬜ Meta Cloud API"]
        tel1["📱 Teléfono pasajero A"]
        tel2["📱 Teléfono conductor A"]
    end
    tunel["Túnel HTTPS público<br/>(dev tunnels / ngrok)<br/>solo para el stack A"]

    subgraph host["Equipo de desarrollo — Docker Engine"]
        subgraph red["Red docker interna 'flowride'"]
            subgraph sa["Stack A (operador demo A)"]
                apia["api-a<br/>imagen flowride:{tag}<br/>HTTPS :8443 (dev-cert)"]
                wrka["worker-a<br/>misma imagen, entrypoint Worker"]
            end
            subgraph sb["Stack B (operador demo B)"]
                apib["api-b<br/>HTTPS :8444"]
                wrkb["worker-b"]
            end
            sim["🧪 whatsapp-sim<br/>DOBLE DE PRUEBA<br/>HTTPS :8445"]
            subgraph pg["postgres (PostgreSQL 17) — un servidor, tres bases, tres usuarios"]
                dba[("flowride_tenant_a<br/>usuario app_a")]
                dbb[("flowride_tenant_b<br/>usuario app_b")]
                dbcp[("flowride_control_plane<br/>usuario plataforma")]
            end
            mig["migrador (ejecución única por base)"]
        end
    end
    admin["👤 Administrador técnico<br/>Swagger / curl en el host"]

    tel1 & tel2 -- "app WhatsApp" --> meta
    meta -- "webhook HTTPS (override del número de prueba)" --> tunel --> apia
    wrka -- "HTTPS Graph API" --> meta
    sim -- "webhook HTTPS firmado (secreto B)" --> apib
    wrkb -- "HTTPS Graph API simulada" --> sim
    apia & wrka -- "TCP 5432 (app_a)" --> dba
    apib & wrkb -- "TCP 5432 (app_b)" --> dbb
    mig -- "TCP 5432" --> dba & dbb & dbcp
    admin -- "HTTPS + X-Api-Key A" --> apia
    admin -- "HTTPS + X-Api-Key B" --> apib
    admin -- "HTTPS /sim/entrantes" --> sim

    classDef doble fill:#fff4e5,color:#7a4a00,stroke:#d48a00,stroke-dasharray: 6 3
    class sim doble
```

**Lo que este despliegue demuestra y lo que no**

| Afirmación | ¿Se demuestra en compose? |
|---|---|
| Cada stack solo tiene la cadena de conexión y el usuario de **su** base (`app_a` no tiene permisos sobre `flowride_tenant_b`) | **Sí**: `PLAT-AIS-01` |
| El simulador solo conoce la URL del stack B; el número de Meta solo apunta al stack A | **Sí** |
| HTTPS en la entrada; HTTP no se publica | **Sí** (puertos HTTPS únicamente) |
| Separación física de **servidores** de BD por operador | **No**: en compose las tres bases comparten un servidor PostgreSQL por economía local. En Azure cada operador tiene su propio Flexible Server (D2). Se declara como diferencia entre entornos, no se oculta |
| Disponibilidad o redundancia | **No**: un host único |

Configuración y secretos: `appsettings.json` con valores no sensibles; secretos (cadena de conexión, API key, token de Meta, app secret, secreto del simulador) en `dotnet user-secrets` o en un `.env` **excluido de Git**; `appsettings.example.json` sin valores reales. Detalle en `delivery/ejecucion-local.md`.

## D2 — Azure (Diseñado, ADR-0009)

```mermaid
flowchart TB
    meta["⬜ Meta Cloud API"]
    subgraph sub["Suscripción Azure — región East US 2 (supuesto de precio)"]
        subgraph plat["Grupo de recursos: plataforma (sin datos operativos)"]
            acr["Azure Container Registry (Basic)"]
            kvp["Key Vault de plataforma<br/>app secret de Meta"]
            law["Log Analytics"]
            cpdb[("PostgreSQL control plane")]
        end
        subgraph opA["Grupo de recursos: operador A  (×N operadores)"]
            subgraph env["Container Apps environment (Consumption)"]
                ca["Container App 'flowride-a'<br/>contenedor api (0,25 vCPU / 0,5 GiB)<br/>contenedor worker (0,25 vCPU / 0,5 GiB)<br/>minReplicas=1 · ingress HTTPS, allowInsecure=false"]
                job["Container Apps Job 'migrar-a'"]
            end
            pga[("PostgreSQL Flexible Server B1ms<br/>32 GB · backup automático<br/>sslmode=require")]
            kva["Key Vault del operador A<br/>BD, token de Meta, API key"]
        end
    end

    meta -- "webhook HTTPS (override por número)" --> ca
    ca -- "HTTPS Graph API" --> meta
    ca -- "TCP 5432 + TLS" --> pga
    job -- "TCP 5432 + TLS" --> pga
    ca -- "HTTPS (identidad administrada)" --> kva
    ca -- "OTLP" --> law
    acr -- "pull de la imagen" --> ca
```

El simulador **no existe** en este despliegue. Las opciones contractuales (réplica adicional, HA de zona en la BD) están en ADR-0009 §4.
