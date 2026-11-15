# TASK-03 — Aceptación humana de la corrección H-01

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión aceptada:** TASK-03-DRAFT-3, corrección focalizada H-01  
**Fecha de aceptación:** 2026-11-15  
**Autoridad:** aceptación humana explícita registrada en la conversación del proyecto.  
**Estado de H-01:** `ACCEPTED — HUMAN APPROVED`

## 1. Alcance de la aceptación

Se acepta la corrección documental H-01 incorporada en TASK-03-DRAFT-3. La aceptación cubre la alineación de SPEC §11, PLAN T03-09, TASKS T03-09 y DESIGN-VALIDATION respecto de la serialización lógica de:

- ADMIT;
- FENCE;
- HANDLER_START_INTENT;
- ADMISSION_REJECTED_EXPIRED.

También se acepta la aclaración que distingue el OS lock de vida del proceso Coordinator de la frontera de serialización lógica intra-proceso.

## 2. Reglas aceptadas

1. Un ADMIT confirmado antes de FENCE conserva el derecho a completar su ciclo.
2. HANDLER_START_INTENT puede confirmarse después de FENCE para esa admisión previa si el Permit sigue vigente en el punto de commit durable.
3. Si el Permit expiró antes del commit de START_INTENT, debe persistirse ADMISSION_REJECTED_EXPIRED. El bloqueo solo se libera tras confirmación durable; fallo o ambigüedad mantiene el bloqueo.
4. Un ADMIT nuevo posterior a FENCE se rechaza.
5. Un START_INTENT confirmado antes de FENCE no se cancela ni se revalida por FENCE.
6. La validación D-01 y la persistencia de START_INTENT o del rechazo son una sola operación lógica serializada; las carreras deben verificarse con barreras/latches deterministas.

## 3. Límites expresos

Esta aceptación **no** constituye aprobación integral del diseño TASK-03 ni autorización de implementación.

- D03-OPEN-01..08 permanecen abiertos.
- En particular, D03-OPEN-02 continúa pendiente: no se aprueba ninguna política automática de tail para HANDLER_START_INTENT ni ADMISSION_REJECTED_EXPIRED. Si aparecen sin checkpoint, se mantiene RECOVERY_REQUIRED hasta decisión explícita.
- TTL, fuente/instante de reloj, precisión, skew y renovación pre-consumo siguen pendientes.
- No se autoriza modificar código/configuración, ejecutar Maven, builds, tests ni experimentos.
- El workspace 009A.1 debe revalidarse read-only (T03-00) antes de cualquier implementación.
- La aprobación de H-01 no aprueba TASK-04 ni tareas posteriores y no abre el gate CRL de producción.

## 4. Estado resultante

- Corrección H-01: `ACCEPTED — HUMAN APPROVED`.
- TASK-03 diseño integral: `DRAFT — PENDING HUMAN REVIEW / OPEN DECISIONS`.
- Implementación TASK-03: `NOT AUTHORIZED`.
- Gate CRL de factibilidad para producción: `CLOSED`.

Este addendum registra únicamente la aceptación humana explícita de H-01 y no modifica retroactivamente el contenido de los documentos snapshot DRAFT-3.
