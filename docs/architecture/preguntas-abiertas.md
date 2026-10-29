# Preguntas abiertas

Datos que no se pudieron verificar o que dependen del cliente. Ninguno bloquea la línea base; cada uno dice qué decisión cambiaría si la respuesta es distinta a la supuesta.

| # | Pregunta | Supuesto actual | Qué cambia si es distinto | Dónde se usa |
|---|---|---|---|---|
| 1 | Tarifa oficial de Colombia para mensajes utility y de servicio (CSV o PDF de Meta en COP o USD) | USD 0,0008–0,0014 (fuentes de terceros) | Solo la cifra de costo variable (ADR-0005 §5); el argumento principal es la experiencia de usuario | ADR-0005, ADR-0009 |
| 2 | ¿El límite de 5 destinatarios del número de prueba está en la documentación oficial vigente? | 5 (fuentes de terceros) | El número de teléfonos de la demo del tenant A | ADR-0005 |
| 3 | Texto vigente del art. 22 de la Ley 769 de 2002 (vigencia de la licencia), modificado por la Ley 1383 de 2010 y la Ley 2251 de 2022 | Se valida contra la fecha de vencimiento registrada | Si la norma fija otra condición (por ejemplo, exámenes), se agrega a la regla de nivel 1 | ADR-0007 |
| 4 | Estado del Decreto 1469 de 2025 (SMMLV 2026) ante la posible suspensión del Consejo de Estado | 1.750.905 COP | Solo la cifra de la multa en el registro de cambios | Registro de cambios #28 |
| 5 | ¿El SLA de 99,9 % de Flexible Server sin HA aplica también al nivel Burstable? ¿El SLA formal de ACA es 99,95 %? | Sí a ambos | El techo de la disponibilidad compuesta (ADR-0009 §2) | ADR-0009 |
| 6 | ¿El backup de Flexible Server es gratis hasta el 100 % del almacenamiento provisionado? | Sí (conocimiento previo) | ~3–4 USD/mes en el costo por operador | ADR-0009 §3 |
| 7 | ¿Existe el código de idioma `es_CO` para plantillas? | Se usa `es` | Solo la metadata de las plantillas | `diseno-detallado.md` §2 |
| 8 | Valores por defecto del catálogo de parámetros (T = 15 min, X = 6 h, margen de confirmación = 90 min, N = 3, anticipación mínima = 120 min, margen de recogida = 15 min) | Propuesta de partida | Ajuste por operador sin despliegue (ADR-0008) | `diseno-detallado.md` §6 |
| 9 | Duración real entre los lugares frecuentes del cliente ancla (hotel ↔ aeropuerto) por franja horaria | Matriz estática con un valor por par | Si la variación por hora supera el margen, se necesita una matriz por franja o una fuente con tráfico (condición de revisión de ADR-0007) | ADR-0007 |
| 10 | ¿Cómo pide hoy el servicio la aerolínea del caso ancla: cada tripulante o un coordinador? | Cada tripulante (F1) | Adelantar la variante del coordinador de V1 | RF-08 |
| 11 | ¿Las plantillas de §2 de `diseno-detallado.md` se aprueban antes del 4-nov? | Pendiente (sometidas por el autor) | La ruta fuera de la ventana pasa de Diseñado a E2E | ADR-0005 §3 |
