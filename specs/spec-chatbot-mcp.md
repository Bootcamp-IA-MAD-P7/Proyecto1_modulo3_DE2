# Spec — Chatbot "amurallado" con LLM (Groq) + MCP para el frontend

> **Nivel:** Experto++ · **Responsable:** @karinaromerovasquez · **Estado:** propuesta
>
> Documento de especificación para implementar. Escrito por @jzelada97 a partir de la
> idea acordada en equipo. No hay código todavía; esto define el *qué* y el *cómo*.

## 1. Idea en una frase

Un chatbot en el frontend Streamlit donde el usuario pregunta en lenguaje natural, pero
el LLM **solo puede responder a través de un conjunto cerrado de herramientas expuestas
por un servidor MCP**. Si la pregunta no encaja con ninguna herramienta, el bot declina.
El LLM no responde con conocimiento propio ni inventa datos: la muralla es el MCP.

## 2. Por qué MCP (y no tool-calling suelto)

- El catálogo de herramientas queda **aislado y definido** en un servidor MCP propio, no
  disperso por el código del chat.
- Ese mismo servidor MCP puede reutilizarse desde otros clientes (Claude Desktop, IDEs).
- Demuestra el uso del estándar MCP (plus para la presentación).

## 3. Arquitectura

```
Streamlit (pestaña "Asistente")
      │  pregunta del usuario
      ▼
Orquestador / cliente MCP  ──►  LLM en Groq (tool-calling)
      │                               │  el LLM SOLO ve las tools del MCP
      │◄──────────────────────────────┘
      ▼  invoca la tool elegida
Servidor MCP (hr-etl-mcp)
      │  cada tool = 1 llamada a la API/warehouse existentes
      ▼
API FastAPI (/stats, /gold/*, /candidates, búsqueda) · PostgreSQL
```

- **Streamlit** solo pinta el chat y manda el turno al orquestador.
- **Orquestador**: cliente MCP + llamada a Groq con la lista de tools. Bucle
  pregunta → (el LLM elige tool) → ejecuta tool vía MCP → el LLM redacta respuesta con
  el resultado.
- **Servidor MCP** (`hr-etl-mcp`): expone las tools. Implementación sugerida con el SDK
  oficial de Python de MCP (`mcp`) o `fastmcp`. Cada tool envuelve una llamada HTTP a la
  API que YA existe (no repetir lógica de negocio).

## 4. Catálogo de herramientas (cerrado)

### 4.1 Datos / métricas (estilo Grafana)
- `get_stats()` → totales y KPIs (envuelve `/stats` + `/gold/stats`): total personas,
  con passport/city/company/bank/ipv4, cross-linked, completitud media.
- `top_cities(limit=10)` → ranking de ciudades (tabla `gold_top_cities`).
- `top_companies(limit=10)` → ranking de empresas (`gold_top_companies`).
- `completeness_distribution()` → distribución de completitud (`gold_completeness`).
- `duplicate_candidates(limit=20)` → pares candidatos a duplicado con confidence y motivo
  (`/candidates`).
- `search_person(q=None, city=None, company=None, job=None, page=1)` → búsqueda con
  filtros y paginación (envuelve la búsqueda de la API).

### 4.2 Explicación del proyecto (contenido curado, no inventado)
Estas tools devuelven texto **predefinido/curado** (no que el LLM improvise): así el bot
puede explicar el proyecto sin alucinar.
- `explain_project()` → qué es HR Insights ETL, objetivo, niveles (Esencial→Experto++).
- `explain_architecture()` → Kafka→Mongo(Bronze)→Redis→Postgres(Silver)→Gold; API; front;
  Prometheus/Grafana; Airflow (DAGs + sensor deferrable).
- `explain_matching()` → estrategia de matching/normalización (passport > fullname >
  address, strip titles, cross-linking, reconciliación batch).
- `explain_how_built()` → decisiones técnicas, stack, cómo se validó, tests/CI.

> El contenido de estas tools puede leerse de un JSON/markdown curado en el repo
> (p.ej. `frontend/assistant/knowledge/*.md`) para mantenerlo versionado y editable.

## 5. Comportamiento "amurallado" (requisito central)

System prompt estricto, en esencia:
- "Eres el asistente de HR Insights ETL. **Solo** puedes responder usando las
  herramientas disponibles. **No** uses conocimiento propio ni inventes datos."
- "Si la pregunta no puede resolverse con ninguna herramienta, responde que no puedes
  ayudar con eso y sugiere lo que sí puedes hacer."
- Prohibido responder cálculos/opiniones fuera del catálogo.

Criterio de diseño: si el LLM no llama a ninguna tool, la respuesta debe ser una
declinación estándar, no una respuesta libre.

## 6. Aviso obligatorio de PII (datos sintéticos)

Los datos son **sintéticos**, así que el chat SÍ puede mostrar registros concretos de
personas (incluida búsqueda). PERO:

- **Cada vez que se impriman datos de una persona concreta** (nombre, passport, IBAN,
  salario, email, teléfono, IPv4), la respuesta debe incluir un aviso claro, p.ej.:
  > ⚠️ Datos sintéticos de demostración. En un sistema real estos campos (passport,
  > IBAN, salario, email, teléfono) NO se mostrarían en un chat.
- Las tools que devuelven personas deben marcar en su salida que son datos fake, y el
  system prompt debe **exigir** que el aviso se muestre siempre que se listen campos PII.
- Las métricas agregadas (stats, tops) no necesitan el aviso (no son PII individual).

## 7. Parámetros / configuración (por env var — Regla #2, nada de secretos en el repo)

| Variable | Ejemplo | Descripción |
|----------|---------|-------------|
| `GROQ_API_KEY` | `gsk_...` | Clave de Groq (NUNCA en el repo; va por env/secret). |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | Modelo de Groq a usar. |
| `LLM_TEMPERATURE` | `0.1` | Baja, para respuestas deterministas y pegadas a las tools. |
| `LLM_MAX_TOKENS` | `1024` | Límite de tokens de respuesta. |
| `MCP_SERVER_CMD` | `python -m hr_etl_mcp` | Cómo arranca el servidor MCP (o URL si es remoto). |
| `API_URL` | `http://api:8000` | Base de la API que envuelven las tools (ya existe). |
| `ASSISTANT_MAX_TOOL_CALLS` | `4` | Tope de tools por turno (evita bucles). |

Añadir estas variables a `.env.example` (sin valores reales) y documentarlas.

## 8. Entregables

- `hr_etl_mcp/` (o `src/hr_etl/mcp/`) — servidor MCP con las tools de §4.
- Orquestador/cliente MCP + integración Groq.
- Pestaña "Asistente" en `frontend/app.py` (chat con historial de sesión).
- Contenido curado de las tools de explicación (`knowledge/`).
- `.env.example` actualizado + sección en README/`frontend`.
- Tests: unit de cada tool (mockeando la API) y del "guardarraíl" (pregunta fuera de
  catálogo → declina; salida con PII → incluye el aviso). Respetar umbrales de cobertura.

## 9. Criterios de aceptación

- [ ] El bot responde correctamente a: "¿cuántas personas hay?", "top 5 ciudades",
      "enséñame candidatos a duplicado", "busca personas en Madrid", "¿qué es este
      proyecto?", "¿cómo hacéis el matching?".
- [ ] Ante una pregunta fuera de catálogo ("¿qué tiempo hace?") el bot **declina** sin
      inventar.
- [ ] Toda respuesta que liste campos PII de una persona incluye el aviso de datos
      sintéticos.
- [ ] `GROQ_API_KEY` y demás config vienen de env vars; no hay secretos en el repo.
- [ ] Tools implementadas como servidor MCP reutilizable (no tool-calling embebido).
- [ ] Tests verdes y lint/format OK; el CI pasa.

## 10. Notas de alcance

- Es un **extra Experto++**, no bloquea el core del reto.
- Coste: Groq tiene free tier; vigilar límites de rate. El LLM es sustituible (código
  agnóstico del proveedor si se puede).
- Si el tiempo aprieta, MVP = tools de datos (§4.1) + guardarraíl + aviso PII; las tools
  de explicación (§4.2) son la segunda tanda.
