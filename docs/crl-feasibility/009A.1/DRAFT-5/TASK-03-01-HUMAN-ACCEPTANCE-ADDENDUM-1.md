# TASK-03-01 — Addendum de aceptación humana de propuestas D03-OPEN-01..08

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Fecha:** 2026-11-15  
**Documento base:** `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1.md`  
**SHA-256 del documento base:** `85ff77f5a496d471116b3c88985b4a138f4c46ab3235260c208132454333a183`  
**Autoridad:** aprobación humana explícita del usuario registrada en la conversación del proyecto.  
**Estado:** `ACCEPTED — HUMAN APPROVED` para las direcciones/recomendaciones enumeradas en este addendum.  
**Alcance de autorización:** decisiones de diseño/documentación. No autoriza implementación, modificación de código/configuración, Maven, builds, tests ni experimentos.

---

## 1. Declaración de aceptación

El usuario aprueba las soluciones conservadoras propuestas para D03-OPEN-01..08 como baseline de diseño para continuar TASK-03. La aprobación acepta las direcciones y salvaguardas descritas en el documento base, con las precisiones de alcance de este addendum.

La aprobación no afirma que la integración con todos los componentes existentes ya esté demostrada ni que no puedan aparecer incompatibilidades. Si el trabajo documental o una implementación futura autorizada detecta una colisión entre estas decisiones, la arquitectura existente, las invariantes aprobadas o las capacidades reales del filesystem, se deberá:

1. detener el avance afectado y describir la colisión con evidencia;
2. identificar componentes, contratos, estados, datos persistidos y escenarios que impacta;
3. evaluar compatibilidad hacia atrás, replay/migración, idempotencia, seguridad, rollback y evidencia de pruebas necesaria;
4. proponer el refactor mínimo suficiente, junto con alternativas y riesgos;
5. someter el cambio de diseño a revisión y aprobación humana antes de modificar código o configuración.

No se resolverán colisiones mediante cambios silenciosos de semántica, relajación de invariantes, fallback no atómico o ampliación implícita de alcance.

## 2. Decisiones aceptadas y alcance exacto

| ID | Estado tras aceptación | Dirección aprobada | Reserva que permanece |
|---|---|---|---|
| D03-OPEN-01 | `DIRECTION APPROVED — DETAIL SPECIFICATION OPEN` | Codec binario tipado, versionado, determinista y length-prefixed; separación de versiones de record/checkpoint/canonicalización; representación explícita de presencia/null/valor; SHA-256 con domain separation para payload y record; recoveryEventId conserva la fórmula normativa R3.1. | Antes de congelar el codec deben especificarse tags, anchos/byte order, domain separators exactos, campos nullable, límites, reglas de unknown fields, genesis hash y compatibilidad de versiones. La aprobación no inventa esos valores. |
| D03-OPEN-02 | `CLOSED — TAIL POLICY APPROVED` | La recuperación automática de tail se limita a ADMIT y COMPLETE, bajo las precondiciones R3.1. FENCE, HANDLER_START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL y epoch advance en tail dejan `RECOVERY_REQUIRED`; no se infiere confirmación, no se envía ACK ni se emite un evento compensatorio por analogía. | Una eventual resolución manual de cada tipo requerirá un contrato separado de evidencia, autoridad, evento durable, idempotencia, STATUS y salida del estado cerrado. No queda autorizada recuperación automática adicional. |
| D03-OPEN-03 | `DIRECTION APPROVED — STARTUP CONTRACT DETAIL OPEN` | Se adopta un evento durable explícito de avance de epoch (nombre provisional `COORDINATOR_EPOCH_ADVANCED`), con incremento estricto `newEpoch = previousEpoch + 1`; un almacén genuinamente vacío podrá iniciar con epoch 1 solo si journal, checkpoint y temporales están todos ausentes. El Coordinator no opera antes del checkpoint del epoch del arranque. | Quedan por especificar el nombre/schema final, startupOperationId, definición operativa de reinicio, overflow, primer-arranque exacto, tail de epoch y recuperación de cortes en cada frontera. Un epoch advance en tail no se ignora ni se duplica: `RECOVERY_REQUIRED`. |
| D03-OPEN-04 | `POLICY APPROVED — PLATFORM EVIDENCE OPEN` | Se distinguen `SUPPORTED_AND_SUCCEEDED`, `UNSUPPORTED` explícito y `FAILED_OR_AMBIGUOUS`. No se clasifican I/O error, acceso denegado o resultado ambiguo como unsupported por heurística. No hay fallback no atómico. Error real/ambiguo mantiene Coordinator cerrado y exige verificación por replay. | La clasificación concreta de excepciones/proveedores depende del JDK/Windows/filesystem ensayado. Directory force unsupported limita los claims; no equivale a durabilidad ante pérdida de energía/kernel/disco/hardware. |
| D03-OPEN-05 | `CLOSED — RESIDUAL TEMP POLICY APPROVED` | Temporales con `CREATE_NEW` en el mismo directorio/filesystem y nombre reservado correlacionado con operación/evento; startup detecta temporales residuales y no los promueve, reemplaza ni elimina automáticamente. | Prefijo, naming final y límites de lectura se concretarán en SPEC. La limpieza manual requiere decisión documentada, autoridad, evidencia/hash previo y resguardo; no forma parte del replay automático. |
| D03-OPEN-06 | `DIRECTION APPROVED — CLASSIFICATION DETAIL OPEN` | Se diferencian `RECOVERING` (fase transitoria), `RECOVERY_REQUIRED` (estado íntegro pero semántica/evidencia no resuelta) y `FAILED_CLOSED` (corrupción, contradicción o propiedad del almacén/lock no confiable). No hay auto-reset desde estados cerrados. Retorno exige acción humana explícita y documentada; `FAILED_CLOSED` requiere reparación/restauración controlada y baseline verificable. | Debe cerrarse la clasificación exacta de lock ocupado por otro Coordinator legítimo frente a propiedad/estado de lock incierto, y la tabla exhaustiva condición→estado→acción→evidencia→autoridad. |
| D03-OPEN-07 | `GATE APPROVED — EXPERIMENT NOT AUTHORIZED` | Las capacidades Windows de FileLock, ATOMIC_MOVE y directory force permanecen `UNVERIFIED` hasta prueba aislada; cada mecanismo se prueba por separado con fixtures sintéticos y captura de JDK/OS/proveedor/filesystem/excepciones/hashes. No se afirma soporte desde teoría o evidencia de otro entorno. | Esta aprobación acepta el gate conservador y la metodología propuesta; **no autoriza ejecutar el experimento**. Se requiere aprobación humana específica, directorio aislado y plan de rollback antes de escribir o probar en Windows. |
| D03-OPEN-08 | `BOUNDARY AND PRECONDITIONS APPROVED — MEMBER EVIDENCE REMAINS TASK-04` | TASK-03 define el contrato/referencia del recovery event y sus invariantes; no ejecuta REBIND. El REBIND conserva requestId/admissionId, incluye targetProcessBootId en la acción canonicalizada y usa recoveryKey/recoveryEventId R3.1. Se exige ledger Member íntegro, receipt durable, ausencia demostrable de START_INTENT/ENTERED, proceso/boot anterior terminado y sin tail/corrupción/contradicción; ausencia de cualquiera => no REBIND. | La evidencia/ledger local y la demostración de terminación corresponden a TASK-04. Este addendum no autoriza TASK-04 ni una implementación anticipada del ledger. `UNKNOWN` y `RECONCILED_UNKNOWN` nunca equivalen a NOT_STARTED. |

## 3. Decisiones heredadas que continúan pendientes

La aceptación de D03-OPEN-01..08 no cierra ni altera:

- D-VD03-1: duración/TTL y referencia temporal del Permit.
- D-VD03-2: fuente de reloj, precisión y tolerancia de desfase.
- D-VD03-3: renovación pre-consumo, si aplica, y su representación.
- D-VD03-4: evidencia temporal necesaria para replay/restart.
- D-ID-1: encoding/entropía de processBootId, requestId y commandId.
- D-EVIDENCE-1: alcance y suficiencia de la evidencia de terminación de proceso, especialmente remota.
- D-FS-1: capacidad real del filesystem.
- D-WIRE-1: framing, encoding y autenticación final del protocolo.

Se preservan sin reinterpretación D-01, D-02, D-03, D-VD03-5 y H-01. En particular, H-01 continúa referido a serialización de operaciones vivas; no aprueba tail recovery de START_INTENT ni ADMISSION_REJECTED_EXPIRED.

## 4. Efecto sobre T03-01 y TASK-03

1. Las recomendaciones D03-OPEN-01..08 quedan aceptadas en los niveles y con las reservas consignadas en §2.
2. T03-01 puede avanzar a una revisión documental consolidada que refleje estas decisiones y complete los detalles todavía abiertos sin inventar parámetros.
3. No se declara todavía congelado el codec, el contrato final de epoch ni la clasificación exhaustiva de estados/locks.
4. La aprobación no constituye aprobación integral del diseño TASK-03 ni autorización de implementación.
5. T03-02..T03-12 siguen sometidas a sus gates y dependencias. El experimento Windows conserva su autorización independiente.
6. No se autoriza publicar este addendum en GitHub ni modificar archivos del repositorio mediante esta aceptación; cualquier publicación será una operación documental separada.

## 5. Registro de evidencia y límites

- Aprobación: manifestación del usuario «son bastante conservadoras las soluciones propuestas, las apruebo, si notamos algo que colisione realizaremos un refactor».
- Artefacto base: `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1.md`.
- Hash del artefacto base registrado en el encabezado.
- Tipo de actividad: aceptación documental de decisiones de diseño.
- Código/configuración modificados: no.
- Maven/build/tests/experimentos ejecutados: no.
- Workspace Windows probado: no.
- Implementación TASK-03: `NOT AUTHORIZED`.
- TASK-04 y tareas posteriores: `NOT AUTHORIZED`.
- Gate de factibilidad CRL para producción: `CLOSED`.

---

**Resultado:** aceptación humana registrada para las ocho propuestas conservadoras, con los detalles bloqueantes y gates independientes expresamente preservados.