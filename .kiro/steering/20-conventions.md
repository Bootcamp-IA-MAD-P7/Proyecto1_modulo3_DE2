---
inclusion: auto
name: conventions
description: Convenciones de código y estructura del proyecto — estructura de carpetas src/hr_etl, estilo Python (black, isort, ruff, type hints), umbrales de tests y Definition of Done. Cargar al escribir o refactorizar código, crear archivos nuevos o preparar PRs.
---

# Convenciones de Código y Estructura

## Estructura de carpetas del proyecto
```
Proyecto1_modulo3_DE2/
├── src/
│   └── hr_etl/
│       ├── __init__.py
│       ├── config.py            # Configuración 12-factor (pydantic-settings)
│       ├── logging_conf.py      # Logging estructurado
│       ├── models/              # Modelos pydantic + esquemas SQLAlchemy
│       ├── consumer/            # Consumer de Kafka
│       ├── lake/                # Escritura cruda en MongoDB (Data Lake)
│       ├── processing/          # Detección de tipo, normalización y JOIN por persona
│       ├── warehouse/           # Escritura en PostgreSQL (Data Warehouse)
│       ├── cache/               # Cliente Redis (buffer/matching intermedio)
│       ├── metrics/             # Métricas Prometheus
│       ├── api/                 # API FastAPI de consulta
│       └── pipeline.py          # Orquestación del flujo
├── frontend/                    # App Streamlit
├── tests/                       # pytest (unit + integración)
├── docker/                      # Dockerfiles
├── docker-compose.yml           # App + Mongo + Postgres + Redis + Prometheus
├── pyproject.toml
├── .env.example
└── README.md
```

## Estilo Python
- PEP 8, formateo con `black`, imports con `isort`, lint con `ruff`.
- Type hints obligatorios en funciones públicas.
- Docstrings en funciones/clases no triviales.
- Modelos de datos con `pydantic`; ORM con `SQLAlchemy`.
- Nada de credenciales hardcodeadas: todo por `config.py` desde variables de entorno.

## Tests
- `pytest`, `pytest-cov`. Umbrales: líneas >=80%, ramas >=75%, funciones >=85%.
- Un archivo de test por módulo fuente. La lógica de join/normalización debe estar bien cubierta.

## Definition of Done (por Issue)
- [ ] Código en rama `feature/*` con commits limpios.
- [ ] Tests unitarios añadidos y en verde.
- [ ] Lint (`ruff`) y formato (`black`) sin errores.
- [ ] Documentación mínima (docstring + nota en README si aplica).
- [ ] PR hacia `dev` revisado por al menos otro miembro.
