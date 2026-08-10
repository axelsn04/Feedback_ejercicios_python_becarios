# Retroalimentación — Proyecto Integrador II, Semana 10
## Red de Logística Distribuida

**Alumno:** Ulises Josue Reyes Martinez  
**Fecha de revisión:** Junio 2026  
**Calificación:** 99 / 100

---

## Calificación final: 99 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Transferencia atómica y modelo de consistencia | 30% | **30 / 30** | Excelente |
| Reconciliación offline | 25% | **25 / 25** | Excelente |
| Sistema completo (API, analítica, Docker) | 25% | **25 / 25** | Excelente |
| decisions.md | 20% | **19 / 20** | Excelente |
| **Total** | 100% | **99 / 100** | |

---

## 1. Transferencia atómica y modelo de consistencia — 30/30

- El estado intermedio no es una convención de aplicación: es un `CHECK` de base de datos (`custodia_coherente`) más un índice único parcial (`WHERE estado='ABIERTA'`) que impiden, a nivel de motor, que un paquete esté en dos lugares o dos camiones a la vez. Recreé tu base con `sql/schema.sql` contra un PostgreSQL 16 real y corrí `sql/test_invariantes.sql`: los 15 tests adversariales se comportan exactamente como se espera, incluyendo los que deben fallar con `ERROR` (`llegada_no_precede_salida`, `progreso_dentro_de_ruta`).
- `tests/test_atomicidad_falla.py` inyecta una excepción real en tres puntos distintos de la transacción (entre transferencia y evento, antes del outbox, a mitad de la llegada) contra PostgreSQL real, no una base simulada. Corrí la suite completa: **24/24 pasan**, incluyendo la verificación de que una falla en el outbox también deshace la transferencia y el evento ya escritos — la prueba de que el outbox vive en la misma transacción atómica y no es "solo logging".
- Encontraste y corregiste dos bugs reales de implementación que solo aparecen al ejecutar el DDL contra un motor real: una `RULE` de solo-lectura que resultó incompatible con `INSERT ... ON CONFLICT` (deshabilitando sin darte cuenta el mecanismo de idempotencia), y una transición de estado inválida (`EN_TRANSITO -> en_transito`) que tu propio índice parcial rechazaba con una excepción sin manejar. Ambos hallazgos están documentados con el error real de PostgreSQL, no descritos de memoria.
- Fuiste más allá del criterio explícito con `tests/test_resiliencia_bd.py`: verificaste que la API responde `503` limpio (no `500` crudo, no colgada) cuando Postgres es inalcanzable, con el tiempo de respuesta medido (<2s). Corrí esta suite también: **12/12 pasan**.

Ejecuté el pipeline completo de verificación (`verificar_todo.sh`) de punta a punta contra un PostgreSQL 16 que instalé para esta revisión: **157 verificaciones individuales pasaron, cero fallaron**, sobre los pasos que no requieren el dataset original. No tengo ninguna crítica para este criterio.

---

## 2. Reconciliación offline — 25/25

- La política diferenciada por severidad (5 categorías, no 3 genéricas) está anclada en evidencia específica: descubriste que el 100% de los 70 casos "el nodo dice entregado, el sistema dice devuelto" traen `metadata.firmado: true`, y que los 58 casos de "en tránsito vs. entregado" **nunca** lo traen — esa correlación exacta es lo que justifica tratarlos con severidad distinta en vez de una regla única para "conflicto terminal".
- Documentas honestamente una limitación real y verificada por diseño: no puedes reconstruir algorítmicamente la categoría `entregado_vs_devuelto` a partir de los tres archivos dados, porque el histórico usa un espacio de identificadores disjunto del snapshot de activos, y el snapshot de "activos" excluye por definición los paquetes ya en estado terminal — lo dejas como pregunta abierta para el generador del dataset en vez de forzar una reconstrucción que los datos no permiten.
- El hallazgo más valioso de este criterio es **ADR-012**: tu primera corrida del motor de reconciliación clasificó los 12,000 eventos en una sola categoría y aplicó cero, incluidos los 70 casos con firma que exigían revisión manual obligatoria. Diagnosticaste la causa raíz (tratar procedencia del nodo y tipo de conflicto como una jerarquía en vez de dos dimensiones ortogonales), la corregiste, y verificaste el resultado exacto tras corregirlo: 287 conflictos de estado (2.392%, coincide con el ground truth), 70 casos a revisión humana (exactamente los que traen firma). Verifiqué la tabla de reglas resultante contra `test_invariantes.sql`: la precedencia queda tal como describes.
- Las reglas de reconciliación viven en una tabla versionada (`reglas_reconciliacion`), no en un `if/elif` — cada conflicto procesado queda trazable a la fila de regla exacta que se le aplicó, y agregar una categoría nueva no exige un deploy.
- Corrí `test_ingesta.py` y `test_reconciliacion.py`: **49/49** y **11/11** respectivamente, incluida la verificación de que reconciliar con 4 workers en paralelo produce exactamente el mismo estado final que con 1 worker secuencial.

No tengo ninguna crítica sustancial para este criterio.

---

## 3. Sistema completo (API, analítica, Docker) — 25/25

- Los endpoints cubren cada capacidad exigida (estado de paquete, ingesta de eventos, reconciliación por HTTP, ambos detectores de riesgo, resumen operacional) sin discrepancia entre lo documentado y lo implementado.
- `docker-compose.yml` está validado en **dos entornos distintos**: tu máquina de desarrollo y una segunda máquina con Windows y Docker Desktop — el único proyecto de esta ronda de revisiones con validación cruzada de entorno real, no solo "funciona en mi máquina". Los tres problemas que encontraste en esa segunda validación (puerto ocupado, Postgres nativo interceptando 5432, un DSN hardcodeado en un test) están documentados con su corrección, y ninguno resultó ser un problema de la lógica de negocio.
- El healthcheck de `db` espera correctamente a que termine de correr `schema.sql` (no solo a que el proceso de Postgres arranque) antes de que `api` se considere lista para levantar — un detalle que varias otras entregas de este mismo ejercicio pasan por alto.

Reproduje tu proceso de verificación completo en un entorno nuevo, uno más donde nunca se había corrido tu código: instalé PostgreSQL 16, cargué `schema.sql`, y corrí `verificar_todo.sh` de punta a punta. Terminó con **exit 0** y cero fallos. No tengo ninguna crítica sustancial para este criterio.

---

## 4. decisions.md — 19/20

- El formato ADR (Contexto/Decisión/Evidencia/Consecuencias) mantiene cada decisión trazable a un número medido, y el documento se corrige a sí mismo cuando la evidencia lo exige — ADR-015 revierte una decisión explícita de ADR-014 con su propia evidencia de que ya no había motivo para no hacerlo bien.
- La sección de limitaciones documenta cinco supuestos con honestidad, incluida la imposibilidad de garantizar deduplicación perfecta sin un `evento_id` real del escáner — no lo escondes, lo mides (`salud_evento_id`) y lo declaras como requisito pendiente, no como detalle menor.

**Lo único que le falta para el máximo:** con 9,814 palabras es, por un margen amplio, el documento más largo de todos los que he revisado para este ejercicio. El contenido lo justifica casi en su totalidad — no hay relleno, casi cada párrafo trae una cifra propia — pero no hay una síntesis de una página al principio para quien solo tiene tiempo de leer lo esencial antes de entrar a los 18 ADRs completos. Es la misma sugerencia que le haría a cualquier bitácora de este nivel de detalle.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- ADR-012 es, de todo lo que he revisado en este ejercicio, el ejemplo más claro de depuración real: no es una narrativa de "funcionó a la primera", es "corrí mi propio motor, clasificó todo mal, encontré por qué, lo corregí, y verifiqué la cifra exacta después". Ese tipo de honestidad sobre los propios errores es exactamente lo que un log de decisiones debería registrar.
- La verificación cruzada en una segunda máquina con un sistema operativo distinto (ADR-016) es un nivel de disciplina de entrega que no vi en ningún otro proyecto de este ejercicio.
- No tengo una pregunta pendiente para una revisión en vivo esta vez. Si tuviera que elegir una, sería que expliques con tus propias palabras la limitación #1 (el espacio de identificadores disjunto entre el histórico y el resto de los archivos) — es la observación más sutil de todo el documento, y la que más deja ver si el hallazgo fue realmente tuyo.