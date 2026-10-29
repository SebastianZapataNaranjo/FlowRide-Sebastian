# Riesgos, sensibilidades y puntos de compromiso

| # | Riesgo | Tipo | Probabilidad / impacto | Mitigación | ADR |
|---|---|---|---|---|---|
| R1 | La matriz de duración es estática y no considera el tráfico: recogidas tardías en horas pico | **Sensibilidad** (la promesa de presentación depende de este parámetro) | Media / alto | `recogida.margen_min` configurable; la promesa queda registrada; condición de revisión cuando exista RF-18 | 0007 |
| R2 | Meta sin SLA comercial; caída del proveedor | Riesgo externo | Baja / alto | Reintentos de Meta (7 días) e inbox; outbox con backoff; disponibilidad compuesta declarada | 0005, 0009 |
| R3 | App secret de Meta comprometido: webhooks forjados hacia todos los operadores | **Punto de compromiso** común | Baja / muy alto | Key Vault de plataforma, rotación con gracia, validación de `phone_number_id`, métrica de firmas inválidas | 0014 |
| R4 | Ventana de 24 h cerrada y plantilla no aprobada: el conductor no recibe la propuesta | Riesgo operativo | Media / medio | Check-in de turno (abre la ventana); expiración → reencolado → alerta | 0005, 0007 |
| R5 | Alerta al operador con su ventana cerrada (sin plantilla aprobada) | Riesgo operativo | Media / alto | El operador escribe al iniciar su jornada; plantilla `alerta_operativa` sometida; `GET /admin/alertas` como respaldo | 0008 |
| R6 | Duplicado residual de una notificación (timeout de Meta sin webhook de estado) | **Trade-off** aceptado | Baja / bajo | `biz_opaque_callback_data` + ventana de confirmación; solo se repite el mensaje, no la transición | 0006 |
| R7 | Margen corto frente al techo de 40 USD (~33–35 USD por operador) | Riesgo de negocio | Media / medio | Medir un mes en el piloto; ajustar el tamaño de la réplica o la retención de logs | 0009 |
| R8 | Override de webhook mal configurado: los mensajes de A llegan a B | Riesgo operativo | Baja / alto | Validación del `phone_number_id` en el stack; rechazo y auditoría | 0002 |
| R9 | API key sin identidad personal | **Trade-off** aceptado | — / medio | Restringida al Administrador técnico; OIDC en V1 | 0003, 0011 |
| R10 | Línea telefónica reasignada a otra persona por el operador móvil | Riesgo de identidad | Baja / alto | Desactivar al usuario desde `/admin`; verificación de reemplazo de número en V1 | 0005 |
| R11 | Un solo servidor PostgreSQL en compose: no demuestra la separación física de servidores | Diferencia entre entornos | — / bajo | Declarada en `despliegue.md`; en Azure hay un servidor por operador | 0002, 0009 |
| R12 | Valores por defecto del catálogo de parámetros no validados con el operador | Supuesto | Media / medio | Parametrizables sin despliegue; `preguntas-abiertas.md` #8 | 0008 |
| R13 | Límite de 5 destinatarios del número de prueba (dato de terceros) | Supuesto de la demo | Baja / bajo | La demo usa 3 números en A; el tenant B no lo necesita | 0005 |
