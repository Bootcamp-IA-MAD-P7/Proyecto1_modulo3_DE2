---
inclusion: always
---

# Reglas Críticas del Proyecto (LEER SIEMPRE)

## ✅ REGLA #1 — Generador de datos (fase de limpieza CERRADA)
La fase de detección/limpieza/normalización de datos está terminada. El generador de
datos ya puede incluirse en el repo y desplegarse si hace falta para la demo/despliegue.

- La lógica de limpieza/matching NO debe basarse en leer el generador: se diseñó de forma
  independiente y así se mantiene. No reintroduzcas conocimiento del generador en el ETL.
- Está permitido incluir el generador en el repo y desplegarlo. Antes de redistribuir su
  código, verifica que su licencia lo permite (si no trae licencia explícita, consúltalo).
- Sigue tratando su salida (los mensajes Kafka) como el contrato de entrada del pipeline.

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
