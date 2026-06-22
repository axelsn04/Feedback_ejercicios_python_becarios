# Retroalimentación — Ejercicio 07: De tu Máquina al Mundo

**Alumno:** Angel Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 83 / 100

---

## Resumen general

La capa de aplicación es la más sofisticada del grupo en E07: `sys.exit()` con
mensajes descriptivos para validación de variables, dos niveles de validación
(variable definida + archivo existe), `CACHE_TTL` conectado correctamente desde
`.env` hasta el código, y `127.0.0.1:8000:8000` en el port binding que evita
exponer el servicio a redes externas — decisión de seguridad que ningún otro
alumno tomó. Los 12 tests incluyen el más completo del grupo para idempotencia:
verifica que el amount original (100.00) no se sobreescribe cuando llega el
mismo `transaction_id` con amount diferente (250.00). El gap es significativo:
`Dockerfile`, `Dockerfile.setup` y el script de setup no fueron entregados —
el 30% más crítico del ejercicio no puede verificarse.

---

## Criterio 1 — Imagen funcional y liviana `17 / 25`

**Lo que el README afirma y el código soporta:**

El README documenta: `python:3.11-slim` como base, multi-stage con `uv` para
la fase de build, imagen final ~175MB. Si es preciso, sería la imagen más
liviana del grupo — Ulises midió 288MB. El app code está preparado para
contenedores: `PYTHONUNBUFFERED` implícito vía `sys.exit()` (termina
inmediatamente en error), logging a stdout, y las rutas vienen de variables de
entorno.

**Validación fail-fast en `db.py` — los dos niveles correctos:**
```python
if not sqlite_path:
    sys.exit("CRITICAL: Environment variable SQLITE_DB_PATH is not set.")
    
if not os.path.exists(sqlite_path):
    sys.exit(f"CRITICAL: SQLite database file not found at: {sqlite_path}. "
             "Make sure the setup container runs successfully first.")
```
El primer `sys.exit` es el error de configuración. El segundo es el error de
runtime (la variable está definida pero el archivo no existe en el volumen
montado). Los mensajes de error son específicos al contexto Docker. Esto es
mejor que un traceback de Python o una excepción genérica.

**DuckDB pre-warmed en lifespan:**
```python
cursor.execute("SELECT COUNT(*) FROM transactions_view;")
```
La primera request a `/analytics/summary` llega a caché precalentada — la
latencia de "primera request fría" se desplaza al arranque del contenedor.

**`Dockerfile` y `Dockerfile.setup` no entregados.** Sin los archivos no se
puede verificar:
- El multi-stage real
- Si pip/setuptools/wheel se eliminan del runtime
- Si hay usuario non-root
- El tamaño medido (no solo declarado)
- Si los `~175MB` son correctos

El criterio evalúa que la imagen construye y pesa menos de 300MB. Con los
archivos faltantes no se puede confirmar ninguna de estas propiedades.

---

## Criterio 2 — docker-compose completo `22 / 30`

**`127.0.0.1:8000:8000` — decisión de seguridad única en el grupo:**
```yaml
ports:
  - "127.0.0.1:8000:8000"
```
El binding a localhost evita que el puerto sea accesible desde la red externa
del host. En un servidor real con una IP pública, `0.0.0.0:8000:8000` expondría
la API directamente a Internet sin un reverse proxy. El README documenta la
razón: "prevents public network exposure". Ningún otro alumno tomó esta
decisión.

**`${VAR:-default}` inline con defaults correctos:**
```yaml
environment:
  - SQLITE_DB_PATH=${SQLITE_DB_PATH:-/data/transactions.db}
  - PARQUET_FILE_PATH=${PARQUET_FILE_PATH:-/data/test_1m_snappy.parquet}
  - CACHE_TTL=${CACHE_TTL:-60}
```
El sistema funciona sin crear `.env` — los defaults son los valores correctos
para la configuración estándar del repo.

**`depends_on: condition: service_completed_successfully` correcto ✓**

**Sin named volume para SQLite — gap de idempotencia:**
```yaml
volumes:
  - ../data:/data
```
Tanto `setup` como `api` montan el directorio del host directamente. El SQLite
vive en `../data/transactions.db` en el host, no en un named volume. Esto
significa que `docker compose down -v` no elimina la SQLite (es un bind mount
al host, no un volumen Docker). El README dice "remover los volúmenes de datos
creados en runtime" con `down -v`, pero eso no aplica a bind mounts. Si la
intención es que una segunda ejecución detecte el SQLite existente y sea
idempotente, funciona — pero la limpieza requiere `rm ../data/transactions.db`
manualmente.

**Sin `:ro` en el Parquet.** Ambos servicios pueden escribir en el directorio
de datos del host. El Parquet de origen podría ser modificado accidentalmente.

**`Dockerfile.setup` no entregado.** No se puede verificar qué hace el
contenedor de setup — si verifica idempotencia, crea índices, o carga los
datos en chunks.

---

## Criterio 3 — Variables de entorno `20 / 20`

**`CACHE_TTL` conectado correctamente — el único alumno que lo resuelve así:**
```python
# main.py
CACHE_TTL = int(os.getenv("CACHE_TTL", "60"))
# ...
cache.set(cache_key, result, CACHE_TTL)
```
La variable viene de `.env`, el compose la pasa al contenedor con default
`${CACHE_TTL:-60}`, y la app la lee con `os.getenv`. El ciclo completo
funciona. A diferencia de Yaanit donde `CACHE_TTL_SUMMARY` estaba en
`.env.example` pero hardcodeada en el código, aquí el valor realmente es
configurable.

**Validación en dos niveles en `db.py`:**
1. Variable no definida → `sys.exit("CRITICAL: ... is not set.")`
2. Archivo no existe → `sys.exit("CRITICAL: ... not found at: {path}.")`

El segundo mensaje menciona explícitamente "Make sure the setup container runs
successfully first" — en el contexto Docker ese es el diagnóstico correcto.

**`.env.example` limpio con exactamente las variables que el usuario necesita:**
```env
SQLITE_DB_PATH=/data/transactions.db
PARQUET_FILE_PATH=/data/test_1m_snappy.parquet
CACHE_TTL=60
```

---

## Criterio 4 — README operacional `24 / 25`

Los cuatro comandos requeridos con `docker compose` v2:
```bash
docker compose up --build   ← ✓
docker compose ps           ← ✓
curl http://localhost:8000/health  ← ✓
docker compose logs -f api  ← ✓
docker compose down -v      ← ✓
```

El README incluye además las instrucciones de instalación de Docker en
Ubuntu/WSL2 — el único del grupo que lo documenta:
```bash
sudo apt-get install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
# Nota: Cerrar y volver a abrir sesión para aplicar los permisos de grupo.
```
Para alguien que recibe este repo en una máquina limpia, ese paso adicional
marca la diferencia.

Un punto menos: el README describe el formato de los logs pero no incluye el
resultado esperado de `curl http://localhost:8000/health`, que ayudaría a
verificar que el sistema está funcionando correctamente.

---

## Sobre la suite de tests y el código de aplicación

El `test_post_batch_idempotency_database` es el test más completo del grupo
para deduplicación:
```python
# Segunda llamada con amount diferente (250.00 vs 100.00 original)
resp2 = client.post("/transactions/batch", json=payload2)
assert resp2.json()["ignored_ids"] == [tx_id]

# Verifica que el amount original (100.00) NO fue sobreescrito
inserted_tx = next((t for t in txs if t["transaction_id"] == tx_id), None)
assert inserted_tx["amount"] == 100.00
```
Verifica que `INSERT OR IGNORE` realmente ignora (no actualiza). La mayoría de
tests del grupo verifica que `inserted == 0`, pero no verifican que el estado
de la base no cambió.

El campo `ignored_ids` en la respuesta del batch (`/transactions/batch`) es
también único:
```python
return {
    "inserted_records": len(to_insert),
    "ignored_records": len(existing_ids),
    "ignored_ids": list(existing_ids)   ← auditability
}
```
Permite al cliente saber exactamente qué IDs fueron rechazados y por qué,
lo que es útil para reconciliación.

---

## Pregunta de seguimiento

> `Dockerfile` y `Dockerfile.setup` no fueron entregados. Comparte los dos
> archivos. En particular: ¿`Dockerfile.setup` incluye pandas o solo duckdb?
> ¿Cuánto pesa la imagen resultante de `docker images` después de un
> `docker compose build`? ¿El setup verifica si la SQLite ya existe antes de
> cargar para garantizar idempotencia?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Imagen funcional y liviana | 25% | 17 / 25 |
| docker-compose completo | 30% | 22 / 30 |
| Variables de entorno | 20% | 20 / 20 |
| README operacional | 25% | 24 / 25 |
| **Total** | **100%** | **83 / 100** |

---

Los Dockerfiles son los archivos más críticos del E07 — sin ellos el criterio
principal (imagen) no puede verificarse. La aplicación en sí es la más completa
del grupo: CACHE_TTL conectado, `ignored_ids` en la respuesta del batch,
validación en dos niveles, middleware de request tracking. Si se entregan los
Dockerfiles y la imagen mide ~175MB, la nota sería considerablemente más alta.