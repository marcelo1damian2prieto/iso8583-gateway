# CRL-FEASIBILITY-009A.1 — Índice de actualización documental TASK-03 DRAFT-5

**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Base:** TASK-03 DRAFT-4, commit `bef7aa7c615745e19f548b47854fb4e37d543b18` de `main`.

## 1. Propósito

Este paquete consolida, mediante enmiendas aditivas, la aceptación humana de las ocho propuestas conservadoras D03-OPEN-01..08, el cierre/aceptación de T03-00 y el procedimiento acordado para analizar incompatibilidades antes de diseñar un posible refactor.

DRAFT-4 permanece intacto como snapshot histórico. Las enmiendas de este paquete actualizan los estados y secciones que enumeran; cuando haya contradicción, prevalece el addendum DRAFT-5 aplicable. El contenido no alcanzado por las enmiendas continúa vigente.

## 2. Archivos

1. `TASK-03-SPEC-DRAFT-5-DECISION-ADDENDUM.md` — baseline T03-00, decisión matriz, políticas de tail/epoch/directory force/temporales/recovery states/REBIND y gobernanza de incompatibilidades.
2. `TASK-03-PLAN-DRAFT-5-DECISION-ADDENDUM.md` — estado de T03-00/T03-01, precondiciones y gates actualizados, secuencia y estrategia de incompatibilidades.
3. `TASK-03-TASKS-DRAFT-5-DECISION-ADDENDUM.md` — estados de tareas y criterios de salida actualizados.
4. `TASK-03-DESIGN-VALIDATION-DRAFT-5-DECISION-ADDENDUM.md` — nueva validación documental de requisitos, escenarios, decisiones, riesgos y gates.

## 3. Aceptaciones relacionadas

Este paquete debe conservar como registros fuente:
- `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1.md` — catálogo de dominio y propuestas originales.
- `TASK-03-01-HUMAN-ACCEPTANCE-ADDENDUM-1.md` — aceptación humana de las ocho propuestas y reservas.
- `TASK-03-01-COLLISION-ANALYSIS-AND-REFACTOR-PLANNING-CLARIFICATION-1.md` — aclaración posterior que reemplaza la interpretación de refactor mínimo inmediato.
- `T03-00-BASELINE-INSPECTION.md` y `WORKSPACE-PROVISIONING-MANIFEST.json` — evidencia de baseline y provisión.

## 4. Decisiones efectivas

- D03-OPEN-01: dirección aprobada, especificación detallada abierta.
- D03-OPEN-02: cerrada; recuperación automática de tail solo ADMIT/COMPLETE.
- D03-OPEN-03: dirección aprobada, detalle de startup epoch abierto; tail epoch detiene.
- D03-OPEN-04: política de clasificación aprobada, evidencia de plataforma abierta.
- D03-OPEN-05: cerrada; temporales residuales detectados y no alterados automáticamente.
- D03-OPEN-06: dirección aprobada, matriz exhaustiva/lock detail abierta.
- D03-OPEN-07: gate metodológico aprobado; experimento Windows no autorizado.
- D03-OPEN-08: boundary y precondiciones aprobadas; evidencia Member pertenece a TASK-04.

D-VD03-1..4, D-ID-1, D-EVIDENCE-1, D-FS-1 y D-WIRE-1 permanecen pendientes. D-01/D-02/D-03/D-VD03-5/H-01 preservadas.

## 5. Procedimiento de incompatibilidades

Analizar cada incompatibilidad individualmente con evidencia, confirmar su naturaleza, mapear impacto y evaluar alternativas. No presuponer refactor. Después del análisis de las incompatibilidades relevantes, preparar un plan consolidado de refactor si corresponde, para revisión/aprobación humana. La implementación de cualquier refactor requiere autorización separada.

## 6. Gates

- T03-00 completada y aceptada como baseline estático con limitación de acceso directo al filesystem.
- T03-01 decisiones aceptadas a nivel de dirección/política; detalles de freeze indicados siguen abiertos.
- Implementación T03-02..T03-12: no autorizada.
- Experimento Windows FileLock/ATOMIC_MOVE/directory force: no autorizado.
- TASK-04 y tareas posteriores: no autorizadas.
- Gate de factibilidad CRL para producción: cerrado.
- No se ejecutaron Maven/build/tests/experimentos; no se modificó código/configuración.

## Publicación

Los once archivos de esta revisión están disponibles individualmente en este directorio de GitHub. El manifiesto de provisión fue saneado únicamente en las rutas absolutas locales; se conservaron inventario, tamaños y SHA-256 declarados. El ZIP de revisión descargable no forma parte de este directorio.
