---
inclusion: always
---

# Equipo y Reparto de Trabajo (2 personas)

## Equipo

| Rol | GitHub | Responsabilidad principal |
|-----|--------|---------------------------|
| Scrum Master | **jzelada97** | lake (MongoDB) + warehouse + api + frontend + docker (infra e2e) + Prometheus + logs + documentación/board/demo/presentación; coordinación del equipo |
| PM | **karinaromerovasquez** | Project Manager; processing (join/normalización, core del ETL) + consumer (Kafka) + cache (Redis) |

Equipo reducido a 2 tras la salida de otros miembros. El reparto es por área de foco, no por nivel.

## Módulos independientes (para paralelizar sin pisarse)
Cada módulo tiene interfaces claras para poder trabajar en paralelo:

| Módulo | Responsabilidad | Interfaz de entrada | Interfaz de salida |
|--------|-----------------|---------------------|--------------------|
| consumer | Leer de Kafka, alto throughput | topic Kafka | mensajes crudos dict |
| lake (mongo-writer) | Guardar crudo en MongoDB | mensaje crudo | doc en Mongo |
| processing | Detectar tipo, normalizar, JOIN por persona | mensajes/docs | registro consolidado |
| warehouse (sql-writer) | Persistir en PostgreSQL | registro consolidado | filas en Postgres |
| cache | Buffer/matching en Redis | fragmentos | fragmentos agrupados |
| api | Consultas sobre Postgres | HTTP | JSON |
| frontend | UI de consulta (Streamlit) | API/Postgres | UI |
| infra/monitoring | docker-compose, Prometheus, logs | - | entorno + métricas |

## Asignación por módulo (2 personas)
- karinaromerovasquez (PM): processing (join/normalización, core del ETL) + consumer + cache.
- jzelada97 (Scrum Master): lake + warehouse + api + frontend + docker + Prometheus + logs + documentación/board/demo/presentación.

## Roles de gestión
- **Scrum Master**: jzelada97 — facilita el proceso, desbloquea al equipo, coordina el día a día.
- **PM (Project Manager)**: karinaromerovasquez — visión de proyecto, alcance y prioridades.

## Cómo usar los agentes del harness
Cada persona puede invocar los agentes definidos en `.kiro/agents/` para su módulo. Los agentes
conocen el contexto (steering) y las reglas críticas automáticamente.
