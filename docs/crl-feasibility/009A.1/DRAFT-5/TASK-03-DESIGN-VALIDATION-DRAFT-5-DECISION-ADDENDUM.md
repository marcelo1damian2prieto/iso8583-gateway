# TASK-03 — Enmienda DRAFT-5 a DESIGN-VALIDATION-DRAFT-4

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-DESIGN-VALIDATION-DRAFT-5-DECISION-ADDENDUM`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Documento base:** `TASK-03-DESIGN-VALIDATION-DRAFT-4.md`  
**Base Git:** `bef7aa7c615745e19f548b47854fb4e37d543b18`.

## 1. Efecto y dictamen actualizado

Esta enmienda es aditiva y se lee junto con DESIGN-VALIDATION-DRAFT-4. Prevalece en los estados de baseline, decisiones y gates que aquí actualiza. La revisión combinada se denomina `DESIGN-VALIDATION-DRAFT-5`.

**Dictamen: `VIABLE WITH CONDITIONS — DOCUMENTARY BASELINE AND DECISION DIRECTIONS ACCEPTED; IMPLEMENTATION NOT READY`.**

La evaluación anterior se actualiza en dos puntos: T03-00 ya está completada/aceptada y las ocho propuestas D03-OPEN-01..08 recibieron aprobación humana a nivel de dirección/política, con detalle abierto y gates reservados. Esto no significa que el diseño integral de TASK-03 esté aprobado, que el codec/epoch estén totalmente especificados o que la implementación esté autorizada.

## 2. Evidencia actualizada del baseline

T03-00 fue ejecutada de forma estática/read-only y el informe `T03-00-BASELINE-INSPECTION.md` fue inspeccionado y aceptado por el usuario. El manifiesto contiene 31 archivos y su tamaño/SHA-256 fue contrastado independientemente contra las entradas del `project.zip`: 31/31 coincidencias, cero faltantes y cero discrepancias. El manifiesto declara que la copia coincide con el origen y que el origen no fue movido ni eliminado. No hubo acceso directo para recalcular los hashes en el disco local C: desde esta sesión.

Baseline observado: POM artifactId `crl-feasibility-009a`, version `1.0.0`, Java 21, Spring Boot 3.5.16 declarado; 21 Java bajo `src/test/java/lab/crl`; 9 fixtures PEM públicos; no `src/main/java` en el inventario; `.git` ausente/excluido; 23 métodos `@Test` identificados por inspección estática, no ejecutados. Maven tree no verificado. El baseline es arnés histórico 009A copiado de forma aislada; no es una implementación nueva de TASK-03.

Se mantienen las limitaciones: TASK-01 no aporta hashes por archivo para comparación hash-by-hash; no se ejecutaron Maven/build/tests/experimentos; Windows FileLock/ATOMIC_MOVE/directory force no han sido probados.

## 3. Estado de decisiones y criterio de diseño

| ID | Estado aprobado | Implicación para validación | Reserva/gate |
|---|---|---|---|
| D03-OPEN-01 | `DIRECTION APPROVED — DETAIL SPECIFICATION OPEN` | Codec binario tipado/versionado/determinista/length-prefixed, presencia explícita, SHA-256 domain-separated. | No freeze sin tags, widths, byte order, domain separators, nullable/limits/unknown fields/genesis/compatibility. |
| D03-OPEN-02 | `CLOSED — TAIL POLICY APPROVED` | Solo ADMIT/COMPLETE con recovery automático R3.1. | Todos los demás tails, incluido epoch, son `RECOVERY_REQUIRED`; no recovery por analogía. |
| D03-OPEN-03 | `DIRECTION APPROVED — STARTUP DETAIL OPEN` | Evento provisional `COORDINATOR_EPOCH_ADVANCED`, incremento estricto +1, genesis epoch1 solo almacén totalmente vacío. | StartupOperationId, definición restart, overflow y crash/tail detail abiertos; tail detiene. |
| D03-OPEN-04 | `POLICY APPROVED — PLATFORM EVIDENCE OPEN` | Estados `SUPPORTED_AND_SUCCEEDED`, `UNSUPPORTED`, `FAILED_OR_AMBIGUOUS`; unsupported solo explícito; no fallback. | Clasificación concreta depende del JDK/proveedor/Windows ensayado. |
| D03-OPEN-05 | `CLOSED — RESIDUAL TEMP POLICY APPROVED` | `CREATE_NEW` mismo filesystem/directorio; residuales detectados; no auto-promote/replace/delete. | Naming y límites finales abiertos; limpieza manual separada. |
| D03-OPEN-06 | `DIRECTION APPROVED — CLASSIFICATION DETAIL OPEN` | Estados distintos RECOVERING/RR/FAILED_CLOSED; sin auto-reset. | Tabla exhaustiva y lock legitimate-busy vs uncertain open. |
| D03-OPEN-07 | `GATE APPROVED — EXPERIMENT NOT AUTHORIZED` | Metodología de experimento Windows aislado aceptada. | La ejecución sigue prohibida hasta autorización específica y rollback. |
| D03-OPEN-08 | `BOUNDARY/PRECONDITIONS APPROVED — TASK-04 EVIDENCE` | TASK-03 define contrato/referencia; no ejecuta REBIND. | Ledger Member y prueba de terminación quedan en TASK-04, no autorizado. |

El addendum de aceptación humana y la aclaración de incompatibilidades son parte de la base de gobierno. La aclaración posterior prevalece sobre cualquier lectura que imponga proponer un refactor mínimo de inmediato: primero se analiza individualmente cada incompatibilidad con evidencia; después, si hay incompatibilidades relevantes, se prepara un plan consolidado para aprobación humana.

## 4. Requisitos y reglas validados

Se consideran aceptadas como baseline de diseño las siguientes reglas: binary deterministic codec direction; allowlist de tail ADMIT/COMPLETE; epoch increment +1 previo al servicio; clasificación explícita de directory force; temporales sin autoridad automática; estados cerrados diferenciados y sin auto-reset; experimento Windows independiente; REBIND limitado por evidencia Member.

No se relajan los requisitos heredados:

- D-01: Permit se consume en el checkpoint durable de START_INTENT; validación y persistencia una sola operación lógica con ADMIT/FENCE.
- D-02: PENDING y UNKNOWN no reconciliado bloquean COMMIT_GLOBAL; RECONCILED_UNKNOWN puede dejar de bloquear con evidencia suficiente, pero conserva outcome UNKNOWN y nunca es COMPLETE.
- D-03: PERMIT_RECEIVED se fuerza en ledger Member antes de ACK; no duplicar escritura central.
- D-VD03-5: rechazo expiry durable antes de liberar bloqueo; fallo/ambigüedad conserva bloqueo; STATUS por misma clave.
- H-01: orden total vivo ADMIT/FENCE/START_INTENT/ADMISSION_REJECTED_EXPIRED; no decide tail recovery.
- TTL/reloj/skew/renovación pre-consumo siguen pendientes.

## 5. Escenarios y pruebas que deberán validar las decisiones

### Positivos

- ADMIT y COMPLETE íntegros en tail recuperados una sola vez conforme R3.1 y `recoveryKey` estable.
- Almacén totalmente vacío genera epoch inicial conforme a la política que se detalle; el Coordinator no opera hasta checkpoint epoch.
- Directory force devuelve éxito soportado o unsupported explícito, con claims limitados.
- Un temporal nuevo `CREATE_NEW` no colisiona con archivos preexistentes; los residuales son detectados y no adquieren autoridad.
- La serialización H-01 conserva admisiones previas a FENCE y rechaza nuevas admisiones tras FENCE.
- REBIND se mantiene fuera de TASK-03 y solo podrá proponerse con evidencia Member completa de TASK-04.

### Negativos y casos borde

- Tail de FENCE/START_INTENT/ADMISSION_REJECTED_EXPIRED/RECONCILE_UNKNOWN/COMMIT_GLOBAL/epoch => `RECOVERY_REQUIRED`, sin ACK/permiso/recovery inferido.
- Registro/encoding ambiguo, null/absent confusos, hash incorrecto, length overflow, secuencia o snapshot contradictorios => cierre sin repair/truncate automático.
- Directory force I/O/access denied/resultado ambiguo no se clasifica `UNSUPPORTED`; después del rename no se da ACK nuevo y se requiere replay/verificación.
- Temporal parcial/completo residual no se promueve, reemplaza ni elimina automáticamente.
- `FAILED_CLOSED` no se resetea por retry/restart; requiere intervención y baseline verifiable.
- `REBIND` sin ledger Member íntegro, con START_INTENT/UNKNOWN, proceso anterior no demostrado muerto o tail/contradicción => denegado.

Estos son criterios/escenarios propuestos para verificación futura. No se ejecutaron.

## 6. Incompatibilidades y refactor — proceso de validación

El diseño deberá ser contrastado con el baseline de T03-00 antes de implementar. Cada potencial incompatibilidad se documentará con evidencia concreta y se clasificará como:

- incompatibilidad confirmada;
- riesgo hipotético;
- falta de información;
- diferencia estructural sin incompatibilidad demostrada.

Para cada incompatibilidad confirmada se analizarán requisitos, invariantes, contratos, ownership, estados, archivos/formatos persistidos, replay, idempotencia, concurrencia, fencing, seguridad, testability, compatibilidad y rollback. Se evaluarán alternativas y no se asumirá que siempre corresponde refactor.

Después de analizar las incompatibilidades relevantes se elaborará, si corresponde, un plan de refactor consolidado. El plan tendrá que incluir evidencia, scope, trazabilidad, alternativas, secuencia/dependencias, datos y formatos, compatibilidad/replay/migración/rollback, impacto de seguridad e idempotencia, pruebas/aceptación y riesgos residuales. Aprobación humana del plan y autorización de implementación son gates separados.

La detección de colisión no revoca automáticamente una decisión aceptada; tampoco autoriza modificar código durante esta etapa documental.

## 7. Riesgos abiertos y gates

1. Detalles finales del codec (D03-OPEN-01) y startup epoch (D03-OPEN-03) continúan bloqueando freeze de esos contratos.
2. La clasificación exhaustiva de estados y el lock busy/uncertain (D03-OPEN-06) debe detallarse antes del contrato de operación de recuperación.
3. Las capacidades de filesystem Windows siguen no verificadas (D03-OPEN-07). No experimento autorizado.
4. No se ha probado durabilidad frente a pérdida de energía, kernel, disco o hardware.
5. No hay ancla externa anti-rollback.
6. REBIND depende de TASK-04 y de evidencia Member; tarea no autorizada.
7. Implementación TASK-03 no autorizada; TASK-04+ no autorizada; production gate cerrado.

## 8. Veredicto

**`VIABLE WITH CONDITIONS — PENDING HUMAN REVIEW`.** T03-00 está completada; las ocho propuestas conservadoras fueron aceptadas en los niveles indicados; la política de tail y la política de temporales están cerradas; los detalles de codec/epoch/states siguen abiertos; el experimento Windows no está autorizado. H-01 y las decisiones TASK-02 permanecen preservadas. No hay evidencia de implementación ni de tests/builds.

La aprobación de esta revisión documental por el usuario no debe interpretarse como autorización de implementación. Se requiere aprobación integral de diseño y autorización humana independiente para implementar. Cualquier experimento Windows necesita su propia autorización.