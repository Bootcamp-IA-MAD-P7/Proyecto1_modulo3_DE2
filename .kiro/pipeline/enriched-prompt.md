# Enriched Prompt

## Summary
Sistema de ingeniería de datos para "HR Insights / HR Pro" que consume en tiempo real miles de
mensajes por segundo desde un servidor Apache Kafka (proporcionado como caja negra), persiste los
mensajes crudos en una base de datos documental (MongoDB, actuando como Data Lake), procesa y
**une los fragmentos de datos de una misma persona** (Personal, Location, Professional, Bank, Net)
en un único registro consolidado, y persiste el resultado limpio y estructurado en una base de
datos relacional PostgreSQL (Data Warehouse). Todo el sistema se despliega en contenedores con
Docker y docker-compose. El proyecto es un trabajo de equipo (4 personas) desarrollado sobre un
Agentic Harness compartido versionado en el repositorio.

## Functional Requirements
- [FR-1] Consumir mensajes del topic de Kafka en tiempo real, soportando alto throughput (miles de msg/s).
- [FR-2] Persistir cada mensaje crudo tal cual llega en MongoDB (Data Lake / staging documental).
- [FR-3] Identificar el tipo de cada mensaje entre los 5 esquemas: Personal, Location, Professional, Bank, Net.
- [FR-4] Agrupar/unir (join) los distintos fragmentos que pertenecen a la misma persona en un único registro.
- [FR-5] Limpiar y normalizar los datos, tolerando inconsistencias (el dataset es deliberadamente "sucio").
- [FR-6] Persistir los registros consolidados en PostgreSQL con un esquema relacional normalizado.
- [FR-7] Sistema de logs estructurado en todos los componentes (Nivel Medio).
- [FR-8] Tests unitarios de la lógica de transformación/join y de los writers (Nivel Medio).
- [FR-9] Dockerizar la aplicación completa con docker y docker-compose (Nivel Medio).
- [FR-10] Usar Redis como caché intermedia para acelerar la captura en tiempo real y el matching de fragmentos (Avanzado).
- [FR-11] Monitorización de métricas: mensajes consumidos, velocidad, tiempo de procesado y de persistencia (Prometheus) (Avanzado).
- [FR-12] API REST para consultar la información final procesada en PostgreSQL (Avanzado).
- [FR-13] Carga continua: las BD se actualizan de forma continua mientras Kafka sigue emitiendo (Experto).
- [FR-14] Frontend sencillo (Streamlit/Gradio) para consultar los datos de clientes (Experto).

## Non-Functional Requirements
- [NFR-1] Alto rendimiento en el consumer: no perder mensajes bajo carga (backpressure, commits controlados).
- [NFR-2] Escalabilidad horizontal del procesamiento (varios consumers en el mismo consumer group).
- [NFR-3] Portabilidad total vía contenedores; arranque con un solo `docker compose up`.
- [NFR-4] Observabilidad: logs + métricas Prometheus + endpoints de health.
- [NFR-5] Resiliencia: reintentos y tolerancia a fallos transitorios de Kafka/DB/Redis.
- [NFR-6] Idempotencia en la persistencia (evitar duplicados al reprocesar).
- [NFR-7] Configuración por variables de entorno (12-factor), sin secretos hardcodeados.
- [NFR-8] Código mantenible y modular para permitir trabajo en paralelo de 4 personas.
- [NFR-9] Cobertura de tests: lineas >= 80%, ramas >= 75%, funciones >= 85%.

## Technical Context
- **Language/Framework**: Python 3.12 (última estable disponible en la máquina: 3.12.2). API con FastAPI. Frontend con Streamlit.
- **Existing patterns**: Repositorio nuevo (vacío salvo git init). Se define arquitectura desde cero.
- **Dependencies**: confluent-kafka o kafka-python, pymongo, SQLAlchemy + psycopg2/asyncpg, redis, pandas, pydantic, prometheus-client, fastapi/uvicorn, streamlit, pytest.
- **Target environment**: Docker + docker-compose en Windows (Docker Desktop presente en las 4 máquinas). Docker 29.x, Compose v5.x.
- **Kafka**: servidor externo proporcionado (caja negra) que se levanta con su propio `docker-compose up --build`. Nuestra app se conecta por red al broker.

## Constraints
- **CRÍTICO**: PROHIBIDO leer, inspeccionar o hacer reverse engineering del código del generador de datos / servidor Kafka. Solo se ejecuta. Violarlo puede suponer descalificación.
- SQL relacional debe ser PostgreSQL (decisión del equipo).
- Base documental: MongoDB.
- Control de versiones: GitHub. Ramas: `main` + `dev` + `feature/*`.
- Gestión de proyecto: GitHub Issues + GitHub Projects (tablero Kanban).
- Equipo de 4 personas con roles por área (ver .kiro/steering/30-team-roles.md).
- Los datos llegan **fragmentados por tipo**, no agrupados; el join por persona es responsabilidad del ETL.
- Los datos pueden ser **inconsistentes** (claves de unión imperfectas: Passport, Fullname, Address).

## Acceptance Criteria
- [AC-1] Al ejecutar el sistema con el Kafka del profesor activo, los mensajes crudos aparecen en MongoDB.
- [AC-2] Se identifican correctamente los 5 tipos de esquema.
- [AC-3] Los fragmentos de una misma persona se consolidan en un único registro en PostgreSQL.
- [AC-4] El proceso tolera datos inconsistentes sin caerse (registra y gestiona los no-emparejables).
- [AC-5] `docker compose up` levanta app + MongoDB + PostgreSQL + Redis (+ Prometheus) sin intervención manual.
- [AC-6] Existen logs consultables por componente.
- [AC-7] Los tests unitarios pasan y se cumplen los umbrales de cobertura.
- [AC-8] La API responde consultas sobre los datos consolidados en PostgreSQL.
- [AC-9] Métricas de rendimiento expuestas en /metrics (Prometheus).
- [AC-10] Frontend muestra datos de clientes desde PostgreSQL.
- [AC-11] El repositorio tiene ramas organizadas, commits limpios, README y documentación.

## Security Considerations
- **Sensitivity level: HIGH** — el dataset contiene PII y datos financieros: nombre, sexo, teléfono,
  pasaporte, email, dirección, IBAN, salario, IPv4.
- **Data types involved**: identificadores personales (Passport, email, teléfono), localización
  (dirección, ciudad), datos financieros (IBAN, salario), datos de red (IPv4). Aunque son datos
  sintéticos, se tratan con las mismas prácticas que datos reales (buena práctica pedagógica).
- Sin secretos en el repo; credenciales de BD por variables de entorno / `.env` (git-ignored).
- Principio de mínimo privilegio en usuarios de BD. Sin exponer puertos innecesarios al host.

## Out of Scope
- Autenticación de usuarios finales / gestión de identidades (no requerido por el enunciado).
- Cualquier análisis del generador de datos o de su lógica interna.
- Despliegue en cloud / CI-CD en plataformas externas (el "harness" aquí es agéntico + docker-compose local; GitHub Actions es opcional/stretch).
- Análisis de negocio de RRHH (dashboards analíticos avanzados) más allá del frontend de consulta.
