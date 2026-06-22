# Ejercicio 7 — De tu Máquina al Mundo

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

Tu sistema corre en tu máquina. El siguiente paso es que corra en cualquier máquina — y que lo puedas entregar a un equipo sin el clásico "en mi máquina funciona". Vas a contenerizar el sistema completo con Docker y definir la infraestructura como código.

---

## Lo que construirás

Una configuración Docker completa para el sistema del E04 o E05 (tú eliges cuál). El sistema debe levantarse con un solo comando desde cualquier máquina que tenga Docker instalado, sin dependencias adicionales.

---

## Prerequisitos

- Docker y Docker Compose instalados en tu máquina
- Sistema del E04 (FastAPI) o E05 (Django) funcionando localmente

---

## Paso 1 — Dockerfile para la API

Escribe un Dockerfile **multi-stage** para la API FastAPI o Django:

- El stage de `build` instala dependencias
- El stage `final` copia solo lo necesario para correr
- Imagen base: `python:3.11-slim`
- La imagen final debe pesar **menos de 300MB**

> No copies datos (`.parquet`, `.db`, `.csv`) dentro de la imagen. Los datos se montan como volumen en runtime.

---

## Paso 2 — docker-compose

Escribe `docker-compose.yml` con dos servicios:

| Servicio | Descripción |
|---|---|
| `api` | El servidor FastAPI o Django en el puerto 8000 |
| `setup` | Corre una sola vez para crear la base SQLite desde el Parquet |

Ambos servicios comparten un volumen para los datos. El servicio `api` debe depender de que `setup` haya terminado exitosamente:

```yaml
depends_on:
  setup:
    condition: service_completed_successfully
```

---

## Paso 3 — Variables de entorno

Mueve todas las rutas y configuraciones hardcodeadas a variables de entorno. Crea un archivo `.env.example` con todas las variables requeridas y sus valores por defecto.

- El `.env` real va en `.gitignore`
- La aplicación debe **fallar con un mensaje claro** si falta una variable requerida — no con un error críptico de Python

---

## Paso 4 — Health check y logs

Agrega un `HEALTHCHECK` en el Dockerfile que llame a `GET /health` cada 30 segundos. Si el endpoint no responde en 5 segundos, el contenedor entra en estado `unhealthy`.

Configura el logging para que salga a stdout en **formato JSON** — un objeto por línea con `timestamp`, `level` y `message`.

---

## Paso 5 — Documentación de operaciones

Escribe un `README.md` operacional con exactamente estos cuatro comandos verificados:

```bash
# Levantar el sistema desde cero
docker compose up --build

# Verificar que está corriendo
docker compose ps
curl http://localhost:8000/health

# Ver los logs en tiempo real
docker compose logs -f api

# Parar y limpiar todo
docker compose down -v
```

> Todos los comandos del README deben ser ejecutables tal como están escritos — sin modificaciones. Lo verificamos en una máquina limpia.

---

## Entregables

```
ejercicio-07-contenedores/
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .dockerignore
└── README.md
```

---

## Cómo se evaluará

| Criterio | Peso | Lo que revisamos |
|---|---|---|
| Imagen funcional y liviana | 25% | La imagen construye sin errores, corre correctamente, pesa menos de 300MB |
| docker-compose completo | 30% | Un solo `docker compose up --build` levanta todo desde cero — lo verificamos en una máquina limpia |
| Variables de entorno | 20% | `.env.example` completo, nada hardcodeado, error claro si falta variable |
| README operacional | 25% | Todos los comandos documentados corren exactamente como están escritos |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 7 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```