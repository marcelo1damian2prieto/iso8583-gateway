# TASK-03-DRAFT-2 — Paquete de diseño para revisión

Este paquete es la segunda revisión documental de TASK-03 para CRL-FEASIBILITY-009A.1. Los cuatro documentos están publicados/previstos bajo `docs/crl-feasibility/009A.1/` en el repositorio `iso8583-gateway` y continúan siendo borradores para revisión humana; su publicación en GitHub no equivale a aprobación.

## Documentos

- `TASK-03-SPEC-DRAFT-2.md`: especificación de persistencia, catálogo de eventos, tail, startup/epoch y decisiones abiertas.
- `TASK-03-PLAN-DRAFT-2.md`: etapas alineadas con T03-00..T03-12 y grafo de dependencias.
- `TASK-03-TASKS-DRAFT-2.md`: backlog con identificadores, prioridades y criterios de salida alineados con el plan.
- `TASK-03-DESIGN-VALIDATION-DRAFT-2.md`: invariantes, escenarios positivos/negativos/borde, crash matrix y trazabilidad.
- `SOURCE-BASIS-AND-LIMITATIONS-DRAFT-2.md`: fuentes consultadas y límites de evidencia.
- `SHA256SUMS-DRAFT-2.md`: hashes SHA-256 de los cinco documentos anteriores y este README.

## Estado

`DRAFT — PENDING HUMAN REVIEW`.

La autorización vigente cubre preparación y validación del diseño. No habilita modificaciones de código/configuración funcional, Maven, tests, builds ni experimentos. La implementación requiere completar la validación y obtener aprobación humana explícita independiente.

## Correcciones respecto de DRAFT-1

1. PLAN y TASKS comparten los mismos identificadores T03-00..T03-12.
2. Idempotencia pasa a P0 y se ubica antes del tail recovery.
3. Se incorpora un catálogo de eventos Coordinator y se limita la recuperación automática a ADMIT/COMPLETE conforme R3.1.
4. Se agrega el avance durable de `coordinatorEpoch` como requisito de startup, dejando su contrato/crash policy como decisión pendiente.
5. Se distingue directory force unsupported de un error real/ambiguo después de `ATOMIC_MOVE`.
6. Se agregan escenarios de expiry D-VD03-5, epoch, tails no cubiertos, temporales residuales y REBIND condicionado al Member ledger.
7. Se agrega una matriz de trazabilidad requisitos → reglas → escenarios → evidencia.

## Revisión recomendada

Revisar como conjunto SPEC, PLAN, TASKS y DESIGN-VALIDATION. Resolver D03-OPEN-01..08 o aceptar explícitamente los bloqueos antes de congelar el diseño. No autorizar implementación por el solo hecho de revisar o publicar estos documentos.
