# TASK-03 — Revisión final de aprobación de DRAFT-3

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-FINAL-APPROVAL-REVIEW-DRAFT-3`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Tipo:** revisión documental cruzada; no ejecución técnica.

## 1. Dictamen ejecutivo

**H-01: RESUELTO DOCUMENTALMENTE.** La corrección alinea la SPEC §11, PLAN T03-09, TASKS T03-09 y DESIGN-VALIDATION. La distinción entre OS lock de proceso y serialización lógica de operaciones también queda explícita.

**TASK-03 completo: VIABLE CON CONDICIONES; NO RECOMIENDO APROBAR TODAVÍA LA IMPLEMENTACIÓN.** D03-OPEN-01..08 continúan abiertos y la propia SPEC los considera bloqueantes para congelar diseño/contratos. Esta revisión no los cierra ni propone decisiones implícitas.

## 2. Validación cruzada de H-01

| Regla | SPEC | PLAN | TASKS | DESIGN-VALIDATION |
|---|---|---|---|---|
| ADMIT antes de FENCE conserva el derecho de completar | §11, regla 1 | T03-09 | T03-09 | Invariante 5, P09 |
| START_INTENT posterior a FENCE permitido para ADMIT previo si Permit vigente al commit durable | §11, regla 2 | T03-09 | Criterio T03-09 | P09, FR-011 |
| Expiry pre-commit produce rechazo durable; solo entonces libera bloqueo | §11, regla 3 | T03-09 | Criterio T03-09 | P10, N15 |
| Persistencia de rechazo fallida/ambigua mantiene bloqueo | §11, regla 3 | T03-09 | Criterio T03-09 | N15, C21 |
| FENCE antes de ADMIT nuevo rechaza | §11, regla 4 | T03-09 | Criterio T03-09 | N16, C21 |
| START_INTENT antes de FENCE no se cancela/revalida | §11, regla 5 | T03-09 | Criterio T03-09 | P11, C21 |
| Orden total determinista entre las cuatro operaciones | §11 | T03-09 y barreras/latches | T03-09 y evidencia requerida | C21 y estrategia de testing |

## 3. Reglas heredadas preservadas

- D-01 se mantiene: consumo en commit durable de START_INTENT; validación y persistencia como una sola operación lógica.
- D-VD03-5 se mantiene: expiry previa exige ADMISSION_REJECTED_EXPIRED durable; persistencia fallida/ambigua mantiene la admisión bloqueante.
- D-02/D-03 no se reinterpretan.
- TTL, instante/fuente de reloj, precisión, skew y renovación pre-consumo permanecen pendientes.
- La ampliación no autoriza recuperación automática de tails START_INTENT/ADMISSION_REJECTED_EXPIRED. D03-OPEN-02 sigue abierta y ambos tails continúan en RECOVERY_REQUIRED.

## 4. Aclaración de lock

La separación se considera suficiente a nivel documental:

- OS lock: exclusión de un segundo proceso Coordinator sobre el almacén local durante la vida operativa del proceso.
- Serialización lógica: orden intra-proceso de las operaciones concurrentes ADMIT/FENCE/START_INTENT/rechazo expiry.

No se atribuye a ninguno consenso multi-host ni fencing distribuido.

## 5. Alcance y regresiones

No se detecta en la revisión documental una ampliación funcional más allá de H-01. IDs/dependencias T03-00..T03-12 se preservan. No se cambia el catálogo de tail ni D03-OPEN-01..08; no se altera Member ledger, wire protocol, HTTP gate, POM, source code ni documentos principales del Gateway.

## 6. Bloqueos que impiden la aprobación integral

1. D03-OPEN-01: encoding, canonicalización, hashes/digests y schema checkpoint.
2. D03-OPEN-02: política tail de FENCE, START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN y COMMIT_GLOBAL.
3. D03-OPEN-03: evento/clave/primer arranque/crash recovery de coordinatorEpoch.
4. D03-OPEN-04: clasificación de directory force unsupported frente a error real/ambiguo.
5. D03-OPEN-05: temporales residuales.
6. D03-OPEN-06: transición RECOVERY_REQUIRED/FAILED_CLOSED.
7. D03-OPEN-07: evidencia del filesystem Windows para FileLock/ATOMIC_MOVE/directory force.
8. D03-OPEN-08: contrato/evidencia Member para REBIND.

Además, se debe revalidar read-only el workspace 009A.1 (T03-00) antes de cualquier implementación. Las pruebas de filesystem Windows requieren autorización específica del experimento.

## 7. Resultado de pruebas/revisión

- Revisión realizada: documental, transversal y basada en DRAFT-2, TASK-02 aprobado y fuente R3.1 §3.18.
- Escenarios H-01 añadidos: P09, P10, P11, N15, N16 y C21.
- Tests/builds/Maven/experimentos ejecutados: ninguno.
- Código/configuración modificados: ninguno.
- Hashes/manifiesto: deberán validarse contra la publicación DRAFT-3.

## 8. Recomendación de aprobación

**Aprobar el contenido de la corrección H-01 como revisión documental, sujeto a aceptación humana explícita. No aprobar todavía la implementación TASK-03.** Para aprobar el diseño integral, resolver o aceptar expresamente como bloqueos las decisiones abiertas de la SPEC y completar la validación correspondiente. La aprobación de diseño no autoriza por sí sola implementación; se requiere autorización humana separada.

El gate de factibilidad CRL para producción permanece cerrado.
