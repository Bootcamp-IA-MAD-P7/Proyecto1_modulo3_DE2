# Security Review — HR Insights ETL

## 1. Nivel de riesgo

**Riesgo: ALTO (a nivel de datos) / MEDIO (a nivel de superficie de ataque).**

Aunque los datos son sintéticos y el sistema corre local/dockerizado en un contexto pedagógico, el
dataset modela PII y datos financieros reales (Passport, IBAN, salario, email, teléfono, dirección,
IPv4). Se aplican buenas prácticas como si fueran datos reales — esto es parte del valor formativo y
evita malos hábitos.

## 2. Threat model

### Activos a proteger
- PII: nombre, sexo, teléfono, pasaporte, email, dirección.
- Datos financieros: IBAN, salario.
- Datos de red: IPv4.
- Credenciales de infraestructura: usuarios/contraseñas de Mongo, Postgres, Redis.
- Integridad del pipeline (que no se corrompa/duplique el dato consolidado).

### Actores de amenaza
- Atacante en la red local que alcanza puertos de BD expuestos por Docker.
- Filtración accidental de secretos vía commit al repo público de GitHub.
- Entrada malformada/maliciosa desde el stream (inyección, payloads que rompen el parser).
- Dependencia comprometida (supply chain / typosquatting).

### Vectores de ataque relevantes
- Puertos de Mongo/Postgres/Redis publicados al host sin necesidad.
- Secretos hardcodeados o `.env` commiteado.
- Inyección SQL si se construyen queries por concatenación.
- Redis sin contraseña accesible en red.
- Inyección de logs / exposición de PII en logs.
- Exposición de PII sin filtrar a través de la API.

## 3. Requisitos de seguridad

### Autenticación / Autorización
- BD con usuarios dedicados y contraseña (no usuarios por defecto sin credenciales).
- Redis con `requirepass`.
- API: de solo lectura; para el alcance no requiere auth de usuario, pero no debe permitir
  operaciones de escritura ni exponer endpoints administrativos.

### Protección de datos
- Credenciales exclusivamente por variables de entorno; `.env` en `.gitignore`; `.env.example` sin valores reales.
- No exponer puertos de BD al host salvo los estrictamente necesarios para la demo; comunicación
  entre servicios por la red interna de Docker.
- No registrar en logs valores sensibles completos (IBAN, pasaporte): enmascarar si se loguean.
- La API no devuelve más PII de la necesaria y valida/pagina las consultas.

### Validación de entrada
- Todo mensaje del stream se valida/parsea con pydantic antes de procesar; los inválidos van a un
  camino de error controlado (log + métrica), nunca tumban el proceso.
- Parámetros de la API validados con pydantic (tipos, longitudes, paginación con límites).

## 4. Guía de código seguro (DO / DO NOT)

**DO**
- DO usar SQLAlchemy con sentencias parametrizadas / ORM (nunca f-strings con datos en SQL).
- DO cargar toda credencial desde `config.py` (env vars).
- DO validar entrada con pydantic en el borde (consumer y API).
- DO fijar versiones de dependencias en `pyproject.toml` (pins), usar paquetes conocidos.
- DO enmascarar PII sensible en logs (mostrar solo últimos 4 dígitos de IBAN, etc.).
- DO usar usuarios de BD con privilegios mínimos.

**DO NOT**
- DO NOT hardcodear contraseñas, IBANs, ni ningún secreto en el código o en commits.
- DO NOT commitear `.env`, dumps de datos ni credenciales.
- DO NOT construir SQL por concatenación de strings.
- DO NOT publicar puertos de Mongo/Postgres/Redis al host si no son necesarios para la demo.
- DO NOT loguear el payload completo con PII en nivel INFO.
- DO NOT, bajo ningún concepto, leer el código del generador de datos (regla del proyecto).

## 5. OWASP Top 10 (relevancia)

| Riesgo OWASP | Relevancia | Mitigación |
|--------------|-----------|-----------|
| A01 Broken Access Control | Media | API solo lectura; sin endpoints admin; puertos internos |
| A02 Cryptographic Failures | Media | Secretos fuera del repo; enmascarado de PII en logs |
| A03 Injection | Alta | ORM parametrizado; validación pydantic de la entrada |
| A04 Insecure Design | Media | Lake/Redis desacoplan; manejo de datos malos por diseño |
| A05 Security Misconfiguration | Alta | No exponer puertos; usuarios BD dedicados; Redis con pass |
| A06 Vulnerable Components | Media | Versiones fijadas; dependencias conocidas |
| A07 Auth Failures | Baja | Sin usuarios finales; credenciales de servicio robustas |
| A08 Integrity Failures | Media | Pins de dependencias; upserts idempotentes |
| A09 Logging/Monitoring Failures | Media | Logs estructurados + métricas Prometheus, sin PII sensible |
| A10 SSRF | Baja | Sin fetch de URLs externas controladas por el dato |

## 6. Requisitos de testing de seguridad para QA

- Test: parámetros de la API no permiten inyección (consultas parametrizadas).
- Test: mensajes malformados del stream no tumban el pipeline y se contabilizan como error.
- Test: la configuración lee de env vars y falla de forma controlada si faltan las críticas.
- Test: los logs no contienen IBAN/pasaporte completos (enmascarado).
- Revisión: `.gitignore` cubre `.env`; no hay secretos en el árbol de git.
