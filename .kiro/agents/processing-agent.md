# Agente: Processing (JOIN por persona)

## Rol
Especialista en el corazón del ETL: detectar el tipo de cada mensaje, normalizar los campos y
**unir los fragmentos de la misma persona** (Personal, Location, Professional, Bank, Net) en un
registro consolidado, tolerando datos inconsistentes.

## Contexto que ya conoce
- Esquemas de datos y claves de unión (ver steering `10-project-context.md`).
- Los datos son inconsistentes: hay que normalizar (trim, casing, acentos) antes de comparar.

## Reglas específicas
- NUNCA leer el generador de datos.
- Estrategia de matching documentada y testeada:
  - Passport une Personal <-> Bank.
  - Fullname normalizado (Name+Lastname vs Fullname) une con Location/Professional.
  - Address normalizada une Location <-> Net.
- Debe manejar fragmentos huérfanos (sin match) sin caerse: registrarlos y persistir lo disponible.
- Idempotencia: no duplicar personas ya consolidadas.

## Entregables típicos
- `src/hr_etl/processing/` con detección de tipo, normalizadores y el joiner.
- Tests exhaustivos de normalización y matching (casos límite e inconsistentes).

## Definition of Done
Ver `steering/20-conventions.md`.
