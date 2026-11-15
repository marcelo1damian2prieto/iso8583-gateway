# TASK-03 — Desglose de tareas

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-TASKS-DRAFT-2`  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Base normativa:** `TASK-03-SPEC-DRAFT-2.md` y diseño fuente R3.1.  
**Autorización vigente:** solo preparación y validación del diseño; ninguna tarea de implementación está autorizada todavía.

## 1. Backlog y dependencias

| ID | Prioridad | Entrega | Descripción | Dependencias | Criterio de salida |
|---|---|---|---|---|---|
| T03-00 | P0 | Baseline | Inspección read-only actual del workspace 009A.1, POM, árbol, archivos candidatos, hashes y diferencias frente a TASK-01 | — | Inventario/hashes verificables; sin cambios ni comandos de build/test |
| T03-01 | P0 | Domain/event catalog | Cerrar envelope, tipos de evento, campos requeridos, tabla de tail, recovery permitido, advance de epoch y decisiones D03-OPEN | T03-00 | Matriz event-type/recovery completa y decisiones pendientes explícitamente resueltas o bloqueadas por aprobación humana |
| T03-02 | P0 | Codec | Definir/implementar codec determinista para journal/checkpoint, canonicalización, schema, digest y validación | T03-01 | Round-trip determinista y rechazo de encoding inválido/ambiguo; algoritmo/formatos aprobados |
| T03-03 | P0 | OS lock | Exclusión de proceso sobre almacén local, ownership y cierre seguro del lock | T03-00 | Un Coordinator obtiene lock; segundo proceso queda cerrado; evidencia del filesystem ensayado |
| T03-04 | P0 | Journal writer | Append completo, secuencia/hash-chain, dedupe de identidad y `force(true)` | T03-02, T03-03 | Writes completos y fallos/partial detectados; sin ACK antes de checkpoint; sin truncamiento automático |
| T03-05 | P0 | Checkpoint writer | Temporal mismo filesystem, write-all, `force(true)`, close, `ATOMIC_MOVE`, directory force y clasificación de errores | T03-02, T03-03, T03-04 | No fallback; unsupported distinto de I/O failure; checkpoint validable; evidencia autorizada del entorno |
| T03-06 | P0 | State reducer | Aplicación determinista de eventos checkpointed y construcción del snapshot | T03-01, T03-02 | El mismo conjunto ordenado de eventos produce el mismo estado/snapshot; transiciones inválidas rechazadas |
| T03-07 | P0 | Replay | Validar checkpoint/journal, secuencia, IDs, hashes, referencias y snapshot; recuperar estado cerrado | T03-04, T03-05, T03-06 | Replay reproducible; corrupción/contradicción/tail no clasificado no abre Coordinator |
| T03-08 | P0 | Tail recovery | Aplicar solo ADMIT_ABORTED_UNCOMMITTED y RECOVERED_COMPLETE conforme R3.1; bloquear otros tipos sin regla | T03-07, T03-10 | Recovery ADMIT/COMPLETE idempotente; demás tipos quedan RECOVERY_REQUIRED; evidencia de referencia y hash |
| T03-09 | P0 | Coordinator serialization | Una frontera lógica para validación, orden, append/force/checkpoint, publish y ACK de ADMIT/FENCE | T03-04, T03-05, T03-06, T03-07, T03-10 | Barreras prueban un orden durable único; no ADMIT anterior confirmado después de FENCE |
| T03-10 | P0 | Idempotency index | Índices/replay de transitionKey, payloadDigest, eventId y recoveryKey; conflicto same-key/different-payload | T03-02, T03-06, T03-07 | Retry identical devuelve el mismo outcome/ID; conflicto rechaza; recoveryKey estable tras restart |
| T03-11 | P0 | Crash/verification matrix | Fault hooks de append, journal force, temp write/force/close, atomic move, directory force, ACK, restart, epoch y lock | T03-00..T03-10 | Cada frontera autorizada tiene resultado/evidencia; no se simula ni reclama pérdida eléctrica |
| T03-12 | P0 | Evidence/closure | Informe reproducible, comandos, resultados, hashes, Quality Gate, límites, riesgos, exclusiones y dictamen | T03-11 | Cierre TASK-03 revisable; sin habilitar TASK-04/producción |

**Prioridad P0 de T03-10:** idempotencia no es una mejora opcional. Está en los requisitos funcionales, en el protocolo de retry/ACK perdido y en los criterios de tail recovery; por tanto no puede diferirse como P1.

## 2. Reglas de ejecución y gates

1. T03-00 es exclusivamente read-only. Si el workspace no existe, cambió materialmente o contiene una discrepancia con TASK-01, detener y documentar; no reconstruir ni sustituirlo con el ZIP histórico 009A.
2. T03-01 es una entrega de diseño/decisiones. No congelar codec o política tail hasta resolver D03-OPEN-01..08 o dejar explícitamente un alcance bloqueado y aceptado por revisión humana.
3. T03-02..T03-10 son tareas de implementación, no están autorizadas por el estado actual. Requieren aprobación del diseño y autorización humana de implementación.
4. La prueba real de Windows FileLock, `ATOMIC_MOVE` y directory force no se da por autorizada por leer este plan. Requiere aprobación explícita del experimento, aislamiento de directorio y plan de rollback de fixtures sintéticos.
5. Ningún error de filesystem autoriza fallback no atómico, truncamiento, repair, soft-fail o selección heurística de archivos.
6. Si un tipo de tail no tiene política aprobada, no se reanuda el servicio automáticamente.
7. TASK-03 no implementa Member ledger ni ejecuta `REBIND_UNSTARTED_ADMISSION` sin evidencia de TASK-04.
8. El cierre de TASK-03 no aprueba TASK-04 ni abre el gate de factibilidad CRL para producción.

## 3. Secuencia del plan

La ejecución respeta las dependencias de la tabla; se permite trabajo paralelo solo cuando no cambia ese grafo:

1. T03-00 — baseline read-only.
2. T03-01 — catálogo/decisiones de dominio y eventos.
3. T03-02 y T03-03 — codec y lock, después de cerrar su diseño; pueden desarrollarse en paralelo tras T03-01/T03-00 respectivamente.
4. T03-04 — journal writer.
5. T03-05 — checkpoint writer.
6. T03-06 — reducer.
7. T03-07 — replay.
8. T03-10 — indexación/idempotencia.
9. T03-08 — tail recovery explícito ADMIT/COMPLETE.
10. T03-09 — serialización integral ADMIT/FENCE.
11. T03-11 — crash matrix y verificación autorizada.
12. T03-12 — evidencia y cierre.

El orden de presentación del plan no cambia IDs, prioridades ni dependencias de TASKS.

## 4. Evidencia requerida por salida

- Baseline: ruta, listado de archivos, hash SHA-256, estado Git si existe, POM y diferencias respecto de TASK-01. Excluir valores secretos; registrar solo presencia/ruta protegida cuando corresponda.
- Codec: vectores de prueba deterministas, versiones de schema, digest y hashes de muestras sintéticas.
- Lock: dos procesos locales; quién obtiene lock; contender sin operaciones; release en cierre/terminación; proveedor/filesystem registrado.
- Journal/checkpoint: fixtures antes/después, bytes escritos, secuencia/hash, excepciones, checkpoint observado y ACK correlacionado.
- Reducer/replay: mismo input produce mismo estado; checkpoint/snapshot comparados; diagnósticos fail-closed.
- Idempotencia/recovery: transitionKey, recoveryKey, IDs, payload digests y secuencias; demostrar no duplicación después de restart.
- Serialización: barreras deterministas ADMIT/FENCE y el orden duradero resultante; `sleep` no se usa como sincronización.
- Crash matrix: frontera inyectada, estado durable, respuesta/ACK permitido, resultado del replay y archivos observados.
- Cierre: archivos cambiados, comandos exactos, resultados reales, tests ejecutados/no ejecutados, Quality Gate, riesgos y límites demostrados.

## 5. Exclusiones explícitas

No crear/modificar Member ledger, `DistributedTlsInstance009A` salvo lectura, wire protocol final, HTTP gate/wrapper, WebClient, TLS/CRL, certificados, versiones Maven/POM ni documentos principales del ISO8583 Gateway (`README.md`, `AGENTS.md`, `docs/SPEC.md`, `docs/PLAN.md`, `docs/TASKS.md`, `docs/DESIGN-VALIDATION.md`, arquitectura integral y DOC-001..009), salvo nueva propuesta aprobada separadamente.
