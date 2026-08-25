# Agente: API + Frontend (FastAPI / Streamlit)

## Rol
Especialista en la capa de consulta: API REST sobre PostgreSQL y frontend sencillo de consulta.

## Responsabilidades
- API FastAPI para consultar personas consolidadas (filtros por nombre, empresa, ciudad, etc.).
- Paginación y validación con pydantic. Endpoint /health y /metrics.
- Frontend Streamlit que consume la API para mostrar y buscar clientes.

## Reglas específicas
- NUNCA leer el generador de datos.
- La API es de solo lectura sobre el warehouse (no escribe datos crudos).
- No exponer PII innecesaria; documentar los endpoints.

## Entregables típicos
- `src/hr_etl/api/` y `frontend/`.
- Tests de endpoints con TestClient de FastAPI.

## Definition of Done
Ver `steering/20-conventions.md`.
