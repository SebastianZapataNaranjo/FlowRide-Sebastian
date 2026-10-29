# Checklist de seguridad aplicado a F1–F3

Base: lista de verificación de seguridad de los lineamientos del curso Ingeniería del Software 2, aplicada a FlowRide. Estado de cada ítem en la línea base.

| Ítem del curso | Aplicación en FlowRide | Estado |
|---|---|---|
| Autenticación apropiada | Firma HMAC del webhook; teléfono → usuario; API key por stack para el Administrador técnico | Diseñado E2E (ADR-0003, ADR-0005) |
| Autorización en el servidor por operación | Pertenencia de la asignación al conductor; rol del remitente según el estado de la conversación | Diseñado E2E (`COND-SEG-01`) |
| El actor se obtiene de la identidad autenticada | El conductor sale del `wa_id`, nunca del payload del botón | Diseñado E2E |
| No se opera fuera del alcance autorizado | Cada stack solo tiene la credencial de su base | Diseñado E2E (`PLAT-AIS-01`) |
| Validación de entrada | Formatos del código y la hora; IDs de listas y botones validados contra la conversación; rangos de los parámetros | Diseñado E2E |
| Consultas parametrizadas | EF Core y SQL parametrizado de Npgsql; sin concatenación | Diseñado E2E |
| Errores sin detalles internos | Catálogo de mensajes al usuario; `problem+json` con código en `/admin` | Diseñado E2E (ADR-0008) |
| Contraseñas con hash | No aplica en F1–F3: no hay contraseñas de usuario (identidad por teléfono; panel con OIDC en V1) | N/A |
| Secretos fuera del repositorio | user-secrets / `.env` ignorado; Key Vault en Azure | Diseñado (ADR-0014) |
| HTTPS/TLS | Kestrel con dev-cert (compose); ingress de ACA con `allowInsecure=false` | Diseñado (ADR-0003) |
| CORS restrictivo | No hay frontend en la entrega; CORS deshabilitado (sin orígenes permitidos) | Diseñado |
| Protección contra abuso | Firma obligatoria; límite de tamaño del cuerpo del webhook; límite de tasa por IP en `/admin` (`Microsoft.AspNetCore.RateLimiting`) | Diseñado |
| Límites de tamaño y paginación | `limit` máximo 100 en `/admin`; listas de WhatsApp ≤ 10 filas | Diseñado |
| Protección de registros y auditoría | Teléfonos enmascarados; sin cuerpos de mensaje en los logs; auditoría de solo inserción | Diseñado (ADR-0010) |
| Dependencias revisadas | `dotnet list package --vulnerable` en CI | Diseñado |
| Pruebas de acceso denegado y manipulación de IDs | `COND-SEG-01`, `CANAL-SEG-01`, `PLAT-AIS-01` | Diseñado |
| Restricciones de integridad en la base | `UNIQUE`, `EXCLUDE`, `CHECK`, claves foráneas | Diseñado (ADR-0004, ADR-0006) |
| No exponer datos personales no autorizados | El pasajero ve solo su itinerario; al conductor se le muestra el nombre del pasajero sin teléfono; `es_personal` reservado para V1 | Diseñado |
| Ley 1581 | FlowRide como encargado y el operador como responsable; minimización en logs; retención del inbox (`mensaje_entrante.contenido`) limitada a 30 días con purga programada | Diseñado; la purga es una tarea del Worker por construir |

**Riesgos residuales**: app secret de Meta compartido entre operadores (ADR-0014); API key sin identidad personal (ADR-0003 §4); red interna del compose sin TLS hacia PostgreSQL (solo entorno local).
