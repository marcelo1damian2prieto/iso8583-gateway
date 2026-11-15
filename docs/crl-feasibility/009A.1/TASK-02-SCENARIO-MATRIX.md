> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Matriz de escenarios

**Fuentes:** diseño fuente R3.1 §§3.8–3.18, 4–9; decisiones humanas TASK-02 D-01 a D-03; TASK-01 VIABLE CON CONDICIONES en conversación; fuentes/logs 009A. El diseño local declara R3.1; la revisión aportada identifica la copia `(7)` también como R3.1. La referencia previa R3.1.1 fue errónea; fuente intacta. No existe informe TASK-01 aparte. **Herencia/decisiones:** fail-closed, GET sintético, vigencia consumida en START_INTENT durable, PERMIT_RECEIVED local durable, ENTERED-before-body, COMPLETE write servidor, UNKNOWN no retry, R3 recovery idempotente. TTL/reloj/tolerancia pendientes. La matriz no se ejecutó.

| ID | Estado inicial / inyección | Secuencia | Resultado y estado final | Evidencia / aceptación |
|---|---|---|---|---|
| S01 | GLOBAL_ACTIVE G1 | ADMIT durable; después FENCE | admission previo queda pending hasta terminal/reconcile; coordinator FENCING | journal ordena ADMIT antes FENCE; permiso único, handler ≤1 |
| S02 | ADMIT concurrente con FENCE | barrera libera contendientes | orden total: ADMIT primero => in-flight drain; FENCE primero => reject | no ADMIT G1 confirmado después de FENCE |
| S03 | FENCE durable | intentar ADMIT G1 | reject antes body; no ADMITTED | seq FENCE y handler count 0 |
| S04 | ADMIT durable, response descartada | retry mismo TransitionKey y STATUS | mismo admissionId, sin segundo evento lógico | journal, receipt local e invocations |
| S05 | write HTTP completado; COMPLETE ACK perdido | retry/query misma clave | COMPLETE único solo con evidencia del write; si incierto UNKNOWN | callback Reactor write + event/checkpoint correlacionados |
| S06 | PERMIT_RECEIVED sin START_INTENT | crash member; recover | NOT_STARTED solo con journals/ledger íntegros, boot muerto, sin tail/corrupción/UNKNOWN; si no UNKNOWN | PID/boot, ledger local durable de receipt, journal; rebind explícito solo si demostrado |
| S07 | HANDLER_START_INTENT durable; caída después del ACK, antes o tras posible llamada | crash del miembro en frontera START_INTENT | UNKNOWN/RECOVERY_REQUIRED, sin reejecución automática | intent durable, terminal ausente, invocations ≤1; ausencia de ENTERED no prueba NOT_STARTED |
| S08 | HANDLER_ENTERED durable | crash dentro del GET | UNKNOWN; no inferir RETURNED/COMPLETE ni reejecutar | ENTERED sin RETURNED/COMPLETE |
| S09 | coordinator con admissions pendientes | restart/replay | RECOVERING/gates closed; conservar pending; consultar mismo boot/status; no commit | journal/checkpoint, pending, cero nuevo ADMIT |
| S10 | permiso de boot anterior | member reinicia y presenta grant viejo | BOOT_CLOSED, bootId nuevo, permiso no heredado | boot IDs y reject del grant viejo |
| S11 | member no registrado | ADMIT/PREPARE unknown member | reject; member cerrado; no barrier/commit | manifest exacto y rechazo |
| S12 | epoch desactualizado | command/permit viejo o ACK tardío | reject; respuesta vieja no revierte estado | epoch/command/event order invariantes |
| S13 | hash CRL distinto | PREPARE/ADMIT hash conflictivo | reject; no activation/commit | digest esperado vs actual |
| S14 | journal/checkpoint corrupto/incompleto | startup replay | FAILED_CLOSED/RECOVERY_REQUIRED según clase; sin auto-truncate/permiso | bytes, seq/hash, checkpoint, estado cerrado |
| S15 | recovery event ya durable | repetir misma recoveryKey/action | reusa eventId, sin duplicar compensación | originalEventId/key/eventId/payload iguales |
| S16 | recovery key/eventId con payload distinto | replay conflictivo | FAILED_CLOSED | conflict digest y ausencia de servicio |
| S17 | Permit expira antes de confirmación durable de HANDLER_START_INTENT | control temporal de prueba; luego solicitar intent | Coordinator rechaza START_INTENT; persiste ADMISSION_REJECTED_EXPIRED y confirma después del commit durable; estado REJECTED_NOT_STARTED, sin HANDLER_ENTERED/body y ya no bloquea FENCE/COMMIT_GLOBAL | evidencia de expiry al boundary, evento correlacionado al ADMIT, ACK posterior a force/checkpoint, invocation count=0, blocksCommit=false |
| S18 | Permit vigente al commit durable de HANDLER_START_INTENT; expira después, incluso antes de ENTERED/body o durante handler | barreras tras intent y antes/dentro del body; avanzar tiempo | Intent consumió permiso; expiry posterior no cancela y no se revalida en ENTERED/body; seguir flujo normal, sin retry | orden temporal vs commit, intent durable, handler invocation ≤1; server write aparte |
| S19 | coordinator inaccesible antes de ADMIT | cortar canal | deny fail-closed, handler=0 | error control, denied count |
| S20 | pérdida de conectividad al coordinator tras ADMIT, antes o después del commit START_INTENT | cortar canal en cada frontera | antes de intent confirmado: no se puede invocar body; después de intent confirmado: permiso consumido, expiry/canal perdido no cancela ejecución; terminal no confirmado permanece pending/UNKNOWN; nuevos ADMIT denegados | ledger in-flight, error de transporte, transición START_INTENT durable/ausente y recovery; timeout no COMPLETE |
| S21 | FENCE con request bloqueado | FENCE, luego liberar latch | FENCING espera terminal/reconcile; sin GLOBAL_ACTIVE mientras pending | barrier/event order y ausencia commit previo |
| S22 | PENDING o UNKNOWN no reconciliado en ledger | intentar COMMIT_GLOBAL | Rechazar; permanece FENCING/closed | state bloqueante presente y ningún COMMIT_GLOBAL |
| S23 | UNKNOWN con evidencia completa de reconciliación | correlacionar exactamente admission/member/boot; demostrar durablemente que proceso murió y no puede continuar; journal/checkpoint y ledger íntegros, sin tail/corrupción/contradicción ni otro boot ejecutable; persistir RecoveryRecord idempotente | pasa a RECONCILED_UNKNOWN: outcome sigue desconocido, deja de bloquear commit solo si se cumplen todas las condiciones; no COMPLETE | evidence de PID+boot, ledger/journal íntegros, recoveryKey/recoveryEventId, `blocksCommit=false`, sin evento COMPLETE |
| S24 | fallo persistiendo HANDLER_ENTERED | inyectar fallo antes del body | GET no invocado | error persistencia e invocation count=0 |
| S25 | PERMIT_RECEIVED ledger local falla | hacer fallar append/force antes de continuar | Member no confirma recepción ni solicita START_INTENT; body no ejecutado; RECOVERY_REQUIRED si persistencia parcial/ambigüa | sin ACK receipt; error y ledger/seq/hash; invocation=0 |
| S26 | STATUS/ACK tardío de boot/epoch viejo | entregar tras transición posterior | solo correlacionar; no rollback de estado | epoch/boot y estado corriente intactos |
| S27 | rechazo por expiry, caída antes del append | solicitar START_INTENT vencido; detener Coordinator antes de append | no hay rechazo durable confirmado; admisión PENDING/bloqueante; no body | sin ACK de rechazo; handler count=0; commit negado |
| S28 | rechazo por expiry, append parcial/ambiguo | inyectar fallo durante append/force | no inferir rechazo durable; conservar bloqueante y fail-closed | journal/tail y STATUS/replay; ningún ACK confirmado; commit negado |
| S29 | rechazo por expiry, caída tras force antes del checkpoint | caer tras force y antes del checkpoint | no liberar hasta reconstruir/confirmar evento durable | evento correlacionado íntegro, replay idempotente, estado bloqueante hasta confirmación |
| S30 | rechazo por expiry, caída tras checkpoint antes del ACK | caer después de checkpoint | evento REJECTED_NOT_STARTED durable; STATUS misma clave devuelve rechazo; no duplicar evento | evento/checkpoint, transition key, body count=0 |
| S31 | ACK de rechazo perdido en transporte | descartar ACK tras persistencia | no tratar timeout como fallo; STATUS misma clave confirma evento/estado durable | mismo admissionId/transition key, un evento, estado REJECTED_NOT_STARTED |
| S32 | repetir rechazo con misma clave y payload | reenviar solicitud idéntica | respuesta idempotente, mismo resultado durable sin transición duplicada | un evento lógico durable, mismo digest y estado |
| S33 | misma clave de rechazo con payload conflictivo | cambiar digest/correlación para clave existente | rechazar conflicto, fail-closed; conservar evidencia y no fabricar otro rechazo | conflict digest, journal intacto y sin segundo evento lógico |
| S34 | carrera entre expiry-reject y FENCE | barrera determinista para serializador ADMIT/intent/reject/FENCE | orden único; FENCE espera rechazo durable cuando este gana; si FENCE gana, no intent viejo; no commit con rechazo ambiguo | orden journal, ACK tras durable, blocksCommit hasta confirmación |
| S35 | rechazo confirmado y verificar GET no invocado | instalar contador/latch del cuerpo; provocar expiry y esperar ACK durable | REJECTED_NOT_STARTED sin HANDLER_ENTERED, GET ni COMPLETE | evento, estado terminal y contador de invocaciones=0 |
| S36 | rechazo durable y COMMIT_GLOBAL | confirmar rechazo, luego intentar commit | admisión rechazada ya no bloquea; COMMIT_GLOBAL puede avanzar solo si los demás criterios se satisfacen | evento durable y correlacionado; blocksCommit=false; ningún COMPLETE sintético |

## Criterio común de prueba

Barreras/latches deterministas, estado journal/checkpoint y ledger antes/después, correlation IDs + boot/epoch, y contador de invocaciones. No usar `sleep` para establecer orden. Timeout es límite de fallo, no sincronización ni terminal. Una respuesta HTTP escrita por servidor no prueba recepción del cliente. RequestId nuevo sin clave de negocio no se asocia a UNKNOWN. RECONCILED_UNKNOWN nunca equivale a COMPLETE. S17–S18 prueban D-01 y S27–S36 el rechazo durable/idempotente y sus fallos; medir tiempo real requiere aún resolver TTL, reloj y tolerancia, sin extrapolar a operaciones financieras.

---

## Addendum de estado — cierre TASK-02 y autorización TASK-03

**Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
**Autoridad:** aprobación humana explícita registrada en la conversación del proyecto.  
**Estado de TASK-02:** `CLOSED — HUMAN APPROVED`.  
**Estado de TASK-03:** `AUTHORIZED — DESIGN PREPARATION AND VALIDATION ONLY`.

La aprobación humana cierra la revisión documental TASK-02-DRAFT-2. Las siete piezas de este paquete quedan aceptadas como documentación de referencia de TASK-02, conservando sus decisiones y trazabilidad.

Se autoriza iniciar TASK-03 para preparar su especificación, plan ejecutable, desglose de tareas y validación del diseño. Esta autorización **no** autoriza todavía modificaciones de código, configuración funcional, ejecución de implementación ni pruebas de integración que formen parte de la implementación. Antes de implementar se deberá completar la validación del diseño de TASK-03 y obtener aprobación humana explícita.

La autorización queda acotada a TASK-03. No autoriza TASK-04 ni tareas posteriores, ni modifica las decisiones arquitectónicas heredadas. En particular:

- Coordinator único de laboratorio; sin consenso ni dependencias externas nuevas.
- Alcance de GET sintético; no extrapolar semánticas a operaciones financieras.
- Fail-closed ante corrupción, ambigüedad o evidencia insuficiente.
- No afirmar durabilidad frente a pérdida de energía/hardware sin evidencia específica.
- Los parámetros temporales pendientes (TTL, reloj, tolerancia y renovación pre-consumo) no deben inventarse ni congelarse por inferencia.
- El gate de factibilidad CRL para producción permanece cerrado.

Este addendum prevalece sobre referencias históricas anteriores de estos documentos que indiquen que TASK-02 sigue pendiente o que TASK-03 no está autorizada. El contenido técnico histórico se conserva sin reescritura para mantener trazabilidad.
