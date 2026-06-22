# Ejercicio 5 — El Backend con Estructura

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

Sabes construir una API desde cero con FastAPI. Ahora vas a reconstruir los mismos endpoints usando Django REST Framework — un framework con más estructura y convenciones. La diferencia entre los dos te va a dar criterio para elegir el correcto en cada proyecto.

---

## Lo que construirás

Una API REST con Django y Django REST Framework que expone los mismos 6 endpoints del E04. Los datos viven en una base gestionada por el ORM de Django — no SQL crudo. Al terminar tendrás también un panel de administración funcional y autenticación por token.

---

## Prerequisitos

- Parquet de 1M filas generado en el Ejercicio 1
- Base SQLite con índices generada en el Ejercicio 3
- API FastAPI funcionando del Ejercicio 4

Si no los tienes disponibles, solicítalos en el canal **#becarios** antes de empezar.

---

## Paso 1 — Configuración del proyecto Django

Crea el proyecto Django con la estructura estándar. Configura la base de datos en `settings.py`. Crea una app `transactions` dentro del proyecto. Instala `djangorestframework` y agrégalo a `INSTALLED_APPS`.

---

## Paso 2 — Modelo de datos y migraciones

Define el modelo `Transaction` en `models.py` usando el ORM de Django. El schema es el mismo del módulo (`transaction_id`, `timestamp`, `user_id`, `merchant_id`, `amount`, `category`, `country_code`, `status`).

Crea y aplica las migraciones. Escribe un management command `load_transactions` que lea el Parquet del E1 e ingeste los datos usando el ORM en chunks.

Los índices del E3 deben replicarse aquí. Usa `Meta.indexes` en el modelo para declarar los mismos índices que diseñaste en el E3 y verifica en la migración generada que aparecen correctamente.

---

## Paso 3 — Serializers y ViewSets

Implementa los 6 endpoints usando DRF:

| Método | Ruta | Backend |
|--------|------|---------|
| GET | `/analytics/summary` | DuckDB sobre el Parquet |
| GET | `/analytics/top-merchants` | DuckDB sobre el Parquet |
| GET | `/users/{user_id}/transactions` | ORM de Django |
| GET | `/users/{user_id}/stats` | ORM de Django |
| POST | `/transactions/batch` | ORM de Django |
| GET | `/health` | Memoria |

Para los endpoints analíticos puedes usar DuckDB directamente sobre el Parquet — DRF no obliga a usar el ORM para todo. Implementa los serializers correspondientes para cada respuesta.

---

## Paso 4 — Autenticación con tokens

Agrega autenticación por token a los endpoints de usuario y escritura usando el `TokenAuthentication` de DRF.

- El endpoint `/health` y los `/analytics/*` quedan **públicos**.
- El `POST /transactions/batch` y los `/users/*` requieren token válido en el header `Authorization: Token <token>`.

---

## Paso 5 — Django Admin

Registra el modelo `Transaction` en `admin.py`. Configura:

- `list_display` con las columnas más útiles
- Filtros por `status` y `country_code`
- Búsqueda por `transaction_id` y `user_id`

---

## Entregables

```
ejercicio-05-django/
├── config/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── transactions/
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── admin.py
│   └── management/
│       └── commands/
│           └── load_transactions.py
├── tests/
│   └── test_api.py
├── requirements.txt
└── README.md
```

El README debe incluir el comando exacto para levantar el servidor, crear el superusuario, cargar los datos y obtener un token.

---

## Cómo se evaluará

| Criterio | Peso | Lo que revisamos |
|---|---|---|
| Modelos, migraciones e índices | 25% | Schema correcto, migraciones limpias, índices del E3 replicados con `Meta.indexes` |
| 6 endpoints funcionales con DRF | 30% | Responden con datos reales, autenticación correcta, códigos HTTP correctos |
| Django Admin configurado | 20% | List display, filtros y búsqueda funcionan — navegable sin errores |
| Tests (mínimo 6) | 25% | Happy path por endpoint, 401 sin token, 422 en batch inválido |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 5 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```