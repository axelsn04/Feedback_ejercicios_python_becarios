# Ejercicio 1 — Formatos Bajo la Lupa

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

Tienes un millón de registros de transacciones. Tu equipo los guarda en CSV porque siempre lo han hecho así. Tu trabajo es demostrar con mediciones reales si esa es la mejor decisión — o no.

---

## Lo que construirás

Una herramienta de benchmarking que genera datos, los almacena en múltiples formatos y compara su rendimiento. Al final tendrás evidencia concreta para recomendar un formato de almacenamiento.

---

## Paso 1 — Genera el dataset

Escribe un script `generate_data.py` que acepte `--size` como argumento (`100k`, `500k`, `1m`) y genere un CSV con exactamente este schema:

| Campo | Tipo | Valores |
|---|---|---|
| transaction_id | string (UUID4) | Único por fila |
| timestamp | datetime | Rango de 1 año hacia atrás desde hoy, distribución uniforme |
| user_id | entero | Entre 1 y 50,000 |
| merchant_id | entero | Entre 1 y 10,000 |
| amount | float | Entre 0.01 y 5,000.00 |
| category | string | 10 valores: Food, Travel, Electronics, Health, Entertainment, Retail, Transport, Education, Services, Other |
| country_code | string | 15 países: MX, CO, BR, AR, CL, PE, EC, VE, BO, PY, UY, CR, GT, PA, DO |
| status | string | completed (85%), failed (10%), pending (5%) |

Este schema es fijo. No lo modifiques. Lo usarás en los cuatro ejercicios del módulo.

---

## Paso 2 — Benchmark de formatos

Crea un módulo `storage_benchmark/` y un CLI `benchmark_cli.py` que reciba `--size` y `--formats` como argumentos. Para cada formato y escala, mide:

- Tiempo de escritura (`time.perf_counter()`, repite 3 veces y reporta el promedio)
- Tiempo de lectura completa del archivo
- Tiempo de lectura selectiva: solo las columnas `amount` y `category`
- Tamaño del archivo en disco (bytes)
- Pico de memoria RAM durante la lectura (`tracemalloc`)

Formatos que debes cubrir: CSV, JSON Lines (JSONL), Parquet sin compresión, Parquet con Snappy, Parquet con Gzip.

> El tiempo de generación del dataset NO cuenta como tiempo de escritura. Genera los datos primero, guarda en memoria, y empieza a medir solo cuando escribes al disco.

---

## Paso 3 — Reporte

Escribe un `report.md` con:

- Una tabla comparativa por escala con todos los formatos y métricas
- Gráficas de barras (matplotlib) para tiempo de lectura y tamaño en disco
- Una sección de conclusiones de mínimo 400 palabras donde expliques **POR QUÉ** ocurren las diferencias que observaste — no solo qué formato ganó
- Una recomendación final: qué formato usarías en producción para este caso de uso y por qué

---

## Entregables

```
ejercicio-01-formatos/
├── generate_data.py
├── benchmark_cli.py
├── storage_benchmark/
├── results/
│   ├── benchmark_100k.json
│   ├── benchmark_500k.json
│   └── benchmark_1m.json
└── report.md
```

El CLI debe ejecutarse así:

```bash
python benchmark_cli.py --size 1m --formats csv jsonl parquet parquet_snappy parquet_gzip
```

---

## Cómo se evaluará

| Criterio | Peso |
|---|---|
| CLI funcional y módulo organizado | 20% |
| Rigor del benchmark | 25% |
| Análisis por escala | 25% |
| Reporte y conclusiones | 30% |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 1 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```