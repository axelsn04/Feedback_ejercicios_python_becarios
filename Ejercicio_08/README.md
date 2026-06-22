# Ejercicio 8 — El Proyecto Final

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 8-10 horas

---

## El problema

Este ejercicio no tiene pasos predefinidos. Tienes todos los componentes del módulo — storage, query engines, capa transaccional, API, pipeline de datos, contenedores. Tu trabajo es integrarlos en un sistema cohesivo que resuelva un problema real y que puedas explicar en una entrevista.

---

## El contexto

Eres el equipo de datos de una fintech LATAM. El equipo de producto necesita un sistema de monitoreo de transacciones con estas capacidades:

- Ver el volumen de transacciones en tiempo real por país y categoría
- Consultar el historial de un usuario específico con filtros de fecha
- Detectar usuarios con patrones anómalos (más de N transacciones fallidas en los últimos 30 días, N parametrizable)
- Ingestar transacciones nuevas desde un archivo CSV externo
- Ver el estado del sistema y métricas de rendimiento en `/health`

---

## Lo que debes construir

Un sistema completo que integre los mejores elementos de E04 a E07. No es necesario usar todo lo que construiste — elige las piezas que tienen sentido para este caso de uso y justifica tus decisiones.

### Componentes obligatorios

**1. API REST** con al menos 6 endpoints que cumplan los SLAs del E04:

| Método | Ruta | SLA |
|---|---|---|
| GET | `/health` | <50ms siempre |
| GET | `/analytics/summary` | <500ms frío / <20ms caliente |
| GET | `/analytics/top-merchants` | <500ms frío / <20ms caliente |
| GET | `/users/{user_id}/transactions` | <80ms |
| GET | `/users/{user_id}/stats` | <80ms |
| POST | `/transactions/batch` | <2s para 500 registros |
| GET | `/anomalies` | <500ms |

**2. Endpoint de detección de anomalías** — dado un umbral N como parámetro, retorna los `user_id` con más de N transacciones con `status: failed` en los últimos 30 días.

**3. Pipeline de ingesta** — acepta un CSV externo, valida el schema con las mismas reglas del E06, carga las filas válidas y retorna un reporte de errores. El pipeline puede estar expuesto como endpoint (`POST /pipeline/ingest`) o como script CLI.

**4. Configuración Docker** — un solo `docker compose up --build` levanta el sistema completo desde cero en una máquina limpia.

**5. Suite de tests** — cobertura de los casos críticos del negocio: SLAs, detección de anomalías, rechazo de schema inválido, deduplicación.

---

## El documento de decisiones

Escribe un `decisions.md` de **mínimo 600 palabras** que responda:

- ¿Qué tecnologías elegiste para cada capa y por qué — con evidencia de los ejercicios anteriores?
- ¿Qué compromisos (trade-offs) tomaste y cuáles son sus consecuencias?
- ¿Qué cambiarías si el dataset fuera de 100M filas en lugar de 1M?
- ¿Qué monitorearías en producción y cómo detectarías un problema antes de que lo reporte un usuario?

> Este documento es la parte más importante del ejercicio. Un sistema que funciona pero no puede ser explicado no es un activo — es una caja negra.

---

## Entregables

```
ejercicio-08-final/
├── app/                 ← o proyecto Django
│   ├── main.py
│   ├── db.py
│   ├── cache.py
│   └── models.py
├── pipeline/
│   └── ingest.py
├── tests/
│   └── test_system.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── decisions.md         ← mínimo 600 palabras
└── README.md
```

El README debe incluir el comando para levantar el sistema, los endpoints disponibles con ejemplos de `curl`, y el comando para correr los tests.

---

## Cómo se evaluará

| Criterio | Peso | Lo que revisamos |
|---|---|---|
| Funcionalidad completa | 40% | 6 endpoints, pipeline, anomalías y Docker — todo funciona desde cero en una máquina limpia |
| Calidad técnica | 35% | SLAs cumplidos, tests que pasan, código organizado, sin configuración hardcodeada |
| decisions.md | 25% | Mínimo 600 palabras, trade-offs justificados con evidencia, respuestas concretas sobre escalabilidad y observabilidad |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 8 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```