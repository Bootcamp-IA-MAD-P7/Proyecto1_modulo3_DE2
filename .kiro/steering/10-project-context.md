---
inclusion: auto
name: project-context
description: Contexto del proyecto HR Insights ETL — qué se construye, esquemas de datos que llegan (Personal, Location, Professional, Bank, Net), claves de unión/matching y stack acordado. Cargar cuando se trabaje en el pipeline, procesamiento, matching o decisiones de arquitectura de datos.
---

# Contexto del Proyecto — HR Insights ETL

## Qué construimos
Pipeline de ingeniería de datos que:
1. Consume mensajes en tiempo real desde un Kafka externo (miles/s).
2. Los guarda crudos en **MongoDB** (Data Lake / staging).
3. Identifica el tipo (Personal, Location, Professional, Bank, Net) y **une los fragmentos de la
   misma persona** en un único registro.
4. Limpia/normaliza (los datos son inconsistentes a propósito).
5. Persiste el registro consolidado en **PostgreSQL** (Data Warehouse).
6. (Avanzado) Redis como caché intermedia, Prometheus para métricas, API FastAPI de consulta.
7. (Experto) Carga continua + frontend Streamlit.

## Esquemas de datos que llegan (fragmentados, uno por mensaje)
- **Personal**: Name, Lastname, Sex, Telfnumber, Passport, E-Mail
- **Location**: Fullname, City, Address
- **Professional**: Fullname, Company, Company Address, Company Telfnumber, Company E-Mail, Job
- **Bank**: Passport, IBAN, Salary
- **Net**: Address, IPv4

## Claves de unión (el reto principal)
No hay un ID único global. Las uniones plausibles:
- Personal.Passport  <-> Bank.Passport
- Personal (Name+Lastname) -> "Fullname" en Location y Professional
- Location.Address <-> Net.Address
Los datos "may not be consistent": hay que normalizar (mayúsculas/espacios/acentos) y decidir una
estrategia de matching robusta. Documentar las decisiones de matching.

## Stack acordado
- Python 3.12, Docker + docker-compose.
- MongoDB (documental), PostgreSQL (relacional).
- Redis (caché), Prometheus (métricas), FastAPI (API), Streamlit (frontend).
- Kafka client: confluent-kafka (preferido por rendimiento) o kafka-python.

## Estrategia de entrega (incremental por niveles)
Arquitectura preparada para Experto desde el inicio, pero se cierra por hitos entregables:
Esencial -> Medio -> Avanzado -> Experto. Cada hito debe quedar funcional.
