# Retroalimentación — Ejercicio 05: El Backend con Estructura

**Alumno:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 60 / 100

---

## Resumen general

La estructura de Django está en su lugar — migraciones, admin, DRF, autenticación
por token — y los 6 endpoints responden. Los cuatro gaps que bajan la calificación
son concretos y corregibles: DuckDB abre una conexión nueva por request (no hay
singleton), la deduplicación del batch está rota porque `transaction_id` tiene
`editable=False` en el modelo y DRF lo ignora en el input, el serializer solo
valida `amount > 0` sin whitelist de `category`/`country_code`/`status`, y el
`load_transactions` carga todo el Parquet a memoria antes de procesarlo.

---

## Criterio 1 — Modelos, migraciones e índices `14 / 25`

**`user_id = CharField(max_length=50)` y `merchant_id = CharField(max_length=50)`
— tipos incorrectos.** El schema del módulo define ambos como enteros. El URL
pattern `<str:user_id>` es consistente con el CharField, pero esto propaga el
problema: el dataset del E1 almacena `user_id` como entero, la base del E5 lo
almacena como string, y cualquier join o comparación cruzada entre ejercicios
falla silenciosamente.

**`amount = DecimalField(decimal_places=2, max_digits=10)`** — la elección más
correcta del grupo para valores monetarios. `FloatField` acumula error de
representación; `DecimalField` es exacto para operaciones financieras.

**Nombres de índices auto-generados:**
```
transaction_user_id_ecc61d_idx      ← Django hash
transaction_country_b023fb023_idx   ← Django hash
```
Sin `name=` explícito en `models.Index(...)`, Django genera el nombre. La
rúbrica pide replicar los índices del E3 (`idx_user_timestamp`,
`idx_country_user`) — el criterio no se cumple si los nombres no coinciden.

**Sin `db_table` en Meta.** La tabla se llama `transactions_transaction`. La
base del E3 usa `transactions`. Cualquier query que cruce ambas bases falla.

**`load_transactions` carga todo el Parquet a memoria antes de chunkear:**
```python
df = pd.read_parquet(file_path)          # 1M filas completas en RAM
for i in range(0, len(df), chunk_size):
    chunk = df.iloc[i:i+chunk_size]       # slice del DataFrame en memoria
    for _, row in chunk.iterrows():       # iterrows — muy lento
        transactions.append(Transaction(...))
```

El objetivo de chunkear es no cargar todo en memoria — aquí se carga todo
primero y luego se procesa en pedazos. `iterrows()` sobre 1M filas es
notoriamente lento porque crea un objeto `Series` por cada fila. La alternativa
correcta es `df.to_dict('records')` que convierte el DataFrame completo a lista
de dicts en una sola operación vectorizada, o mejor, leer el Parquet con
`pyarrow.parquet.iter_batches()` que sí es streaming real.

La ruta del Parquet está hardcodeada (`'../data/benchmark_1m.parquet'`) sin
argumento CLI ni variable de entorno — el comando falla si se corre desde
cualquier directorio que no sea el esperado.

`bulk_create(ignore_conflicts=True)` ✓ — al menos la idempotencia está
garantizada por la PK.

---

## Criterio 2 — 6 endpoints funcionales con DRF `17 / 30`

**DuckDB por request — sin singleton.** Cada llamada a `/analytics/*` ejecuta:
```python
result = duckdb.query(query).fetchone()
```
`duckdb.query()` es la función de módulo que abre una conexión `:memory:`
temporal y la cierra al terminar. El overhead de abrir esa conexión más
registrar la vista sobre el Parquet — medido en ~88ms en el E4 — se paga en
cada request. El SLA caliente de 20ms es imposible de alcanzar.

**Deduplicación del batch rota por `editable=False` en el modelo:**

El modelo declara:
```python
transaction_id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
```

Cuando `editable=False`, DRF's `ModelSerializer` convierte el campo en
`read_only=True` automáticamente. Los campos `read_only` se excluyen del
`validated_data` — el `transaction_id` que viene en el request se ignora
silenciosamente. En la view:

```python
transactions = [Transaction(**item) for item in serializer.validated_data]
Transaction.objects.bulk_create(transactions, ignore_conflicts=True)
```

Como `item` no tiene `transaction_id`, Django ejecuta el `default=uuid.uuid4`
y genera un UUID nuevo para cada transacción. Resultado: `bulk_create` nunca
encuentra un conflicto de PK, `ignore_conflicts=True` nunca actúa, y el mismo
payload enviado 10 veces inserta 10 filas con UUIDs distintos. La deduplicación
es estructuralmente imposible con este modelo y serializer.

La corrección es remover `editable=False` y agregar la validación de UUID4 en el
serializer:
```python
transaction_id = models.UUIDField(primary_key=True)
# En el serializer:
transaction_id = serializers.UUIDField()
```

**Serializer valida solo `amount` — sin whitelist de dominio:**
```python
def validate_amount(self, value):
    if value <= 0:
        raise serializers.ValidationError("Amount must be positive")
    return value
```
No hay validación de `category` (10 valores), `country_code` (15 países), ni
`status` (3 valores). Una transacción con `category="Gambling"` o
`country_code="ZZ"` pasa sin error. El `test_batch_invalid_category` que tienen
Antonio y el alumno anónimo no está en la suite de Yaanit — si estuviera,
fallaría.

**`TransactionBatchSerializer` está definido pero nunca se usa.** La view
`ingest_batch` usa `TransactionSerializer`, no `TransactionBatchSerializer`. El
segundo serializer es código muerto.

**`GET /users/{id}/transactions` — sin 404.** Si `user_id` no tiene
transacciones, retorna `{"user_id": id, "page": 1, "transactions": []}` con
200. La URL usa `<str:user_id>` — correcto para el CharField del modelo.

**`GET /users/{id}/stats` — retorna 404 correctamente** pero sin `country_code`
en el response. El E4 requería este campo.

**`POST /transactions/batch` retorna 200 en lugar de 201.** En REST, una
inserción exitosa usa 201 Created.

**`duckdb.query(query, params)` en `top_merchants` — la API de parámetros es
dudosa.** `duckdb.query()` no soporta parámetros posicionales de la misma forma
que `connection.execute(sql, params)`. El filtro por `country` puede no
aplicarse, retornando todos los merchants sin filtrar.

---

## Criterio 3 — Django Admin `15 / 20`

Los tres elementos requeridos por la rúbrica están presentes:
- `list_filter = ('status', 'country_code')` ✓
- `search_fields = ('transaction_id', 'user_id')` ✓
- `list_display` con columnas ✓

Gaps: `timestamp` y `category` no están en `list_display` — las dos columnas
más útiles para triage operacional. Sin `list_per_page`, Django intenta cargar
100 filas por página sobre 1M registros. Hay una importación duplicada de
`django.contrib.admin` que se dejó sin limpiar.

---

## Criterio 4 — Tests `14 / 25`

7 tests. Cubren el mínimo (6) pero varios pasan por razones incorrectas.

**`test_batch_valid` con `transaction_id: "tx-test-999"` pasa porque el campo
es ignorado, no porque sea válido.** `editable=False` hace que DRF descarte
`transaction_id` del input. Un UUID inválido y un UUID válido producen
exactamente el mismo resultado — el serializer genera uno nuevo en ambos casos.
Este test no valida deduplicación ni formato de UUID.

**`test_batch_invalid` con `amount=-100 → 422` es el único test correcto del
batch.** La validación funciona porque `validate_amount` existe en el
serializer. Pero no hay tests para `category` inválida, `country_code` inválido
ni `status` inválido — y no podrían pasar porque el serializer no los valida.

**`test_analytics_summary` fallará sin Parquet.** No hay skip logic. En CI sin
el archivo, la suite se rompe con `FileNotFoundError` en lugar de un skip
claro.

**`test_user_transactions` pasa porque el endpoint no retorna 404.** El test
acepta `status == 200` para un usuario sin datos — correcto respecto al
comportamiento actual pero incorrecto respecto al spec. Si se corrige el
endpoint para retornar 404, el test falla aunque el comportamiento mejore.

Sin tests de: deduplicación (que en realidad no funciona), top-merchants,
paginación, 401 en `/users/{id}/stats`, usuario inexistente en stats.

---

## Sobre el uso de herramientas de IA

La estructura de Django está en su lugar — settings, migraciones, admin, DRF —
y los endpoints responden. El gap más revelador es el `editable=False`: es el
default del scaffold de Django para PKs autogeneradas, y dejarlo sin revisar
rompe silenciosamente la función central del batch endpoint. El E4 resolvió la
deduplicación explícitamente; conectar esa decisión con cómo DRF trata los
campos `read_only` era el paso que faltó aquí.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 6:

> Cambia `transaction_id` de `UUIDField(editable=False)` a `UUIDField()` y
> agrega validación de UUID4 en el serializer. Luego escribe un test que envíe
> el mismo `transaction_id` en dos requests separados y verifique que el segundo
> retorna `inserted=0`. ¿Qué pasa si envías el mismo `transaction_id` dos veces
> en un solo batch? ¿El resultado es correcto?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Modelos, migraciones e índices | 25% | 14 / 25 |
| 6 endpoints funcionales con DRF | 30% | 17 / 30 |
| Django Admin configurado | 20% | 15 / 20 |
| Tests (mínimo 6) | 25% | 14 / 25 |
| **Total** | **100%** | **60 / 100** |

---

Los cuatro cambios concretos antes de E06, en orden de impacto:

1. **Remover `editable=False` de `transaction_id`** — desbloquea la
   deduplicación completa del batch.
2. **Singleton de DuckDB con `threading.Lock`** — el SLA caliente de 20ms
   requiere reutilizar la conexión.
3. **Nombres de índices explícitos** — `name="idx_user_timestamp"` replica el E3.
4. **Whitelist de `category`, `country_code`, `status` en el serializer** —
   actualmente ninguna de esas reglas de negocio se aplica.