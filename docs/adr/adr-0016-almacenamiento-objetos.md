# ADR-0016: almacenamiento de objetos por operador para archivo de auditoría y exportes (Diseñado V1)

| Campo | Valor |
|---|---|
| Estado | **Proposed** · Diseñado V1 |
| Fecha | 2026-10-28 |
| Reemplaza | ADR-0013 del xlsx |

**Contexto.** La auditoría se conserva 5 años y los datos históricos se archivan a los 24 meses (contexto v10 §8.5). Los exportes de reportes y los soportes documentales del conductor (licencia, SOAT) no deben vivir en la base transaccional ni en el disco del contenedor, que no persiste (ADR-0009). En F1–F3 no hay archivos: la auditoría vive en la base del operador y los backups los gestiona Flexible Server.

**Decisión (dirección).** Azure Blob Storage con un contenedor por operador y acceso solo con la identidad del stack; niveles Hot y Cool/Archive con política de ciclo de vida; retención inmutable para el archivo de auditoría. Puerto `IAlmacenObjetos` en `Servicios.Output.Ports` (hoy no se crea).

**Cuándo se activa.** Al acercarse los 24 meses de operación de un operador, al entrar los reportes del panel, o si se cargan los soportes documentales del conductor.

**Escenarios.** `OPLOG-AUD-01` (retención de 5 años), `PAS-AUD-02` (V1).
