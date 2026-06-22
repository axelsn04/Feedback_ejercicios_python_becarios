# Retroalimentación — Ejercicio 06: El Pipeline de Datos

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 86 / 100

---

## Resumen general

El pipeline tiene la separación de capas más limpia que se puede pedir: `extract` solo normaliza, `transform` solo valida, `load` solo inserta — ninguna capa hace el trabajo de otra. Los tests cubren cada tipo de error individualmente y la idempotencia está demostrada correctamente, con la salvedad honesta de que `uuid4()` no es determinista incluso con seed fijo. El gap real está en las reglas de negocio: `status` nunca se valida y `user_id`/`merchant_id` solo verifican null, sin rango. Además, el reporte ofrece una explicación sobre la distribución de errores que no coincide con lo que hace `data_source.py`.

---

## Criterio 1 — Capas separadas y cohesivas `24 / 25`

La separación es ejemplar. `extract()` normaliza formato (timestamp a ISO, country_code a mayúsculas, amount redondeado, strip de whitespace) y **nunca rechaza una fila** — esa responsabilidad es exclusiva de `transform()`. El propio reporte lo explica con precisión: "la decisión de rechazar pertenece a transform... esta separación ayuda a que se puedan cambiar las reglas de validación sin tocar la normalización, o al revés." Esa es exactamente la razón de separar capas en un pipeline ETL.

`load()` usa `total_before`/`total_after` con `COUNT(*)` para derivar `inserted` y `duplicates` sin necesidad de inspeccionar cada fila individualmente — una forma simple y correcta de contar el efecto neto de `INSERT OR IGNORE`.

Un punto menos: `data_source.py` no valida que `--batch-size` esté en el rango 100-1,000 que pide el enunciado — el argparse acepta cualquier entero sin límites.

---

## Criterio 2 — Validación y cuarentena `22 / 30`

Las 5 reglas que sí están implementadas son correctas: UUID4 con verificación de round-trip (`str(parsed) != str(tid).lower()`), rango de `amount`, whitelist de `category`, whitelist de `country_code`, y timestamp futuro con margen de 1 hora.

**Gap 1 — `status` nunca se valida.** El schema fijo del módulo define `status` como `completed`/`failed`/`pending`. `_validate_row()` no tiene ninguna regla para este campo — una fila con `status: "cancelled"` o `status: null` pasaría sin ser rechazada. Es el único campo del schema sin cobertura.

**Gap 2 — `user_id` y `merchant_id` solo verifican null, no rango.** El schema define `user_id` entre 1-50,000 y `merchant_id` entre 1-10,000. El código solo hace `if row.get('user_id') is None: reasons.append(...)` — un `user_id = -5` o `user_id = 99999999` pasaría como válido.

**El reporte tiene una explicación que no coincide con el código.** Sobre por qué category y UUID parecen los errores más frecuentes, el reporte dice: *"un `null_field` a veces cae en un campo sin validación estricta (como `status`), y entonces no se rechaza."* Pero revisando `data_source.py`:

```python
field = random.choice(['user_id', 'merchant_id', 'amount', 'category'])
```

`status` **nunca** es elegido por `null_field` — no está en la lista. Esa hipótesis del reporte no está respaldada por el código que el propio alumno escribió. Además, la diferencia entre 17 (categoría inválida) y 12 (amount/timestamp) sobre ~68 rechazos totales con 5 tipos de error elegidos uniformemente es estadísticamente consistente con varianza aleatoria simple — no hay una asimetría real que requiera explicación causal. El análisis construye una narrativa técnica plausible sobre un patrón que probablemente es ruido estadístico, y la justifica con un mecanismo que no existe en el código.

---

## Criterio 3 — Idempotencia verificada `19 / 20`

`test_idempotency` es el test correcto: inserta la misma fila dos veces y verifica `COUNT(*) == 1`, no solo que el segundo insert reporte 0. Eso confirma el estado final de la base, no solo el valor de retorno de la función.

La observación más valiosa de la sección de idempotencia es la honestidad sobre la limitación real: con `--seed 42`, `data_source.py` reproduce las mismas filas estadísticamente (mismos montos, mismas categorías, mismos tipos de error) pero **no los mismos `transaction_id`**, porque `uuid.uuid4()` usa el generador del SO y no respeta `random.seed()`. Eso significa que correr `pipeline.py` dos veces con el mismo seed no demuestra idempotencia end-to-end — solo los tests la demuestran, porque generan la fila explícitamente con un UUID fijo. Reconocer esa brecha entre "lo que demuestran los tests" y "lo que demuestra el CLI real" es la señal de que el alumno entendió el problema completo, no solo lo hizo pasar.

---

## Criterio 4 — Reporte de ejecución `21 / 25`

El JSON tiene exactamente la estructura pedida con ambas invariantes verificadas en código (`assert`, no solo en el reporte). El desglose `rejected_by_type` con conteos por tipo de error es claro y el README documenta el schema del JSON completo.

La sección "Distribución de errores detectados" es donde se pierden puntos — no por el dato en sí (correcto: 68 de 500, 13.6%), sino por la interpretación causal incorrecta del criterio anterior. Un reporte de pipeline de datos vive o muere por la confiabilidad de su análisis; una explicación que no se valida contra el código propio es el tipo de error que en un pipeline de producción se propaga sin que nadie lo detecte.

---

## Sobre el uso de herramientas de IA

El registro de tiempos (8h, con desglose por fase) y la honestidad sobre la limitación de UUID4 vs idempotencia son señales de trabajo genuino. La explicación incorrecta sobre `status` y `null_field` es más consistente con una narrativa generada para sonar técnica que con una verificación línea por línea del propio `data_source.py` — vale la pena revisar ese hábito antes de E07 y E08, donde los reportes técnicos pesan más.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 7:

> Agrega la regla de validación de `status` a `transform.py` (whitelist de `completed`/`failed`/`pending`) y rangos explícitos para `user_id` (1-50,000) y `merchant_id` (1-10,000). Corre el pipeline de nuevo con el mismo `--seed 42` y compara la nueva distribución de `rejected_by_type` contra la que reportaste. ¿Cambia el total de filas rechazadas? ¿Por qué sí o por qué no, dado que `data_source.py` no genera errores de `status` ni de rango en `user_id`/`merchant_id`?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Capas separadas y cohesivas | 25% | 24 / 25 |
| Validación y cuarentena | 30% | 22 / 30 |
| Idempotencia verificada | 20% | 19 / 20 |
| Reporte de ejecución | 25% | 21 / 25 |
| **Total** | **100%** | **86 / 100** |

---

La arquitectura del pipeline es sólida y la honestidad sobre la limitación de UUID4 demuestra comprensión real del problema. El gap de validación (`status`, rangos) y la explicación no verificada contra el código son los dos puntos concretos a corregir — ambos son rápidos de arreglar y la pregunta de seguimiento te lleva directo a la corrección.