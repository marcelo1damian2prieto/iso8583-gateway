# TASK-03-DRAFT-3 — Corrección focalizada H-01

Este paquete publica una corrección documental acotada a H-01 sobre la serialización lógica de ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED. Parte de DRAFT-2 y conserva sus decisiones abiertas, límites y exclusiones.

## Documentos

- `TASK-03-SPEC-DRAFT-3.md`: amplía §11, aclara §6.3 (OS lock frente a serialización de operaciones), actualiza FR-011 y aceptación.
- `TASK-03-PLAN-DRAFT-3.md`: amplía T03-09 y su estrategia de concurrencia.
- `TASK-03-TASKS-DRAFT-3.md`: actualiza el alcance/criterio de salida de T03-09.
- `TASK-03-DESIGN-VALIDATION-DRAFT-3.md`: agrega los seis escenarios H-01 y trazabilidad.
- `TASK-03-FINAL-APPROVAL-REVIEW-DRAFT-3.md`: dictamen de revisión cruzada y preparación para aprobación humana.
- `SOURCE-BASIS-AND-LIMITATIONS-DRAFT-3.md`: procedencia normativa y límites.
- `SHA256SUMS-DRAFT-3.md`: SHA-256 de los siete documentos anteriores.

## Corrección H-01

1. ADMIT confirmado antes de FENCE permanece como admisión previa que puede completar su ciclo.
2. START_INTENT posterior a FENCE para esa admisión se permite si el Permit sigue vigente en el punto de commit durable.
3. Expiry anterior al commit produce ADMISSION_REJECTED_EXPIRED; solo el rechazo durable libera el bloqueo. Fallo/ambigüedad mantiene el bloqueo.
4. FENCE anterior a un ADMIT nuevo hace que el ADMIT se rechace.
5. START_INTENT confirmado antes de FENCE no se cancela ni se revalida por FENCE.
6. Las cuatro transiciones comparten una frontera lógica intra-proceso con orden total, verificada mediante barreras/latches deterministas.

La separación OS lock (exclusión de otro Coordinator durante la vida del proceso) vs serialización lógica de operaciones (orden intra-proceso) queda explícita.

## Límites preservados

- La política tail para HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED permanece abierta en D03-OPEN-02. No se autoriza recuperación automática de esos tails.
- D03-OPEN-01..08 siguen abiertos; TTL/reloj/skew/renovación pre-consumo también siguen pendientes.
- No se altera el scope de T03-00..T03-12 fuera del ajuste de T03-09.
- No se modifican código, configuración funcional ni artefactos principales del Gateway.
- No se ejecutan Maven, tests, builds ni experimentos.

## Estado

`DRAFT — PENDING HUMAN REVIEW`.

Este commit publica documentos para revisión; no es aprobación del diseño ni autorización de implementación. La eventual aprobación de diseño y la autorización independiente para implementar son gates separados. El gate CRL productivo permanece cerrado.
