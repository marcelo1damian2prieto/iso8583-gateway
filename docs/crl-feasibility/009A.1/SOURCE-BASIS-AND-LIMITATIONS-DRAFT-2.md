# TASK-03-DRAFT-2 — Base documental y limitaciones

**Fecha:** 2026-11-15  
**Propósito:** trazabilidad de fuentes y límites de la revisión documental cruzada.

## Fuentes consultadas

- `CRL-FEASIBILITY-009A.1-DESIGN(9).md`, revisión R3.1 del 2026-11-14; §§3.2, 3.11, 3.15, 3.17, 3.18 y 3.20. Archivo consultado desde los adjuntos del proyecto.
- Paquete TASK-02-DRAFT-2 publicado en `docs/crl-feasibility/009A.1/` de GitHub `main`, incluidos `TASK-02-DECISIONS-REQUIRED.md`, `TASK-02-DOMAIN-MODEL.md` y `TASK-02-DESIGN-VALIDATION.md`, con addendum de cierre/aprobación humana.
- `TASK-03-SPEC-DRAFT-1.md`, `TASK-03-PLAN-DRAFT-1.md`, `TASK-03-TASKS-DRAFT-1.md` y `TASK-03-DESIGN-VALIDATION-DRAFT-1.md`, leídos desde GitHub `main`.
- `SOURCE-BASIS-AND-LIMITATIONS.md` de DRAFT-1 y el informe TASK-01 compartido anteriormente (`Markdown pegado.md`).
- Historial conversacional donde se aprobaron D-01, D-02, D-03, D-VD03-5 y la preparación documental TASK-03.

## Aclaraciones normativas aplicadas

- R3.1 §3.15 fija el orden append → journal force → checkpoint temporal → force temporal → close → `ATOMIC_MOVE` → intento de directory force si soportado → ACK; define al checkpoint como punto de transición confirmada.
- R3.1 §3.15 detalla recuperación de tail para ADMIT y COMPLETE. DRAFT-2 no extrapola esa política a otros tipos.
- R3.1 §3.17 define `recoveryKey=(originalEventId,recoveryAction)` y `recoveryEventId=SHA-256(canonical(recoveryKey))`; contempla `REBIND_UNSTARTED_ADMISSION(targetProcessBootId)`, que requiere evidencia Member.
- R3.1 §3.18 exige `HANDLER_ENTERED` durable antes del cuerpo GET; el Member ledger está fuera de TASK-03.
- R3.1 §3.2 exige avanzar y persistir `coordinatorEpoch` en restart antes de habilitar operaciones. La identidad del evento/clave y recuperación ante corte de esa transición no están cerradas en la documentación disponible; DRAFT-2 los deja como decisiones.
- TASK-02 aprobó el punto de consumo D-01, las distinciones D-02, la persistencia local D-03 y el rechazo por expiry D-VD03-5. TTL/reloj/skew/renovación pre-consumo siguen pendientes.

## Estado de evidencia

| Tema | Evidencia | Clasificación |
|---|---|---|
| Diseño fuente | R3.1, archivo `(9)`, §§ citadas | Fuente normativa |
| Decisiones TASK-02 | Paquete publicado y addendum aprobado | Decisión humana cerrada |
| Workspace 009A.1 | Informe TASK-01 estático anterior | Debe revalidarse read-only antes de implementación |
| Código GitHub | Repositorio público contiene documentación, no `pom.xml` ni Java del arnés | No disponible en repo |
| ZIP histórico 009A | Evidencia de baseline anterior | No acredita identidad del workspace 009A.1 |
| Windows filesystem | FileLock/ATOMIC_MOVE/directory force no ensayados | No validado; requiere autorización específica |
| Builds/tests DRAFT-2 | Ninguno ejecutado | No ejecutados |

## Limitaciones y prohibiciones de inferencia

- No atribuir al repositorio GitHub el código del laboratorio 009A.1.
- No presentar el ZIP 009A ni dependency tree artifact 008 como baseline exacto 009A.1.
- No afirmar compatibilidad Windows de FileLock, atomic move o directory force sin prueba del filesystem destino.
- No afirmar durabilidad frente a pérdida eléctrica/hardware por el hecho de usar `force(true)`.
- No presentar hash-chain/checkpoint local como defensa contra reemplazo por copia antigua íntegra.
- No inventar políticas tail para FENCE, START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL o epoch.
- No ejecutar REBIND sin ledger Member y evidencia verificable.
- No fijar TTL, reloj, skew ni renovación pre-consumo.
- No ejecutar Maven, tests, builds, experimentos o modificaciones de código dentro de esta preparación.
