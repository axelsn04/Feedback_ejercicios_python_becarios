# Retroalimentación — Ejercicio 07: De tu Máquina al Mundo

**Alumno:** Ulises Josue Reyes Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 98 / 100

---

## Resumen general

La entrega más completa del grupo en E07. Cinco decisiones la distinguen: la
imagen final pesa 288MB después de un proceso de optimización documentado con
`docker history --no-trunc` y `du -sh` reales (376MB → 288MB con cuatro
optimizaciones específicas y sus trade-offs); el healthcheck usa Python en
lugar de `curl` eliminando 13.5MB sin perder funcionalidad; el script
`setup.py` implementa chunking real por LIMIT/OFFSET sobre la vista DuckDB
(no carga 1M filas de golpe); `entrypoint.sh` valida tanto que las variables
existen como que los archivos físicamente existen en el volumen montado; y el
decisions.md documenta el razonamiento de cada decisión con números medidos,
incluyendo la limitación honesta del logging (8/10 líneas JSON, 2 son `print()`
del E4 que no se modificaron). El único gap concreto: el build context en la
raíz del repo requiere copiar `.dockerignore` manualmente antes del primer
`docker compose up --build`.

---

## Criterio 1 — Imagen funcional y liviana `25 / 25`

**288MB verificado con docker images después de cuatro optimizaciones:**

La ruta de 376MB → 288MB está documentada con origen de cada MB:

| Componente | Tamaño | Modificable |
|---|---|---|
| Debian trixie | 87.4MB | No |
| Python 3.11.15 | 48.4MB | No |
| DuckDB binary .so | 58MB | No |
| pip + setuptools + wheel | 28-29MB | → eliminados del venv final |
| uvicorn\[standard\] extras | 18.5MB | → uvicorn sin extras |
| curl | 13.5MB | → reemplazado por Python urllib |
| \_\_pycache\_\_ y .pyc | 11.8MB | → --no-compile + PYTHONDONTWRITEBYTECODE |

El análisis viene de `docker history --no-trunc` combinado con `du -sh`
dentro del contenedor — no son estimaciones, son mediciones reales.

**Multi-stage correcto:**
```dockerfile
FROM python:3.11-slim AS build
# instala deps en /opt/venv, luego elimina pip/setuptools/wheel
RUN pip install ... && rm -rf /opt/venv/lib/python3.11/site-packages/pip ...

FROM python:3.11-slim AS final
COPY --from=build /opt/venv /opt/venv
# solo el código de app/, no datos
```

**Healthcheck sin curl — ahorra 13.5MB:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=5)" || exit 1
```
`urlopen` lanza `HTTPError` para 4xx/5xx y `URLError` si no hay respuesta —
ambos casos producen exit != 0. Funcionalidad idéntica a `curl -f` sin el
binario extra. Verificado en Python real con puerto cerrado.

**Trade-offs documentados con honestidad:**
- Eliminar `uvicorn[standard]` pierde `uvloop` (event loop más rápido en C)
  y `httptools` (parser HTTP en C). El trade-off está documentado. Se verificó
  que `pyyaml` sigue disponible para `--log-config` al agregar `pyyaml`
  explícitamente en requirements.txt.
- Eliminar `pip` del venv final significa que no se puede hacer `pip install`
  dentro del contenedor para debugging.

**Non-root user `appuser` — sin costo funcional.**

---

## Criterio 2 — docker-compose completo `28 / 30`

**Patrón de imagen compartida entre servicios:**
```yaml
setup:
  build:
    context: ..
    dockerfile: ejercicio-07-contenedores/Dockerfile
  image: transacciones-api:latest
  entrypoint: ["python", "scripts/setup.py"]   ← override del ENTRYPOINT

api:
  build:
    context: ..
    dockerfile: ejercicio-07-contenedores/Dockerfile
  image: transacciones-api:latest              ← misma imagen
  depends_on:
    setup:
      condition: service_completed_successfully
```

`image: transacciones-api:latest` en ambos servicios le indica a Docker
Compose que son la misma imagen — la construye una sola vez y la reutiliza.
`setup` sobreescribe el `ENTRYPOINT` del Dockerfile para correr `setup.py`
en lugar de `entrypoint.sh`. El decisions.md documenta que sin esta solución
el primer intento generó un "conflicto de tags al construir dos servicios en
paralelo" — esto solo se detecta corriendo, no leyendo el código.

**`depends_on: condition: service_completed_successfully` — correcto.**
`api` no arranca hasta que `setup` termina con código 0. `setup.py` retorna 0
tanto en el caso de "generé la DB" como en el de "la DB ya existía
(idempotente)".

**`setup.py` con chunking real por LIMIT/OFFSET:**
```python
while True:
    rows = conn.execute(f"""
        SELECT ... FROM transactions
        LIMIT {CHUNK_SIZE} OFFSET {offset}
    """).fetchall()
    if not rows:
        break
    sqlite_conn.executemany("INSERT OR IGNORE ...", rows)
    total_inserted += len(rows)
    offset += CHUNK_SIZE
```
LIMIT/OFFSET sobre la vista DuckDB pagina la lectura del Parquet sin cargar
1M filas de golpe. Documentado en el código con la referencia explícita a
`ingest.py` del E3 y `bulk_create` del E5. `PRAGMA journal_mode=WAL` ✓.
Crea `idx_user_timestamp` con el mismo nombre del E3 ✓.

**Idempotencia del servicio `setup` verificada:**
```python
if os.path.exists(db_path):
    print(f"setup: {db_path} ya existe -- nada que hacer (idempotente).")
    return 0
```
Segunda corrida de `docker compose up` termina en segundos. La salida del
decisions.md lo confirma: `"setup-1 | setup: /data/transactions.db ya existe -- nada que hacer (idempotente)."`.

**Gap concreto — `.dockerignore` en la raíz del repo:**
El build context es `..` (raíz del repo), lo que requiere que `.dockerignore`
esté en la raíz, no en `ejercicio-07-contenedores/`. El README documenta el
paso:
```bash
cp .dockerignore ../.dockerignore
```
Esto es un paso manual antes del primer `docker compose up --build` que
no es obvio para alguien que clona el repo. El decisions.md lo documenta como
una consecuencia intencional de la Decisión 2 (build context en la raíz), pero
el costo operacional es real: en una máquina limpia falta ese paso manual.
La alternativa habría sido duplicar `app/` dentro de `ejercicio-07-contenedores/`
para usar esa carpeta como build context — decisión con un trade-off peor
(código duplicado), documentado correctamente.

---

## Criterio 3 — Variables de entorno `20 / 20`

**`.env.example` completo:**
```env
PARQUET_PATH=/data/transactions_1m_parquet_snappy.parquet
DB_PATH=/data/transactions.db
ANALYTICS_TTL=300
```
Las tres variables que necesita el sistema. `ANALYTICS_TTL` tiene default
en `entrypoint.sh` (`${ANALYTICS_TTL:-300}`) — es opcional pero documentada.

**Validación en dos niveles en `entrypoint.sh`:**
```sh
# Nivel 1: variable definida
for var in $required_vars; do
    eval "value=\$$var"
    if [ -z "$value" ]; then
        echo "ERROR: la variable de entorno '$var' es requerida..." >&2
        exit 1
    fi
done

# Nivel 2: archivo existe en el volumen montado
if [ ! -f "$PARQUET_PATH" ]; then
    echo "ERROR: PARQUET_PATH='$PARQUET_PATH' no existe dentro del contenedor." >&2
    echo "Verifica que el volumen de datos esté montado y que el servicio 'setup' haya corrido." >&2
    exit 1
fi
```
El mensaje del nivel 2 menciona el servicio `setup` explícitamente — en el
contexto de Docker ese es el diagnóstico correcto. El decisions.md documenta
el razonamiento: `entrypoint.sh` da el error más rápido y más específico;
`init_connections()` en `db.py` sigue siendo la garantía de último recurso
si el sistema corre fuera de Docker.

**Compatible con `sh` (dash)** — verificado con `dash` que es `/bin/sh` de
`python:3.11-slim`. El decisions.md documenta este chequeo explícitamente.

---

## Criterio 4 — README operacional `25 / 25`

Los cuatro comandos requeridos, exactamente como están escritos:

```bash
docker compose up --build        # ✓ Paso: levantar desde cero
docker compose ps                # ✓ Paso: verificar estado
curl http://localhost:8000/health # ✓ Paso: verificar health
docker compose logs -f api       # ✓ Paso: logs en tiempo real
docker compose down -v           # ✓ Paso: parar y limpiar
```

Todos verificables tal como están escritos. El README también documenta:
- La nota sobre `-v` (elimina volúmenes anónimos pero no el bind mount
  `./data`) — alguien que ejecute `docker compose down -v` esperando borrar
  todo y luego se pregunta por qué `transactions.db` sigue ahí, lo va a
  agradecer.
- La respuesta esperada de `/health` con el JSON completo.
- Equivalentes en PowerShell para Windows.

**Limitación conocida documentada con honestidad:**
```
Limitación conocida: app/main.py (E4) usa dos print() directos en el
lifespan. Esas dos líneas salen como texto plano, no como JSON.
Verificado localmente: de 10 líneas de log, 8 son JSON válido y 2 son
texto plano.
```
El porcentaje exacto (8/10) confirma que corrió el ciclo completo. La
decisión de no modificar `app/main.py` (código evaluado del E4) es correcta
y está bien justificada.

---

## Sobre el uso de herramientas de IA

El viaje de 376MB → 288MB documentado con `docker history --no-trunc` y
`du -sh` en el contenedor real, la detección del conflicto de tags al
construir dos servicios en paralelo (solo visible corriendo), la validación
de `dash` para `entrypoint.sh`, y el ciclo completo de 10 líneas de log con
el conteo exacto de 8/10 JSON son indicadores de que el sistema se corrió
real. El decisions.md con los segundos exactos por actividad y las
"sorpresas" de CRLF y archivos vacíos que describe es el tipo de narración
que no se genera — se vive.

---

## Pregunta de seguimiento

> El build context es `..` (raíz del repo) y `.dockerignore` debe estar en la
> raíz. Una alternativa habría sido crear un symlink en lugar de copiar:
> `ln -s ejercicio-07-contenedores/.dockerignore .dockerignore`. ¿Eso
> resolvería el problema en Linux/Mac? ¿Y en Windows con Git for Windows?
> ¿Cuál es la solución más portable para un equipo que trabaja en ambos sistemas?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Imagen funcional y liviana | 25% | 25 / 25 |
| docker-compose completo | 30% | 28 / 30 |
| Variables de entorno | 20% | 20 / 20 |
| README operacional | 25% | 25 / 25 |
| **Total** | **100%** | **98 / 100** |

---

El único gap concreto es el `.dockerignore` en la raíz — consecuencia
documentada y razonada de la Decisión 2 (build context en raíz del repo), pero
que representa un paso manual no obvio en una máquina limpia. Todo lo demás:
imagen optimizada con mediciones reales, chunking correcto en setup.py,
validación en dos niveles en entrypoint.sh, logging JSON con la limitación
honestamente documentada.