# Ejercicio 4 — El Sistema Completo

**Módulo:** Python para Sistemas de Datos Modernos  
**Duración estimada:** 6-8 horas

---

## El problema

Tienes dos bases de datos del módulo: el Parquet de 1M transacciones (E1) y la base SQLite con índices (E3). El objetivo es construir una API REST que sirva ambas de forma correcta — cada endpoint conectado al motor que corresponde según el patrón de acceso — con cache para los endpoints analíticos y validación de schema en las escrituras.

---

## Lo que construirás

Una API FastAPI con arquitectura dual DuckDB + SQLite, cache en memoria con TTL, validación Pydantic, tests automatizados y un benchmark de latencia que demuestre que cada endpoint cumple su SLA.

---

## Prerequisitos

- Parquet de 1M filas generado en el Ejercicio 1
- Base SQLite con índices generada en el Ejercicio 3

Si no los tienes disponibles, solicítalos en el canal **#becarios** antes de empezar.

---

## Los 6 endpoints que debes implementar

| Método | Ruta | Backend | SLA |
|---|---|---|---|
| GET | `/analytics/summary` | DuckDB | <500ms frío / <20ms caliente |
| GET | `/analytics/top-merchants` | DuckDB | <500ms frío / <20ms caliente |
| GET | `/users/{user_id}/transactions` | SQLite | <80ms |
| GET | `/users/{user_id}/stats` | SQLite | <80ms |
| POST | `/transactions/batch` | SQLite | <2s para 500 registros |
| GET | `/health` | Memoria | <50ms siempre |

---

## Especificaciones por endpoint

### GET /analytics/summary
Retorna totales globales del dataset: conteo total de transacciones, monto total, promedio, y breakdown por país y por categoría. Los datos cambian solo con un nuevo pipeline de ingesta — no con cada POST al batch.

### GET /analytics/top-merchants
Retorna los N merchants con mayor volumen de transacciones. Parámetros opcionales: `limit` (default 10, máximo 100) y `country` (código ISO de 2 letras para filtrar por país).

### GET /users/{user_id}/transactions
Retorna las transacciones de un usuario ordenadas de más reciente a más antigua, con paginación. Parámetros: `page` (default 1) y `page_size` (default 20, máximo 100). Retorna 404 si el usuario no existe.

### GET /users/{user_id}/stats
Retorna el total de transacciones, monto acumulado, categoría más frecuente y país del usuario. Retorna 404 si el usuario no existe.

### POST /transactions/batch
Recibe hasta 500 transacciones, valida el schema, deduplica por `transaction_id` e inserta las nuevas en SQLite. Si cualquier campo es inválido retorna 422 con el detalle antes de tocar la base.

Schema de cada transacción:
```json
{
  "transaction_id": "string",
  "timestamp": "ISO8601",
  "user_id": 1,
  "merchant_id": 1,
  "amount": 99.99,
  "category": "Food",
  "country_code": "MX",
  "status": "completed"
}
```

### GET /health
Retorna el estado del sistema: uptime, estado de las conexiones a DuckDB y SQLite, y estadísticas del cache (hits, misses, hit rate). No debe tocar las bases de datos — solo leer estado en memoria.

---

## Requerimientos técnicos obligatorios

### Conexiones en el lifespan
Las conexiones a DuckDB y SQLite se abren **una sola vez** al arrancar el servidor, en el `lifespan` de FastAPI, y se reutilizan en todos los requests. Abrir una conexión dentro de un endpoint es un error de arquitectura — el benchmark de latencia lo delataría inmediatamente.

### Cache con TTL
Los endpoints `/analytics/*` deben usar cache en memoria con TTL configurable. El cache debe:
- Tener TTL diferente por endpoint (los datos de summary y top-merchants pueden tener distinta frecuencia de actualización)
- Exponer métricas de hit rate en `/health`
- Invalidar entradas cuando corresponda

### Validación Pydantic
El `POST /transactions/batch` debe rechazar automáticamente con 422 cualquier transacción con:
- `amount` fuera del rango 0.01-5000.00
- `category` fuera del dominio definido en E1
- `country_code` fuera de los 15 países del dataset
- `status` distinto de `completed`, `failed` o `pending`
- Campos faltantes o con tipos incorrectos

### Deduplicación en el batch
Si un `transaction_id` ya existe en la base o aparece más de una vez en el mismo batch, debe ser ignorado silenciosamente. El response debe reportar cuántas transacciones fueron insertadas y cuántas fueron descartadas como duplicados.

---

## Tu tarea: justificar las decisiones

Escribe un `architecture_decision.md` que responda:
- ¿Por qué cada endpoint usa el backend que elegiste?
- ¿Por qué cacheas los endpoints analíticos y no los de usuario?
- ¿Qué TTL elegiste y por qué?
- ¿Cómo manejas la concurrencia en las conexiones?

> Estas decisiones deben estar respaldadas por los datos de E2 y E3. Si en E3 mediste que DuckDB tarda 88ms en P1 vs 0.033ms de SQLite, úsalo como evidencia.

---

## Tests automatizados (mínimo 8)

Escribe una suite de tests con pytest que cubra como mínimo:

- Happy path de cada uno de los 6 endpoints
- Usuario inexistente → 404
- Batch con schema inválido → 422
- Paginación fuera de rango
- Deduplicación en el batch (primer insert = 1, segundo = 0)
- Verificación de SLA: al menos un endpoint debe tener un test que valide la latencia

---

## Benchmark de latencia

Con el servidor corriendo, ejecuta al menos 100 requests por endpoint. Para los analíticos mide dos regímenes:
- **Cold** — cache vacío, la query llega a DuckDB
- **Warm** — cache caliente, la respuesta viene de memoria

Reporta p50, p95 y p99 por endpoint. Demuestra el impacto del cache con el ratio cold/warm.

---

## Entregables

```
ejercicio-04-sistema/
├── app/
│   ├── main.py              ← FastAPI: lifespan + 6 endpoints
│   ├── db.py                ← conexiones DuckDB y SQLite
│   ├── cache.py             ← TTLCache con métricas
│   └── models.py            ← modelos Pydantic
├── tests/
│   └── test_api.py          ← mínimo 8 tests
├── benchmarks/
│   └── latency_report.md    ← p50/p95/p99 cold vs warm
├── architecture_decision.md
└── README.md                ← instrucciones para arrancar el servidor
```

---

## Cómo se evaluará

| Criterio | Peso |
|---|---|
| 6 endpoints funcionales con datos reales | 25% |
| Architecture decision justificado | 20% |
| Cache con TTL y medición de impacto | 20% |
| Tests con pytest | 20% |
| Benchmark de latencia con análisis | 15% |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 4 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```