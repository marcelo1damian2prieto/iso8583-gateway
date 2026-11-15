> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Modelo de dominio y límites

**Fuentes:** diseño R3.1 (§§3.1–3.20, 4–9); decisiones humanas TASK-02 posteriores a validación (D-01 a D-03); dictamen TASK-01 VIABLE CON CONDICIONES y hallazgos consignados en la conversación; POM/fuentes/informe/ZIP/logs de 009A. No existe informe TASK-01 separado en los directorios temporales inspeccionados. El árbol Maven disponible se atribuye al artifact 008, no se presenta como resolución exacta de 009A. El archivo fuente local declara R3.1; la revisión aportada informa que la copia `(7)` también declara R3.1, aunque no está disponible localmente para comparación textual. La referencia previa “R3.1.1” fue errónea. No se modifica la fuente ni se inventa una revisión.

**Decisiones aprobadas heredadas:** coordinator único de laboratorio; gate HTTP síncrono por solicitud, fail-closed; TLS antiguo puede completar handshake pero no ejecutar handler sin autorización; sin consenso/DB externa/OCSP/soft-fail; GET sintético; UNKNOWN no se reejecuta automáticamente; R3 recoveryKey y `HANDLER_ENTERED` durable antes del cuerpo GET; COMPLETE solo tras HTTP write completion del servidor. Sin aprobación CRL productiva.

**Pendiente:** TTL, fuente de reloj, tolerancia de desfase y renovación pre-consumo; no alteran el punto de consumo D-01 aprobado. Véase `TASK-02-DECISIONS-REQUIRED.md`. No habilita TASK-03.

## 1. Reglas comunes

- IDs opacos, no reutilizables dentro de su ámbito durable; validar siempre payload además del ID.
- Canonicalización versionada, length-prefixed y no ambigua para claves/hash (R3 §3.17).
- No derivar TTL ni expiración de `createdAt` o reloj de pared: punto de consumo/efecto temporal D-01 está fijado; parámetros temporales permanecen pendientes VD-03.
- Hash de CRL refiere al DER comparable entre miembros; UUID/version local de snapshot no representa identidad global.

## 2. Conceptos

| Concepto | Responsabilidad / identidad | Datos e invariantes | Persistencia, ciclo y rechazo |
|---|---|---|---|
| `Coordinator` | Autoridad única del laboratorio; identidad de proceso/configuración local; no autoridad multi-host. | Estado rollout, enrollment, epoch, journal/checkpoint y admissions pendientes. Servicio solo desde estado durable validado. | Journal/checkpoint; inicia RECOVERING. Corrupción, lock o contradicción => RECOVERY_REQUIRED/FAILED_CLOSED. Actual 009A: coordinator de test, mapas y append texto. |
| `Member` | JVM identificada por `(membershipVersion, memberId, processBootId)`. | Reporta generation/hash, epoch, contexto/listener y estado. Boot nuevo no hereda permiso. | Ledger/local state según R3; nuevo boot cerrado. El state file 009A solo guarda generation/hash. |
| `MembershipManifest` | Conjunto exacto para rollout; clave `(membershipVersion,digest)`, asociado a rolloutId. | schemaVersion, IDs ordenados sin duplicados, digest, configurationVersion, generación/hash target. No vacío; igual a enrollment exacto. | Durable antes de PREPARE; inmutable durante rollout. Unknown/subset/superset/version mismatch => rechazo. |
| `Rollout` | Unidad de transición de generación; `rolloutId`. | Epoch, membership, target, hash, fases y observaciones por miembro. GLOBAL_ACTIVE solo con verificación exacta + commit durable. | Derivado de journal/checkpoint; ACK aislado no es commit. Timeout ambiguo => closed/recovery. |
| `CoordinatorEpoch` | Fencing epoch monotónico de coordinator único. | Asociado a eventos y permisos; autoridad para negar mensajes viejos. | Durable antes de habilitar tras restart. No da consenso ni detecta rollback íntegro local. Orden requiere serializador ADMIT/FENCE a implementar/probar. |
| `ProcessBootId` | Identifica una ejecución de miembro; nuevo por arranque. | Correlaciona PID, permiso, ledger y STATUS. PID no equivale a boot identity ni prueba de muerte. | Registrado durablemente donde aplique; boot nuevo no reclama admisión previa. Encoding/entropía pendientes. |
| `RequestId` | Clave lógica del request sintético del caller. | Reintento de mismo request conserva ID/payload; ID nuevo sin correlación negocio no se asocia a UNKNOWN anterior. Payload distinto bajo igual key => conflicto. | Clave de consulta/dedupe junto a member/boot; no equivale a AdmissionId. |
| `AdmissionId` | Identidad única de permiso consumible, asignada por coordinator; PK del ledger in-flight. | Ligado a request/member/boot/epoch/generation/hash/rollout y ADMIT. No reutilizar, incluso si tail ADMIT se aborta. | Durable antes de emitir Permit; terminal/recovery correlacionados. Timeout no genera uno paralelo. |
| `EventId` | Identidad de evento durable. | UUID propuesto por R2; secuencia, tipo, payload digest, epoch, refs y hash-chain. Mismo ID con payload distinto es conflicto. | Journal; replay valida orden/referencias/unicidad. Forma final ligada al contrato de persistencia. |
| `TransitionKey` | Idempotencia lógica distinta de EventId. | ADMIT `(memberId,processBootId,requestId,ADMIT)`; COMPLETE `(admissionId,COMPLETE)` (R2 §3.15); otros comandos usan ID estable + payload digest. | Indexada durable por dueño de transición. Retry idéntico consulta resultado; conflicto rechaza/audita. |
| `Permit` | Prueba autenticada/correlacionada de ADMIT concedido. | admissionId, requestId, member, boot, epoch, rollout, generación/hash, grantDigest. D-01: vigente al confirmar durablemente HANDLER_START_INTENT, punto de consumo; vencimiento posterior no cancela ejecución autorizada. | Emitido después de ADMIT durable; PERMIT_RECEIVED se persiste en ledger local del miembro. TTL, reloj, tolerancia y renovación siguen pendientes. ACK perdido consulta misma key/STATUS. |
| `AdmissionLedger` | Estado por admissionId de autorización y ejecución. | ADMITTED → PERMIT_RECEIVED → START_INTENT → ENTERED → RETURNED → COMPLETE; rama terminal pre-consumo REJECTED_NOT_STARTED mediante ADMISSION_REJECTED_EXPIRED; además PENDING, UNKNOWN y RECONCILED_UNKNOWN; máximo una entrada; ENTERED durable antes del body. | PERMIT_RECEIVED durable en ledger local del miembro antes de continuar; Member envía PERMIT_RECEIVED_ACK al Coordinator solo tras force local. Fallo detiene flujo y deja recovery requerido. Evidencia local: admissionId, grantDigest, bootId, secuencia/hash. No existe en 009A. |
| `AdmissionRejectedExpired` | Evento durable del Coordinator que rechaza HANDLER_START_INTENT por expiración previa al consumo. | Correlaciona admissionId, requestId, memberId, processBootId, evento ADMIT original, clave de transición estable y payload digest cuando aplique. | Append/force + checkpoint según journal existente; ACK de rechazo solo después de persistencia confirmada. Idempotente; fallo/ambigüedad conserva estado bloqueante. No es resultado de negocio ni finalización HTTP. |
| `RecoveryRecord` | Evento durable de reconciliación, no reemplaza original. | originalEventId, action/key/eventId, expected/result state, digest. R3: `recoveryKey=(originalEventId,recoveryAction)`; recoveryEventId SHA-256 de canonical. | Replay valida referencia, orden, acción y payload. Duplicado conflictivo => FAILED_CLOSED. RECONCILED_UNKNOWN conserva resultado desconocido; evidencia durable suficiente de muerte/no continuidad puede sacarlo del conjunto bloqueante, nunca convertirlo en COMPLETE. |
| `CrlSnapshot` | CRL validada para contexto de generación. | Bytes DER/hash comparable, PKIX/frescura y metadatos locales; ID/versión local no comparable entre JVMs. | Inmutable mientras la usa contexto; hash/validación inesperados => no habilitar. Existe manager/snapshot en 009A. |
| `Generation` | Bundle de generación/hash/contexto TLS/listener miembro. | generationId, hash CRL, epoch/membership y evidencia contexto/listener. | Local prepared/active; 009A persiste solo generation/hash. State local nunca autoriza servicio por sí solo. |

## 3. Relaciones

```text
Rollout 1—1 MembershipManifest; Rollout 1—N Member observations
RequestId 1—0..1 AdmissionId por TransitionKey admitida
AdmissionId 1—N EventId; AdmissionId 1—1 AdmissionLedger state machine
EventId 1—0..N RecoveryRecord (acciones distintas explícitamente válidas)
Generation → CrlSnapshot(hash DER) + contexto/listener local
```

## 4. Hechos, límites y pendientes

El código 009A (`RotationCoordinator009A`, `DistributedTlsInstance009A`, `CrlSnapshot`) solo prueba capacidades actuales de coordinación de test, proceso hijo, TLS/CRL y estado simple. Los objetos, ledger y garantías anteriores son diseño, no implementación. POM declara Boot 3.5.16/Java 21/WebFlux/Reactor Netty sin versión directa y BC 1.81 test; el tree archivado está rotulado 008, por lo que no prueba resolución transitiva exacta 009A. Quedan pendientes: TTL/fuente de reloj/tolerancia/renovación antes de consumo; encoding de bootId/requestId/commandId; evidencia de muerte de proceso remoto; binding autenticado del permiso; almacenamiento y recuperación real. Si START_INTENT se rechaza por expiry, el Coordinator persiste idempotentemente ADMISSION_REJECTED_EXPIRED, correlacionado con el ADMIT original. Solo tras confirmar esa persistencia durable la admisión pasa a REJECTED_NOT_STARTED y deja el conjunto bloqueante; una persistencia fallida o ambigua la mantiene bloqueante. Timeout o ACK perdido no prueban rechazo: se consulta STATUS con la misma clave. Sin clave negocio no hay dedupe entre distintos requestId. D-01/02/03 aplican solo al GET sintético; no se agregan políticas financieras.

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
