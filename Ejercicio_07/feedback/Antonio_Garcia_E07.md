# Retroalimentación — Ejercicio 07: De tu Máquina al Mundo

**Alumno:** Antonio Garcia
**Fecha de revisión:** Mayo 2026
**Calificación:** 96 / 100

---

## Resumen general

El Dockerfile más optimizado del grupo. Tres decisiones lo distinguen del
resto: `strip --strip-unneeded` sobre los archivos `.so` del venv (ningún otro
alumno hizo esto), la exclusión explícita de pandas y pyarrow del
`requirements-e4.txt` con el razonamiento correcto ("DuckDB reads Parquet
natively, so pyarrow/pandas are not needed"), y las variables `CACHE_TTL_*`
conectadas correctamente desde `.env` → compose `environment` → contenedor,
con defaults inline. Los únicos gaps: no hay usuario non-root y `setup_db.py`
no fue entregado para verificación directa.

---

## Criterio 1 — Imagen funcional y liviana `24 / 25`

**Optimización `strip --strip-unneeded` sobre `.so` — único en el grupo:**
```dockerfile
find /venv -name "*.so" -exec strip --strip-unneeded {} + && \
```
El binario de DuckDB es un `.so` de 58MB sin strip. `strip --strip-unneeded`
elimina los símbolos de debug sin afectar la funcionalidad del binario en
runtime. Para paquetes con extensiones en C compiladas (DuckDB, pydantic-core,
numpy si estuviera), el ahorro puede ser de varios MB por archivo. Es la
optimización que más diferencia hace en este stack y ningún otro alumno la
implementó.

**Sin pandas ni pyarrow en `requirements-e4.txt`:**
El comentario en el Dockerfile lo justifica correctamente: "DuckDB reads
Parquet natively, so pyarrow/pandas are not needed. This keeps the final image
well under 300 MB." Con solo fastapi + uvicorn + duckdb en el runtime, la
imagen es significativamente más pequeña que las de alumnos que incluyen pandas
(~35MB) o pyarrow (~45MB).

**Limpieza del venv completa:**
```dockerfile
rm -rf /venv/lib/python*/site-packages/pip* \
       /venv/lib/python*/site-packages/wheel* \
       /venv/lib/python*/site-packages/setuptools* \
       /venv/lib/python*/site-packages/pkg_resources* && \
find /venv -name "*.pyc" -delete && \
find /venv -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
```
`pkg_resources` (parte de setuptools) se elimina explícitamente — el único
alumno que lo hace. Junto con pip, wheel y setuptools, eso son ~30MB que no
llegan a la imagen final.

**`PYTHONDONTWRITEBYTECODE=1` y `PYTHONUNBUFFERED=1` ✓**

**Healthcheck con Python urllib:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
```
`start-period=15s` reconoce que la API necesita tiempo para cargar conexiones
DuckDB y SQLite antes de responder al primer healthcheck. Evita que Docker
marque el contenedor como unhealthy en el arranque.

**Sin usuario non-root — único gap concreto.** `USER appuser` en el runtime
stage sería una línea. Sin ella el proceso uvicorn tiene privilegios de root
dentro del contenedor.

**`python:3.12-slim` en lugar de 3.11-slim.** Válido — 3.12 tiene mejoras de
rendimiento del intérprete. Solo relevante si el E04 depende de APIs de 3.11
(no parece ser el caso).

---

## Criterio 2 — docker-compose completo `28 / 30`

**`:ro` en el bind mount del Parquet — correcto:**
```yaml
volumes:
  - ${DATA_DIR:-../data}:/data:ro
  - db_vol:/db
```
El Parquet del host está montado como solo lectura en ambos servicios. El
SQLite generado por `setup` vive en el named volume `db_vol`, separado
del Parquet de entrada.

**`CACHE_TTL` variables conectadas con defaults inline:**
```yaml
api:
  environment:
    E4_PARQUET: /data/benchmark_1m/transactions.snappy.parquet
    E4_DB: /db/transactions.db
    CACHE_TTL_SUMMARY: ${CACHE_TTL_SUMMARY:-30}
    CACHE_TTL_TOP_MERCHANTS: ${CACHE_TTL_TOP_MERCHANTS:-30}
```
La sintaxis `${CACHE_TTL_SUMMARY:-30}` significa: leer del entorno del host
(desde `.env`), y si no está definida usar 30. Las variables realmente llegan
al contenedor y la app del E04 puede leerlas con `os.getenv()`. A diferencia
de Yaanit donde los TTL estaban en `.env.example` pero hardcodeados en el
código, aquí el pipeline completo funciona.

**`restart: unless-stopped` en el servicio `api`:**
El API se reinicia automáticamente si el proceso falla o el host se reinicia,
excepto si fue detenida manualmente. Es la política correcta para un servicio
en producción. Ningún otro alumno especificó `restart` explícitamente.

**CLI flags al `setup_db.py`:**
```yaml
setup:
  command:
    - python
    - /app/ejercicio-04-sistema/setup_db.py
    - --parquet
    - /data/benchmark_1m/transactions.snappy.parquet
    - --db
    - /db/transactions.db
```
Pasar las rutas como argumentos CLI en lugar de solo leerlos desde env vars
hace el script más flexible — se puede probar manualmente fuera de Docker sin
variables de entorno. El `--parquet` y `--db` son los mismos args que
describía el README del E03.

**E4_PARQUET y E4_DB con defaults en ENV del Dockerfile:**
```dockerfile
ENV E4_PARQUET="/data/benchmark_1m/transactions.snappy.parquet" \
    E4_DB="/db/transactions.db"
```
Permite hacer `docker run` directamente sobre la imagen sin docker-compose, si
los volúmenes están montados en las rutas correctas.

**Dos puntos menos** porque `setup_db.py` no fue entregado. No se puede
verificar la idempotencia, el chunking, ni la creación de índices que el README
describe.

---

## Criterio 3 — Variables de entorno `19 / 20`

**`.env.example` limpio con solo las variables del usuario:**
```env
DATA_DIR=../data
CACHE_TTL_SUMMARY=30
CACHE_TTL_TOP_MERCHANTS=30
```

`E4_PARQUET` y `E4_DB` son variables internas del contenedor, configuradas
en `docker-compose.yml` y en el `ENV` del Dockerfile. No aparecen en
`.env.example` porque el usuario no necesita cambiarlas. Esta separación es
correcta: el `.env.example` solo expone las variables que el usuario necesita
configurar.

El README documenta en la tabla de variables:
- `DATA_DIR` — ruta al directorio del Parquet (bind mount)
- `CACHE_TTL_SUMMARY` — TTL para `/analytics/summary`
- `CACHE_TTL_TOP_MERCHANTS` — TTL para `/analytics/top-merchants`
- Y explica que `E4_PARQUET` y `E4_DB` se configuran internamente

**El comentario en `.env.example` documenta opcionalidad:**
```
# Cache TTL in seconds for analytics endpoints (optional, defaults shown)
```
El usuario sabe que tiene defaults si no configura estas variables.

**Un punto menos** porque `setup_db.py` no fue entregado y no se puede
verificar si hay validación explícita de variables con mensajes claros (como
`os.environ["PARQUET"]` que lanzaría `KeyError` vs un mensaje descriptivo).

---

## Criterio 4 — README operacional `25 / 25`

Los cuatro comandos exactos con `docker compose` v2:
```bash
docker compose up --build   ← ✓
docker compose ps           ← ✓
curl http://localhost:8000/health  ← ✓
docker compose logs -f api  ← ✓
docker compose down -v      ← ✓
```

El README también:
- Documenta el prerequisito (Parquet de E01) con los comandos exactos para
  generarlo si no existe ✓
- Distingue `docker compose down` (para servicios) de `docker compose down -v`
  (elimina también el volumen SQLite) ✓
- Incluye ejemplos de curl para todos los endpoints ✓
- Diagrama ASCII de la arquitectura con rutas exactas ✓
- Tabla de variables con defaults ✓
- Documenta que los logs están en formato JSON ✓

---

## Sobre el uso de herramientas de IA

El `strip --strip-unneeded` sobre `.so` files, la exclusión explícita de
pandas/pyarrow con el razonamiento técnico correcto, los defaults inline en
compose para las TTL variables, y el `start-period=15s` en el healthcheck son
decisiones que requieren entender el stack. El `pkg_resources` en la lista de
eliminación explícita (que otros alumnos no incluyen) sugiere que se verificó
qué quedaba en el venv después de la instalación.

---

## Pregunta de seguimiento

> El Dockerfile usa `python:3.12-slim` y el `strip --strip-unneeded` en los
> `.so` del venv. ¿Cuánto pesa la imagen final (`docker images`)? ¿Cuánto
> ahorra el `strip` respecto a no hacerlo? Ejecuta `docker build --no-cache`
> dos veces — una con la línea de `find ... strip` y otra sin ella — y compara
> el tamaño. ¿El ahorro vale la complejidad adicional en el Dockerfile?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Imagen funcional y liviana | 25% | 24 / 25 |
| docker-compose completo | 30% | 28 / 30 |
| Variables de entorno | 20% | 19 / 20 |
| README operacional | 25% | 25 / 25 |
| **Total** | **100%** | **96 / 100** |

---

El `strip --strip-unneeded` y la exclusión de pandas/pyarrow son las dos
optimizaciones más técnicas del grupo en E07. El único cambio concreto antes
del E08: agregar `RUN useradd --create-home appuser` y `USER appuser` al
runtime stage del Dockerfile.