---
inclusion: always
---

# Reglas Críticas del Proyecto (LEER SIEMPRE)

## 🚫 REGLA #1 — PROHIBIDO LEER EL GENERADOR DE DATOS
El servidor Kafka y el generador de datos se proporcionan como **CAJA NEGRA**.

- NUNCA abras, leas, inspecciones ni hagas reverse engineering del código del generador de datos.
- NUNCA leas archivos dentro del repositorio del generador salvo su README de instalación.
- El generador SOLO se ejecuta con `docker compose up --build`. Nada más.
- Violar esta regla puede suponer la descalificación del proyecto.
- Si un archivo o carpeta parece pertenecer al generador (contiene la lógica de creación de
  Personal/Location/Professional/Bank/Net data), NO lo leas y avisa al usuario.

## 🔒 REGLA #2 — Datos sensibles
El dataset contiene PII y datos financieros (Passport, IBAN, salario, email, teléfono, IPv4).
Aunque son sintéticos, se tratan como reales: sin secretos en el repo, credenciales por env vars.

## 🌿 REGLA #3 — Flujo de ramas
- `main`: solo código estable y entregable. Protegida.
- `dev`: integración. Las features se mergean aquí primero.
- `feature/<area>-<descripcion>`: ramas de trabajo. Ej: `feature/consumer-kafka`, `feature/processor-join`.
- Commits limpios y descriptivos (en español o inglés, consistentes). Formato sugerido: `tipo(area): descripcion`.

## 📋 REGLA #4 — Gestión con GitHub Issues + Projects
Todo trabajo se corresponde con un Issue. Las ramas referencian el número de issue cuando aplica.
