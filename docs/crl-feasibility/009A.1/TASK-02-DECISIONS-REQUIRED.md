> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Decisiones requeridas

**Fuentes:** diseño fuente R3.1 §§3.3–3.18, 4–9; decisiones humanas posteriores a TASK-02 (D-01 a D-03); TASK-01 conversacional (VIABLE CON CONDICIONES); fuentes/evidencia 009A. No existe archivo TASK-01 separado; dependency tree se atribuye a artifact 008. El diseño local declara R3.1; la revisión aportada identifica la copia `(7)` como R3.1. La referencia R3.1.1 previa fue errónea; no se modifica fuente. **Decisiones heredadas:** coordinator único lab, fail-closed, GET sintético, UNKNOWN no retry, ENTERED durable-before-body, COMPLETE server-write only, recoveryKey R3. **Pendiente:** parámetros TTL/tiempo y revisión humana explícita.

## 1. Bloqueantes y pendientes

| ID | Decisión requerida | Alternativas / impacto | Recomendación no aprobada | Dependencias |
|---|---|---|---|---|
| D-VD03-1 | Valor/duración TTL y referencia temporal para comprobar vigencia en el commit durable START_INTENT | D-01 ya fija punto exacto de consumo; aún debe parametrizar cuánto dura y desde qué instante se mide | No inventar valor; decidir antes de congelar campos temporales | Permit, validación START_INTENT, tests deterministas |
| D-VD03-2 | Fuente de reloj, precisión y tolerancia de desfase entre coordinator/member | monotónico local no comparable por sí solo entre procesos; reloj pared puede variar | Elegir con evidencia del modelo local; no asumir distribución remota | serialización, restart/replay, expiry boundary |
| D-VD03-3 | ¿Hay renovación antes de START_INTENT y cómo se representa/persiste? | D-01 no autoriza renovación/cancelación después del consumo | No post-consumo renewal; pre-consumo mechanism TBD | Permit schema, journal, TASK-03/05 |
| D-VD03-4 | Evidencia requerida para replay/restart al determinar vigencia | coordinator restart no debe liberar admissionId ni revalidar intent ya consumido | Conservar hora/decisión durable del intent y parámetros aprobados; precisión exacta pendiente | journal/checkpoint, recovery, tests |
| D-ID-1 | Encoding/entropía de processBootId/requestId/commandId | campos de rol están definidos, encoding no | Validar más tarde, sin fijar aquí | domain y protocolo |
| D-EVIDENCE-1 | Evidencia suficiente de proceso muerto, especialmente remoto | PID local + waitFor/boot; remoto no probado por harness | Acotar prueba a procesos locales; no afirmar garantía remota | reconciliation/rebind |
| D-FS-1 | Capacidad filesystem para atomic move/directory force | no validable en inspección read-only | Experimento aprobado en TASK futura; sin asumir durabilidad de energía | journal/checkpoint TASK-03/04 |
| D-WIRE-1 | Framing/encoding/auth final de mensajes | ejemplos JSON solo ilustrativos; transporte existente es line TCP loopback | No tratar contratos TASK-02 como wire protocol aprobado | TASK-05 |

## 2. Confirmadas por R1–R3, no reabrir aquí

- Single coordinator de laboratorio; no consenso/DB/Kafka/Redis/HA.
- Gate síncrono HTTP por request y fail-closed; coordinator caído => nuevas requests denegadas; handshake TLS antiguo puede completar, handler no.
- GET sintético, sin extrapolación a transacciones financieras.
- D-01: Permit debe estar vigente cuando coordinator confirma durablemente HANDLER_START_INTENT; la validación + persistencia es una operación lógica serializada con ADMIT y FENCE. Ese commit consume el permiso; expiración anterior rechaza sin body, posterior no cancela y no se vuelve a comprobar en HANDLER_ENTERED ni en el cuerpo. Exclusivo GET sintético.
- D-02: PENDING y UNKNOWN no reconciliado bloquean COMMIT_GLOBAL. RECONCILED_UNKNOWN conserva resultado desconocido, pero evidencia durable suficiente de muerte/no continuidad puede sacarlo del conjunto bloqueante; nunca COMPLETE.
- D-03: PERMIT_RECEIVED se fuerza en ledger local del miembro antes de continuar; solo después, el Member envía PERMIT_RECEIVED_ACK al Coordinator. El fallo local impide continuar; no se agrega escritura central redundante.
- D-VD03-5 (resuelta): si el permiso venció antes del commit durable de HANDLER_START_INTENT, el Coordinator persiste ADMISSION_REJECTED_EXPIRED, correlacionado con la admisión original, y confirma el rechazo solo después de esa persistencia. La admisión pasa a REJECTED_NOT_STARTED y deja de bloquear FENCE/WAIT_IN_FLIGHT/COMMIT_GLOBAL únicamente tras confirmación durable; fallo o ambigüedad la mantiene bloqueante. Rechazo idempotente; ACK perdido se resuelve por STATUS con la misma clave. No se infiere rechazo por timeout.
- ADMIT se ordena/durabiliza antes del permiso; FENCE se serializa con ADMIT; WAIT no convierte timeout en COMPLETE.
- PERMIT_RECEIVED es receipt; START_INTENT puede preceder a ejecución; HANDLER_ENTERED durable antes del cuerpo; HANDLER_RETURNED después del body; COMPLETE solo tras write HTTP server y no prueba receipt cliente.
- START_INTENT sin evidencia posterior suficiente => UNKNOWN/RECOVERY_REQUIRED, no retry automático. Nuevo requestId sin correlación no se vincula automáticamente. RECONCILED_UNKNOWN requiere reconciliación explícita durable, correlation admission/boot, proceso muerto/no continuidad demostrada, journal+ledger íntegros y sin tail/corrupción/contradicción; outcome queda UNKNOWN.
- Recovery R3: `recoveryKey=(originalEventId,recoveryAction)`, `recoveryEventId=SHA-256(canonical(...))`; mismo key idempotente, conflicto fail-closed.
- No autorización de CRL producción ni implementación por este documento.

## 3. Parámetros y decisiones que siguen pendientes

TTL/duración y punto de referencia, fuente de reloj, tolerancia de desfase y renovación pre-consumo siguen sin definir. D-01 sí cierra el punto de consumo/efecto expiry anterior/posterior para GET sintético. Deben resolverse antes de congelar contrato temporal e implementación. Política financiera no definida.

## 4. Condición de continuación y trazabilidad de diseño

La fuente documental se normaliza como R3.1: el diseño local `%TEMP%\crl-feasibility-009a1\CRL-FEASIBILITY-009A.1-DESIGN.md` declara R3.1 en metadatos y §3.20; la revisión aportada indica que la copia `(7)` también lo declara, aunque esa copia no está disponible localmente para comparación textual. No hay evidencia de fuente R3.1.1; esa etiqueta se retira de TASK-02 como error de referencia, sin cambiar el archivo fuente. Los documentos siguen `PROPOSED — PENDING HUMAN REVIEW`; no habilitan TASK-03.

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
