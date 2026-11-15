> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Máquinas de estados

**Fuentes:** diseño R3.1 §§3.1–3.20, 4–9; decisiones humanas TASK-02 posteriores a validación D-01 a D-03; dictamen TASK-01 VIABLE CON CONDICIONES registrado en conversación; fuentes Java/POM 009A. TASK-01 no se localizó como informe independiente; el dependency-tree disponible corresponde a artifact 008. El diseño local declara R3.1; la revisión aportada informa que la copia `(7)` también declara R3.1. La referencia R3.1.1 anterior fue errónea; fuente no modificada. **Herencia:** single coordinator lab, fail-closed, GET sintético, UNKNOWN no retry, ENTERED durable-before-body, COMPLETE solo server write. **Pendiente:** TTL/reloj/tolerancia/renovación; no iniciar TASK-03.

`D` = transición durable antes del ACK confirmatorio; `V` = estado derivado/verificado; `P` = estado de proceso no durable por sí mismo. Todo fallo/ambigüedad mantiene gate cerrado.

## 1. Coordinator

Estados heredados: `RECOVERING`, `IDLE`, `PREPARING`, `PREPARED_BARRIER`, `FENCING_OLD_EPOCH`, `ACTIVATING`, `VERIFYING`, `GLOBAL_ACTIVE`, `FAILED_CLOSED`.

| Origen | Evento | Precondición/validación | Efecto durable | Destino y fallo |
|---|---|---|---|---|
| proceso nuevo | START/REPLAY | lock exclusivo; validar journal/checkpoint/config | Replay seq/hash/pending; persistir nuevo epoch antes de operar | RECOVERING → IDLE solo íntegro y reconciliado; contradicción => FAILED_CLOSED, sin permiso |
| IDLE | BEGIN_ROLLOUT/PREPARE manifest | enrollment exacto, no vacío, hash/contexto válido, sin rollout conflictivo | manifest + ROLLOUT_REQUESTED/PREPARING | PREPARING; timeout no implica éxito/fallo, consultar misma key/STATUS |
| PREPARING | PREPARE_ACK/STATUS | miembro exacto, boot/epoch/hash/rollout válidos | ACK/observación idempotente | PREPARING; error => closed; todos exactos preparados => persistir PREPARED_BARRIER |
| PREPARING | barrier completa | membership/cardinalidad/hash exactos | evento durable de barrier | PREPARED_BARRIER; subset nunca basta |
| PREPARED_BARRIER/GLOBAL_ACTIVE previo | FENCE(oldEpoch) | serializado con ADMIT, rollout/epoch válido | persistir FENCE en mismo orden total ADMIT/FENCE; negar admissions antiguas | FENCING_OLD_EPOCH; ACK solo tras durable; ambiguo => STATUS/replay |
| FENCING_OLD_EPOCH | COMPLETE/RECONCILE/WAIT | admission conocido, evidencia válida, journal y ledger íntegros | COMPLETE válido o RECONCILED_UNKNOWN idempotente; actualizar conjunto bloqueante | sigue FENCING mientras haya PENDING/UNKNOWN sin reconciliar; timeout no resuelve |
| FENCING_OLD_EPOCH | drain resuelto | cero PENDING ni UNKNOWN no reconciliado; cada RECONCILED_UNKNOWN cumple criterios de reconciliación durable abajo | persistir barrera/epoch recovery aplicable | ACTIVATING; outcome desconocido permanece, no se reescribe a COMPLETE |
| ACTIVATING | ACTIVATE_INTENT/STATUS | miembro exacto, comando estable, target/boot/hash correctos | ACTIVATE_INTENT antes de envío; ACK/estado correlacionado durable | ACTIVATING; ACK perdido => ACK_UNKNOWN + STATUS same commandId; mismatch => FAILED_CLOSED |
| ACTIVATING | todos target | cada boot/member exacto autenticado | observaciones durable | VERIFYING; parcial => closed/recovery |
| VERIFYING | verificación integral | enrollment exacto, estado fresco, hash/epoch/gen correctos; cero PENDING/UNKNOWN no reconciliado; RECONCILED_UNKNOWN solo si satisface criterios R2/R3 | COMMIT_GLOBAL + checkpoint antes de permitir G2 | GLOBAL_ACTIVE; ACK después de commit durable. Reconciliado conserva resultado UNKNOWN |
| cualquier | corrupción, auth, contradicción, error durable | evidencia inválida o insuficiente | no emitir permiso; registrar si es seguro | FAILED_CLOSED/RECOVERY_REQUIRED |
| GLOBAL_ACTIVE | solicitud | member/boot/epoch/hash vigentes, no fence | máquina request serializada | continúa GLOBAL_ACTIVE; timeout resuelto por misma transition key |

No se afirma orden total porque `record()` sea `synchronized`: el mecanismo debe cubrir validación + estado + persistencia + decisión de ACK para ADMIT/FENCE.

## 2. Member

Estados heredados: `BOOT_CLOSED`, `ACTIVE_CLOSED`, `PREPARED_CLOSED`, `ACTIVATED_CLOSED`, `SERVING_AUTHORIZED`, `FENCED`, `FAILED_CLOSED`. Listener/contexto pueden existir sin habilitar handler. La durabilidad de fase requiere ledger member aún inexistente en 009A.

| Origen | Trigger | Validación/efecto | Destino/ACK | Fallo |
|---|---|---|---|---|
| arranque | BOOT | crear bootId nuevo; consultar coordinator; validar config/CRL/contexto sin abrir gate | BOOT_CLOSED; STATUS | coordinator ausente, hash/epoch/state inválido => FAILED_CLOSED |
| BOOT_CLOSED | enrollment/active válido | member, membership, boot, epoch, hash exactos | ACTIVE_CLOSED; reportar status; cada request aún requiere ADMIT | permanece cerrado |
| ACTIVE_CLOSED | PREPARE(target) | command idempotente; manifest/hash/protocolo/CRL/contexto validos | guardar prepared si durabilidad definida; PREPARED_CLOSED | timeout => STATUS misma key; mismatch => reject |
| PREPARED_CLOSED | FENCE | epoch/rollout aplicables; detener admisiones locales nuevas | FENCED | contradicción => FAILED_CLOSED |
| FENCED/PREPARED_CLOSED | ACTIVATE | intent/commandId target correctos; mantener gate cerrado; construir/bind/persistir | ACTIVATED_CLOSED; ACK/STATUS | error => FAILED_CLOSED |
| ACTIVATED_CLOSED | COMMIT_GLOBAL vigente | commit durable autenticado y boot/epoch/hash exactos | SERVING_AUTHORIZED; gate lógico solo para solicitar ADMIT por request | ACK/estado ambiguo => cerrado |
| cualquier | restart | bootId nuevo; nunca heredar permiso anterior | BOOT_CLOSED | journal/state mismatch => FAILED_CLOSED |
| estado activo | FENCE/mismatch/control denial | impedir trabajo nuevo; requests existentes siguen VD-03; no false COMPLETE | FENCED/FAILED_CLOSED | ambigüedad permanece pending |

## 3. Solicitud por AdmissionId

```text
ADMITTED → PERMIT_RECEIVED → HANDLER_START_INTENT → HANDLER_ENTERED
         → HANDLER_RETURNED → SERVER_WRITE_COMPLETED (COMPLETE)
         ↘ REJECTED_NOT_STARTED (ADMISSION_REJECTED_EXPIRED durable)
         ↘ UNKNOWN / RECOVERY_REQUIRED (sin retry automático)
```

| Transición | Regla durable | Fallo/recovery |
|---|---|---|
| none → ADMITTED | coordinator serializa con FENCE; append+force+checkpoint ADMIT antes de permiso | ACK perdido lookup por TransitionKey; no crear ID paralelo |
| ADMITTED → PERMIT_RECEIVED | Member escribe y fuerza PERMIT_RECEIVED en su ledger append-only local, incluyendo admissionId, grantDigest y processBootId, antes de continuar; luego envía PERMIT_RECEIVED_ACK al Coordinator | No hay append central redundante requerido. Fallo local => no envía ACK, no continúa ni solicita START_INTENT; ledger incompleto/corrupto => RECOVERY_REQUIRED. Evidencia: ledger local correlacionado por IDs/seq/hash y STATUS; cualquier incertidumbre mantiene gate cerrado |
| PERMIT_RECEIVED → START_INTENT | Member solicita intent; Coordinator serializa validación de vigencia + persistencia/checkpoint de HANDLER_START_INTENT con ADMIT y FENCE. Permiso debe estar vigente en el instante de confirmación durable del intent | Este commit consume el permiso. Si venció antes, Coordinator persiste idempotentemente ADMISSION_REJECTED_EXPIRED, correlacionado al ADMIT original, y confirma rechazo solo tras durabilidad; entonces transición a REJECTED_NOT_STARTED y retiro del conjunto bloqueante. Fallo/ambigüedad conserva PENDING/bloqueante; ACK perdido => STATUS misma clave. Si está vigente, expiración posterior no cancela; no recheck en ENTERED/body. Intent sin terminal = UNKNOWN |
| PERMIT_RECEIVED → REJECTED_NOT_STARTED (expiry) | HANDLER_START_INTENT rechazado por expiración previa al consumo; Coordinator valida y persiste idempotentemente `ADMISSION_REJECTED_EXPIRED`, correlacionado con ADMIT original, serializado con ADMIT/FENCE | La persistencia durable confirmada del evento es el punto de rechazo: estado derivado `REJECTED_NOT_STARTED` y retiro del conjunto bloqueante. El ACK comunica al Member el resultado ya persistido; su entrega no es condición adicional. ACK perdido: Member o Coordinator consultan STATUS con la misma clave idempotente. Persistencia fallida/ambigua conserva PENDING y bloqueante. Nunca `HANDLER_ENTERED`, cuerpo GET ni `COMPLETE` tras rechazo confirmado |
| START_INTENT → ENTERED | receipt local previa; primera instrucción wrapper persiste HANDLER_ENTERED antes del body | fallo impide ejecutar body; partial/ambiguous no demuestra NOT_STARTED |
| ENTERED → RETURNED | GET retorna, luego persiste HANDLER_RETURNED | crash sin marca durable => UNKNOWN |
| RETURNED → COMPLETE | Reactor Netty confirma fin de response write; coordinator persiste terminal antes de retirar pending | no prueba entrega cliente; timeout no COMPLETE; ack recovery misma clave |
| no terminal → PENDING | permiso/admission existe y ejecución todavía podría iniciarse/continuar, o evidencia no descarta ejecución; incluye rechazo por expiry cuyo evento terminal no esté durablemente confirmado | bloquea COMMIT_GLOBAL |
| no terminal → UNKNOWN | resultado incierto y aún no se ha probado que ejecución dejó de ser posible | bloquea COMMIT_GLOBAL; no reejecución; solo RECONCILE_UNKNOWN explícito |
| UNKNOWN → RECONCILED_UNKNOWN | evidencia durable y explícita prueba proceso/boot dueño muerto y que no puede continuar esa admission; journal+ledger íntegros, sin tail/corrupción/contradicción; vínculo exacto de admissionId/boot; recovery record idempotente durable | outcome sigue desconocido; puede dejar de bloquear COMMIT_GLOBAL, nunca COMPLETE. Evidencia insuficiente conserva UNKNOWN |

`PERMIT_RECEIVED` no es ejecución; START_INTENT solo acredita posibilidad; ENTERED acredita frontera durable previa al cuerpo; RETURNED no acredita write; COMPLETE acredita solo write del servidor. El orden coincide con diseño fuente R3.1 §§3.13/3.18. En D-01, vigencia se comprueba en el commit durable START_INTENT, serializada lógicamente con ADMIT/FENCE. Expiry previo rechaza; posterior no cancela ni se revalida en HANDLER_ENTERED/body. Siguen pendientes TTL, fuente de reloj, tolerancia y renovación pre-consumo. Ninguna política se extrapola a operaciones financieras.

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
