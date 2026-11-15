# TASK-03 — Plan de persistencia durable del Coordinator

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-PLAN-DRAFT-4`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Implementación:** no autorizada.  
**Especificación normativa:** `TASK-03-SPEC-DRAFT-4.md`  
**Desglose normativo:** `TASK-03-TASKS-DRAFT-4.md`

## 1. Objetivo

Planificar la entrega del subsistema durable del Coordinator manteniendo consistencia entre SPEC, PLAN y TASKS. El plan adopta los identificadores T03-00..T03-12 definidos en TASKS; no introduce numeración paralela.

El plan es una propuesta de ejecución futura. La autorización vigente cubre únicamente preparación/validación del diseño; ninguna implementación, prueba de código, Maven, build o experimento está autorizada actualmente.

## 2. Baseline y workspace

El informe histórico TASK-01 inspeccionó el laboratorio 009A en:

`C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a\project`

No acredita que el workspace aislado 009A.1 haya sido creado o inspeccionado. El diseño R3.1 propone una raíz nueva `%TEMP%\crl-feasibility-009a1`, conservando 009A como evidencia inmutable.

La captura de Explorador de archivos aportada por el usuario el 2026-11-15 muestra que la raíz 009A.1 contiene los ocho documentos Markdown visibles, pero no muestra el subdirectorio `project`. El usuario confirmó que eso es todo lo que contiene. El estado Git, los archivos ocultos y cualquier contenido no visible no se consideran inspeccionados.

**Estado:** T03-00 está `BLOCKED — WORKSPACE NOT PROVISIONED`. No se puede inspeccionar POM, árbol de fuentes ni hashes del proyecto 009A.1 mientras falte `%TEMP%\crl-feasibility-009a1\project`.

**Gate previo de provisión (fuera de T03-00):** la copia aislada debe planificarse y recibir autorización humana independiente. El origen candidato es `%TEMP%\crl-feasibility-009a\project`, según el informe TASK-01; no se moverá ni modificará. Deben resolverse antes de copiar el tratamiento de `.git`, `target/`, evidencia histórica, keystores y secretos locales, y generarse un manifiesto de archivos copiados/excluidos con SHA-256. Ver `WORKSPACE-PROVISIONING-REQUIREMENTS-DRAFT-4.md`.

El repositorio público `iso8583-gateway` contiene documentación, no el `pom.xml` ni las fuentes Java del arnés local. No usar la raíz de GitHub, el ZIP histórico ni el proyecto 009A como si fueran el workspace 009A.1.

**Precondición T03-00:** después de que la provisión haya sido autorizada y completada, inspeccionar read-only la ruta hija `project`, estado Git si corresponde, POM, árbol, archivos candidatos, hashes y diferencias con el informe histórico. T03-00 no crea ni copia ese workspace.

```text
T03-00 ──> T03-01 ──> T03-02 ──> T03-04 ──> T03-05 ──┐
   │          │                                      │
   └────────> T03-03 ────────────────────────────────┘
              │          T03-06 <── T03-01,T03-02    │
              └─────────────────────────────────────┘
                                      T03-07 <── T03-04,T03-05,T03-06
                                      T03-10 <── T03-02,T03-06,T03-07
                                      T03-08 <── T03-07,T03-10
                                      T03-09 <── T03-04,T03-05,T03-06,T03-07,T03-10
                                      T03-11 <── T03-00..T03-10
                                      T03-12 <── T03-11
```

El grafo es vinculante; la secuencia operativa de la sección 4 respeta estas relaciones. T03-02 y T03-03 pueden progresar en paralelo una vez cerradas sus precondiciones de diseño.

## 4. Etapas alineadas con TASKS

### T03-00 — Baseline read-only

**Precondición:** debe existir `%TEMP%\crl-feasibility-009a1\project`, provisionado mediante un gate separado y autorizado. La raíz 009A.1 con documentación, por sí sola, no satisface esta precondición.

- Confirmar la ruta hija `project` y que corresponde al workspace aislado, no al origen 009A.
- Inventariar `pom.xml`, árbol, fuentes, tests, evidencia y estado Git si existe.
- Capturar SHA-256 de los archivos candidatos relevantes, POM y archivos de coordinación.
- Comparar con TASK-01 señalando que TASK-01 es evidencia histórica de 009A, no baseline hash-equivalente de 009A.1.
- Confirmar Java/stack solo desde evidencia atribuible al workspace aislado.
- Identificar ruta de datos y filesystem destino sin escribir ni probar.

**Si falta `project`:** cerrar como `BLOCKED — WORKSPACE NOT PROVISIONED`, sin copiar, crear, mover, reconstruir ni sustituir el workspace.

**Salida:** inventario y hashes verificables cuando la precondición se cumpla; ningún archivo modificado; ninguna build/test/experimento.

### T03-01 — Domain/event catalog y cierre de decisiones

- Definir envelope lógico por tipo, campos obligatorios/opcionales y referencias.
- Enumerar eventos: ADMIT, COMPLETE, FENCE, HANDLER_START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL, avance de epoch y recovery events.
- Congelar política tail solo para ADMIT/COMPLETE según R3.1; documentar el resto como `RECOVERY_REQUIRED` mientras no exista decisión explícita.
- Resolver D03-OPEN-01..08 de SPEC, incluyendo canonicalización, epoch, temporales y filesystem.
- Preservar D-01/D-02/D-03/D-VD03-5 y parámetros temporales aún pendientes.

**Salida:** matriz de eventos/tail y decisiones documentadas. Si quedan decisiones bloqueantes, no congelar contratos de implementación.

### T03-02 — Codec

- Definir schema versionado y serialización determinista del JournalRecord y Checkpoint.
- Congelar canonicalización, payloadDigest, recordHash, valores ausentes y versión de canonicalización.
- Validar secuencias, IDs, referencias y encoding.

**Salida:** contrato codec aprobado y vectores de prueba sintéticos. Implementación solo con autorización posterior.

### T03-03 — OS lock

- Definir recurso protegido, apertura/ownership/cierre del lock y comportamiento ante lock no disponible.
- Diseñar prueba de dos procesos locales y shutdown anormal.
- No extrapolar semántica a filesystem de red ni multi-host.

**Salida:** contrato de exclusión local y plan de prueba aislado.

### T03-04 — Journal writer

- Append de todos los bytes del registro bajo lock.
- Mantener secuencia monotónica y hash-chain según codec aprobado.
- `force(true)` del journal antes de continuar al checkpoint.
- Tratar write parcial/IOException como no confirmado y no emitir ACK.
- No truncar ni ignorar registros parciales automáticamente.

**Salida:** evidencia de write completo, fallo/partial y continuidad de secuencia/hash.

### T03-05 — Checkpoint writer

- Crear temporal en el mismo filesystem/directorio que checkpoint destino.
- Escribir todos los bytes, `force(true)`, cerrar y sustituir exclusivamente con `ATOMIC_MOVE`.
- Intentar directory force cuando sea soportado y registrar capacidad/resultado.
- Clasificar `ATOMIC_MOVE` unsupported/failure separadamente de directory force unsupported y error real/ambiguo posterior al rename.
- No implementar fallback no atómico ni promover temporales residuales por inferencia.

**Salida:** evidencia autorizada del filesystem real y política de error aplicada. La prueba efectiva Windows requiere aprobación específica del experimento.

### T03-06 — State reducer

- Aplicar eventos checkpointed en orden y producir snapshot determinista.
- Validar transiciones legales, referencias y estado derivado.
- Mantener separados estado confirmado y candidato no checkpointed.

**Salida:** mismo input produce el mismo estado/snapshot; transiciones inválidas se rechazan.

### T03-07 — Replay

- Adquirir lock antes de leer/reconstruir.
- Validar checkpoint, journal, schema, secuencias, IDs, payloadDigest, hash-chain y referencias.
- Reconstruir estado confirmado y comparar snapshot.
- Clasificar tail por el catálogo T03-01; no habilitar servicio mientras haya tail no clasificado.

**Salida:** replay determinista y cierre ante corrupción, contradicción, temporal ambiguo o tipo de tail no autorizado.

### T03-10 — Idempotency index

- Construir índices durables/reconstruibles de transitionKey, payloadDigest, eventId, recoveryKey y recoveryEventId.
- Reutilizar resultado ante retry same-key/same-payload.
- Rechazar same-key/different-payload y duplicados incompatibles.
- Preservar canonicalización versionada del recoveryKey definida por R3.1.

**Salida:** evidencia de retry y restart sin duplicación. Esta tarea es P0, no P1.

### T03-08 — Tail recovery ADMIT/COMPLETE

- Para ADMIT íntegro en tail sin checkpoint, generar `ADMIT_ABORTED_UNCOMMITTED` según R3.1; no emitir Permit ni crear otro admissionId para la misma clave.
- Para COMPLETE íntegro y válido, verificar ADMIT pendiente del mismo Member/boot y resultado permitido; generar/reutilizar `RECOVERED_COMPLETE` una sola vez.
- Para FENCE, START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL y epoch, detenerse hasta política explícita aprobada.
- Para `REBIND_UNSTARTED_ADMISSION`, exigir evidencia Member íntegra de TASK-04; no ejecutar dentro de TASK-03.

**Salida:** recovery idempotente de los dos tipos permitidos; resto fail-closed.

### T03-09 — Coordinator serialization

- Definir una frontera lógica intra-proceso que serialice ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED como una unidad de orden total.
- Para una admisión checkpointed antes de FENCE, permitir START_INTENT después de FENCE si el Permit sigue vigente en el punto de commit durable; FENCE no revoca esa admisión previa.
- Si el Permit expiró antes del commit, persistir ADMISSION_REJECTED_EXPIRED y liberar el bloqueo solo después de confirmación durable; fallo/ambigüedad conserva el bloqueo.
- Rechazar ADMIT nuevo cuando FENCE ya confirmó; preservar START_INTENT confirmado antes de FENCE sin cancelarlo ni revalidarlo.
- Cubrir validación D-01 + persistencia de su resultado dentro de la misma operación lógica; no separar check y commit.
- Publicar memoria, ACK, Permit o rechazo solo después del punto de confirmación durable.
- Retry/STATUS de resultado ambiguo consulta la misma clave; no crea una transición paralela.

**Salida:** barreras/latches deterministas prueban los seis escenarios focales H-01, orden durable único, ACK perdido y ausencia de liberación prematura.

### T03-11 — Crash/verification matrix

- Inyectar cortes en append, journal force, temp write/force/close, atomic move, directory force y ACK.
- Ensayar restart/replay y lock con proceso real.
- Ensayar epoch advancement y sus cortes únicamente tras definir contrato y obtener autorizaciones.
- Separar terminación de proceso de pérdida de energía/hardware.

**Salida:** matriz completa de frontera/estado durable/resultado replay/ACK/evidencia.

### T03-12 — Evidence/closure

- Generar informe con archivos exactos, hashes, comandos, resultados y tests.
- Documentar Quality Gate, riesgos abiertos, límites del filesystem y diferencias respecto del baseline.
- Registrar explícitamente pruebas no ejecutadas y claims no demostrados.
- Solicitar revisión humana de cierre TASK-03.

**Salida:** dictamen técnico reproducible. No habilita TASK-04 ni gate CRL de producción.

## 5. Estrategia de pruebas

- **Unitarias:** canonicalización, codec, digest/hash, secuencia, reducer, checkpoint round-trip, transiciones, transitionKey/recoveryKey.
- **Integración de archivos:** append/force, checkpoint temporal/force/atomic move, lectura y replay.
- **Procesos:** OS lock con segundo proceso, terminación Coordinator y restart/replay.
- **Concurrencia:** ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED con barreras/latches; no `sleep` como sincronización.
- **Fault injection:** cada frontera precisa del protocolo de persistencia, directory force y ACK.
- **Seguridad:** logs sin tokens/secretos; temporales con ubicación/permisos acotados al soporte de plataforma.
- **Epoch:** crash/restart en los límites una vez que su evento y su política estén aprobados.
- **Filesystem:** distinguir unsupported capability de I/O error; ningún claim más allá de evidencia real.

## 6. Riesgos y gates de parada

| Riesgo | Severidad | Mitigación/gate |
|---|---|---|
| `ATOMIC_MOVE` no soportado | Crítico | No fallback; no ACK; Coordinator cerrado |
| Directory force no soportado | Alto | Registrar capability; claims limitados a process termination ensayada |
| Error real después del rename | Crítico | No clasificar como unsupported; replay/verificación cerrada |
| Tail de evento sin regla | Crítico | `RECOVERY_REQUIRED`; no inferir transición |
| Epoch no confirmado | Crítico | No habilitar Coordinator |
| Doble Coordinator | Crítico | OS lock exclusivo; contender sin servicio |
| Carrera entre ADMIT/FENCE/START_INTENT/rechazo expiry | Crítico | Frontera lógica integral y pruebas deterministas de los seis escenarios H-01 |
| Recovery duplicado | Alto | recoveryKey/recoveryEventId estables; conflicto fail-closed |
| Rebind sin ledger Member | Crítico | Requiere evidencia TASK-04; excluido de TASK-03 |
| Rollback de copia antigua íntegra | Limitación aceptada | Documentar; no ancla externa en alcance |
| Workspace divergente | Alto | T03-00 read-only; detener si cambia materialmente |

Detenerse y solicitar decisión si el workspace aislado no está provisionado, si no coincide materialmente con el manifiesto de provisión aprobado o si su procedencia no puede verificarse; si se requiere tocar componentes excluidos; si el filesystem tiene comportamiento ambiguo; si se exige fallback, truncamiento, soft-fail o auto-repair; si se pretende fijar TTL/reloj/skew/renewal; o si se requiere dependencia, DB, HA/consenso o alcance financiero.

## 7. Estado final esperado

TASK-03 quedará cerrada solo cuando el almacén, checkpoint, lock, replay, tail recovery permitido, idempotencia y serialización satisfagan SPEC con evidencia reproducible y límites explícitos. Cierre TASK-03 no autoriza TASK-04 ni tareas posteriores. El gate de factibilidad CRL para producción permanece cerrado.
