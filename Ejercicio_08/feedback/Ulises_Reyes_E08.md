# Retroalimentación — Ejercicio 08: El Proyecto Final

**Alumno:** Ulises Josue Reyes Martinez  
**Fecha de revisión:** Mayo 2026  
**Calificación:** 99 / 100

---

## Resumen general

La entrega más sólida del módulo en E08, y probablemente la más sólida del módulo completo. La decisión arquitectónica central — "SQLite es la fuente de verdad viva, DuckDB la consulta con `sqlite_scanner`" — no es una preferencia sino la consecuencia directa de un test que falló y obligó a razonar sobre el problema real. El `decisions.md` es el único del grupo que documenta ese proceso de forma honesta: diseñé X, encontré Y, cambié a Z por esta razón concreta. El `conftest.py` construye un dataset determinista con anomalías conocidas (user 7 con 8 fallidas, user 42 con 6, user 99 con 2), lo que convierte los tests de anomalías de "responde 200" a "detecta exactamente a quien debe detectar". Las invariantes matemáticas en el pipeline como `assert` son verificación en tiempo de ejecución, no en tests. El Dockerfile tiene `USER appuser` — el único del módulo que cierra ese gap. La imagen mide 335MB: supera el límite de 300MB del spec, pero el `decisions.md` lo documenta con el número exacto, el desglose por componente, y el razonamiento concreto para aceptar el trade-off.

---

## Criterio 1 — Funcionalidad completa `40 / 40`

**Los 9 endpoints implementados + pipeline + Docker:**

| Endpoint | Estado | Nota |
|---|---|---|
| GET `/health` | ✓ | `transactions_in_db`, `duckdb_connected`, `sqlite_connected`, `cache_hit_rate`, `uptime_seconds` |
| GET `/analytics/summary` | ✓ | Vista unificada vía DuckDB+sqlite_scanner, cache con TTL configurable |
| GET `/analytics/top-merchants` | ✓ | `limit` + `country` (normalizado a upper), cache por clave compuesta |
| GET `/analytics/anomalies` | ✓ | `threshold` + `window_days` parametrizables, umbral desde env var |
| GET `/users/{id}/transactions` | ✓ | `date_from`, `date_to`, paginación con `total_pages`, 404 limpio |
| GET `/users/{id}/stats` | ✓ | `transaction_count`, `total_amount`, `most_frequent_category`, `country_code` |
| POST `/transactions/batch` | ✓ | Hasta 500, `inserted`, `duplicates_skipped`, validación Pydantic |
| POST `/pipeline/ingest` | ✓ | Upload CSV, pipeline E→T→L, reporte con invariantes verificadas en respuesta |
| Pipeline ETL | ✓ | Capas del E6 reutilizadas intactas + `csv_source.py` nuevo |
| Docker | ✓ | Multi-stage, `USER appuser`, setup idempotente, `sqlite_scanner` preinstalada offline |

**La decisión arquitectónica central — documentada con el proceso que la produjo:**

El `decisions.md` describe cómo el primer diseño (UNION ALL entre Parquet y SQLite) fallaba porque las consultas por usuario tendrían que escanear el Parquet de 1M filas, inutilizando el índice `idx_user_timestamp` del E3. El cambio al "Modelo A" (SQLite como fuente viva, DuckDB con `sqlite_scanner`) fue forzado por ese test, no elegido por preferencia. Eso es exactamente el nivel de razonamiento que el spec pedía: una decisión con consecuencia medible.

**`setup.py` es genuinamente idempotente:**
```python
if os.path.exists(db_path):
    print(f"setup: {db_path} ya existe -- nada que hacer (idempotente).")
    return 0
```
Si la base ya existe, el servicio termina con código 0 sin tocar nada. `docker compose restart setup` no destruye los datos. Angel hizo lo contrario — borrar y recrear — que es el anti-patrón. Aquí está bien resuelto.

**Invariantes matemáticas en el reporte del pipeline:**
```python
assert n_csv == len(extracted) + len(parse_errors)
assert len(extracted) == len(valid) + len(rejected)
assert inserted + duplicates == len(valid)

return {
    ...
    "invariants": {
        "csv_eq_extracted_plus_parse_errors": n_csv == ...,
        "extracted_eq_valid_plus_rejected": len(extracted) == ...,
        "inserted_plus_duplicates_eq_valid": inserted + duplicates == len(valid),
    },
}
```
Las invariantes son `assert` en tiempo de ejecución Y están en la respuesta JSON para que el cliente las verifique. Si alguna se rompe, el pipeline falla con traza clara en lugar de devolver un reporte silenciosamente incorrecto. El test `test_ingest_csv_invariants` verifica que `all(r.json()["invariants"].values())`.

**Tres niveles de error en `csv_source.py` — más fino que el E6:**
El E6 distinguía errores de formato (extract) vs errores de negocio (transform). El E8 agrega un tercer nivel: errores de estructura del archivo (columna faltante, archivo vacío, no parseable). `csv_source.py` valida la estructura antes de pasar una sola fila a extract. El test `test_ingest_csv_missing_column` verifica que si falta `amount`, el error dice exactamente `"amount"`.

---

## Criterio 2 — Calidad técnica `34 / 35`

**26 tests con dataset determinista — el enfoque correcto:**

```python
# conftest.py — anomalías conocidas, resultados exactos
for i in range(8):  # user 7: 8 fallidas recientes
    recientes.append(("recent-f7-...", ..., 7, ..., "failed"))
for i in range(6):  # user 42: 6 fallidas recientes
    recientes.append(("recent-f42-...", ..., 42, ..., "failed"))
for i in range(2):  # user 99: 2 fallidas (NO anómalo con N=5)
    recientes.append(("recent-f99-...", ..., 99, ..., "failed"))
```

Y los tests que lo explotan:
```python
def test_anomalies_threshold_5(client):
    flagged = {u["user_id"] for u in body["anomalous_users"]}
    assert 7 in flagged
    assert 42 in flagged
    assert 99 not in flagged
    assert body["total_flagged"] == 2

def test_anomalies_threshold_7(client):
    flagged = {u["user_id"] for u in r.json()["anomalous_users"]}
    assert flagged == {7}
```

No es "el endpoint responde 200 y tiene la forma correcta". Es "con estos datos exactos, el sistema detecta exactamente a estos usuarios y no a este otro". Si alguien cambia `>` por `>=` en el HAVING, `test_anomalies_threshold_7` falla porque user 42 (6 fallidas) aparecería con threshold=7. Eso es un test que protege la lógica de negocio.

**`conftest.py` con puente pandas para analytics — honestidad técnica documentada:**
```python
# Para los endpoints de analytics, que en producción usan DuckDB con la
# extensión sqlite_scanner, los tests usan un puente pandas: registran la
# tabla SQLite como una vista DuckDB. Es equivalente funcional al ATTACH
# y permite correr la suite en cualquier entorno sin descargar la extensión.
conn.register("txn_table_global", txn_df)
```
La diferencia entre el entorno de test y producción está documentada en el `conftest.py` *y* en el `decisions.md`. El test no miente — dice exactamente qué está probando y qué no.

**`extract.py` con separación de responsabilidades precisa:**
```python
# Un amount=-50.0 pasa por esta capa sin problema — es un float perfectamente
# formateado. transform.py lo rechazará porque viola la regla de negocio.
# Si extract.py rechazara amounts negativos, estaría mezclando responsabilidades.
```
La docstring no solo explica qué hace — explica qué *no* hace y por qué esa distinción importa. Esta precisión conceptual se refleja en el código: `extract` tiene `parse_errors` para errores de formato irrecuperables (timestamp no parseable, amount no convertible a float), y `transform` tiene `rejected` para violaciones de negocio.

**Dockerfile con `USER appuser` — único del módulo:**
```dockerfile
RUN useradd --create-home --shell /bin/bash appuser
...
COPY --from=build --chown=appuser:appuser /root/.duckdb /home/appuser/.duckdb
USER appuser
```
El `--chown=appuser:appuser` en el `COPY` de `.duckdb` es la corrección explícita del problema de capa duplicada que el propio `decisions.md` documenta: "primera versión copiaba `/root/.duckdb` y luego hacía `chown` en un `RUN` separado, generando dos capas de 34.7MB cada una". La solución usa `COPY --chown` en una sola instrucción. Ernesto, Angel y Antonio nunca llegaron a este nivel de análisis del Dockerfile.

**Imagen de 335MB — único en el módulo que documenta el exceso con honestidad:**
El spec pide <300MB. La imagen pesa 335MB. El `decisions.md` documenta:
- El desglose exacto: base 155MB, venv+duckdb 75.3MB, `sqlite_scanner` 34.7MB, código ~185KB
- Por qué el 35MB extra: imagen offline vs descarga en cada arranque desde cero
- El número real en lugar de afirmar "<300MB" sin matices

Ese nivel de honestidad técnica es más valioso que haber llegado a 299MB ocultando el trade-off. Un punto menos por el exceso técnico sobre el límite del spec; cero puntos menos por cómo está documentado.

**`window_days` en anomalías — parámetro adicional vs spec:**
```python
def test_anomalies_invalid_window(client):
    assert client.get("/analytics/anomalies?window_days=0").status_code == 422
    assert client.get("/analytics/anomalies?window_days=999").status_code == 422
```
El spec pedía `threshold` parametrizable y 30 días fijo. Ulises agrega `window_days` con validación `ge=1, le=365`, igual que Antonio. Ambos llegan a la misma extensión de forma independiente — señal de que es la extensión obvia.

---

## Criterio 3 — decisions.md `25 / 25`

**El mejor `decisions.md` del módulo — cada afirmación tiene número de ejercicio, cifra concreta, y consecuencia operativa.**

| Sección | Evidencia concreta |
|---|---|
| FastAPI vs Django | Comparación operativa: async + `asyncio.Lock` para CSV, DRF más torpe para modelo de datos dual |
| DuckDB+sqlite_scanner | "el mismo número que el E4 y el E5 con el mismo Parquet" — verificación cruzada de consistencia |
| SQLite índices | "idx_user_timestamp del E3 cubre exactamente este patrón" — el índice justifica la decisión de capa |
| UNION ALL → Modelo A | Describe el test que falló, el problema que reveló, y por qué el cambio fue forzado por el requisito |
| Cache con invalidación | "TTL de 300 segundos significa fresco cada 5 minutos, no tiempo real" — diferencia entre el claim y la realidad |
| 100M filas | PostgreSQL con MVCC, DuckDB sobre Parquet particionado, cola de trabajos para ingesta, setup incremental |
| Monitoreo | p99 por endpoint, tasa de rechazo del pipeline, hit rate del cache como señal de TTL mal configurado |
| Docker — 335MB | Desglose exacto por componente, razonamiento para aceptar el exceso, cómo se midió |
| Validación con datos reales | Tabla de resultados con números del dataset de 1M — `total_amount: 2,500,147,886.54`, mismo que E4 y E5 |

**La sección de invalidación de cache — la única del módulo que identifica la tensión entre "tiempo real" y TTL:**
> "El enunciado pide datos en tiempo real, pero el cache del E4 con TTL de 300 segundos significa 'fresco cada cinco minutos', que no es lo mismo."

Esa frase sola vale más que tres párrafos de justificación técnica. Reconocer la tensión entre el requisito del negocio y la implementación concreta, y documentar cómo se resuelve (invalidación dirigida tras escritura), es exactamente lo que separa un sistema explicable de una caja negra.

**La sección de monitoreo responde la pregunta del spec:**
> "si el conteo de transacciones deja de crecer al ritmo esperado, el pipeline de ingesta probablemente está fallando aguas arriba"

Eso es proactividad: detectar la tendencia antes del umbral de falla. No esperar a que un usuario reporte que los datos están desactualizados — monitorear la señal que precede al problema.

**Validación cruzada con datos reales:**
La tabla de resultados con Docker y el dataset de 1M verifica que `analytics/summary` devuelve `total_transactions: 1,000,002` después de ingestar un CSV de 2 filas válidas. Eso confirma que la invalidación de cache funciona, que el pipeline se integra correctamente con el histórico, y que DuckDB+sqlite_scanner suma correctamente las 1M filas del histórico más las 2 nuevas. Ningún otro alumno verificó esto.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Funcionalidad completa | 40% | 40 / 40 |
| Calidad técnica | 35% | 34 / 35 |
| decisions.md | 25% | 25 / 25 |
| **Total** | **100%** | **99 / 100** |

---

El punto restante es técnico: la imagen supera el límite de 300MB del spec. La forma en que está documentado ese exceso — con desglose exacto, razonamiento concreto y el número real — convierte un gap técnico en una decisión informada. Si el criterio fuera "imagen <300MB o 0 puntos", la nota sería diferente. El criterio es "imagen funcional y liviana": funcional cumple, liviana es relativa dado el componente `sqlite_scanner` que el diseño requiere.

El `conftest.py` con dataset determinista y anomalías conocidas, las invariantes como `assert` en tiempo de ejecución más como campos en la respuesta JSON, la separación precisa entre errores de formato y errores de negocio en el pipeline, y el `decisions.md` que documenta el proceso de cambiar de arquitectura a mitad del desarrollo — eso es el nivel correcto para un sistema de datos en producción.