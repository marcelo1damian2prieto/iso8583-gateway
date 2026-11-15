# TASK-03-DRAFT-4 — Corrección de baseline y provisión de workspace

Este paquete conserva la corrección H-01 ya aceptada y corrige la trazabilidad del baseline de TASK-01 y el estado real de provisión de 009A.1. No cambia decisiones de dominio ni autoriza copiar o implementar el proyecto.

## Documentos

- `TASK-03-SPEC-DRAFT-4.md`: corrige §3; separa informe TASK-01 de 009A y workspace aislado 009A.1.
- `TASK-03-PLAN-DRAFT-4.md`: incorpora el gate previo de provisión y la precondición de T03-00.
- `TASK-03-TASKS-DRAFT-4.md`: ajusta T03-00 y sus reglas de bloqueo.
- `TASK-03-DESIGN-VALIDATION-DRAFT-4.md`: incorpora el bloqueo de baseline en la validación.
- `TASK-03-FINAL-APPROVAL-REVIEW-DRAFT-4.md`: dictamen documental actualizado y estado T03-00.
- `SOURCE-BASIS-AND-LIMITATIONS-DRAFT-4.md`: evidencia, procedencia y límites.
- `WORKSPACE-PROVISIONING-REQUIREMENTS-DRAFT-4.md`: requisitos para una eventual copia aislada; no autoriza ejecutarla.
- `SHA256SUMS-DRAFT-4.md`: SHA-256 de los ocho documentos anteriores.

## Baseline y provisión

- El informe TASK-01 inspeccionó `009a\\project`, no `009a1\\project`.
- La captura del 2026-11-15 muestra la raíz 009A.1 con ocho documentos Markdown y sin subdirectorio `project`; el usuario confirmó que eso es todo su contenido.
- T03-00 está `BLOCKED — WORKSPACE NOT PROVISIONED`.
- Provisionar `009a1\\project` requiere un plan y autorización humana separados. T03-00 seguirá siendo read-only.

## Corrección H-01 preservada

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

H-01 permanece `ACCEPTED — HUMAN APPROVED`. D03-OPEN-01..08 siguen abiertas. T03-00 permanece bloqueada. No se autoriza implementación, copia/provisión del workspace, Maven, builds, tests ni experimentos.

Este commit publica documentos para revisión; no es aprobación del diseño ni autorización de implementación. La eventual aprobación de diseño y la autorización independiente para implementar son gates separados. El gate CRL productivo permanece cerrado.
