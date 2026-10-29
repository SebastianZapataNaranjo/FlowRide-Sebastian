# Ramas, integración continua y calidad (especificación)

> **Estado: Diseñado.** El pipeline se crea con la línea base.

## Ramas y pull requests

- `main` estable y protegida: solo se integra por PR con el pipeline en verde.
- Ramas `feature/…`, `docs/…`, `fix/…` y `chore/…` (esta documentación vive en `docs/das-f1-f3`).
- Commits con mensaje descriptivo; PR con descripción, escenarios afectados y evidencia (salida de las pruebas).
- Revisión humana obligatoria de los cambios generados por asistentes de código (lineamientos del curso Ingeniería del Software 2).

## Pipeline de CI (GitHub Actions)

| Paso | Herramienta | Criterio para pasar |
|---|---|---|
| 1. Restaurar y compilar | `dotnet build -warnaserror` | Sin errores ni advertencias |
| 2. Formato | `dotnet format --verify-no-changes` | Sin diferencias |
| 3. Pruebas de dominio y casos de uso | xUnit | 100 % en verde |
| 4. **Pruebas arquitectónicas** | NetArchTest (reglas C1–C8 y M1–M4 de `paquetes.md`) | 100 % en verde |
| 5. Integración y concurrencia | xUnit + Testcontainers (PostgreSQL 17) | `OPLOG-CONF-03`, `CANAL-CONF-01`, `COND-SEG-01`, `COND-ACC-01`, `PAS-CONF-03`, `COND-CONF-02` en verde |
| 6. Contrato del webhook | xUnit (firma, duplicados, `phone_number_id`) | `CANAL-SEG-01` en verde |
| 7. Cobertura | coverlet | ≥ 80 % en `FlowRide.Dominio` y `FlowRide.Servicios`; en infraestructura no se fija porcentaje, la cubren las pruebas de integración |
| 8. Análisis estático | SonarCloud | Quality gate sin vulnerabilidades ni *bugs* nuevos |
| 9. Dependencias y secretos | `dotnet list package --vulnerable`, escaneo de secretos de GitHub | Sin vulnerabilidades altas; sin secretos |
| 10. Imagen | `docker build` | Imagen construida y etiquetada con el commit |

Las pruebas de carga (k6) y la prueba de aislamiento con compose (`PAS-DISP-01`, `COND-DISP-01`, `PLAT-AIS-01`, `PAS-DISP-03`) corren de forma manual antes de la sustentación y su salida se adjunta como evidencia. No corren en cada PR por duración y porque necesitan los dos stacks.

Advertencia del curso que se respeta: pasar el análisis estático **no** demuestra que la arquitectura sea limpia; por eso existen el paso 4 y la revisión en el PR (`paquetes.md` §3, limitación declarada).
