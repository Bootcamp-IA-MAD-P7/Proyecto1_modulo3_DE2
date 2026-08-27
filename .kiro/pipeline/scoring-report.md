# Development Scoring Report

## Overall Score: 8.6 / 10.0 — PASS ✅

> Threshold: 7.0 to PASS, 5.0-6.9 CONDITIONAL (minor fixes needed), <5.0 FAIL (rework required)

## Dimension Scores

| Dimension | Score | Weight | Weighted | Notes |
|-----------|-------|--------|----------|-------|
| Requirements Compliance | 9/10 | 25% | 2.25 | Todos los FR implementados (Esencial→Experto) |
| Architecture Compliance | 9/10 | 20% | 1.80 | Estructura exacta al diseño; patrones bien aplicados |
| Security Compliance | 9/10 | 20% | 1.80 | PII masking, env vars, ORM parametrizado, non-root Docker |
| Code Quality | 8/10 | 15% | 1.20 | Buenos type hints y docstrings; black no aplicado en 17 archivos |
| Error Handling | 8/10 | 10% | 0.80 | Pipeline resiliente; falta retry con backoff en persistencia |
| Testability | 9/10 | 10% | 0.90 | 91% cobertura, mocks bien usados, fixtures claras |
| **TOTAL** | | **100%** | **8.75** | |

---

## Requirements Compliance Detail (9/10)

### Implemented ✅

| Req | Descripción | Implementación |
|-----|-------------|----------------|
| FR-1 | Consumer Kafka alto throughput | `consumer/kafka_consumer.py` — confluent-kafka, commit manual, import diferido |
| FR-2 | Persistir crudo en MongoDB | `lake/mongo_lake.py` — store_raw + batch mode con flush por tamaño/tiempo |
| FR-3 | Identificar tipo (5 esquemas) | `processing/detector.py` — cobertura de keys >= 50%, soporta variaciones |
| FR-4 | JOIN fragmentos por persona | `processing/matcher.py` + `cache/redis_buffer.py` + cross-linking via alias |
| FR-5 | Normalización de datos sucios | `processing/normalizer.py` — accents, whitespace, salary, key aliases |
| FR-6 | Persiste en PostgreSQL | `warehouse/person_repo.py` — upsert ON CONFLICT con COALESCE |
| FR-7 | Logs estructurados | `logging_conf.py` — JSON format, PIIMaskingFilter |
| FR-8 | Tests unitarios | 15 módulos de test, 79 tests pasando |
| FR-9 | Docker completo | docker-compose.yml con 6 servicios + healthchecks + override |
| FR-10 | Redis como caché | `cache/redis_buffer.py` — buffer con TTL + alias cross-linking |
| FR-11 | Métricas Prometheus | `metrics/prometheus.py` — 7 métricas; Prometheus scraping app y API |
| FR-12 | API REST | `api/routes.py` — /health, /metrics, /persons (paginado+filtros+búsqueda), /stats |
| FR-13 | Carga continua | `__main__.py` — loop infinito (max_records=None), at-least-once delivery |
| FR-14 | Frontend Streamlit | `frontend/app.py` — dashboard con métricas, gráficos, búsqueda, detalle |

### Partially Implemented ⚠️

| Aspecto | Detalle |
|---------|---------|
| NFR-2 Escalabilidad horizontal | El consumer group está configurado pero no hay documentación de cómo escalar a N consumers (solo se menciona en arquitectura) |
| NFR-5 Reintentos con backoff | El pipeline captura excepciones pero NO reintenta con backoff exponencial; simplemente descarta el mensaje y cuenta error |
| AC-4 Gestión de no-emparejables | Se cuentan y loguean como warnings pero no hay un "dead-letter queue" explícito ni tabla de huérfanos en Postgres |

### Missing ❌

Ningún requisito funcional está ausente.

---

## Architecture Compliance Detail (9/10)

### Followed ✅

| Patrón/Estructura | Implementación |
|-------------------|----------------|
| Pipes & Filters por etapas | `pipeline.py` orquesta decode→detect→lake→buffer→consolidate→warehouse |
| Data Lake (MongoDB) | `lake/mongo_lake.py` — schema-less, payload original intacto |
| Data Warehouse (PostgreSQL) | `warehouse/` — modelo relacional normalizado con SQLAlchemy |
| Redis buffer con TTL | `cache/redis_buffer.py` — acumula fragmentos hasta min_fragments |
| Repository pattern | `person_repo.py` y `mongo_lake.py` encapsulan acceso a datos |
| Strategy pattern | Matcher permite cambiar lógica de matching sin tocar consolidator |
| Adapter/DI | Clientes inyectables en Pipeline; `__main__.py` compone todo |
| Config 12-factor | `config.py` con pydantic-settings, todo desde env vars |
| Estructura de carpetas | Exacta al diseño en `architecture.md` y `20-conventions.md` |
| Dockerfiles multi-stage | Build vs runtime separados, non-root user |
| Healthchecks en compose | Mongo, Postgres, Redis con checks configurados |
| Red interna Docker | Puertos de BD no expuestos en compose base (solo en override) |

### Deviations ⚠️ (menores)

| Esperado | Actual | Impacto |
|----------|--------|---------|
| `docker/` con 3 Dockerfiles | ✅ Correcto | — |
| Tests en `tests/` con naming `test_<module>.py` | También hay `test_coverage_gaps.py` y `test_mongo_lake_integration.py` (extras) | Positivo: más cobertura |
| Batch mode en lake (opcional) | Implementado con `buffer_raw()` + `flush()` | Positivo: mayor throughput |
| Pipeline usa `store_raw()` (sync, uno a uno) | No usa el batch mode en el pipeline real | Menor: el store por mensaje es suficiente para el ejercicio |

---

## Security Compliance Detail (9/10)

### Satisfied ✅

| Requisito de seguridad | Implementación |
|------------------------|----------------|
| Sin secretos hardcodeados | `config.py` lee todo de env vars; `.env.example` tiene placeholders |
| `.env` en `.gitignore` | ✅ Incluido; también `*.env` excepto `.env.example` |
| SQL parametrizado (no concatenación) | SQLAlchemy ORM + `pg_insert` con `on_conflict_do_update` |
| Validación de entrada | Detector/Normalizer toleran cualquier mensaje; API con `Query(ge=, le=)` |
| PII masking en logs | `PIIMaskingFilter` cubre passport, IBAN, email; tests lo verifican |
| Redis con contraseña | `--requirepass` en compose; `redis_password` en config |
| Usuarios BD dedicados | `POSTGRES_USER=hr_user`, `MONGO_INITDB_ROOT_USERNAME` |
| Non-root en Docker | `USER appuser` (uid 10001) en los 3 Dockerfiles |
| API solo lectura | Solo endpoints GET; no hay POST/PUT/DELETE |
| Puertos internos por defecto | `docker-compose.yml` base no expone puertos de BD al host |
| Dependencias pinneadas | Versiones exactas en `pyproject.toml` |

### Minor Observations ⚠️

| Aspecto | Detalle | Recomendación |
|---------|---------|---------------|
| API sin rate limiting | El endpoint /persons acepta hasta 500 items por página | Añadir rate limiting (ej: slowapi) para entornos compartidos |
| IBAN/salary visibles en API | `/persons` devuelve IBAN y salary sin enmascarar | Para un entorno real, enmascarar en respuesta (solo últimos 4 del IBAN) |
| Mongo user root | El container usa `MONGO_INITDB_ROOT_USERNAME` (admin) | Crear un usuario con permisos solo sobre `hr_lake` |

---

## Code Quality Issues

### Critical (must fix)

Ninguno.

### Warnings (should fix)

| File | Área | Issue | Recomendación |
|------|------|-------|---------------|
| 17 archivos | Formato | `black --check` reporta 17 archivos con diferencias (mayormente líneas >100 chars) | Ejecutar `black src/ tests/` una vez y commitear |
| `api/routes.py:67,69` | Filtro city | `PersonRow.city == city.strip().lower()` — hace lowercase en Python pero los datos en BD pueden estar en su formato original | Usar `.ilike()` en vez de `==` para consistencia (como ya se hace con company/job) |
| `pipeline.py` | Cross-link | La función `_register_cross_link` normaliza el mensaje de nuevo (ya se hizo en matcher) | Cachear el mensaje normalizado para evitar doble trabajo |

### Suggestions (nice to have)

| File | Área | Issue | Recomendación |
|------|------|-------|---------------|
| `redis_buffer.py` | TTL alias | El alias TTL es `ttl * 3` hardcodeado | Extraer a parámetro configurable |
| `pipeline.py` | Buffer clear | Tras consolidar, los fragmentos NO se borran del Redis (`clear()` nunca se llama en el flujo normal) | Llamar `self._buffer.clear(key)` tras consolidar exitosamente para liberar memoria |
| `person_repo.py` | Dual path | Hay dos rutas de upsert (SQLite-compatible y native Postgres) | Documentar cuándo se usa cada una; actualmente el pipeline usa `upsert()` (compatible) |
| `frontend/app.py` | Error UX | Si la API está caída, `st.stop()` sale sin contexto | Mostrar instrucciones de troubleshooting al usuario |
| `consumer/kafka_consumer.py` | Timeout | El poll timeout es hardcoded como parámetro de `consume()` | Moverlo a Settings para configurabilidad |
| README.md | Cobertura | El README dice "~94% líneas, ~90% ramas" | La medición real es 91% combinado; actualizar cifra |

---

## Complexity Analysis

| File | Cyclomatic Complexity | Cognitive Complexity | Assessment |
|------|----------------------|---------------------|------------|
| `pipeline.py` (process_message) | 8 | Medio | OK — flujo lineal con guards |
| `normalizer.py` (clean_salary) | 5 | Bajo | OK |
| `consolidator.py` (fragment_to_person) | 6 | Bajo | OK — switch por tipo |
| `detector.py` (detect_type) | 4 | Bajo | OK |
| `person_repo.py` (upsert) | 4 | Bajo | OK |
| `api/routes.py` (list_persons) | 6 | Medio | OK — filtros opcionales |
| `frontend/app.py` | 5 | Medio | OK — script lineal |

Ningún archivo supera umbrales preocupantes. El código es claro y con baja complejidad cognitiva.

---

## Fortalezas Destacadas 💪

1. **Cross-linking inteligente**: El sistema de alias en Redis (name → passport) es una solución elegante al problema de matching distribuido. No es trivial y demuestra comprensión profunda del dominio.

2. **Upsert con COALESCE**: El `ON CONFLICT DO UPDATE SET x = COALESCE(persons.x, EXCLUDED.x)` garantiza que nunca se pierde información — un fragmento nuevo solo rellena huecos.

3. **Testing sin infraestructura**: El uso combinado de mongomock, fakeredis y SQLite in-memory permite tests rápidos y reproducibles sin Docker. Los tests de integración con infra real están separados correctamente.

4. **Batch mode en lake**: El `buffer_raw()` con flush por tamaño o intervalo es una optimización real para alto throughput que va más allá de lo mínimo requerido.

5. **README excepcional**: Documentación exhaustiva con diagramas, tablas de decisiones, queries Prometheus, y limitaciones conocidas honestas. Nivel de README profesional.

6. **Métricas bien pensadas**: Las 7 métricas Prometheus cubren exactamente lo que un operador necesita (throughput, errores, latencia de proceso y persistencia, fragmentos pendientes).

---

## Verdict

### Decision: PASS ✅

El proyecto cumple **todos los requisitos funcionales** de los 4 niveles (Esencial, Medio, Avanzado, Experto). La arquitectura es limpia y coherente, la seguridad está bien implementada, y los tests son robustos. El código es legible, bien documentado y con complejidad controlada.

### Fixes Recomendados Antes de Entregar (mejoran la nota)

1. **Aplicar `black`**: `black src/ tests/` y commitear. Tarda 2 segundos y elimina los 17 archivos con formato inconsistente.

2. **Filtro city en API**: Cambiar `PersonRow.city == city.strip().lower()` a `PersonRow.city.ilike(f"%{city.strip()}%")` para consistencia con los demás filtros.

3. **Llamar `buffer.clear(key)`** en `pipeline.py` después de consolidar exitosamente para liberar fragmentos ya procesados del Redis.

4. **Actualizar cifras de cobertura** en el README (91% real vs ~94% declarado).

### Recommendations for QA Agent

- Verificar que el pipeline end-to-end funciona con el generador real del profesor (docker compose up ambos stacks).
- Stress test: consumir 10K+ mensajes y validar que no hay memory leaks en el buffer Redis (si `clear()` no se llama, los fragmentos consolidados persisten hasta el TTL).
- Verificar el edge case del orden de llegada: Personal DESPUÉS de Location (el cross-link no se resuelve retroactivamente).
- Validar que el frontend muestra datos correctamente cuando hay datos bancarios con IBAN (no enmascarado en la API actual).
- Confirmar que la red `data-generator_default` se resuelve correctamente en Docker Desktop Windows.

---

## Resumen para el Profesor

Este proyecto implementa un pipeline ETL completo de nivel **Experto** que:
- Consume en tiempo real de Kafka con alto throughput (confluent-kafka nativo).
- Almacena crudos en MongoDB y consolidados en PostgreSQL.
- Resuelve el reto del matching sin ID global usando una estrategia por prioridad (passport > nombre > dirección) con cross-linking via Redis.
- Incluye API REST con búsqueda, frontend interactivo, métricas Prometheus.
- Todo dockerizado con docker-compose, healthchecks, y non-root containers.
- 79 tests con 91% de cobertura, lint limpio (ruff 0 errores), y documentación extensa.

Los puntos débiles son menores: formato black pendiente (cosmético), un filtro de ciudad inconsistente en la API, y los fragmentos no se limpian del buffer tras consolidar (se auto-expiran por TTL pero consumen memoria innecesariamente).
