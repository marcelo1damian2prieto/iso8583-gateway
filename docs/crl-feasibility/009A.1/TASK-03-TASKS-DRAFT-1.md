# TASK-03 — Desglose de tareas

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-TASKS-DRAFT-1`  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`

| ID | Prioridad | Entrega | Descripción | Dependencias | Criterio de salida |
|---|---|---|---|---|---|
| T03-00 | P0 | Baseline | Inspección read-only actual de workspace 009A.1, POM, archivos, hashes, secretos excluidos y comparación con TASK-01 | — | Inventario/hashes; sin cambios ni comandos de build/test |
| T03-01 | P0 | Domain/Schema | Congelar envelope de evento, campos, tipos, canonicalización, seq, eventId, transitionKey, digest y checkpoint schema | T03-00 | Esquema versionado documentado y matriz event-type/recovery aprobada |
| T03-02 | P0 | Codec | Codec determinista de journal/checkpoint; validaciones de schema y encoding | T03-01 | Round-trip y rechazo de encoding ambiguo/inválido |
| T03-03 | P0 | Lock | Exclusión de proceso a nivel OS sobre journal/store | T03-00,T03-01 | Un solo Coordinator obtiene lock; contender queda cerrado |
| T03-04 | P0 | Journal writer | Append completo, secuencia/hash-chain y `force(true)`; errores parciales sin ACK | T03-02,T03-03 | Writes completos y fallos probados; no truncamiento automático |
| T03-05 | P0 | Checkpoint writer | Archivo temporal, write-all, `force(true)`, close, `ATOMIC_MOVE`, directory force si soportado; sin fallback | T03-02,T03-03 | Reemplazo atómico probado; unsupported/failure fail-closed; limitación documentada |
| T03-06 | P0 | State reducer | Aplicación determinista de eventos y construcción del snapshot del Coordinator | T03-01,T03-02 | Mismo journal produce mismo estado/snapshot |
| T03-07 | P0 | Replay | Cargar checkpoint/journal, validar seq/hash/refs/snapshot, reconstruir epoch, membership y estado | T03-04,T03-05,T03-06 | Replay válido abre solo tras completar; corrupción/contradicción no abre |
| T03-08 | P0 | Tail recovery | Reconciliar ADMIT no checkpointed y COMPLETE íntegro según R3.1; recoveryKey estable | T03-07 | Recovery event único, idempotente tras crash repetido; tipos desconocidos permanecen cerrados |
| T03-09 | P0 | Coordinator serialization | Orden total ADMIT/FENCE abarcando validación, append, force, checkpoint, publish y ACK | T03-04,T03-05,T03-06 | Barreras concurrentes prueban que no hay ADMIT viejo confirmado tras FENCE |
| T03-10 | P1 | Idempotencia | Index/replay de transitionKey, payload digest, eventId y recoveryKey | T03-06,T03-07,T03-08 | Same-key/same-payload retorna mismo outcome; conflicto rechaza |
| T03-11 | P0 | Crash matrix | Fault hooks de append/force/temp/force/close/rename/dir force/ACK y process restart | T03-04..T03-10 | Cada frontera tiene outcome esperado y evidencia; no se prueba pérdida eléctrica |
| T03-12 | P0 | Evidence/closure | Informe final, comandos, tests, hashes, riesgos abiertos y verificación de exclusiones | T03-11 | Cierre técnico TASK-03 revisable; sin TASK-04 ni producción |

## Exclusiones de implementación de TASK-03

No crear/modificar Member ledger, `DistributedTlsInstance009A` salvo lectura, wire protocol final, HTTP gate, handler wrapper, WebClient, certificados/CRL, Maven versions ni POM salvo que una dependencia imprescindible sea propuesta y aprobada aparte. No modificar README/AGENTS/SPEC/PLAN/TASKS/DOC-001..009 del ISO8583 Gateway.
