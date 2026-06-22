# Ejercicio 6 — El Pipeline de Datos

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

Hasta ahora el dataset de 1M transacciones era estático. En producción los datos llegan continuamente de múltiples fuentes con calidad variable. Tu trabajo es construir el pipeline que los ingesta, valida y transforma de forma reproducible.

---

## Lo que construirás

Un pipeline ETL (Extract, Transform, Load) que lee transacciones nuevas desde una fuente simulada, aplica validaciones y transformaciones, y las carga en la base SQLite del E3. El pipeline debe ser idempotente — correrlo dos veces con los mismos datos debe dar el mismo resultado que correrlo una vez.

---

## Prerequisitos

- Base SQLite con índices generada en el Ejercicio 3

Si no la tienes disponible, solicítala en el canal **#becarios** antes de empezar.

---

## Paso 1 — Fuente de datos

Escribe `data_source.py` que simule la llegada de transacciones nuevas. El script genera batches de entre 100 y 1,000 transacciones con el schema del módulo, pero introduce errores deliberados: montos negativos, categorías inválidas, timestamps futuros, campos nulos.

Acepta `--batch-size` y `--error-rate` como argumentos. Agrega un flag `--seed` para reproducibilidad.

---

## Paso 2 — Capa de extracción

Escribe `extract.py` que lea desde la fuente y normalice los datos:

- Timestamps a ISO 8601
- `country_code` a mayúsculas
- `amount` redondeado a 2 decimales
- Strip de whitespace en campos de texto

Esta capa **no valida reglas de negocio** — solo normaliza tipos y formatos. La decisión de rechazar una fila pertenece exclusivamente a `transform.py`.

---

## Paso 3 — Capa de transformación y validación

Escribe `transform.py` con las siguientes reglas de negocio:

| Regla | Acción |
|---|---|
| `amount` fuera del rango 0.01–5,000.00 | Rechazar |
| `category` fuera de los 10 valores del schema | Rechazar |
| `country_code` fuera de los 15 países | Rechazar |
| `timestamp` con más de 1 hora de adelanto | Rechazar |
| `transaction_id` no es UUID4 válido | Rechazar |
| `status` distinto de `completed`, `failed` o `pending` | Rechazar |
| `user_id` fuera del rango 1–50,000 | Rechazar |
| `merchant_id` fuera del rango 1–10,000 | Rechazar |

Las filas rechazadas **no se pierden** — van a `quarantine/YYYY-MM-DD.jsonl` con el motivo del rechazo. Las filas válidas pasan a la capa de carga.

---

## Paso 4 — Capa de carga

Escribe `load.py` que inserte las filas válidas en SQLite con `INSERT OR IGNORE` por `transaction_id`. La carga debe ser transaccional — si falla a mitad, no queda nada a medias.

Reporta `inserted` y `duplicates` derivándolos de `COUNT(*)` antes y después de la operación, no inspeccionando fila por fila.

---

## Paso 5 — Orquestador y métricas

Escribe `pipeline.py` que encadena `extract → transform → load` y genera un reporte de ejecución por corrida en `results/run_YYYYMMDD_HHMMSS.json` con:

```json
{
  "run_id": "run_20250501_143022",
  "timestamp": "2025-05-01T14:30:22Z",
  "duration_seconds": 1.84,
  "rows_extracted": 500,
  "rows_valid": 432,
  "rows_rejected": 68,
  "rejected_by_type": {
    "invalid_amount": 12,
    "invalid_category": 17,
    "invalid_country_code": 9,
    "future_timestamp": 16,
    "invalid_uuid": 14
  },
  "rows_inserted": 398,
  "rows_duplicate": 34
}
```

Las dos invariantes deben verificarse con `assert` en el código, no solo reportarse:

- `rows_valid + rows_rejected == rows_extracted`
- `rows_inserted + rows_duplicate == rows_valid`

---

## Sobre idempotencia

Correr el pipeline dos veces con los mismos datos debe producir el mismo estado final en la base. Tus tests deben demostrar esto directamente — insertar la misma fila dos veces y verificar `COUNT(*) == 1` en la base, no solo que el segundo insert reporte 0.

> Nota: con `--seed 42`, `data_source.py` reproduce la misma distribución estadística pero **no los mismos `transaction_id`**, porque `uuid.uuid4()` usa el generador del SO y no respeta `random.seed()`. Eso significa que correr el CLI dos veces con el mismo seed no demuestra idempotencia end-to-end. Solo los tests la demuestran, porque generan la fila explícitamente con un UUID fijo. Reconoce esa diferencia en tu reporte.

---

## Entregables

```
ejercicio-06-pipelines/
├── data_source.py
├── extract.py
├── transform.py
├── load.py
├── pipeline.py
├── quarantine/          ← en .gitignore
├── results/             ← en .gitignore
├── tests/
│   └── test_pipeline.py
├── requirements.txt
└── README.md
```

El README debe incluir el comando exacto para correr el pipeline completo y una descripción de la estructura del reporte JSON generado.

---

## Cómo se evaluará

| Criterio | Peso | Lo que revisamos |
|---|---|---|
| Capas separadas y cohesivas | 25% | Cada archivo tiene una responsabilidad clara — sin lógica mezclada entre capas |
| Validación y cuarentena | 30% | Todos los tipos de error son detectados y llegan al quarantine con motivo |
| Idempotencia verificada | 20% | Correr el pipeline dos veces con los mismos datos produce el mismo resultado final |
| Reporte de ejecución | 25% | JSON con todas las métricas por corrida — verificamos que los números cuadran |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 6 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```