# Retroalimentación — Ejercicio 07: De tu Máquina al Mundo

**Alumno:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 68 / 100

---

## Resumen general

La estructura Docker está en su lugar — multi-stage Dockerfile, dos servicios,
`depends_on: condition: service_completed_successfully`, bind mount compartido
y healthcheck. Los cuatro gaps concretos: `setup_db.py` usa `if_exists="replace"`
en `pd.to_sql` que borra y recrea la tabla en cada corrida (no es idempotente);
la imagen probablemente supera los 300MB porque `curl` se instala por separado
y no se aplican las optimizaciones de pip/bytecode; dos variables del
`.env.example` (`CACHE_TTL_SUMMARY`, `CACHE_TTL_MERCHANTS`) están declaradas
pero el código usa TTLs hardcodeados; y el README usa `docker-compose` (v1)
en lugar de `docker compose` (v2).

**Nota:** `api/db.py` y `requirements.txt` no fueron entregados. Sin
`requirements.txt` no se puede calcular el tamaño exacto de la imagen.

---

## Criterio 1 — Imagen funcional y liviana `15 / 25`

**Multi-stage correcto en estructura:**
```dockerfile
FROM python:3.11-slim AS builder
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
```
El builder instala deps; el final copia solo librerías y binarios. La idea es
correcta.

**Imagen probablemente supera 300MB — cuatro optimizaciones faltantes:**

`curl` se instala en el stage final para el healthcheck (+13.5MB medido en
`docker history`). La alternativa correcta es Python con `urllib.request`:
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=5)" || exit 1
```
El binario de DuckDB pesa 58MB fijo. `pandas`, que `setup_db.py` requiere para
leer el Parquet y está en la misma imagen que la API, agrega ~35MB. Sin los
recortes que lograron otros alumnos (eliminar `pip`/`setuptools`/`wheel`,
`PYTHONDONTWRITEBYTECODE=1`, `--no-compile`), la imagen base de
`python:3.11-slim` más estas dependencias probablemente supera los 300MB.

Sin `requirements.txt` no se puede calcular con precisión. Si la imagen mide
menos de 300MB, este criterio sería más alto.

**Sin usuario non-root.** El proceso uvicorn corre como root dentro del
contenedor. `useradd --create-home appuser` + `USER appuser` es una línea.

**`COPY . .` copia todo el contexto.** Sin `.dockerignore` entregado, no es
posible verificar que archivos `.parquet`, `.db` u otros artefactos no entran
en la imagen.

---

## Criterio 2 — docker-compose completo `20 / 30`

**`depends_on: condition: service_completed_successfully` correcto:**
```yaml
api:
  depends_on:
    setup:
      condition: service_completed_successfully
```
`api` no arranca hasta que `setup` termina con código 0. ✓

**Bind mount compartido `../data:/data` — funcional:**
Ambos servicios montan la misma carpeta del host. `setup` escribe la SQLite
ahí; `api` la lee. ✓

**`setup_db.py` no es idempotente — `if_exists="replace"` borra los datos:**
```python
df.to_sql("transactions", conn, if_exists="replace", index=False)
```
`replace` hace `DROP TABLE IF EXISTS transactions` antes de crear la nueva.
En la primera corrida: funciona. En la segunda corrida de `docker compose up`:
el servicio `setup` vuelve a correr, borra toda la tabla y la recarga desde
cero. El enunciado pide idempotencia explícitamente: "correrlo dos veces con
los mismos datos debe dar el mismo resultado". Con `replace`, la tabla se
destruye y se recrea — los mismos datos pero con overhead innecesario y riesgo
de inconsistencia si la carga falla a mitad.

La corrección: verificar si `DB_PATH` ya existe antes de hacer cualquier cosa,
exactamente como Ulises hace en `setup.py`:
```python
if os.path.exists(db_path):
    print(f"setup: {db_path} ya existe -- nada que hacer.")
    return 0
```

**Sin índices en `setup_db.py`.** `pd.to_sql` crea la tabla pero no crea
`idx_user_timestamp` ni `idx_country_user`. El E3 demostró que esos índices
son lo que permite cumplir el SLA de <80ms para los endpoints de usuario.
Sin índices, el servicio arranca pero el SLA del E4 no se cumple.

**Sin chunking — carga 1M filas a memoria de golpe:**
```python
df = pd.read_parquet(parquet_path)   # 1M filas a RAM
df.to_sql("transactions", ...)
```
Para el contexto de Docker con posibles límites de memoria por contenedor,
cargar 1M filas con pandas ocupa ~500MB-1GB en RAM dependiendo del tipo de
datos. Chunking real (LIMIT/OFFSET sobre DuckDB o `iter_batches` de pyarrow)
evita ese pico.

**Sin WAL mode** en la conexión SQLite de `setup_db.py`.

---

## Criterio 3 — Variables de entorno `15 / 20`

**`.env.example` más completo del grupo — 6 variables:**
```env
DB_PATH=/data/transactions.db
PARQUET_PATH=/data/benchmark_1m.parquet
API_HOST=0.0.0.0
API_PORT=8000
CACHE_TTL_SUMMARY=60
CACHE_TTL_MERCHANTS=30
```

`CACHE_TTL_SUMMARY` y `CACHE_TTL_MERCHANTS` son los únicos TTLs con
valores diferentes por endpoint — diseño correcto.

**`CACHE_TTL_SUMMARY` y `CACHE_TTL_MERCHANTS` declaradas pero nunca leídas:**
```python
# api/main.py
cache.set(cache_key, payload, ttl_seconds=60)   # hardcodeado
cache.set(cache_key, payload, ttl_seconds=30)   # hardcodeado
```
Las variables existen en `.env.example` y se cargan con `env_file: .env`,
pero el código ignora `os.getenv("CACHE_TTL_SUMMARY")` y usa literales.
Es el mismo patrón que `VALID_STATUSES` en los ejercicios anteriores: la
variable está declarada pero desconectada del código.

**Validación de variables en `setup_db.py` — `raise ValueError` genera un
traceback de Python, no un mensaje limpio:**
```python
if not parquet_path:
    raise ValueError("Missing environment variable: PARQUET_PATH")
```
El contenedor termina con un stacktrace, no con el mensaje `raise` aislado.
La alternativa limpia es `print(..., file=sys.stderr); sys.exit(1)`, que da
el mensaje sin el traceback.

`API_HOST` y `API_PORT` están en `.env.example` pero no se usan desde
variables de entorno — el `command` en docker-compose los hardcodea:
`uvicorn api.main:app --host 0.0.0.0 --port 8000`.

---

## Criterio 4 — README operacional `18 / 25`

Los cuatro comandos requeridos están presentes. El problema es la sintaxis:

```bash
docker-compose up --build     ← v1 (con guion)
docker-compose ps             ← v1
docker-compose logs -f api    ← v1
docker-compose down -v        ← v1
```

El spec y el entorno de verificación usa Docker Compose v2 (`docker compose`
sin guion, plugin nativo de Docker). En macOS con Docker Desktop moderno y en
Linux con instalación reciente, `docker-compose` puede no estar instalado como
binario standalone. En una máquina limpia con solo Docker v2, estos comandos
fallan.

La sección "Comportamiento esperado" describe qué hace el sistema pero no
incluye la respuesta JSON esperada de `/health`, ni ejemplos de otros
endpoints, ni instrucciones para obtener un token si los endpoints de usuario
están protegidos.

No hay documentación de cuándo usar `--build` vs cuándo no (después de la
primera vez).

---

## Sobre `api/main.py` y el sistema del E4/E5

`api/main.py` tiene los 6 endpoints con la arquitectura dual DuckDB/SQLite y
caché TTL diferenciado — los conceptos son correctos. Sin `api/db.py` no se
puede verificar cómo se inicializan y comparten las conexiones. La nota en la
retroalimentación del E05 sobre el mismo bug de paginación (404 solo en
`page == 1`) sigue presente en este archivo — los endpoints de usuario
retornan 200 vacío si se pide `page=2` de un usuario inexistente.

---

## Sobre el uso de herramientas de IA

El código funciona en sus partes verificables, pero el lenguaje de los
comentarios y docstrings — "Dual-engine CQRS API", "crush strict latency SLAs",
"ultra-fast RAM store", "mathematical hit-rate coefficient" — es el tipo de
prosa que describe el código en lugar de documentar decisiones. Los comentarios
técnicos útiles documentan por qué se tomó una decisión, no qué hace el código
(el código ya lo dice). El gap de `if_exists="replace"` (no idempotente) y las
dos variables de entorno que no se leen son los mismos patrones de los
ejercicios anteriores — variables declaradas pero desconectadas.

---

## Pregunta de seguimiento

> `setup_db.py` usa `df.to_sql(..., if_exists="replace")`. Cambia esto para
> que el script sea idempotente: si `DB_PATH` ya existe, imprime un mensaje y
> termina con código 0 sin tocar nada. Si no existe, crea la tabla con
> `if_exists="fail"` en lugar de `"replace"`. Luego agrega los dos índices del
> E3 con `cursor.execute("CREATE INDEX ...")` antes de `conn.commit()`. ¿Por
> qué `"fail"` es más seguro que `"replace"` para una primera carga?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Imagen funcional y liviana | 25% | 15 / 25 |
| docker-compose completo | 30% | 20 / 30 |
| Variables de entorno | 20% | 15 / 20 |
| README operacional | 25% | 18 / 25 |
| **Total** | **100%** | **68 / 100** |

---

Los cuatro cambios concretos antes del E08:
1. `if_exists="replace"` → check si DB existe, si no usar `"fail"` + crear índices
2. Reemplazar `curl` en el healthcheck por Python urllib (−13.5MB, elimina `apt-get`)
3. Conectar `CACHE_TTL_SUMMARY` y `CACHE_TTL_MERCHANTS` al código con `os.getenv`
4. `docker-compose` → `docker compose` (v2) en el README