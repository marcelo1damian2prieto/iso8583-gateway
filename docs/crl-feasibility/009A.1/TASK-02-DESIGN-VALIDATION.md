> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Revisión crítica y trazabilidad

**Fuentes:** diseño fuente R3.1 §§3.1–3.20, 4–9; decisiones humanas TASK-02 D-01 a D-03; dictamen TASK-01 VIABLE CON CONDICIONES/hallazgos en conversación; POM/fuentes/logs/ZIP 009A. TASK-01 no existe como archivo separado en el directorio. Árbol Maven archivado identifica artifact 008, por lo que no prueba resolución exacta de 009A. La referencia R3.1.1 previa se normaliza a R3.1 con evidencia del archivo local y la copia `(7)` indicada; fuente no modificada.

## 1. Resultado

Los siete documentos TASK-02 están **READY FOR HUMAN REVIEW**, estado `PROPOSED — PENDING HUMAN REVIEW`; no son diseño aprobado ni congelado. Se incorporan D-01/02/03. VD-03 conserva parámetros TTL/reloj/tolerancia/renovación pendientes, por lo que aún no se debe congelar el esquema temporal completo del Permit ni comenzar TASK-03. No hubo implementación ni ejecución.

## 2. Coherencia de estados y contratos

| Elemento | Semántica consistente | Estado/contrato | Escenario asociado |
|---|---|---|---|
| `ADMIT` / `FENCE` | orden total; mismo serializador lógico abarca validación y persistencia; FENCE primero impide intent/admission viejo; ADMIT/intent previo queda in-flight | coordinator FSM; ADMIT, FENCE, WAIT_IN_FLIGHT | S01–S03, S21 |
| `PERMIT_RECEIVED` (D-03) | Member escribe y fuerza ledger local antes de continuar; después envía PERMIT_RECEIVED_ACK al Coordinator; fallo local impide START_INTENT/body; no escritura central redundante | máquina de request; contrato receipt local | S06, S25 |
| consumo de Permit (D-01) | Vigencia comprobada al confirmar durablemente HANDLER_START_INTENT; check + persistencia/checkpoint es una operación lógica serializada con ADMIT/FENCE | contrato HANDLER_START_INTENT | S17, S18 |
| rechazo por expiry pre-consumo (D-VD03-5 resuelta) | Coordinator persiste ADMISSION_REJECTED_EXPIRED; tras confirmar durabilidad, estado REJECTED_NOT_STARTED, sin body y no bloqueante | evento idempotente correlacionado al ADMIT original; fallo/ambigüedad conserva bloqueante; ACK perdido se resuelve por STATUS misma clave | S17, S27–S36 |
| expiry antes de intent | Rechazar intent durablemente; no HANDLER_ENTERED ni GET | REJECTED_NOT_STARTED solo tras commit de ADMISSION_REJECTED_EXPIRED; no se retira pending antes | S17, S35 |
| expiry posterior | intent ya consumió permiso; no cancela ni se revalida en HANDLER_ENTERED/body; ejecución autorizada sigue in-flight hasta terminal/reconcile | sin transición de cancelación por TTL | S18 |
| `HANDLER_ENTERED` | durable en primera instrucción protegida antes del GET; fallo durable => no invocar | transición local del wrapper | S24 |
| `HANDLER_RETURNED` / `COMPLETE` | retorno GET ≠ HTTP write; COMPLETE solo tras server write completion y no demuestra receipt cliente | contrato COMPLETE | S05 |
| `PENDING` | ejecución podría todavía producirse/continuar o evidencia no lo descarta; bloquea commit | AdmissionLedger/WAIT | S22 |
| `UNKNOWN` no reconciliado (D-02) | resultado incierto y ejecución aún no descartada; bloquea commit; no retry | recovery/COMMIT_GLOBAL | S07–S09, S22 |
| `RECONCILED_UNKNOWN` (D-02) | outcome de negocio sigue desconocido; proceso/boot probado muerto y no puede continuar esa admission; evidencia y recovery durable íntegros; puede salir del conjunto bloqueante | RECONCILE_UNKNOWN/WAIT/COMMIT | S23 |
| `RECONCILED_UNKNOWN` vs COMPLETE | no se transforma en COMPLETE, no prueba write ni receipt de cliente | eventos distintos, outcome sigue UNKNOWN | S23 |
| ACK/timeout | ambiguo no es éxito ni fallo; query misma key; timeout nunca terminal | STATUS + idempotency | S04/S05/S26 |

**Condiciones acumulativas para que RECONCILED_UNKNOWN no bloquee COMMIT_GLOBAL:** (1) admissionId y processBootId propietario correlacionados inequívocamente; (2) evidencia durable suficiente de que el proceso anterior terminó y ya no puede continuar esa admisión; (3) journal/checkpoint e `AdmissionLedger` íntegros, sin tail parcial/corrupción ni contradicción; (4) no hay otro proceso/boot ejecutable reclamando la admisión; (5) `RecoveryRecord` explícito durable con `recoveryKey=(originalEventId,recoveryAction)` y `recoveryEventId=SHA-256(canonical(...))`, idempotentemente validado; (6) estado final `RECONCILED_UNKNOWN`, outcome continúa desconocido; (7) todos los demás criterios del commit global se cumplen. Falta de cualquier evidencia deja UNKNOWN/PENDING bloqueante. Nunca synthesize COMPLETE.

## 3. Trazabilidad requisitos → artefactos

| Requisito/diseño R3.1 | Modelo/contrato | Estado/invariante | Escenarios |
|---|---|---|---|
| membership exacta §§3.1/3.12 | MembershipManifest, PREPARE/STATUS | desconocido/mismatch => closed | S11, S13 |
| journal/recovery §§3.2/3.11/3.15/3.17 | EventId, TransitionKey, RecoveryRecord | replay, fail-closed, recovery idempotente | S09, S14–16 |
| ADMIT/FENCE §§3.3/3.8 | Permit, ADMIT, FENCE | total order, no nueva admisión tras fence | S01–04, S21–22 |
| handler §§3.13/3.18 | ledger y contratos de cada frontera | ENTERED antes de body, UNKNOWN no retry | S05–08, S24 |
| permiso temporal D-01 | Permit/START_INTENT | consumo en intent durable; sin cancelación posterior | S17–18 |
| `PERMIT_RECEIVED` D-03 | Ledger local durable antes de seguir; ACK Member → Coordinator solo después del force | S06, S25 |
| rechazo pre-consumo D-VD03-5 | ADMISSION_REJECTED_EXPIRED → REJECTED_NOT_STARTED | durable, idempotente, ACK tras persistencia; bloqueante hasta confirmación | S17, S27–S36 |
| D-02 reconciliation | RecoveryRecord/RECONCILE/COMMIT | distinctions PENDING/UNKNOWN/RECONCILED_UNKNOWN; reconciled no COMPLETE | S22–23 |
| ACK/idempotencia §3.5/R3 | STATUS/TransitionKey | respuesta tardía no revierte estado | S04–05, S15–16, S26 |
| TLS/restart §§3.6–3.7 | Member/boot/Generation | boot nuevo closed, handshake ≠ autorización HTTP | S10, S12, S13, S19–20 |

## 4. Revisión de requisitos y testing

- Funcionales §4 y no funcionales §5 quedan mapeados a estados/contratos/escenarios; no se declaran implementados.
- Matriz incluye positivos, negativos, ACK perdido, expiry antes/después del punto aprobado, caídas, recovery duplicado/corrupto, conectividad, in-flight, PERMIT_RECEIVED local y commit con UNKNOWN/reconciled.
- Barreras/latches y snapshots journal/ledger permiten prueba determinista; `sleep` no establece orden. Los experimentos solo pertenecen a tareas futuras autorizadas.
- `force(true)`, atomic move y durabilidad de filesystem aún no validados; proceso muerto/rebind remoto no queda demostrado por PID local.
- Criterios TLS 1.2/1.3, gate, persistencia, secretos y ZIP necesitan TASK-07..09; no están satisfechos por revisión documental.

## 5. Contradicciones, procedencia y riesgos

1. Las versiones TASK-02 anteriores rotulaban fuente como R3.1.1. El diseño local tiene metadata R3.1 y §3.20 “revisión R3.1”; la copia `(7)` nombrada por revisión también se declara R3.1. Se normaliza la trazabilidad TASK-02 a **R3.1**; no se modifica la fuente ni se acuña revisión nueva. Si la copia `(7)` difiere materialmente en contenido, esa comparación exacta no es posible porque no está disponible localmente; la revisión solicitante confirma su etiqueta/metadata.
2. R3 §3.3 no fija valores TTL/clock/skew/renewal. D-01 resuelve punto de consumo y efecto expiry antes/después, no parámetros.
3. No hay contradicción entre PERMIT_RECEIVED local y las transiciones siguientes; no agregar escritura central redundante.
4. 009A actual no implementa ADMIT/FENCE/gate, ledger, replay ni recovery; PREPARE/ACTIVATE/TLS son capacidades parciales existentes.
5. Riesgos heredados: TLS resumption puede omitir callback PKIX; coordinator síncrono afecta disponibilidad; single-host no es HA; rollback de copia local íntegra no se detecta; local PID no demuestra muerte remota; bearer loopback no es auth productiva; requestId nuevo sin correlación no deduplica UNKNOWN.
6. El GET sintético no determina política de transacciones financieras. No extrapolar D-01/02/03 a negocio real.

## 6. Dependencias TASK-03..TASK-09

| Tarea | Insumos TASK-02 | Condición |
|---|---|---|
| TASK-03 | event schema, IDs, FSM, ACK/replay, FENCE order, D-01 consumption point and D-02 terminal/reconciliation distinctions | no freeze TTL/clock fields hasta decisiones restantes; requiere aprobación explícita TASK-03 |
| TASK-04 | boot identity, member ledger, receipt local D-03, local UNKNOWN evidence | definir formato/ownership sin cambiar semántica |
| TASK-05 | contratos lógicos, correlation/auth/errors/idempotency | JSON es ilustrativo, wire protocol pendiente |
| TASK-06 | wrapper exterior, local ledger, intent gate y body ordering | no invocar body antes de durable ENTERED |
| TASK-07 | generation/snapshot/member lifecycle | interfaces previas listas |
| TASK-08 | scenario IDs, fault boundaries y evidence | instrumentation futura autorizada |
| TASK-09 | acceptance matrix, evidence/redaction, filesystem test | todas anteriores completas; tests con autorización |

## 7. Pendientes humanos y veredicto

Continúan pendientes el valor y referencia temporal del TTL, reloj/precisión/tolerancia de desfase, renovación previa al consumo (si aplica), encoding de IDs, framing final y evidencia de muerte de proceso/filesystem. No reabren D-01–03. Ninguna política financiera está definida. La decisión R3.1/R3.1.1 se normaliza a R3.1 con la limitación de copia señalada.

**Veredicto documental: READY FOR HUMAN REVIEW.** Los siete documentos reflejan las decisiones recibidas y mantienen explícitas las cuestiones pendientes. No equivale a aprobación del diseño, no habilita TASK-03 ni implementación.

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
