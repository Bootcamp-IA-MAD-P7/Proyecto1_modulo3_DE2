# Development Report — HR Insights ETL

## Archivos creados
### Configuración e infraestructura
- `pyproject.toml`, `.gitignore`, `.env.example`
- `docker-compose.yml` (app + api + frontend + mongo + postgres + redis + prometheus)
- `docker/Dockerfile.app`, `docker/Dockerfile.api`, `docker/Dockerfile.frontend`
- `monitoring/prometheus.yml`, `sonar-project.properties`

### Código fuente (`src/hr_etl/`)
- `config.py` — configuración 12-factor (pydantic-settings), DSN de Postgres.
- `logging_conf.py` — logging estructurado JSON + `mask_secret` para PII.
- `models/raw.py` — FragmentType + firmas de claves por esquema.
- `models/person.py` — modelo consolidado Person con `merge()` idempotente.
- `models/db_models.py` — tabla SQLAlchemy `persons`.
- `processing/normalizer.py` — normalización de texto/claves, `clean_salary` (formato europeo).
- `processing/detector.py` — detección de tipo por cobertura de claves.
- `processing/matcher.py` — clave de persona por prioridad (passport > nombre > address).
- `processing/consolidator.py` — mapeo fragmento→Person y merge de fragmentos.
- `lake/mongo_lake.py` — escritura cruda en Mongo con metadatos.
- `cache/redis_buffer.py` — buffer de fragmentos por clave con TTL.
- `warehouse/engine.py` — engine/sesión + init de esquema.
- `warehouse/person_repo.py` — upsert idempotente por `match_key`.
- `metrics/prometheus.py` — contadores e histogramas.
- `consumer/kafka_consumer.py` — wrapper de Kafka (import diferido, inyectable).
- `api/main.py`, `api/routes.py` — API FastAPI de solo lectura (/health, /metrics, /persons).
- `pipeline.py` — orquestación por mensaje.
- `__main__.py` — entrypoint ejecutable que cablea toda la infra.

### Frontend
- `frontend/app.py` — Streamlit de consulta vía API.

## Cumplimiento de arquitectura
- [x] Pipes & Filters por etapas (decode → detect → lake → buffer → consolidate → warehouse).
- [x] Data Lake (Mongo) + Data Warehouse (Postgres) + Redis para matching en tiempo real.
- [x] Patrones Repository, Strategy (matcher), Adapter (clientes inyectables), DI en `pipeline.py`/`__main__.py`.
- [x] Estructura de carpetas conforme a `20-conventions.md`.

## Cumplimiento de seguridad
- [x] Sin secretos hardcodeados; todo por env vars (`config.py`).
- [x] `.env` en `.gitignore`; `.env.example` sin valores reales.
- [x] SQLAlchemy parametrizado (sin concatenación SQL).
- [x] Validación de entrada (decode tolerante, pydantic en modelos/API, límites de paginación).
- [x] `mask_secret` disponible para no filtrar PII en logs.
- [x] Redis con `requirepass`; usuarios de BD dedicados en compose.
- [x] Pipeline nunca cae por un mensaje malo (try/except + métrica de fallo).

## Decisiones técnicas
- Import diferido de `confluent-kafka` para permitir tests sin la librería nativa.
- `match_key` con prefijo de tipo (`passport:` / `name:` / `addr:`) para trazabilidad.
- `clean_salary` maneja separadores europeos y símbolos de moneda.
- Upsert que solo rellena huecos (no pisa datos buenos) → idempotente y tolerante a reprocesos.
- Detección por ratio de cobertura de claves (≥0.5) para tolerar variaciones de nombres de campo.

## Regla crítica
- No se ha leído ni inspeccionado el generador de datos. Los nombres de campo provienen del README público.

## Verificación realizada
- Entorno virtual creado, dependencias instaladas, paquete importable.
- Smoke test de lógica: detección correcta, join Personal+Bank por passport, salary europeo parseado.

## Testing hints para QA
- Cubrir normalizer (acentos, espacios, salary europeo/None/basura).
- Cubrir detector con los 5 esquemas + desconocido + vacío.
- Cubrir matcher (prioridad passport/nombre/address y fragmento huérfano).
- Cubrir consolidator (merge multi-fragmento, sin clave → None).
- Cubrir person_repo (insert + upsert que rellena huecos) con SQLite en memoria.
- Cubrir mongo_lake con mongomock; redis_buffer con fakeredis.
- Cubrir API con TestClient + session_factory inyectada (SQLite).
- Seguridad: mensaje malformado no rompe pipeline; logs enmascaran PII.

## Dependencias añadidas
Ver `pyproject.toml`. Runtime: confluent-kafka, pymongo, SQLAlchemy, psycopg2-binary, redis,
pydantic(+settings), prometheus-client, fastapi, uvicorn, python-dotenv. Dev: pytest, pytest-cov,
mongomock, fakeredis, httpx, ruff, black. Frontend: streamlit, requests.
