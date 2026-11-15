# TASK-03 — Enmienda DRAFT-5 al desglose TASKS-DRAFT-4

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-TASKS-DRAFT-5-DECISION-ADDENDUM`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Documento base:** `TASK-03-TASKS-DRAFT-4.md`  
**Base Git:** `bef7aa7c615745e19f548b47854fb4e37d543b18`.

## 1. Efecto

Esta enmienda es aditiva y actualiza el estado operativo del backlog y los criterios de salida de TASKS-DRAFT-4. Los IDs, prioridades y grafo T03-00..T03-12 se conservan. En caso de conflicto prevalece esta enmienda para los estados que contiene. La revisión combinada se denomina `TASKS-DRAFT-5`.

## 2. Estado de tareas

| ID | Estado actualizado | Evidencia / reserva |
|---|---|---|
| T03-00 | `COMPLETED — STATIC READ-ONLY BASELINE WITH ACCESS LIMITATION` | Informe `T03-00-BASELINE-INSPECTION.md` aceptado por el usuario. Manifiesto con 31 archivos; 31/31 tamaño+SHA-256 contrastados con `project.zip`; sin `.git`; 21 Java bajo `src/test/java/lab/crl`; no se ejecutó Maven/build/test/experimento. Se mantiene la limitación de no acceso directo al disco C: desde esta sesión. |
| T03-01 | `DOCUMENTARY CATALOG ACCEPTED — DETAIL FREEZE PARTIALLY OPEN` | Aprobación humana de las ocho propuestas D03-OPEN-01..08 con estados diferenciados y reservas. La tabla tail queda cerrada; el codec, epoch, clasificación de estados/locks y contrato detallado conservan puntos abiertos. El catálogo no autoriza implementación. |
| T03-02 | `NOT STARTED — DESIGN DETAIL REQUIRED` | Antes de congelar codec definir tags, anchos, byte order, domain separators, nullability, límites, unknown fields, genesis hash y compatibilidad/versiones. |
| T03-03 | `NOT STARTED — CONTRACT / PLATFORM EVIDENCE REQUIRED` | El contrato OS lock debe mantenerse local. Evidencia efectiva Windows requiere autorización experimental independiente. |
| T03-04 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Append completo, force, secuencia/hash-chain y dedupe; respetar D03-OPEN-04/05. |
| T03-05 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | `CREATE_NEW` mismo directorio/filesystem, force/close/ATOMIC_MOVE; sin fallback; distinguir directory-force outcomes; detectar temporales residuales sin promoción/borrado automático. |
| T03-06 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Reducer determinista según eventos y snapshot aprobados. |
| T03-07 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Replay con tail allowlist estricta ADMIT/COMPLETE; otros tipos, incluido epoch, `RECOVERY_REQUIRED`. |
| T03-08 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Recovery idempotente solo ADMIT/COMPLETE bajo R3.1; no extrapolar. |
| T03-09 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Preservar H-01: orden total lógico ADMIT/FENCE/START_INTENT/rechazo expiry; separar OS lock de serialización intra-proceso. |
| T03-10 | `NOT STARTED — IMPLEMENTATION NOT AUTHORIZED` | Idempotencia P0; transitionKey/recoveryKey; retry y payload conflictivo. |
| T03-11 | `NOT STARTED — EXPERIMENTS NOT AUTHORIZED` | Crash matrix propuesta; experimentos Windows separados requieren aprobación específica, aislamiento y rollback. |
| T03-12 | `NOT STARTED` | Cierre solo tras evidencia verificable y revisión humana; no habilita TASK-04 ni producción. |

## 3. Resultado de T03-01 y reservas

El usuario aceptó las direcciones y salvaguardas de D03-OPEN-01..08. Los estados exactos son:

- D03-OPEN-01: dirección de codec binario aprobada; especificación detallada abierta.
- D03-OPEN-02: política de tail cerrada; automatización solo para ADMIT y COMPLETE.
- D03-OPEN-03: dirección de epoch aprobada; detalles del contrato de startup abiertos.
- D03-OPEN-04: política de clasificación aprobada; evidencia específica de plataforma abierta.
- D03-OPEN-05: política de temporales residuales cerrada.
- D03-OPEN-06: dirección de clasificación de estados aprobada; tabla exhaustiva/lock detail abierta.
- D03-OPEN-07: gate metodológico aprobado; experimento Windows no autorizado.
- D03-OPEN-08: frontera y precondiciones de REBIND aprobadas; evidencia Member queda en TASK-04.

D-VD03-1..4, D-ID-1, D-EVIDENCE-1, D-FS-1 y D-WIRE-1 siguen pendientes. D-01/D-02/D-03/D-VD03-5 y H-01 no se alteran.

## 4. Nuevas reglas para la ejecución futura

1. No congelar un contrato técnico a partir de una dirección de diseño aceptada si faltan detalles normativos que afecten bytes, compatibilidad o replay.
2. Tail recovery automático queda limitado a ADMIT/COMPLETE; el resto de tipos en tail mantiene el gate cerrado `RECOVERY_REQUIRED`.
3. Temporales residuales se detectan, registran y no se promueven, reemplazan ni eliminan automáticamente.
4. `FAILED_CLOSED` no tiene auto-reset; la recuperación requiere intervención controlada y baseline verificable.
5. No ejecutar pruebas reales del filesystem Windows sin el gate experimental D03-OPEN-07 autorizado por separado.
6. TASK-03 define la referencia de REBIND, pero no implementa ni ejecuta Member ledger/REBIND. TASK-04 sigue separado y no autorizado.

## 5. Procedimiento de incompatibilidades y refactor

Ante cada posible colisión:

- registrar evidencia exacta;
- clasificarla como incompatibilidad demostrada, riesgo hipotético, falta de información o diferencia estructural;
- mapear requisitos/invariantes/contratos/estados/datos persistidos/seguridad/replay/idempotencia afectados;
- evaluar alternativas, incluida la opción de no refactor;
- no alterar silenciosamente decisiones aceptadas ni relajar garantías;
- analizar las incompatibilidades relevantes individualmente antes de crear un plan de refactor consolidado.

El plan consolidado deberá especificar inventario, evidencia, objetivo/alcance, trazabilidad, alternativas, secuencia/dependencias, compatibilidad de formatos/datos, replay/migración/rollback, impactos de idempotencia/concurrencia/fencing/seguridad, testing/aceptación, riesgos residuales y gates. El plan se revisa y aprueba humanamente antes de implementar. No se presume que todas las incompatibilidades requieran refactor ni que siempre corresponda el refactor mínimo.

## 6. Criterios de salida actualizados para T03-01

T03-01 podrá cerrarse como entrega documental cuando:

1. el catálogo de dominio/eventos separa Coordinator, Member y STATUS;
2. D03-OPEN-01..08 conserva los estados/alcances/reservas aceptados arriba;
3. el catálogo de tails define solo ADMIT/COMPLETE como recovery automático y deja explícitos los demás tipos en `RECOVERY_REQUIRED`;
4. la política de temporales y estados cerrados está expresada conforme a las aceptaciones;
5. H-01 y D-01/D-02/D-03/D-VD03-5 no fueron reinterpretadas;
6. los detalles de freeze restantes quedan enumerados y no se rellenan por inferencia;
7. Windows experiment remains unauthorized; no test/evidence claim is fabricated.

La finalización documental de T03-01 no autoriza T03-02 ni tareas posteriores; esas tareas requieren sus gates aplicables.

## 7. Gates de gobierno

- Implementación T03-02..T03-12: `NOT AUTHORIZED`.
- Experimento Windows D03-OPEN-07: `NOT AUTHORIZED`.
- TASK-04 y tareas posteriores: `NOT AUTHORIZED`.
- Gate de factibilidad CRL para producción: `CLOSED`.
- Sin modificaciones de código/configuración ni ejecución de Maven/build/tests/experimentos en esta entrega.