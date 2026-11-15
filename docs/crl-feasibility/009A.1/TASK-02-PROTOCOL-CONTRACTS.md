> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Contratos del protocolo

**Fuentes:** diseño R3.1 §§3.3–3.8, 3.15, 3.17–3.18, 4–9; decisiones humanas TASK-02 D-01 a D-03; inspección TASK-01 registrada en conversación; POM/fuentes/evidencia 009A. No existe informe TASK-01 separado; dependency tree local rotulado artifact 008. El diseño local declara R3.1 y la revisión aportada informa la misma revisión para la copia `(7)`; referencias anteriores R3.1.1 eran erróneas. Fuente no modificada. **Herencia:** single coordinator lab, fail-closed, GET sintético, UNKNOWN no retry, ENTERED durable-before-body, COMPLETE server-write only, R3 recoveryKey. **Pendiente:** TTL/reloj/tolerancia/renovación; no TASK-03.

## 1. Convenciones y campos

JSON abajo es ilustrativo, no wire format aprobado. Common fields: `protocolVersion`, `commandId` o `transitionKey`, `coordinatorEpoch`, `rolloutId` si aplica, `memberId`, `processBootId`, `payloadDigest`. Cada respuesta correlaciona key + outcome durable observado. Autenticación loopback heredada para el laboratorio; no implica protección productiva. El 009A actual usa TCP line-based con bearer, no JSON ni los contratos siguientes.

| Contrato | Emisor → receptor; propósito | Preconditions / request fields | Durable effect / response | Idempotencia, error, timeout/recovery |
|---|---|---|---|---|
| `ADMIT` | Gate member → coordinator; autorización por request | member/boot/membership/epoch/gen/hash/rollout/requestId; GLOBAL_ACTIVE; no FENCE; key `(memberId,processBootId,requestId,ADMIT)` | ADMIT event + admissionId; Permit solo tras punto durable; ACK incluye ID/digest/binding | misma key/payload retorna mismo resultado; payload distinto conflict; timeout lookup STATUS/key, nunca nuevo ID |
| `PERMIT_RECEIVED` | Member persiste receipt en ledger local; luego Member → Coordinator: `PERMIT_RECEIVED_ACK` | admissionId, grantDigest, processBootId; boot destinatario y binding válido | append+force en ledger local del Member antes de continuar; el ACK se envía al Coordinator solo tras persistencia. No se agrega escritura central redundante | Duplicate exacto devuelve receipt; digest/boot distinto rechaza. Fallo local => no ACK, no START_INTENT/body y RECOVERY_REQUIRED; ACK perdido => Coordinator consulta STATUS de la misma admisión; recovery usa ledger local correlacionado + STATUS/journal |
| `HANDLER_START_INTENT` | Member → Coordinator | admissionId/boot/key; ADMIT y receipt local válidos, pendiente, no consumido; Permit vigente al punto durable | En una operación lógica serializada con ADMIT/FENCE: validar vigencia y append/checkpoint intent antes ACK. Ese commit es el punto de consumo | Si expiry anterior, el Coordinator persiste `ADMISSION_REJECTED_EXPIRED` con correlación de la admisión; solo tras confirmarlo durablemente responde el rechazo y fija `REJECTED_NOT_STARTED`. Idempotente; fallo/ambigüedad deja la admisión bloqueante; ACK perdido => STATUS misma clave. Si vigente al commit, expiry posterior no cancela ni recheck en ENTERED/body. Intent sin terminal => UNKNOWN, no reejecutar |
| `ADMISSION_REJECTED_EXPIRED` | Coordinator → Member: rechazo correlacionado de HANDLER_START_INTENT | admissionId, requestId, memberId, processBootId, referencia a ADMIT original, clave de transición estable y payloadDigest cuando aplique | Evento durable en journal/checkpoint del Coordinator; ACK/rechazo solo tras persistencia confirmada. Estado `REJECTED_NOT_STARTED`; no es resultado de negocio ni finalización HTTP | Idempotente; misma clave/payload devuelve mismo rechazo, conflicto falla cerrado. Fallo/resultado ambiguo mantiene bloqueante; ACK perdido => STATUS con la misma clave; nunca liberar por timeout |
| `HANDLER_ENTERED` | Wrapper → member ledger | admissionId/boot/startIntent event; primera instrucción antes del body | persistir durable antes de entrar GET; éxito habilita body | persist failure => body no ejecutado; estado parcial/ambiguo UNKNOWN |
| `HANDLER_RETURNED` | Wrapper → member ledger | admissionId/boot/entered ref; body GET retornó | marca durable posterior al retorno | crash sin marca => UNKNOWN; no inferir de timeout/log |
| `COMPLETE` | Reactor Netty/member → coordinator | admissionId/boot/key `(admissionId,COMPLETE)`; evidencia response write complete | persistir antes de retirar pending; `HTTP_RESPONSE_WRITE_COMPLETED` | solo significa write servidor, no client receipt; ACK perdido query misma key si resultado conocido, si no UNKNOWN |
| `FENCE` | Coordinator → coordinator/member | commandId, rolloutId, oldEpoch/target; serializado con ADMIT | durable antes de ACK; cerrar nuevas admissions viejas | retry igual retorna status; perdido => STATUS/replay; payload conflictivo reject. Si START_INTENT ya está durable, request es in-flight y no se cancela por expiry posterior (D-01); sin START_INTENT no invocar body. Una admisión rechazada por expiry deja de bloquear solo tras persistir `ADMISSION_REJECTED_EXPIRED` |
| `WAIT_IN_FLIGHT` | Coordinator interno | admissions pre-FENCE clasificados PENDING, UNKNOWN no reconciliado o RECONCILED_UNKNOWN | mantener conjunto bloqueante y esperar eventos/barrera; reconciliados salen del conjunto bloqueante solo al persistir criterios exactos | pending/UNKNOWN no reconciliado mantiene FENCING; timeout nunca COMPLETE; reconciliation no cambia outcome |
| `COMMIT_GLOBAL` | Coordinator interno → gate/member | todos exactos verificados target/boot/epoch/hash; cero PENDING y UNKNOWN no reconciliado; cada RECONCILED_UNKNOWN cumple evidencia durable de proceso/boot muerto/no continuación, ledger+journal íntegros, sin tail/corrupción/contradicción y RecoveryRecord idempotente | COMMIT_GLOBAL + checkpoint antes de admitir G2 | RECONCILED_UNKNOWN puede no bloquear pero conserva resultado desconocido; nunca se transforma a COMPLETE. Cualquier caso insuficiente bloquea; ACK perdido STATUS; contradicción => FAILED_CLOSED |
| `RECONCILE_UNKNOWN` | Operator/test → coordinator | admissionId, evidence, operator/testId; correlación inequívoca de miembro/boot/admission; evidencia durable de que el proceso anterior terminó y no puede continuar esa admisión; journal/checkpoint y ledger local íntegros, sin tail/corrupción/contradicción ni otro boot ejecutable para ese ID; autorización explícita | Persistir RecoveryRecord idempotente con recoveryKey/eventId R3; estado `RECONCILED_UNKNOWN`, outcome sigue desconocido | No equivale a COMPLETE ni client receipt. Evidencia insuficiente => permanece UNKNOWN y bloquea commit; conflict => fail-closed |
| `REBIND_UNSTARTED_ADMISSION` | Coordinator recovery → nuevo boot | original ADMIT, targetProcessBootId, recoveryKey R3; journals/ledger íntegros, old boot muerto, sin tail/corrupción/UNKNOWN, NOT_STARTED estricto | recovery event durable; conserva requestId/admissionId y no emite ADMIT nuevo | same key/action idempotent; START_INTENT o UNKNOWN nunca rebind automático; RECONCILED_UNKNOWN no se rebind y no pasa a COMPLETE |
| `STATUS` | caller/coordinator ↔ member | query member/boot/rollout/command/admission; nonce donde definido | solo observación, no muta ni confirma transición | autenticar/correlacionar; respuesta tardía no revierte epoch/estado; timeout => unresolved |
| `PREPARE` | Coordinator → member | commandId, rollout/epoch, manifest, target generation, CRL hash | construir/validar CRL/contexto/protocolo; permanecer cerrado; PREPARED response tras validación | mismo command devuelve estado; mismatch/reject; timeout STATUS misma key; no asumir preparado |
| `ACTIVATE` | Coordinator → member | durable ACTIVATE_INTENT/commandId; target/hash/epoch exactos | instalar generación/listener permaneciendo ACTIVATED_CLOSED; persist status | ACK unknown => STATUS con commandId/boot; retry solo misma key bajo status; discrepancia => FAILED_CLOSED |

## 2. Auth, errores y códigos

Autenticar cada control command, validar rol, membership/boot/epoch, y nunca loggear bearer, PSK/tickets, llaves o password. Categorías propuestas (no wire codes aprobados): `AUTH_DENIED`, `UNKNOWN_MEMBER`, `STALE_BOOT`, `STALE_EPOCH`, `MEMBERSHIP_MISMATCH`, `GENERATION_MISMATCH`, `CRL_HASH_MISMATCH`, `PERMIT_EXPIRED`, `NOT_GLOBAL_ACTIVE`, `FENCED`, `DUPLICATE_PAYLOAD_CONFLICT`, `INVALID_STATE`, `JOURNAL_INVALID`, `RECOVERY_REQUIRED`, `FAILED_CLOSED`, `TIMEOUT/ACK_UNKNOWN`. Separar rechazo lógico de error de transporte. `PERMIT_EXPIRED` rechaza START_INTENT sin body mediante persistencia durable e idempotente de `ADMISSION_REJECTED_EXPIRED`; ante ambigüedad la admisión sigue bloqueante. Status/cuerpo HTTP de denegación no se fija en el diseño R3.1.

## 3. JSON de ejemplo

### ADMIT / ACK

```json
{"protocolVersion":1,"type":"ADMIT","transitionKey":["member-A","boot-x","request-x","ADMIT"],"rolloutId":"rollout-x","membershipVersion":4,"coordinatorEpoch":12,"generationId":"G2","crlContentSha256":"<sha256-der>","requestId":"request-x"}
```

```json
{"type":"ADMIT_ACK","transitionKey":["member-A","boot-x","request-x","ADMIT"],"outcome":"ADMITTED","admissionId":"epoch-sequence-x","grantDigest":"<digest>","permitValidity":"TBD-VD-03"}
```

### FENCE / STATUS

```json
{"type":"FENCE","commandId":"fence-x","rolloutId":"rollout-x","oldCoordinatorEpoch":12,"targetGeneration":"G2"}
```

```json
{"type":"STATUS","nonce":"query-x","memberId":"member-A","processBootId":"boot-x","commandId":"fence-x","activeGeneration":"G1","crlContentSha256":"<sha256-der>","admissions":[{"admissionId":"epoch-sequence-x","state":"RECONCILED_UNKNOWN","outcome":"UNKNOWN","blocksCommit":false,"recoveryEventId":"<sha256>"}]}
```

### Handler intent y COMPLETE

```json
{"type":"HANDLER_START_INTENT","admissionId":"epoch-sequence-x","memberId":"member-A","processBootId":"boot-x","transitionKey":["epoch-sequence-x","HANDLER_START_INTENT"]}
```

```json
{"type":"COMPLETE","transitionKey":["epoch-sequence-x","COMPLETE"],"admissionId":"epoch-sequence-x","processBootId":"boot-x","result":"HTTP_RESPONSE_WRITE_COMPLETED"}
```

Ejemplos no fijan framing, firma/cifrado, encoding, campos opcionales definitivos, valores TTL, reloj, tolerancia, renovación, HTTP status/body ni política productiva. D-01 fija que check de vigencia + persistencia START_INTENT se serializan lógicamente con ADMIT/FENCE; commit de intent consume, expiry anterior rechaza y posterior no cancela/no se revalida. Operaciones financieras quedan expresamente fuera. Ningún ejemplo constituye implementación.

## 4. Estado real

`RotationCoordinator009A.java` tiene PREPARE/ACTIVATE/STATUS por TCP simple; `DistributedTlsInstance009A.java` STATUS/PREPARE/ACTIVATE/STOP; el handler responde generación/hash sin ADMIT. Los contratos anteriores son propuestos, no capacidades existentes.

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
