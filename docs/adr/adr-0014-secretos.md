# ADR-0014: secretos fuera del repositorio, con un ámbito por operador y uno de plataforma

| Campo | Valor |
|---|---|
| Estado | **Proposed** · la parte local es E2E (user-secrets / `.env` ignorado); Key Vault está Diseñado |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0014 del xlsx |

**Contexto.** Cada stack necesita: credencial de su base, token de Meta de su número, API key de administración y, en compose, el secreto de firma con el simulador (stack B). La plataforma necesita el **app secret de Meta**, que firma los webhooks de **todos** los números de la app (ADR-0005). El curso prohíbe secretos en Git, en la imagen y en los logs (lineamientos del curso Ingeniería del Software 2).

**Decisión.**
- **Local (línea base):** `dotnet user-secrets` para ejecutar fuera de Docker; `.env` **en `.gitignore`** para compose; `appsettings.example.json` con claves y sin valores. Un secreto distinto por stack (A y B), de modo que la credencial de A no sirve en B (`PLAT-AIS-01`).
- **Azure (Diseñado):** un Key Vault por operador, leído con la identidad administrada de su Container App; un Key Vault de plataforma para el app secret. Rotación de la contraseña de la BD cada 90 días o ante un incidente; rotación del app secret con período de gracia: el stack acepta la firma con el secreto anterior o el nuevo durante la ventana que permite Meta.

**Riesgo declarado.** El app secret es un **punto de compromiso común** a todos los operadores: con él se pueden forjar webhooks hacia cualquier stack. Mitigaciones: acceso restringido y auditado al Key Vault de plataforma; validación del `phone_number_id` en cada stack (un webhook forjado tendría además que acertar el identificador de ese operador); detección por la métrica de firmas inválidas. En el simulador, cada stack tiene su propio secreto, así que el contraste con producción se presenta explícitamente.

**Escenarios.** `CANAL-SEG-01`, `PLAT-AIS-01`.
