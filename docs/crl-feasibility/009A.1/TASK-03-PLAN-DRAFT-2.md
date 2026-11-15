# TASK-03 — Plan de persistencia durable del Coordinator

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-PLAN-DRAFT-2`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Implementación:** no autorizada.  
**Especificación normativa:** `TASK-03-SPEC-DRAFT-2.md`  
**Desglose normativo:** `TASK-03-TASKS-DRAFT-2.md`

## 1. Objetivo

Planificar la entrega del subsistema durable del Coordinator manteniendo consistencia entre SPEC, PLAN y TASKS. El plan adopta los identificadores T03-00..T03-12 definidos en TASKS; no introduce numeración paralela.

El plan es una propuesta de ejecución futura. La autorización vigente cubre únicamente preparación/validación del diseño; ninguna implementación, prueba de código, Maven, build o experimento está autorizada actualmente.

## 2. Baseline y workspace

TASK-01 ubicó el arnés 009A.1 en:

`C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a1\project`

Las fuentes candidatas fueron reportadas bajo `src/test/java/lab/crl`. El repositorio público `iso8583-gateway` contiene documentación, no el `pom.xml` ni las fuentes Java del arnés 009A.1. No usar la raíz de GitHub como workspace de implementación.

**Precondición T03-00:** antes de cualquier modificación futura, inspeccionar read-only el workspace local, estado Git si corresponde, POM, árbol, archivos candidatos, hashes y diferencias respecto del informe TASK-01. El ZIP histórico 009A es antecedente, no reemplazo ni prueba de la copia 009A.1 actual.

## 3. Grafo de dependencias

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

- Inventariar `pom.xml`, árbol de fuentes, archivos de evidencia y estado Git si existe.
- Capturar SHA-256 de los candidatos relevantes, POM y archivos de coordinación.
- Comparar con TASK-01 e identificar cambios/ausencias.
- Confirmar Java/stack solo desde evidencia atribuible a este workspace.
- Identificar ruta de datos y filesystem destino sin escribir ni probar.

**Salida:** inventario y hashes; ningún archivo modificado; ninguna build/test/experimento.

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

- Una frontera lógica única cubre check, lectura de estado, validación, orden, append/force/checkpoint, publicación y ACK.
- ADMIT y FENCE comparten el mismo orden total.
- Publicar memoria solo después del punto de confirmación.
- Retry de resultado ambiguo consulta misma transitionKey; no crea transición en paralelo.

**Salida:** pruebas deterministas de carreras ADMIT/FENCE y ACK perdido.

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
- **Concurrencia:** ADMIT vs FENCE con barreras/latches; no `sleep` como sincronización.
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
| Carrera ADMIT/FENCE | Crítico | Serialización lógica integral y pruebas deterministas |
| Recovery duplicado | Alto | recoveryKey/recoveryEventId estables; conflicto fail-closed |
| Rebind sin ledger Member | Crítico | Requiere evidencia TASK-04; excluido de TASK-03 |
| Rollback de copia antigua íntegra | Limitación aceptada | Documentar; no ancla externa en alcance |
| Workspace divergente | Alto | T03-00 read-only; detener si cambia materialmente |

Detenerse y solicitar decisión si el workspace no coincide con TASK-01; si se requiere tocar componentes excluidos; si el filesystem tiene comportamiento ambiguo; si se exige fallback, truncamiento, soft-fail o auto-repair; si se pretende fijar TTL/reloj/skew/renewal; o si se requiere dependencia, DB, HA/consenso o alcance financiero.

## 7. Estado final esperado

TASK-03 quedará cerrada solo cuando el almacén, checkpoint, lock, replay, tail recovery permitido, idempotencia y serialización satisfagan SPEC con evidencia reproducible y límites explícitos. Cierre TASK-03 no autoriza TASK-04 ni tareas posteriores. El gate de factibilidad CRL para producción permanece cerrado.
