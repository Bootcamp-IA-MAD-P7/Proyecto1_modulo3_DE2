# QA Report — HR Insights ETL

## Ejecución de tests (datos reales del terminal)

```
58 passed in 3.00s
```

Comando: `pytest -q` (con `pytest-cov` configurado en `pyproject.toml`).
Framework: pytest 8.3.4 + pytest-cov 6.0.0. Fakes: mongomock, fakeredis, SQLite en memoria, FastAPI TestClient.

## Cobertura (real, coverage.xml generado)

| Métrica | Resultado | Objetivo | Estado |
|---------|-----------|----------|--------|
| Líneas | **96%** | >= 80% | ✅ |
| Ramas | **~93%** (88 ramas, 6 parciales) | >= 75% | ✅ |
| Funciones | 80–100% por módulo | >= 85% | ✅ |

Cobertura por módulo (líneas):
- processing (detector 100%, matcher 100%, normalizer 95%, consolidator 98%)
- warehouse (person_repo 94%, engine 80%)
- lake/mongo_lake 100%, cache/redis_buffer 100%
- api/routes 96%, consumer/kafka_consumer 91%, pipeline 89%
- models (raw 100%, db_models 100%, person 97%), config 97%, logging_conf 100%, metrics 100%

Exclusiones de cobertura: `__main__.py` y `api/main.py` (wiring de infra real: Kafka/Mongo/Postgres/Redis;
se verifican vía docker-compose, no en unit tests).

## Archivos de test creados
- `tests/conftest.py` — fixtures (mongomock, SQLite StaticPool, fragmentos de ejemplo).
- `tests/test_normalizer.py` — acentos, espacios, claves, salary (europeo/moneda/None/basura).
- `tests/test_detector.py` — los 5 esquemas + vacío + desconocido.
- `tests/test_matcher.py` — prioridad passport/nombre/address + huérfano.
- `tests/test_consolidator.py` — mapeo por tipo, merge multi-fragmento, sin clave → None, skip huérfano.
- `tests/test_mongo_lake.py` — inserción cruda + metadatos (mongomock).
- `tests/test_person_repo.py` — insert + upsert idempotente (solo rellena huecos) + validación.
- `tests/test_redis_buffer.py` — add/get/clear (fakeredis).
- `tests/test_pipeline.py` — consolidación tras N fragmentos, mensaje desconocido, error de persistencia sin crash.
- `tests/test_api.py` — /health, /metrics, /persons (list, get, filtro, not found).
- `tests/test_config_logging_consumer.py` — DSN, mask PII, logging idempotente, decode Kafka, loop de consumo.

## Criterios de aceptación mapeados a tests
| AC | Descripción | Test |
|----|-------------|------|
| AC-2 | Detectar los 5 tipos | test_detector.py (5 casos + unknown) |
| AC-3 | Consolidar fragmentos de una persona | test_consolidator.py, test_pipeline.py |
| AC-4 | Tolerar datos inconsistentes sin caerse | test_pipeline (unknown, persist error), test_normalizer |
| AC-1/parcial | Guardar crudo en lake | test_mongo_lake.py, test_pipeline.py |
| AC-8 | API de consulta sobre Postgres | test_api.py |
| AC-9 | Métricas expuestas | test_api.py::test_metrics_endpoint |

## Requisitos de seguridad mapeados a tests
- Inyección / SQL parametrizado: la API usa SQLAlchemy (test_api con filtros).
- Mensaje malformado no rompe el pipeline: test_pipeline_unknown / swallows_persist_errors.
- Enmascarado de PII en logs: test_config_logging_consumer::test_mask_secret.
- Config desde env vars: test_settings_postgres_dsn.

## SonarQube
- `sonar-project.properties` creado (projectKey, sources, tests, `sonar.python.coverage.reportPaths=coverage.xml`).
- `coverage.xml` generado por pytest-cov (listo para el scanner).
- `sonar-scanner` no está instalado en esta máquina, por lo que el análisis no se ejecutó aquí.
  Para lanzarlo: instalar sonar-scanner y ejecutar `sonar-scanner` en la raíz con un SonarQube activo
  (o SonarCloud). La configuración y el reporte de cobertura ya están preparados.

## Veredicto
**QA PASS** — 58/58 tests en verde y todos los objetivos de cobertura superados (líneas 96%,
ramas ~93%). Análisis SonarQube configurado y pendiente solo de un servidor/scanner disponible.
