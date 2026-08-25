# Architecture — HR Insights ETL

## 1. Estilo arquitectónico y justificación

**Estilo: Pipeline de streaming por etapas (Staged Event-Driven / Pipes & Filters) con arquitectura
de datos Lake + Warehouse, empaquetado como microservicios ligeros en contenedores.**

Justificación:
- El problema es intrínsecamente un flujo continuo de eventos (Kafka → procesado → almacenamiento).
  Un pipeline por etapas (extract → stage → transform/join → load) modela esto de forma natural.
- **Lake (MongoDB)** absorbe el volumen crudo sin fricción (schema-less), cumpliendo el requisito de
  Data Lake y desacoplando la ingesta de la transformación (si el procesado se ralentiza, no se
  pierden datos).
- **Warehouse (PostgreSQL)** guarda el dato curado y consolidado, listo para consulta/analítica.
- **Redis** como caché intermedia resuelve el reto de *matching en tiempo real*: los fragmentos de
  una persona no llegan juntos, así que se bufferizan por clave hasta poder consolidar.
- Modularidad por componentes = 4 personas trabajando en paralelo con interfaces claras.

Se descarta un monolito puro (no paraleliza bien el trabajo ni escala el consumer) y una
arquitectura event-sourcing completa (sobredimensionada para el alcance pedagógico).

## 2. Estructura del proyecto (concreta)

```
Proyecto1_modulo3_DE2/
├── src/hr_etl/
│   ├── __init__.py
│   ├── config.py                 # pydantic-settings; lee env vars
│   ├── logging_conf.py           # logging estructurado JSON
│   ├── models/
│   │   ├── __init__.py
│   │   ├── raw.py                # tipos de fragmento + detección de tipo
│   │   ├── person.py            # modelo pydantic del registro consolidado
│   │   └── db_models.py         # tablas SQLAlchemy (Postgres)
│   ├── consumer/
│   │   ├── __init__.py
│   │   └── kafka_consumer.py    # consumer group, commits, backpressure
│   ├── lake/
│   │   ├── __init__.py
│   │   └── mongo_lake.py        # insert crudo + metadatos
│   ├── cache/
│   │   ├── __init__.py
│   │   └── redis_buffer.py      # buffer de fragmentos por clave de persona
│   ├── processing/
│   │   ├── __init__.py
│   │   ├── detector.py          # identifica Personal/Location/Professional/Bank/Net
│   │   ├── normalizer.py        # trim, casing, quitar acentos, normalizar address/fullname
│   │   ├── matcher.py           # claves de unión y estrategia de matching
│   │   └── consolidator.py      # une fragmentos -> Person consolidada
│   ├── warehouse/
│   │   ├── __init__.py
│   │   ├── engine.py            # engine/session SQLAlchemy + init schema
│   │   └── person_repo.py       # upsert idempotente del registro consolidado
│   ├── metrics/
│   │   ├── __init__.py
│   │   └── prometheus.py        # contadores/histogramas
│   ├── api/
│   │   ├── __init__.py
│   │   ├── main.py              # FastAPI app
│   │   └── routes.py            # endpoints de consulta
│   └── pipeline.py               # orquesta consumer->lake->cache->consolidate->warehouse
├── frontend/
│   └── app.py                    # Streamlit
├── tests/
│   ├── conftest.py
│   ├── test_detector.py
│   ├── test_normalizer.py
│   ├── test_matcher.py
│   ├── test_consolidator.py
│   ├── test_mongo_lake.py
│   ├── test_person_repo.py
│   └── test_api.py
├── docker/
│   ├── Dockerfile.app
│   ├── Dockerfile.api
│   └── Dockerfile.frontend
├── monitoring/
│   └── prometheus.yml
├── docker-compose.yml
├── pyproject.toml
├── sonar-project.properties
├── .env.example
├── .gitignore
└── README.md
```

## 3. Componentes

| Componente | Responsabilidad | Depende de |
|-----------|-----------------|-----------|
| config | Cargar configuración desde env vars | - |
| logging_conf | Logging estructurado común | config |
| consumer.kafka_consumer | Leer del topic con alto throughput | config, metrics |
| lake.mongo_lake | Persistir mensaje crudo en Mongo | config, models.raw |
| cache.redis_buffer | Bufferizar fragmentos por clave hasta consolidar | config |
| processing.detector | Detectar el tipo de fragmento | models.raw |
| processing.normalizer | Normalizar campos (casing, acentos, espacios) | - |
| processing.matcher | Calcular clave de unión de persona | normalizer |
| processing.consolidator | Unir fragmentos en Person | matcher, models.person |
| warehouse.person_repo | Upsert idempotente en Postgres | engine, db_models |
| metrics.prometheus | Exponer métricas de rendimiento | - |
| api | Consultas de solo lectura sobre Postgres | warehouse |
| frontend | UI de consulta | api |
| pipeline | Orquestación end-to-end | todos |

## 4. Patrones de diseño aplicados

- **Pipes & Filters**: cada etapa transforma y pasa al siguiente.
- **Repository**: `person_repo` / `mongo_lake` encapsulan el acceso a datos.
- **Strategy**: `matcher` permite cambiar la estrategia de emparejamiento sin tocar el consolidador.
- **Adapter**: clientes de Kafka/Mongo/Redis/Postgres tras interfaces propias (facilita tests/mocks).
- **Dependency Injection ligera**: dependencias construidas en `pipeline.py` y pasadas a los componentes.
- **Settings/Config object**: un único objeto de configuración inmutable.

## 5. Decisiones tecnológicas (con rationale)

- **confluent-kafka** sobre kafka-python: mejor rendimiento (binding librdkafka) para miles de msg/s.
  Fallback a kafka-python si hay fricción de instalación en Windows.
- **PostgreSQL + SQLAlchemy 2.x**: ORM maduro, migración fácil, upserts con `ON CONFLICT`.
- **MongoDB + pymongo**: staging schema-less para el crudo.
- **Redis**: buffer por clave con TTL para el matching de fragmentos que llegan desasociados.
- **pydantic v2 / pydantic-settings**: validación y configuración 12-factor.
- **FastAPI + uvicorn**: API async performante y con OpenAPI automática.
- **Streamlit**: frontend mínimo con muy poco código (apto para el perfil menos técnico).
- **prometheus-client**: instrumentación estándar.
- **pytest + pytest-cov + mongomock + fakeredis**: tests sin infra real.

## 6. Modelo de datos relacional (Postgres, borrador)

Tabla `persons` (consolidado, 1 fila por persona):
`id (pk)`, `passport (unique, nullable)`, `full_name`, `name`, `lastname`, `sex`, `phone`, `email`,
`city`, `address`, `company`, `company_address`, `company_phone`, `company_email`, `job`,
`iban`, `salary`, `ipv4`, `created_at`, `updated_at`.
Índices: `passport`, `full_name_normalized`. Upsert por `passport` cuando exista; si no, por
`full_name_normalized`.

En una iteración avanzada puede normalizarse en varias tablas (person, employment, finance, network)
si el equipo lo prefiere; el borrador plano es suficiente para Esencial/Medio.

## 7. Estrategia de matching (el núcleo del reto)

1. Normalizar todos los campos de texto (strip, lower, quitar acentos, colapsar espacios).
2. Determinar clave de persona por prioridad:
   - `passport` si el fragmento lo tiene (Personal, Bank).
   - si no, `full_name_normalized` (Location, Professional).
   - `address_normalized` como puente Location <-> Net.
3. Bufferizar fragmentos en Redis por clave con TTL. Cuando hay suficientes tipos (o al expirar),
   consolidar y persistir en Postgres.
4. Fragmentos huérfanos o inconsistentes: se registran (log + métrica) y se persiste lo disponible;
   nunca tumban el pipeline.

## 8. Cross-cutting concerns

- **Error handling**: excepciones específicas por capa; reintentos con backoff en I/O (Kafka/DB/Redis);
  el pipeline nunca cae por un mensaje malo (dead-letter lógico vía log/métrica).
- **Logging**: estructurado (JSON) con nivel configurable; correlación por offset/clave de persona.
- **Config**: 100% por variables de entorno; `.env.example` documentado; sin secretos en git.
- **DI**: composición en `pipeline.py`; componentes reciben sus dependencias (testeable).
- **Observabilidad**: `/metrics` (Prometheus) con consumidos, velocidad, latencia de procesado y de
  persistencia; `/health` en la API.
- **Idempotencia**: upserts por clave para evitar duplicados en reprocesos.
- **Escalabilidad**: varios consumers en el mismo consumer group; etapas desacopladas vía Lake/Redis.

## 9. Mapa niveles → componentes

- **Esencial**: consumer + lake(Mongo) + processing(detector/normalizer/matcher/consolidator) + warehouse(Postgres).
- **Medio**: logging_conf + tests + docker-compose + Dockerfiles.
- **Avanzado**: cache(Redis) + metrics(Prometheus) + api(FastAPI).
- **Experto**: carga continua (loop del pipeline siempre activo) + frontend(Streamlit).
