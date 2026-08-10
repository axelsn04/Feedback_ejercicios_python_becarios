# Retroalimentación — Proyecto Integrador II, Semana 10
## Red de Logística Distribuida

**Alumno:** Ernesto Cuapantecatl  
**Fecha de revisión:** Julio 2026  
**Calificación:** 98 / 100

---

## Calificación final: 98 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Transferencia atómica y modelo de consistencia | 30% | **29 / 30** | Excelente |
| Reconciliación offline | 25% | **25 / 25** | Excelente |
| Sistema completo (API, analítica, Docker) | 25% | **24 / 25** | Excelente |
| decisions.md | 20% | **20 / 20** | Excelente |
| **Total** | 100% | **98 / 100** | |

---

## 1. Transferencia atómica y modelo de consistencia — 29/30

- El estado "en tránsito" no es una etiqueta — es una restricción real de la base de datos: `CHECK (state = 'en_transito' AND current_node IS NULL) OR (state <> 'en_transito' AND current_node IS NOT NULL)`, más un índice único parcial que impide que un paquete esté en dos camiones a la vez. Esos constraints encontraron dos bugs reales durante el desarrollo (la reconciliación intentaba escribir un estado inconsistente), lo cual obligó a que la reconciliación pase por las mismas operaciones atómicas (`depart`/`arrive`) que el tráfico en vivo — una sola definición de qué significa mover un paquete, no dos que puedan desincronizarse.
- `test_crash_between_debit_and_credit_leaves_no_inconsistency` es el test que la rúbrica pide, y hay evidencia de que no es decorativo: se cambió deliberadamente `ROLLBACK` por `COMMIT` en el código para confirmar que el test *fallaba* como debía, y se restauró después. Además hay un segundo test (`test_crash_during_arrival_leaves_package_in_flight`) que cubre la falla en la segunda fase, no solo la primera — cubre ambas transiciones del modelo de dos fases.
- La decisión de no cachear el endpoint del cliente está respaldada con una medición propia (p99 de 0.237ms sin carga, 1.32ms end-to-end vía HTTP — 151 veces por debajo del presupuesto de 200ms) en vez de asumir que hacía falta. Es exactamente el tipo de "medir antes de suponer" que separa una decisión de una intuición.

Corrí la suite de transferencias completa: 11/11 pasan, incluyendo ambos escenarios de falla a mitad de transacción.

**Lo único que dejaría perfecto:** el benchmark de motor (SQLite vs. la opción de Postgres) se corrió dos veces en la misma máquina; el propio documento lo reconoce con honestidad ("el throughput varía ~13% entre corridas"), pero una tercera corrida, o correrlo en una máquina distinta, cerraría del todo esa variabilidad reconocida.

---

## 2. Reconciliación offline — 25/25

Este es, de los proyectos que he revisado para este ejercicio, el único que resuelve correctamente el punto más importante de todo el criterio:

- **El detector de conflictos no lee la columna `conflicto_tipo` del archivo para decidir nada** — la usa solo al final, para verificar que su propia clasificación coincide con lo que el generador del dataset quiso simular. Esa etiqueta se usa como evidencia de validación, no como insumo. La clasificación real (`classify()` en `pipeline/reconcile.py`) compara cada evento contra el estado canónico que efectivamente está en la base de datos.
- Ese cuidado no fue en vano: el propio ejercicio de verificación encontró que **ninguno de los 70 paquetes etiquetados "el sistema dice devuelto" está realmente en estado `devuelto`** en la base — la etiqueta describe la intención del generador, no la realidad. Un reconciliador que hubiera confiado en la columna habría resuelto conflictos imaginarios mientras ignoraba los reales.
- La política no es una sola regla aplicada uniformemente — usa idempotencia, cuarentena, autoridad del nodo, y revisión manual, cada una para una clase de conflicto distinta, con el criterio de decisión explícito: la asimetría de costo entre un falso positivo (una persona revisando cinco minutos) y un falso negativo (un reclamo financiero falso que cierra el caso y nadie vuelve a buscar el paquete).
- Se encontró y corrigió un bug real de idempotencia en la *cola de revisión humana* (no en el estado del paquete, que ya era idempotente): reconciliar el mismo lote dos veces duplicaba casos porque el mismo evento se clasificaba distinto en la segunda pasada. La solución (un índice único sobre la identidad del evento, no del tipo de conflicto) está razonada, no es un parche.
- El caso real del dataset (`PKG-65E9A6D479FC`) se explica desde las tres perspectivas (operador, sistema, cliente), y se argumenta honestamente por qué "no sabemos" es la respuesta correcta y no una limitación a esconder.

No tengo ninguna crítica sustancial para este criterio.

---

## 3. Sistema completo (API, analítica, Docker) — 24/25

- `docker compose up --build` levanta todo con un solo servicio: `entrypoint.sh` valida el entorno, comprueba que los cuatro archivos de datos existan (con mensajes de error específicos si falta alguno), carga el estado canónico solo si no existe ya (idempotente), reconcilia el lote offline, y arranca la API — sin rutas absolutas del disco de nadie, todo relativo a `./data`.
- Los 12 endpoints cubren exactamente las capacidades de la tabla de garantías de consistencia de `decisions.md` — no hay discrepancia entre lo documentado y lo implementado.
- Corrí la suite completa: 39 tests pasan, y los que dependen del dataset real (`test_e2e.py`) se saltan limpiamente en este entorno por falta de los archivos de origen, sin fallar de forma confusa.

**Lo único que dejaría perfecto:** `uvicorn` corre con `--workers 1`. Es una limitación reconocida y discutida en la sección de escalabilidad (no un descuido), pero para un sistema que se presenta como listo para producción a esta escala, valdría la pena que el propio `docker-compose.yml` dejara ese único worker como una variable de entorno fácil de subir, en vez de un valor fijo en el `entrypoint.sh`.

---

## 4. decisions.md — 20/20

Este documento cumple, en cada sección, el estándar más alto que la rúbrica describe:

- Cada decisión trae su cifra, y cuando la cifra tiene ruido (el benchmark de motor varía ~13% entre corridas), lo dice explícitamente en vez de presentar una precisión que no tiene — "presentar 572.9 como si fuera reproducible sería mentira" es exactamente la actitud que la rúbrica premia.
- El criterio de reversión aparece de forma consistente ("si el margen hubiera sido de 2x, la conversación sería distinta") — no es solo qué se decidió, sino bajo qué evidencia futura la decisión cambiaría.
- La investigación de la bandera `sla_vencido` del propio dataset (comparando cuatro variables candidatas, todas estadísticamente idénticas entre grupos, concluyendo que probablemente es aleatoria) es el tipo de escepticismo hacia los datos de entrada que ninguna otra entrega mostró para este ejercicio.
- La sección de monitoreo prioriza explícitamente una señal de *correctitud* (invariantes rotos) por encima de las señales de disponibilidad, con la justificación de que "un sistema que responde en 5ms con datos inconsistentes es peor que uno caído".
- El documento hace lo que muy pocos hacen: entra en detalle procesable para su propia rúbrica ("el pdf pide dos capacidades que parecen una sola pero no lo son") sin sonar como que está tratando de adivinar qué quiere leer el evaluador.

No tengo ninguna crítica sustancial para este documento.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- El historial de 13 commits, junto con detalles como "esto lo encontré con el test end-to-end, los tests unitarios usaban lotes de cuatro eventos y nunca reconciliaban dos veces", describe el proceso real de encontrar un bug, no una narrativa reconstruida después de que todo ya funcionaba.
- La verificación deliberada de que el test de rollback no fuera decorativo (romper el código a propósito para confirmar que el test lo detecta, y luego restaurarlo) es una práctica de ingeniería que se nota cuando se ha hecho de verdad, no cuando se describe de oídas.
- No tengo una pregunta pendiente para una revisión en vivo esta vez — si tuviera que elegir una, sería pedir que caminara por qué decidió no incluir `conflict_type` en el índice de identidad de un caso de conflicto (la sección "La reconciliación es idempotente, y eso costó un índice"), porque es la clase de decisión sutil que vale la pena que explique con sus propias palabras.