---
inclusion: always
---

# Equipo y Reparto de Trabajo (4 personas)

## Perfiles
- **3 perfiles técnicos** — desarrollo de componentes.
- **1 perfil menos técnico** — documentación, gestión del tablero (GitHub Projects), testing manual
  de la demo, y preparación de la presentación técnica. También puede llevar el README y los
  diagramas con apoyo de los agentes.

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

## Sugerencia de asignación inicial (ajustable)
- Persona A (técnica): consumer + cache.
- Persona B (técnica): processing (join/normalización) — es el corazón del proyecto.
- Persona C (técnica): warehouse + api + frontend.
- Persona D (menos técnica): lake básico (guardar crudo, sencillo) + documentación + board + demo + presentación.

## Cómo usar los agentes del harness
Cada persona puede invocar los agentes definidos en `.kiro/agents/` para su módulo. Los agentes
conocen el contexto (steering) y las reglas críticas automáticamente.
