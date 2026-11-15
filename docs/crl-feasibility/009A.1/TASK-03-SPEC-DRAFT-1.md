# TASK-03 — Especificación de persistencia durable del Coordinator

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-SPEC-DRAFT-1`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Autorización vigente:** preparar y validar diseño/plan; implementación no autorizada.

## 1. Propósito

Definir la persistencia durable local del Coordinator del laboratorio CRL-FEASIBILITY-009A.1 para que su estado pueda reconstruirse de forma determinista después de reinicios y fallos de proceso, conservando idempotencia, integridad, orden de transiciones y comportamiento fail-closed.

La tarea cubre el journal append-only, el checkpoint, el replay, la reconciliación tipada de eventos posteriores al checkpoint, la exclusión de escritores concurrentes y la serialización lógica de transiciones del Coordinator. No busca convertir el laboratorio en un servicio distribuido ni demostrar durabilidad ante pérdida de energía o fallas físicas del almacenamiento.

## 2. Fuentes normativas y precedencia

1. Diseño fuente `CRL-FEASIBILITY-009A.1-DESIGN`, revisión R3.1, especialmente §§3.2, 3.8–3.18, 3.20, 4–9.
2. Paquete TASK-02-DRAFT-2 cerrado y aprobado por el usuario: decisiones D-01, D-02, D-03 y resolución D-VD03-5.
3. Informe de inspección TASK-01, dictamen `VIABLE WITH CONDITIONS`, compartido como `Markdown pegado.md`.
4. Evidencia histórica 009A, utilizada solo como antecedente y no como prueba del estado exacto del workspace 009A.1.

En caso de conflicto, prevalecen las decisiones humanas aprobadas en TASK-02 y el diseño R3.1. Las frases históricas de estado anterior dentro de los documentos TASK-02 quedan subordinadas al addendum de cierre aprobado.

## 3. Contexto del baseline

El informe TASK-01 describe el laboratorio aislado bajo `%TEMP%\\crl-feasibility-009a1\\project`, con fuentes bajo `src/test/java/lab/crl`. El arnés existente es de pruebas, no una aplicación de producción.

La inspección TASK-01 identifica `RotationCoordinator009A.java` como coordinador actual con mapas en memoria y journal de texto append + `FileChannel.force(true)`, sin replay, checkpoint versionado, hash-chain, membership durable ni recovery. También identifica un fallback de movimiento no atómico en `DistributedTlsInstance009A.persist`; ese componente es miembro y queda fuera de la implementación de TASK-03.

**Limitación de esta especificación:** el repositorio GitHub `iso8583-gateway` contiene documentación, no el proyecto Java 009A.1. El source set exacto del workspace local 009A.1 no fue inspeccionado directamente en esta sesión. Las rutas y clases se consideran candidatas de TASK-01 y deben confirmarse mediante inventario read-only antes de cualquier implementación. El ZIP de evidencia 009A es baseline histórico, no sustituto del source actual 009A.1.

## 4. Alcance

### Incluido

- Modelo y codec versionados para eventos durables del Coordinator.
- Journal append-only con secuencia monotónica, identidad de evento, referencias, digest y cadena de hashes.
- Checkpoint versionado que contiene última secuencia/hash confirmada y snapshot derivado del estado.
- Escritura completa, `force(true)`, archivo temporal, `force(true)`, cierre y reemplazo exclusivamente atómico del checkpoint.
- Lock exclusivo de sistema operativo para impedir dos Coordinators escritores sobre el mismo journal local.
- Replay completo y validación cruzada journal/checkpoint/snapshot.
- Detección de corrupción, registro parcial, truncamiento, secuencias inválidas, referencias faltantes y contradicción.
- Reconciliación idempotente de los casos de tail que R3.1 define expresamente para ADMIT y COMPLETE.
- Recuperación idempotente mediante `recoveryKey`/`recoveryEventId` según R3.1 §3.17.
- Serialización de la operación lógica Coordinator para que validación, orden, persistencia, publicación del estado y decisión de ACK sean una unidad para ADMIT/FENCE.
- Estado `RECOVERING`/`RECOVERY_REQUIRED`/`FAILED_CLOSED` y prohibición de servir/admitir antes de terminar la recuperación.
- Pruebas de codec, replay, integridad, idempotencia, exclusión de proceso y crash boundaries propias del almacén del Coordinator.

### Excluido

- Ledger durable local del Member y `PERMIT_RECEIVED` local (TASK-04). D-03 se conserva: el Member fuerza el receipt local antes de enviar `PERMIT_RECEIVED_ACK` al Coordinator; no se agrega escritura central redundante.
- Transporte/wire protocol final, framing, autenticación nueva o cambios del servidor TCP loopback (TASK-05).
- Wrapper/gate HTTP Reactor Netty y persistencia de `HANDLER_ENTERED` local (TASK-06).
- Integración completa de TLS, dos miembros JVM, conexiones persistentes y rotación (TASK-07).
- Matriz integral de fault injection end-to-end y paquete reproducible global (TASK-08/09), sin perjuicio de incluir hooks de persistencia y pruebas focalizadas de TASK-03.
- Consenso, quorum, HA, multi-host, multirregión, replicación, base de datos externa, Kafka, Redis o servicios nuevos.
- Ancla anti-rollback externa. Una copia antigua pero íntegra de journal + checkpoint no puede detectarse solo con archivos locales; esta limitación se mantiene explícita.
- Cualquier afirmación de PCI, FIPS, HSM certificado, compatibilidad con redes o uso productivo.

## 5. Reglas heredadas de TASK-02

- D-01: para el GET sintético, el Permit se consume en la confirmación durable de `HANDLER_START_INTENT`. Validación de vigencia + persistencia forman una operación lógica serializada con ADMIT/FENCE. La expiración previa rechaza; la posterior no cancela y no se revalida en `HANDLER_ENTERED` ni en el body.
- D-02: `PENDING` y `UNKNOWN` no reconciliado bloquean `COMMIT_GLOBAL`. `RECONCILED_UNKNOWN` conserva outcome desconocido, puede dejar de bloquear solo con evidencia durable suficiente y nunca equivale a `COMPLETE`.
- D-03: `PERMIT_RECEIVED` se fuerza en el ledger local del Member antes del ACK al Coordinator; no debe añadirse una escritura central redundante.
- D-VD03-5: rechazo por expiry se confirma con persistencia durable de `ADMISSION_REJECTED_EXPIRED` en Coordinator; después deriva a `REJECTED_NOT_STARTED` y deja de bloquear. Si persistencia es ambigua permanece bloqueante. ACK perdido se resuelve con `STATUS`/misma clave. No se invoca `HANDLER_ENTERED`, GET ni `COMPLETE` tras rechazo confirmado.
- TTL, referencia temporal, fuente de reloj, precisión/tolerancia y renovación pre-consumo siguen pendientes. TASK-03 no puede inventarlos, congelarlos ni inferirlos de `createdAt`.

## 6. Modelo persistente

### 6.1 Journal record

Cada registro durable debe incluir, como mínimo, los campos requeridos por R3.1 y TASK-02:

- `schemaVersion`;
- secuencia monotónica;
- `eventId`;
- `rolloutId` cuando aplique;
- `coordinatorEpoch`;
- tipo de evento;
- campos de estado/miembro/referencia que apliquen (`memberId`, `requestId`, `admissionId`, `processBootId`, generación/hash, etc.);
- `transitionKey` cuando aplique;
- `previousEventId` o referencia al ADMIT original cuando aplique;
- `payloadDigest`;
- `previousHash` y `recordHash`.

La serialización de clave/hash será versionada, length-prefixed y no ambigua. No se fija aquí un encoding binario/JSON ni un algoritmo concreto de `recordHash` si no está expresamente definido por R3.1; deben resolverse en la validación del diseño antes de congelar el codec. La fórmula de `recoveryEventId` sí queda fijada por R3.1: `SHA-256(canonical(recoveryKey))`.

### 6.2 Checkpoint

El checkpoint contendrá:
- `schemaVersion`;
- última secuencia confirmada;
- hash del último registro confirmado;
- snapshot determinista del estado derivado del Coordinator;
- metadata necesaria para validar su correspondencia con el journal.

El snapshot no sustituye el journal como historial. No se compacta ni trunca el journal en esta fase.

### 6.3 Lock

Se requiere exclusión de proceso a nivel de sistema operativo sobre el journal/almacén local. Si no se obtiene el lock o su estado es incierto, el Coordinator no puede operar. Este lock solo evita escritores simultáneos locales en el entorno ensayado; no es consenso ni fencing entre hosts.

## 7. Protocolo de confirmación durable

R3.1 §3.15 define el siguiente orden lógico para una transición:

1. Serializar el comando y validar precondiciones contra el estado confirmado actual.
2. Construir el evento canonicalizado y asignar secuencia/IDs sin publicar todavía el estado nuevo.
3. Append de todos los bytes del registro bajo lock.
4. `force(true)` del journal.
5. Escribir checkpoint temporal completo.
6. `force(true)` del archivo temporal.
7. Cerrar el temporal.
8. Reemplazar el checkpoint vigente mediante `ATOMIC_MOVE`.
9. Intentar `force(true)` del directorio padre si el filesystem/JDK lo soporta y registrar explícitamente la capacidad.
10. Publicar el nuevo estado en memoria y responder ACK/permiso solo después del punto de confirmación definido por R3.1.

No se permite fallback silencioso a rename no atómico. Si `ATOMIC_MOVE` no está soportado o falla, no se envía ACK y el Coordinator permanece cerrado/recuperable. Si el force del directorio no está soportado, se documenta esa limitación y no se reclama durabilidad frente a pérdida de energía/kernel/disco/hardware. Un error conocido o ambiguo en la secuencia impide habilitar servicio hasta replay/recovery.

El checkpoint —no la mera presencia de un evento en el journal— determina qué transición fue confirmada por el protocolo. Un tail válido posterior al checkpoint es evidencia a reconciliar, no autorización por sí misma.

## 8. Replay y recuperación

### 8.1 Validación base

- Adquirir lock exclusivo antes de servir.
- Leer/validar checkpoint, schema, secuencia/hash y snapshot.
- Recorrer journal y validar schema, secuencia contigua, identidad, referencias, payload digest y hash-chain.
- Comprobar que el journal contiene exactamente el registro indicado por la secuencia/hash del checkpoint.
- Reconstruir estado derivado y compararlo con el snapshot.
- Journal más corto que el checkpoint, registro parcial, secuencia repetida/saltada, hash inválido, referencia ausente, snapshot discordante o lock no disponible => `FAILED_CLOSED`/`RECOVERY_REQUIRED`, sin permisos.

### 8.2 Tail posterior al checkpoint

Un tail íntegro posterior al checkpoint se trata por tipo y no se descarta ni se confirma ciegamente:

- **ADMIT íntegro no checkpointed:** el orden R3.1 asegura que no se envió ACK/permiso. Escribir y checkpointar `ADMIT_ABORTED_UNCOMMITTED` como recovery explícito. El mismo `transitionKey` devuelve `NOT_ADMITTED/ABORTED_UNCOMMITTED`; no se crea otro admissionId para ese requestId.
- **COMPLETE íntegro no checkpointed:** validar referencia a un ADMIT pendiente del mismo miembro/boot y resultado permitido; escribir/checkpointar `RECOVERED_COMPLETE` una sola vez, vinculado al `eventId` original. Solo entonces devolver COMPLETE al retry/STATUS.
- **Evento parcial, hash/schema/referencia inválidos o contradicción:** `FAILED_CLOSED`; no truncar, reparar ni elegir un estado por conveniencia.
- **Otros tipos de evento en tail:** no inventar política de recuperación. Deben tener regla explícita aprobada o permanecer `RECOVERY_REQUIRED`/`FAILED_CLOSED`.

### 8.3 Idempotencia del recovery

Para cada acción de recovery:

`recoveryKey = (originalEventId, recoveryAction)`  
`recoveryEventId = SHA-256(canonical(recoveryKey))`

El mismo recoveryKey reutiliza el mismo evento/resultado a través de reinicios. Evento duplicado con payload incompatible, referencias inválidas o más de un eventId para la misma clave => `FAILED_CLOSED`.

## 9. Serialización de ADMIT y FENCE

No basta con que `record()` sea `synchronized`. La misma frontera lógica debe cubrir:

- lectura de estado confirmado;
- validación de miembro/epoch/estado/fence;
- determinación del orden;
- construcción del evento;
- append + force + checkpoint + reemplazo atómico;
- publicación del estado derivado;
- decisión de ACK/permiso.

Si ADMIT confirma antes de FENCE, permanece como admisión previa in-flight y debe resolverse por COMPLETE o reconciliación. Si FENCE confirma primero, no se confirma un ADMIT antiguo después. Un error/timeout no transforma la operación en éxito ni autoriza una transición paralela. La implementación exacta del serializador (lock, actor o mecanismo equivalente) queda abierta a diseño; se exige demostrar el orden con barreras deterministas.

## 10. Requisitos funcionales

| ID | Requisito | Criterio verificable |
|---|---|---|
| T03-FR-001 | Persistir eventos versionados y canonicalizados | Cada record tiene schema, secuencia, eventId, tipo, refs, digest y hashes verificables |
| T03-FR-002 | Append durable del journal | El evento no se confirma/ACKea antes de escritura completa y `force(true)` |
| T03-FR-003 | Checkpoint atómico | Temporal completo, force, close y `ATOMIC_MOVE`; sin fallback no atómico |
| T03-FR-004 | Lock exclusivo de Coordinator | Segundo proceso no puede operar sobre el mismo almacén |
| T03-FR-005 | Replay completo | Estado reconstruido y snapshot/checkpoint coinciden antes de habilitar operaciones |
| T03-FR-006 | Detectar corrupción/truncamiento/contradicción | Cualquier inconsistencia mantiene servicio cerrado, sin truncamiento automático |
| T03-FR-007 | Reconciliar tail ADMIT/COMPLETE tipado | ADMIT no confirmado se aborta; COMPLETE válido se recupera una vez; otros tipos no definidos quedan cerrados |
| T03-FR-008 | Idempotencia de transición y recovery | Misma clave/mismo payload devuelve evento/resultado existente; misma clave/payload distinto se rechaza |
| T03-FR-009 | Serializar ADMIT/FENCE integralmente | El orden durable coincide con el orden de la operación lógica; no hay ADMIT antiguo confirmado después de FENCE |
| T03-FR-010 | Recuperación repetida | Crashes en recovery reutilizan el mismo recoveryKey/recoveryEventId |
| T03-FR-011 | Startup fail-closed | Estado `RECOVERING` hasta lock, replay y reconciliación válidos |
| T03-FR-012 | Preservar decisiones TASK-02 | D-01/02/03 y D-VD03-5 no se reinterpretan ni amplían |

## 11. Requisitos no funcionales

- NFR-001: single Coordinator local de laboratorio; no HA, consenso ni multi-host.
- NFR-002: sin nuevas dependencias externas. Reutilizar JDK y stack declarado; no actualizar versiones por prevención.
- NFR-003: journal append-only y sin compactación en esta fase.
- NFR-004: serialización determinista, versionada y no ambigua.
- NFR-005: fallos y ambigüedad deniegan servicio; no hay soft-fail ni fallback no atómico.
- NFR-006: pruebas deterministas con barreras/latches y fault hooks; no usar `sleep` como sincronización.
- NFR-007: logs sin bearer tokens, claves, PSK/tickets, secretos, certificados privados ni material sensible.
- NFR-008: medir latencia p50/p95/p99 del camino de persistencia/Coordinator cuando esté autorizado; no definir SLO ni presupuesto fijo sin evidencia.
- NFR-009: distinguir terminación de proceso de pérdida de energía; no reclamar lo segundo sin pruebas específicas.
- NFR-010: la limitación anti-rollback de copia antigua íntegra queda documentada.

## 12. Criterios de aceptación

1. Journal y checkpoint nuevos pasan round-trip determinista y validación completa.
2. Rebuild tras reinicio produce el mismo estado confirmado.
3. Checkpoint corrupto, journal truncado respecto de checkpoint, registro parcial, hash/seq/referencia inválidos o snapshot discordante no habilitan el Coordinator.
4. Segundo proceso no adquiere el lock ni escribe; el proceso que no lo obtiene permanece cerrado.
5. `ATOMIC_MOVE` no soportado/fallido no activa un fallback y no emite ACK.
6. Tail ADMIT se reconcilia como abort no autorizado; tail COMPLETE válido se recupera una sola vez; recovery repetido reusa la misma key/ID.
7. ADMIT/FENCE concurrentes producen un orden durable único y verificable.
8. ACK perdido se resuelve consultando la misma clave; no duplica evento/admission.
9. Conflicto de payload para una transitionKey no altera el evento previamente confirmado.
10. El Coordinator no sirve durante `RECOVERING`, `RECOVERY_REQUIRED` ni `FAILED_CLOSED`.
11. No se modifica Member ledger, HTTP gate, wire protocol ni tareas posteriores.
12. Toda afirmación sobre filesystem Windows se acompaña de evidencia del entorno ensayado y se limita al nivel realmente demostrado.

## 13. Condiciones de aprobación

Este documento es una propuesta para revisión. El diseño de TASK-03 solo podrá aprobarse cuando estén validados el inventario real del workspace, el formato exacto de record/checkpoint, la matriz completa de tipos de evento y recuperación, el manejo de errores después de `ATOMIC_MOVE`, el lock de proceso en Windows y la estrategia de pruebas/fault injection.

La aprobación de esta SPEC/PLAN/TASKS no autoriza automáticamente implementar. Se requiere aprobación humana explícita independiente para iniciar modificaciones de código.
