# Agentic Harness — Proyecto HR Insights ETL

Este directorio es el **harness agéntico compartido** del equipo. Todo lo que hay aquí viaja por
git, así que cuando clonas el repo heredas el mismo contexto, reglas y agentes que el resto del
equipo. Objetivo: que los 4 trabajemos con IA de forma consistente y sin pisarnos.

## Qué hay aquí

- `steering/` — **Contexto y reglas persistentes** que se cargan automáticamente en cada sesión:
  - `00-critical-rules.md` — reglas innegociables (incluida la de NO leer el generador de datos).
  - `10-project-context.md` — qué construimos, esquemas de datos y claves de unión.
  - `20-conventions.md` — estructura de carpetas, estilo, tests y Definition of Done.
  - `30-team-roles.md` — reparto de trabajo entre las 2 personas del equipo.
- `agents/` — **Agentes especializados** por módulo (consumer, processing, warehouse, qa, etc.).
  Cada uno arranca ya sabiendo las reglas y el contexto.
- `hooks/` — **Automatismos** compartidos (lint/tests al guardar, recordatorio de la regla del generador).
- `specs/` — **Especificaciones** (requisitos/diseño/tareas) que el equipo completa.
- `pipeline/` — Artefactos del pipeline de orquestación (prompt enriquecido, arquitectura, seguridad, reports).

## Cómo trabajar con esto (para cada miembro)

1. `git clone` del repo y `git checkout dev`.
2. Crea tu rama: `git checkout -b feature/<area>-<desc>` (ej. `feature/consumer-kafka`).
3. Abre el proyecto en Kiro. Los steering se cargan solos.
4. Trabaja tu módulo apoyándote en el agente correspondiente de `.kiro/agents/`.
5. Respeta la Definition of Done. PR hacia `dev`.

## Regla de oro
Nunca leas ni inspecciones el código del generador de datos. Solo se ejecuta. Ver `steering/00-critical-rules.md`.
