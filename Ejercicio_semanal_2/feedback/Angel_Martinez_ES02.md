# Retroalimentación — Proyecto Integrador II, Semana 10
## Red de Logística Distribuida

**Alumno:** Angel Martinez  
**Fecha de revisión:** Julio 2026  
**Calificación:** 59 / 100

---

## Calificación final: 59 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Transferencia atómica y modelo de consistencia | 30% | **20 / 30** | Bueno |
| Reconciliación offline | 25% | **17 / 25** | Bueno |
| Sistema completo (API, analítica, Docker) | 25% | **10 / 25** | Regular |
| decisions.md | 20% | **12 / 20** | Regular |
| **Total** | 100% | **59 / 100** | |

---

## 1. Transferencia atómica y modelo de consistencia — 20/30

**Lo que funciona bien:**
- El uso de PostgreSQL con `SELECT ... FOR UPDATE` para el bloqueo de fila es una elección sólida y más realista que un motor embebido para el estado canónico central — corrí `test_atomic_transfer_rollback_on_failure` y confirma que, tras una falla simulada, el paquete queda exactamente en su nodo y estado original.
- El modelo de consistencia del endpoint de seguimiento (`/packages/{id}/estado`) es genuinamente sofisticado: lectura monotónica real vía el encabezado `X-Last-Seen-Timestamp` (si el caché es más viejo que esa marca, se salta a PostgreSQL), `freshness_seconds` calculado de verdad a partir de `committed_at` (no un valor fijo), e invalidación write-through de Redis tras cada transferencia. Esto coincide exactamente con lo que describe `decisions.md`.

**Dos problemas concretos que bajan la nota:**

1. **La transferencia nunca se cierra en el sistema en vivo.** `execute_atomic_transfer` pone `current_node = nodo_destino` y `last_type = 'en_transito'` en la misma operación — no existe ningún endpoint de "confirmar llegada" que después marque el paquete como `recibido` o `entregado`. Busqué en todo el código dónde se asignan esos dos valores a `last_type` y solo aparecen en `pipeline/local_offline_node.py` (un script de simulación de eventos offline), nunca en la API en vivo. Esto significa que, para un paquete que viaja entre nodos que están en línea todo el tiempo, no hay forma de registrar su llegada real — se queda en `en_transito` indefinidamente desde la perspectiva del sistema, y el propio comentario en el código ("se escaneará como 'recibido' en el siguiente paso") describe una funcionalidad que no llegó a implementarse.
2. **El parámetro `simulate_fail` de las pruebas quedó expuesto en la API pública.** `TransferRequest.simulate_fail` es un campo real del cuerpo de `POST /transfers` (`app/schemas.py`), no un mecanismo interno de pruebas. Cualquier cliente de la API puede mandar `"simulate_fail": true` y forzar una falla 500 a propósito. Es un mecanismo de testing que se filtró al contrato público en lugar de quedar aislado en la suite de pruebas.

---

## 2. Reconciliación offline — 17/25

**Lo que funciona bien:**
- Las cuatro categorías de conflicto documentadas tienen una política concreta y razonada, no genérica, y una de ellas cita un paquete real del dataset (`PKG-6A5DD0F0553E`) como evidencia del caso, no una afirmación abstracta.
- El bulk-fetch de paquetes activos en lotes de 10,000 antes de procesar el archivo evento por evento es una buena práctica de rendimiento que no todos los proyectos de este nivel consideran.

**Lo que baja la nota — una inconsistencia verificable entre el documento y el código:**
Para el conflicto `nodo_dice_entregado_sistema_dice_devuelto`, `decisions.md` dice: *"se actualiza el estado central (Autoridad del Nodo)"* — es decir, que la entrega física reportada por el nodo debería sobrescribir el estado de "devuelto" del sistema. Pero en `pipeline/reconcile.py`, esa rama (`node_delivered_system_returned`) solo hace `summary["rejected"] += 1` y agrega el caso a los detalles para revisión manual — **nunca toca el objeto `package`**. El estado central se queda como estaba, exactamente lo contrario de lo que dice el documento. No es casualidad que esta sea también la única de las cuatro categorías que no tiene un test dedicado en `test_reconciliation.py` (que cubre las otras tres) — escribir ese test habría hecho evidente la discrepancia.

---

## 3. Sistema completo (API, analítica, Docker) — 10/25

**Lo que funciona bien:**
- Los endpoints de negocio están completos y razonablemente organizados: salud, estado de paquete, transferencia, reconciliación, reporte operacional.
- Los healthchecks de PostgreSQL y Redis en `docker-compose.yml` están bien configurados (`pg_isready`, `redis-cli ping`) con reintentos.

**El problema más serio de esta revisión:** `docker-compose.yml` monta, tanto en `setup` como en `api`, una ruta absoluta fija del disco del autor:

```yaml
volumes:
  - /home/angel/PythonTich/guide/ProyectoIntegradorLogistica:/source_data:ro
```

en lugar del patrón `${DATA_DIR:-./data}` que sí usan otros proyectos de este mismo curso. Esto significa que `docker compose up --build` — el comando exacto que el README indica ejecutar — **no puede funcionar en ninguna máquina que no sea la del autor**, porque esa carpeta no existe en ningún otro lugar. No es un paso manual adicional documentado (como en otros proyectos que piden correr un script antes); es una ruta que hay que saber que existe y editar antes de que el sistema arranque en absoluto, y no está mencionado en ningún lugar del README.

Además, como se explicó en el punto 1, no hay forma en la API en vivo de marcar un paquete como recibido o entregado — una funcionalidad de negocio central (confirmar la llegada) que sólo existe en el pipeline de simulación offline, no en el sistema que un operador en un nodo conectado usaría.

---

## 4. decisions.md — 12/20

El documento está bien organizado, con tablas claras por tema, y en general describe con precisión lo que el código hace — el modelo de consistencia del punto 4 (lectura monotónica, invalidación write-through) es un ejemplo de documentación que sí coincide con la implementación real, y se agradece.

**Lo que baja la nota:** dos afirmaciones concretas no se sostienen al verificarlas contra el código:
- La política de "el nodo gana, se actualiza el estado central" para el conflicto de entrega-vs-devolución (ya explicado en el punto 2) no ocurre en el código.
- La descripción del estado "en tránsito" implica un ciclo que se cierra ("no ha sido recibido en la de destino", dando a entender que después sí lo será) que, como se explica en el punto 1, no tiene ninguna vía de cierre en el sistema en vivo.

Ninguna de las dos es un error de escritura menor — son las dos afirmaciones más centrales de sus respectivas secciones.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- La elección de PostgreSQL + Redis + DuckDB (en vez del par DuckDB/SQLite más común en este curso) y el mecanismo de lectura monotónica muestran que sí se entendió el problema de consistencia distribuida a un nivel más profundo que "cachear con un TTL" — ese diseño específico no es algo que se copie sin pensarlo.
- Al mismo tiempo, dos de los hallazgos más importantes de esta revisión (el estado que nunca se cierra, la política documentada que no coincide con el código) son exactamente el tipo de brecha que aparece cuando se escribe la documentación describiendo la intención del diseño antes de terminar o probar esa parte del código. Vale la pena, antes de la próxima entrega, revisar `decisions.md` una última vez línea por línea contra el código ya terminado, no contra el plan original.
- La ruta absoluta en `docker-compose.yml` es fácil de perder de vista porque el sistema sí funciona en la propia máquina — por eso vale la pena, como última prueba antes de entregar, clonar el propio repositorio en una carpeta limpia y correr `docker compose up --build` ahí, exactamente como lo haría alguien más.