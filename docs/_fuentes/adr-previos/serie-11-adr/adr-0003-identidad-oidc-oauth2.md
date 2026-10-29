# ADR-0003: desacoplar identidad mediante OIDC/OAuth2 y revocación controlada

## Metadatos

| Campo | Valor |
|---|---|
| ID | ADR-0003 |
| Fecha | 2026-09-09 |
| Estado | **Accepted** |
| Participantes | Sebastián Zapata Naranjo, Andrés David Ortiz Mejía, Jhonny Estiven Benítez Caro |
| Curso | ISS0666 — Ingeniería del Software 2 (UCO, semestre 2-2026) |
| Profesor evaluador | Wider Farid Sánchez Garzón |
| Reemplaza | ADR-03 histórico de JWT propio HMAC-SHA256. |

## Contexto

FlowRide tiene dos superficies de identidad: el Panel Web usado por Administrador del operador y Operador logístico, y WhatsApp usado por Pasajero y Conductor. La implementación previa generaba JWT propios con HMAC-SHA256 y usaba `empresaId` como claim de aislamiento, lo cual ya no coincide con el modelo Silo.

El requisito vigente es autenticación segura basada en tokens, RBAC y capacidad de revocar acceso en un máximo de 60 segundos. La selección no debe acoplar el dominio a un producto específico.

## Decisión

Adoptar **OpenID Connect / OAuth 2.0** como contrato de identidad para usuarios interactivos del Panel Web, mediante un proveedor de identidad externo o autoalojado compatible con estándares. FlowRide valida tokens y traduce claims externos a un modelo interno de usuario/rol/tenant mediante un `IdentityProviderAdapter`.

Para Pasajero y Conductor, el canal WhatsApp identifica el número/cuenta remitente y lo mapea a un usuario interno; el backend no expone tokens al usuario de WhatsApp. Las acciones sensibles se autorizan contra el usuario interno y el servicio/asignación correspondiente.

La revocación debe surtir efecto en ≤60 s mediante sesión corta, introspección/estado de cuenta, lista de revocación o mecanismo equivalente soportado por el proveedor elegido.

## Alternativas consideradas

### Alternativa A — JWT propio HMAC-SHA256
Descartada como decisión principal. Reduce dependencias, pero obliga al equipo a asumir ciclo de vida de credenciales, recuperación, revocación, MFA y hardening del emisor.

### Alternativa B — Auth0
Válida como proveedor inicial si costo y límites del plan se ajustan al tenant. Buena integración OIDC, pero introduce dependencia comercial.

### Alternativa C — Amazon Cognito
Válida si el despliegue se concentra en AWS. Escalable y basado en estándares, pero aumenta lock-in de plataforma.

### Alternativa D — Keycloak/IdentityServer compatible
Válida si se desea control operativo propio. Reduce dependencia SaaS, pero aumenta costo de operación y parcheo.

## Consecuencias positivas

- El dominio no depende de la implementación concreta del proveedor.
- Facilita MFA, recuperación y políticas de identidad sin reconstruirlas desde cero.
- Elimina el uso de `empresaId` como mecanismo de aislamiento.
- Permite reemplazar proveedor conservando contratos OIDC/OAuth2.

## Consecuencias negativas

- Existe dependencia operativa del proveedor seleccionado.
- Hay que mapear cuidadosamente roles, usuarios y tenant interno.
- La revocación ≤60 s puede requerir estado adicional y reducir el carácter puramente stateless de la validación.

## Tácticas de Bass aplicadas

- **Autenticar actores** — validación de identidad antes de ejecutar acciones.
- **Autorizar actores** — RBAC y autorización contextual por recurso.
- **Revocar acceso** — corte de acceso en el umbral definido.
- **Abstraer plataforma** — adapter interno sobre OIDC/OAuth2.

## Escenarios y requisitos relacionados

- `ADM-SEG-01` — revocación de acceso ≤60 s.
- `COND-SEG-01` — validación de identidad del conductor en WhatsApp.
- `PAS-SEG-01` — limitación de exposición de solicitudes personales.
- `RF-01` — autenticación basada en token.
- `RF-02` — autorización basada en roles.

## ADRs relacionados

- **ADR-0002** — el tenant resuelto determina el Silo, no un claim `empresaId`.
- **ADR-0005** — WhatsApp aporta identidad de canal para Pasajero y Conductor.
- **ADR-0011** — registra eventos de seguridad y auditoría.

## Notas de implementación

- El Panel Web debe usar Authorization Code Flow con PKCE cuando aplique al cliente web.
- Las claves/secretos del proveedor se almacenan fuera del repositorio.
- El adapter debe mapear identificador externo → `Usuario` interno → rol → tenant.
- Desactivar un usuario en FlowRide debe invalidar el acceso aunque el token externo aún no haya expirado.

## Referencias

- Microsoft Learn — OpenID Connect en ASP.NET Core 10.
- `Drivers_Arquitectonicos_FlowRide_FINAL.docx`.
- `RestriccionesTecnicas_FlowRide_ACTUALIZADO.xlsx`.
- `tradeoff_FlowRide_ACTUALIZADO.xlsx` — T-01 y T-09.
