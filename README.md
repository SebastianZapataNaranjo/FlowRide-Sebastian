# FlowRide

FlowRide es una plataforma SaaS para operadores de transporte corporativo interno: empresas que llevan a tripulaciones, cuadrillas o personal de un cliente empresarial con horarios estrictos.
Reemplaza la coordinación manual por WhatsApp y Excel con un motor de asignación configurable por operador, manteniendo WhatsApp como único canal para pasajeros y conductores.
Cada operador tiene su propio stack aislado (modelo Silo).

## Alcance de esta entrega

Documentación de arquitectura de tres funcionalidades encadenadas, que se construirán de extremo a extremo con HTTPS en la línea base:

| | Funcionalidad |
|---|---|
| **F1** | El pasajero registra su itinerario por WhatsApp |
| **F2** | El sistema asigna conductor y vehículo con las reglas del operador |
| **F3** | El conductor recibe la propuesta y la acepta o rechaza por WhatsApp |

Es el **diseño a implementar**: todavía no hay código. Los componentes están en estado *Diseñado* y los ADR en estado *Proposed*, cada uno con el criterio que lo haría pasar a *Accepted*.

## Por dónde empezar

1. [Índice y ruta de lectura](docs/README.md)
2. [ADR-0001 — Monolito modular](docs/adr/adr-0001-monolito-modular.md) y el [registro de ADR](docs/adr/README.md)
3. [C4 nivel 1 — Contexto](docs/architecture/diagramas/c4-contexto.md) y [C4 nivel 2 — Contenedores](docs/architecture/diagramas/c4-contenedores.md)
4. [Secuencias F1–F3](docs/architecture/diagramas/secuencias-f1-f3.md)
5. [Trazabilidad](docs/architecture/trazabilidad.md)

## Estructura de `docs/`

| Carpeta | Contenido |
|---|---|
| `docs/adr/` | Decisiones de arquitectura (ADR-0001 a ADR-0016) |
| `docs/architecture/` | Diseño detallado, trazabilidad, riesgos, preguntas abiertas y registro de cambios frente al trabajo previo |
| `docs/architecture/diagramas/` | C4 niveles 1 y 2, componentes, paquetes, secuencias y despliegue (Mermaid) |
| `docs/quality/` | Escenarios de calidad y tácticas por escenario |
| `docs/delivery/` | Ejecución local, ramas e integración continua, checklist de seguridad |
| `docs/_fuentes/` | Trabajo previo del semestre usado como insumo (no se edita) |

---

**Autor:** Sebastián Zapata Naranjo — Ingeniería del Software 2, Universidad Católica de Oriente (UCO) — Prof. Wider Farid Sánchez Garzón — 28-oct-2026
