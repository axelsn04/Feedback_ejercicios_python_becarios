# Retroalimentación — Ejercicio 07: De tu Máquina al Mundo

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 91 / 100

---

## Resumen general

La entrega más cuidadosa del grupo en arquitectura de volúmenes: el bind mount
del Parquet tiene el flag `:ro` (solo lectura) — el único alumno que lo
implementó, y es la práctica correcta cuando un contenedor no necesita escribir
en sus datos de entrada. La separación entre un named volume para SQLite
(`db-data`) y el bind mount para el Parquet del host es limpia y bien
justificada en el reporte. El `--prefix=/install` en la instalación de
dependencias evita contaminar la imagen final con pip/setuptools sin necesitar
limpieza manual. El setup aplica el feedback del E03: usa `iter_batches` para
chunking real, no `pd.read_parquet()` que carga todo a memoria. El único gap
concreto: sin usuario non-root y sin `PYTHONDONTWRITEBYTECODE`.

---

## Criterio 1 — Imagen funcional y liviana `21 / 25`

**Enfoque `--prefix=/install` — solución limpia a la contaminación de pip:**
```dockerfile
FROM python:3.11-slim AS builder
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /install /usr/local
```

`pip install --prefix=/install` instala los paquetes en `/build/install/`
dentro del builder stage. `pip`, `setuptools` y `wheel` quedan en el location
default del builder (`/usr/local/lib/python3.11/site-packages/`) — no se
copian. El `COPY --from=builder /install /usr/local` trae solo las
dependencias declaradas en `requirements.txt` a la imagen final, sin el
overhead del gestor de paquetes. Es una alternativa válida y más elegante que
el patrón venv + `rm -rf pip*`.

**Python urllib para healthcheck — sin curl:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
```
El reporte documenta explícitamente la decisión: "no vale la pena instalar
paquetes extra solo para el healthcheck". Correcto — curl agrega ~13.5MB a la
imagen por un uso que Python stdlib cubre completamente.

**`<300MB` confirmado en el reporte.** "La imagen final quedó por debajo de
los 300MB requeridos" — con FastAPI, uvicorn, DuckDB, PyArrow y Pandas. Sin
requirements.txt entregado no se puede calcular el margen exacto, pero el
criterio se cumple.

**Sin usuario non-root.** El proceso uvicorn corre como root dentro del
contenedor. En producción, `RUN useradd --create-home appuser` + `USER appuser`
es el estándar. Sin esto el proceso tiene privilegios de root si el contenedor
se escapa.

**Sin `PYTHONDONTWRITEBYTECODE=1`.** Python generará archivos `.pyc` en el
contenedor al importar módulos. No afecta funcionalidad pero ocupa espacio
innecesario.

---

## Criterio 2 — docker-compose completo `27 / 30`

**Named volume para SQLite + bind mount `:ro` para Parquet — la separación más
correcta del grupo:**
```yaml
volumes:
  - db-data:/data/db                            # SQLite, persiste
  - ${PARQUET_DIR:-./../data}:/data/parquet:ro  # Parquet, solo lectura
```

Dos decisiones en esta línea:
1. `db-data` es un named volume de Docker — persiste entre reinicios y se
   puede eliminar explícitamente con `docker compose down -v`.
2. El Parquet del host se monta con `:ro` (read-only) — el contenedor no puede
   modificar los datos de entrada accidentalmente. Ningún otro alumno del grupo
   implementó esto.

**`${PARQUET_DIR:-./../data}` con default en compose:**
Si `PARQUET_DIR` no está definida en `.env`, el compose usa `./../data` como
valor por defecto. Esto permite que funcione sin ningún cambio desde una
instalación limpia, y también permite override en `.env` sin cambiar el compose.

**`depends_on: condition: service_completed_successfully` correcto.** ✓

**Setup idempotente con `iter_batches` — aplicando el feedback del E03:**

El reporte documenta el setup_db.py:
1. Verifica si la base ya tiene datos → si sí, termina con código 0
2. Usa `iter_batches` para leer el Parquet en chunks (no carga 1M filas a RAM)
3. Crea los índices `idx_user_timestamp` e `idx_country_user`
4. Inserta con `INSERT OR IGNORE`

El punto 2 aplica explícitamente el feedback del E03. El punto 1 garantiza
idempotencia: `docker compose up` por segunda vez termina `setup` en segundos
sin recargar datos.

Tres puntos menos porque `setup_db.py` no fue entregado para verificación
directa del código — se evalúa el comportamiento descrito en el reporte.

---

## Criterio 3 — Variables de entorno `18 / 20`

**`.env.example` con tres variables bien separadas:**
```env
SQLITE_PATH=/data/db/transactions.db
PARQUET_PATH=/data/parquet/transactions_1m_none.parquet
PARQUET_DIR=./../data
```

`PARQUET_DIR` opera en el nivel de compose (para el bind mount). `SQLITE_PATH`
y `PARQUET_PATH` operan dentro del contenedor. La separación es correcta.

**Error claro si falta variable — documentado en el reporte:**
```
ERROR: la variable de entorno SQLITE_PATH es requerida.
Revisa .env.example para los valores esperados.
```
Mensaje específico sobre qué variable falta y dónde buscar la solución. Mejor
que un traceback de Python genérico.

**Sin validación de existencia de archivos.** El reporte describe validación de
variables pero no de si los archivos existen en el volumen montado. Si `PARQUET_PATH`
apunta a un archivo que no existe, el error llega más tarde (cuando setup.py
intenta abrir el archivo) en lugar de inmediatamente al arrancar. La verificación
de `os.path.exists(parquet_path)` antes de intentar leer sería el nivel
siguiente de robustez.

Un punto menos porque `setup_db.py` no está disponible para verificar que la
validación realmente está implementada tal como se describe en el reporte.

---

## Criterio 4 — README operacional `25 / 25`

Los cuatro comandos requeridos, exactamente como están escritos, con `docker compose`
v2 (sin guion):

```bash
docker compose up --build       ← ✓
docker compose ps               ← ✓
curl http://localhost:8000/health ← ✓
docker compose logs -f api      ← ✓
docker compose down -v          ← ✓
```

El README también:
- Explica qué hacen los tres pasos de `docker compose up --build` en orden
- Documenta cómo generar el Parquet si no existe (`uv run python generate_data.py`)
- Incluye ejemplos de curl para todos los endpoints
- Documenta el significado del flag `-v` en `down -v`
- Incluye un diagrama ASCII de la arquitectura completa con los volúmenes
- Documenta las tres variables de entorno con sus valores y propósito
- Muestra el formato esperado de los logs JSON

El diagrama de arquitectura es el único del grupo:
```
┌──────────┐         ┌──────────────────────┐
│  setup   │────────→│       api            │
│(run once)│  done   │  FastAPI :8000       │
└──────┬───┘         └──────────┬───────────┘
       └──────────┬─────────────┘
              ┌───┴──────┐
              │ db-data  │  ← volumen compartido
              │(SQLite)  │
              └──────────┘
```

---

## Sobre el reporte técnico

El reporte documenta siete decisiones de diseño con razonamiento claro: la
elección de E4 sobre E5 (un punto de entrada vs setup multi-paso), la separación
de volúmenes, el `:ro` en el Parquet, `iter_batches` vs `pd.read_parquet`,
Python urllib vs curl, el setup como servicio separado vs script de entrypoint,
y por qué los datos no van en la imagen. La referencia explícita al feedback
del E03 para el chunking ("aplicando el feedback que recibí en el tercer
ejercicio") demuestra que el ciclo de retroalimentación funciona.

---

## Pregunta de seguimiento

> El Parquet está montado con `:ro` (read-only), lo que significa que ningún
> contenedor puede modificarlo. El named volume `db-data` para SQLite persiste
> entre reinicios. ¿Qué pasa si corres `docker compose down` (sin `-v`) y
> luego `docker compose up --build`? ¿El setup vuelve a correr? ¿Inserta datos
> o lo detecta como idempotente? ¿Y si corres `docker compose down -v`?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Imagen funcional y liviana | 25% | 21 / 25 |
| docker-compose completo | 30% | 27 / 30 |
| Variables de entorno | 20% | 18 / 20 |
| README operacional | 25% | 25 / 25 |
| **Total** | **100%** | **91 / 100** |

---

El `:ro` en el bind mount del Parquet y la separación named volume/bind mount
son las dos decisiones arquitectónicas más correctas del grupo para el E07. Los
dos cambios antes del E08: agregar `USER appuser` (non-root) y
`PYTHONDONTWRITEBYTECODE=1` al Dockerfile, y verificar la existencia del archivo
además de la variable de entorno en `setup_db.py`.