> **Tarea:** CRL-FEASIBILITY-009A.1 / TASK-02  
> **Revisión documental:** TASK-02-DRAFT-2 (posterior a validación humana)  
> **Fecha:** 2026-11-15 (America/Argentina/Buenos_Aires)  
> **Estado:** `APPROVED — HUMAN REVIEW CLOSED`
> **Implementación:** TASK-03 autorizada solo para especificación, planificación y validación de diseño; implementación aún no autorizada.

# TASK-02 — Análisis de semántica del permiso (VD-03)

**Fuentes:** diseño fuente R3.1 §§3.3, 3.8–3.9, 3.15, 3.18, 4–9; decisiones humanas TASK-02 posteriores a validación (D-01); dictamen TASK-01 en conversación y evidencia 009A. No hay informe TASK-01 separado. La referencia previa R3.1.1 en documentos TASK-02 fue errónea: el archivo local declara R3.1 y la revisión aportada indica esa misma revisión para la copia `(7)`, no disponible localmente para comparación; la fuente no se modifica. **Aprobado D-01:** permiso vigente al commit durable HANDLER_START_INTENT; ese instante lo consume; expiración posterior no cancela; anterior rechaza. **Pendiente:** TTL/reloj/tolerancia/renovación; no TASK-03.

## 1. Política aprobada para GET sintético

La comprobación de vigencia del permiso y la persistencia/checkpoint de `HANDLER_START_INTENT` forman una única operación lógica, serializada con `ADMIT` y `FENCE`. El punto de consumo es el instante de confirmación durable de `HANDLER_START_INTENT`:

- Si el permiso venció antes de ese commit: Coordinator rechaza el intent y persiste durablemente `ADMISSION_REJECTED_EXPIRED`, asociado al ADMIT original; solo después confirma el rechazo. La admisión queda `REJECTED_NOT_STARTED` y sale del conjunto bloqueante únicamente tras esa confirmación durable. No se invoca el cuerpo GET.
- Si está vigente en el commit: queda consumido. Su vencimiento posterior no cancela la ejecución autorizada.
- No volver a comprobar vencimiento en `HANDLER_ENTERED` ni al invocar el cuerpo.
- Si la persistencia del intent o del rechazo es ambigua, no se autoriza reintento: resolver por STATUS/recovery con la misma clave; el rechazo no confirmado mantiene la admisión bloqueante.
- El handler ya autorizado sigue in-flight hasta `HANDLER_RETURNED`/COMPLETE o reconciliación. Timeout no equivale a terminal.

Esta decisión rige solo el GET sintético del laboratorio. No define ni permite inferir política financiera.

## 2. Comparación de alternativas

| Criterio | A — vigencia hasta consumo en intent durable | B — vencimiento absoluto durante ejecución | C — lease renovable durante ejecución |
|---|---|---|---|
| Semántica | Aprobada por D-01 para GET: vigencia se valida en el commit durable START_INTENT; después no se revalida ni cancela. | Expiry posterior que invalida/cancela contradice D-01. Variante que solo impide iniciar después del vencimiento es equivalente a A. | Renewal/revocación post-consumo que pueda cancelar contradice D-01; no se adopta. |
| Seguridad/fencing | Check de vigencia + intent durable serializado con ADMIT/FENCE; FENCE primero impide intent viejo; intent vigente primero queda admitido y debe drenarse. | El corte no demuestra que proceso dejó de ejecutar; cancelar puede dejar estado/efecto incierto. | Lease perdido no prueba handler parado; necesita además fencing y evidencia de terminación. |
| Coordinator caído | Nuevos ADMIT/intents no confirmados se deniegan. Intent ya confirmado vigente no se cancela por expiry o pérdida posterior; permanece in-flight. | No puede cambiar el resultado aprobado para request ya consumido. | Renewal no se usa para revocar ejecución consumida bajo D-01. |
| In-flight | Ledger pendiente hasta COMPLETE o reconciliation; compatible con WAIT_IN_FLIGHT. | No cancelar ni marcar COMPLETE por vencimiento; incompatible si expiración pretende detener handler. | Renovación post-consumo no forma parte de D-01. |
| Complejidad/prueba | Orden durable y barreras deterministas; valores de tiempo/reloj aún requieren decisión. | Temporizadores, skew y cancelación agregan estados/races sin cambiar D-01. | Scheduler, renewal/jitter, canal y recovery agregan complejidad sin formar parte de D-01. |

## 3. Parámetros que siguen pendientes

D-01 no inventa ni aprueba: (1) valor/duración del TTL; (2) punto de referencia desde el cual se evalúa vigencia al confirmar START_INTENT; (3) fuente de reloj, precisión y tolerancia de desfase entre coordinator/member; (4) renovación antes del consumo, si se requiere; (5) encoding/representación temporal persistida y replay al reiniciar. Estos parámetros deben definirse antes de congelar campos/validación temporal del protocolo. No introducen una nueva comprobación en HANDLER_ENTERED/body ni renovación post-consumo.

## 4. Efecto sobre contratos y recovery

`ADMIT` asigna el permiso ligado a admissionId/boot/epoch, pero no lo consume. `HANDLER_START_INTENT` es el punto de validación-consumo atómico con persistencia durable. ACK perdido se consulta por TransitionKey/STATUS; no emitir otro permiso lógico. Si no puede demostrarse si el intent se checkpointó, mantener `PENDING`/`UNKNOWN`, no ejecutar de nuevo. Bajo D-02, UNKNOWN no reconciliado bloquea COMMIT_GLOBAL; `RECONCILED_UNKNOWN` requiere evidencia durable de proceso/boot muerto/no continuación, ledger y journal íntegros y RecoveryRecord idempotente; conserva outcome UNKNOWN y nunca es COMPLETE. Si expiry rechaza START_INTENT, el Coordinator persiste `ADMISSION_REJECTED_EXPIRED` (clave estable, correlación a ADMIT original, idempotencia y journal/checkpoint existentes); tras confirmación durable la admisión pasa a `REJECTED_NOT_STARTED` y deja de bloquear FENCE/WAIT_IN_FLIGHT/COMMIT_GLOBAL. Fallo o ambigüedad conserva PENDING/bloqueante; ACK perdido se resuelve por STATUS con la misma clave. `HANDLER_ENTERED` sigue durable-before-body. FENCE y ADMIT/START_INTENT se serializan bajo el mismo orden global definido por el diseño; request cuyo intent se confirma antes de FENCE es in-flight y no se cancela por expiry posterior.

## 5. Laboratorio vs operaciones financieras

El GET sintético no tiene efecto de negocio y permite barreras/latches. D-01 no define política financiera, cancelación, retry, side effects, deduplicación entre requestIds distintos ni outcome de negocio. Un requestId nuevo sin correlación explícita no se asocia a UNKNOWN previo.

**Resultado VD-03:** punto de consumo y conducta ante expiry anterior/posterior aprobados e incorporados; D-VD03-5 queda resuelta mediante el cierre durable e idempotente ADMISSION_REJECTED_EXPIRED/REJECTED_NOT_STARTED. TTL, reloj, desfase y renovación continúan pendientes humanos. Paquete listo para revisión; contrato temporal no debe congelarse ni implementarse hasta resolver esos parámetros.

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
