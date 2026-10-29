# Ejecución local de la línea base (especificación para construir)

> **Estado: Diseñado.** Describe cómo **deberá** levantarse la línea base. Todavía no hay código; nada de esto se ha ejecutado.

## 1. Servicios de `docker compose`

| Servicio | Imagen | Puerto publicado | Rol | Depende de |
|---|---|---|---|---|
| `postgres` | `postgres:17` | — (solo red interna) | Bases `flowride_control_plane`, `flowride_tenant_a` y `flowride_tenant_b`, con los usuarios `plataforma`, `app_a` y `app_b` (script de inicio) | — |
| `migrador` | `flowride:{tag}` (entrypoint de migraciones) | — | Aplica las migraciones de EF Core a cada base y termina | `postgres` sano |
| `api-a` | `flowride:{tag}` (`FlowRide.Inicializador.Api`) | `8443` (HTTPS) | Stack A: webhook de Meta (número de prueba) y `/admin` | `migrador` completado |
| `worker-a` | `flowride:{tag}` (`FlowRide.Inicializador.Worker`) | — | Stack A: inbox, outbox, barridos | `migrador` completado |
| `api-b` | `flowride:{tag}` | `8444` (HTTPS) | Stack B: webhook del simulador y `/admin` | `migrador` completado |
| `worker-b` | `flowride:{tag}` | — | Stack B | `migrador` completado |
| `whatsapp-sim` | `flowride-sim:{tag}` | `8445` (HTTPS) | **Doble de prueba**: Graph API simulada y webhooks firmados hacia `api-b` | — |

El puerto HTTP no se publica en ningún servicio: una conexión HTTP no llega (ADR-0003).

## 2. HTTPS con el certificado de desarrollo de .NET

Fuente: [Hosting ASP.NET Core images with Docker over HTTPS (.NET 10)](https://learn.microsoft.com/en-us/aspnet/core/security/docker-https?view=aspnetcore-10.0), consulta 2026-10-28.

1. `dotnet dev-certs https -ep ./certs/flowride-dev.pfx -p <contraseña-local>` y `dotnet dev-certs https --trust`. La carpeta `certs/` está en `.gitignore`.
2. Montar `./certs` en `/https` (solo lectura) en `api-a`, `api-b` y `whatsapp-sim`.
3. Variables de entorno por servicio:
   - `ASPNETCORE_URLS=https://+:8443` (u 8444 / 8445)
   - `ASPNETCORE_Kestrel__Certificates__Default__Path=/https/flowride-dev.pfx`
   - `ASPNETCORE_Kestrel__Certificates__Default__Password` ← desde `.env`
4. En .NET 10 el certificado incluye `host.docker.internal` como nombre alternativo, así que la comunicación `worker-b → whatsapp-sim` y `whatsapp-sim → api-b` valida el certificado sin configuración extra. Se confía en el certificado de desarrollo **solo** en el entorno `Development`; nunca se desactiva la validación TLS en el código.

## 3. Configuración y secretos

`appsettings.json` (versionado) lleva solo valores no sensibles. `appsettings.example.json` (versionado) lista todas las claves con valores de ejemplo vacíos. Los secretos van en `.env` (en `.gitignore`) para compose, o en `dotnet user-secrets` si se ejecuta fuera de Docker.

| Clave | Stack A | Stack B | Secreto |
|---|---|---|---|
| `ConnectionStrings__Operador` | `…Database=flowride_tenant_a;Username=app_a;…` | `…flowride_tenant_b;Username=app_b;…` | Sí |
| `Canal__PhoneNumberId` | id del número de prueba de Meta | id ficticio del simulador | No |
| `Canal__GraphBaseUrl` | `https://graph.facebook.com/{version}` | `https://host.docker.internal:8445/{version}` | No |
| `Canal__AccessToken` | token del sistema de Meta | token de prueba del simulador | Sí |
| `Canal__AppSecret` | app secret de la app de Meta | secreto propio del stack B con el simulador | Sí |
| `Canal__VerifyToken` | cadena elegida para el handshake | ídem | Sí |
| `Admin__ApiKey` | clave A | clave B (distinta) | Sí |

## 4. Meta (tenant A) — pasos previos a la demo

1. Crear la app en Meta for Developers con el producto WhatsApp. Se crean automáticamente el número de prueba y la WABA de prueba ([Get Started](https://developers.facebook.com/documentation/business-messaging/whatsapp/get-started)).
2. Registrar como destinatarios los teléfonos del pasajero, el conductor y el operador (límite de 5 según fuentes de terceros; ver `preguntas-abiertas.md`).
3. Abrir un túnel HTTPS público hacia `https://localhost:8443` (dev tunnels o ngrok).
4. Configurar el callback `https://{tunel}/webhooks/whatsapp` con el `VerifyToken` y suscribir el campo `messages`. Opcional: fijar el override del número de prueba con `POST /{PHONE_NUMBER_ID}` y `webhook_configuration` ([Webhook overrides](https://developers.facebook.com/docs/whatsapp/embedded-signup/webhooks/override)).
5. Someter las plantillas de `architecture/diseno-detallado.md` §2. La demo **no** depende de que se aprueben.

## 5. Datos semilla (por `/admin`, con la API key de cada stack)

Por stack: 1 empresa cliente, 3 lugares frecuentes con su matriz de duración, 1 pasajero, 2–3 conductores (uno con el SOAT vencido, para mostrar `OPLOG-CONF-04`), 2 vehículos, turnos de hoy y mañana, un usuario operador con `recibe_alertas = true`, y parámetros por defecto. Los teléfonos del stack A son los reales registrados en Meta; los del stack B son ficticios (solo existen en el simulador).

## 6. Guion de demostración

1. Conductor A: "INICIO TURNO" → respuesta (abre la ventana).
2. Pasajero A: registra el itinerario (F1, 7 pasos).
3. El Worker asigna (F2); `GET /admin/solicitudes/{id}` muestra las `razones` (incluido el conductor descartado por SOAT vencido).
4. Conductor A: Aceptar (F3) → el pasajero recibe la confirmación.
5. Stack B, por `POST /sim/entrantes`: un conductor rechaza y el otro deja expirar → reencolado → alerta al operador B (`GET /sim/bandeja/{tel}`).
6. Aislamiento: webhook firmado para B enviado a A → 401/403; consulta con `app_a` a `flowride_tenant_b` → permiso denegado (`PLAT-AIS-01`).
7. `GET /health/ready` en ambos stacks.

## 7. Salud y observabilidad local

`/health/live` y `/health/ready` por servicio (`healthcheck` de compose con `curl -fsk`). Logs JSON por consola con `correlation_id`, `operador` y `componente` (`docker compose logs -f api-a worker-a`).
