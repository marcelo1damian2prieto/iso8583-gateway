# Paquete de revisión — CRL-FEASIBILITY-009A.1 / TASK-03 DRAFT-5

Este paquete combina cuatro enmiendas documentales DRAFT-5 con sus fuentes de aceptación y baseline. Las enmiendas no borran ni sustituyen físicamente los documentos DRAFT-4; establecen precedencia para las materias que actualizan. La lectura normativa es DRAFT-4 + enmiendas DRAFT-5.

## Enmiendas DRAFT-5

- `TASK-03-SPEC-DRAFT-5-DECISION-ADDENDUM.md`
- `TASK-03-PLAN-DRAFT-5-DECISION-ADDENDUM.md`
- `TASK-03-TASKS-DRAFT-5-DECISION-ADDENDUM.md`
- `TASK-03-DESIGN-VALIDATION-DRAFT-5-DECISION-ADDENDUM.md`
- `TASK-03-DRAFT-5-REVIEW-INDEX.md`

## Fuentes de aceptación y baseline

- `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1.md`
- `TASK-03-01-HUMAN-ACCEPTANCE-ADDENDUM-1.md`
- `TASK-03-01-COLLISION-ANALYSIS-AND-REFACTOR-PLANNING-CLARIFICATION-1.md`
- `T03-00-BASELINE-INSPECTION.md`

## Estado y límites

- T03-00: completada y aceptada como inspección estática read-only, con limitación de acceso directo al filesystem.
- D03-OPEN-01..08: aceptadas en los niveles y con las reservas enumeradas en el addendum.
- Diseño integral TASK-03: sigue pendiente de revisión humana; no se considera freeze aprobado.
- Implementación T03-02..T03-12: no autorizada.
- Experimento Windows FileLock/ATOMIC_MOVE/directory force: no autorizado.
- TASK-04 y tareas posteriores: no autorizadas.
- Gate CRL de producción: cerrado.

No se modificó código/configuración ni se ejecutaron Maven, builds, tests o experimentos.

## Publicación en GitHub

Los documentos de esta revisión se publican individualmente en este directorio. El manifiesto público conserva inventario y hashes, pero sustituye las rutas absolutas locales por `%TEMP%` para no divulgar nombres de usuario ni rutas privadas del equipo.
