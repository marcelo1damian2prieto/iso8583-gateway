# TASK-03 — Validación del diseño

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-DESIGN-VALIDATION-DRAFT-2`  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Tipo de validación:** revisión documental cruzada; no se ejecutaron builds, tests, experimentos ni modificaciones de código.  
**Documentos relacionados:** SPEC-DRAFT-2, PLAN-DRAFT-2 y TASKS-DRAFT-2.

## 1. Dictamen preliminar

**VIABLE CON CONDICIONES PARA CONTINUAR LA PREPARACIÓN DOCUMENTAL.**

La propuesta conserva el modelo append-only, checkpoint atómico, replay, tail recovery acotado, idempotencia, serialización ADMIT/FENCE y fail-closed. No se considera lista para aprobación de implementación mientras permanezcan abiertas las decisiones D03-OPEN-01..08 de SPEC.

## 2. Requisitos funcionales y no funcionales

Los requisitos normativos están en `TASK-03-SPEC-DRAFT-2.md` §§12–13. Los ejes revisados son:

- Journal versionado, secuencial, append-only e íntegro.
- Checkpoint versionado, determinista y sustituido con `ATOMIC_MOVE`.
- Lock exclusivo de Coordinator local.
- Replay del estado checkpointed y validación del snapshot.
- Política de tail por tipo, con recuperación automática solo para ADMIT/COMPLETE conforme R3.1.
- Idempotencia por transitionKey/recoveryKey.
- Orden total lógico de ADMIT/FENCE.
- Startup fail-closed y avance durable de CoordinatorEpoch antes de servicio.
- Preservación de D-01/D-02/D-03/D-VD03-5.
- Exclusión del Member ledger, wire protocol, HTTP gate y tareas posteriores.

## 3. Reglas de negocio e invariantes

1. Solo el estado checkpointed habilita ACK/permiso; un tail íntegro no es autorización por sí mismo.
2. El Coordinator permanece cerrado durante RECOVERING, RECOVERY_REQUIRED y FAILED_CLOSED.
3. El lock exclusivo se obtiene antes de inspeccionar/servir el almacén.
4. No se habilitan operaciones antes de persistir el epoch de ese arranque según política aprobada.
5. ADMIT confirmado antes de FENCE permanece como admisión previa in-flight y debe resolverse; ADMIT antiguo no puede confirmarse después de FENCE.
6. ADMIT íntegro en tail se aborta explícitamente; no se emite Permit ni se crea otra admisión para la misma transitionKey.
7. COMPLETE íntegro en tail solo se aplica si referencia un ADMIT pendiente del mismo Member/boot y tiene resultado permitido; se recupera una sola vez.
8. Ningún otro tipo tail recibe regla por analogía. Mientras su política no esté aprobada, queda RECOVERY_REQUIRED.
9. Retry misma clave/mismo payload recupera mismo resultado; misma clave/payload distinto se rechaza.
10. D-01 consume el Permit en START_INTENT confirmado durablemente; expiración previa provoca rechazo durable; expiración posterior no cancela.
11. `ADMISSION_REJECTED_EXPIRED` solo libera bloqueo después de confirmación durable; fallo/ambigüedad conserva bloqueo.
12. PENDING y UNKNOWN no reconciliado bloquean COMMIT_GLOBAL; RECONCILED_UNKNOWN no equivale a COMPLETE.
13. PERMIT_RECEIVED pertenece al ledger local Member; no se agrega escritura central redundante.
14. REBIND requiere ledger Member íntegro y evidencia de terminación/no continuidad; TASK-03 no puede producir esa evidencia.
15. No fallback no atómico, truncamiento, repair automático ni timeout como transición terminal.
16. Una hash-chain local no detecta sustitución conjunta por copia antigua íntegra sin ancla externa.
17. La prueba de terminación de proceso no demuestra pérdida de energía/kernel/disco/hardware.

## 4. Matriz de eventos y tail

| Evento | Caso normal | Tail esperado en DRAFT-2 | Escenario/evidencia |
|---|---|---|---|
| ADMIT | Checkpoint antes del permiso | `ADMIT_ABORTED_UNCOMMITTED`, checkpointed, no handler | T03-TL-01: event original, recoveryKey/ID y seq/hash |
| COMPLETE | Tras write HTTP server | `RECOVERED_COMPLETE` una sola vez si referencia válida al ADMIT pendiente same Member/boot | T03-TL-02: originalEventId, recoveryEventId, estado y checkpoint |
| FENCE | Cierra nuevas admisiones | `RECOVERY_REQUIRED`; no inferir que fence confirmó | T03-TL-03: tail presente, no servicio/reapertura |
| HANDLER_START_INTENT | Consume Permit según D-01 | `RECOVERY_REQUIRED`; no generar ACK/permiso desde tail | T03-TL-04: START_INTENT tail y handler no autorizado |
| ADMISSION_REJECTED_EXPIRED | Rechazo durable conforme D-VD03-5 | Política pendiente; no liberar bloqueo por tail sin checkpoint | T03-TL-05: tail, status y admission aún bloqueante |
| RECONCILE_UNKNOWN | Outcome desconocido con evidencia explícita | `RECOVERY_REQUIRED`; no liberar ni COMPLETE por tail | T03-TL-06: no commit global |
| COMMIT_GLOBAL | Commit durable tras comprobar condiciones | `RECOVERY_REQUIRED`; no publicar GLOBAL_ACTIVE por tail | T03-TL-07: gate cerrado |
| Coordinator epoch advance | Persistir antes de cada restart | Política pendiente; no habilitar operaciones | T03-TL-08: startup cerrado hasta epoch confirmado |
| Recovery event | Acción estable y correlacionada | Reusar event válido para acción aprobada; otros tipos requieren regla | T03-TL-09: restart repetido no duplica |

## 5. Escenarios positivos

| ID | Escenario | Resultado esperado | Evidencia futura |
|---|---|---|---|
| T03-P01 | Append, journal force, checkpoint temp/force/close y atomic move exitosos | Estado checkpointed; ACK posterior al punto de confirmación | Journal/checkpoint seq/hash y ACK correlacionados |
| T03-P02 | Restart desde journal/checkpoint íntegros sin tail | Estado reconstruido igual al snapshot; luego epoch durable y servicio | Snapshot replay y evento/registro de epoch |
| T03-P03 | Retry same transitionKey y same payload | Mismo eventId/admissionId/outcome; sin transición duplicada | Índice y journal antes/después |
| T03-P04 | ADMIT confirmado antes de FENCE | Admission previa queda in-flight hasta resolución válida | Orden durable y ledger de referencia |
| T03-P05 | COMPLETE válido en tail | `RECOVERED_COMPLETE` una sola vez | Original, recoveryKey/ID y checkpoint |
| T03-P06 | Recovery reiniciado repetidamente | Mismo recoveryEventId; ninguna duplicación | Secuencias y hashes por reinicio |
| T03-P07 | Directory force declarado unsupported pero atomic move/checkpoint verificable | Registrar limitación; claims de proceso limitados al FS ensayado | Provider, excepción/capability y checkpoint |
| T03-P08 | ACK perdido después de checkpoint | STATUS por misma clave recupera resultado | Misma transitionKey y un solo evento lógico |

## 6. Escenarios negativos

| ID | Escenario | Resultado esperado | Evidencia futura |
|---|---|---|---|
| T03-N01 | Registro parcial/hash/schema inválido | FAILED_CLOSED; no truncar ni continuar | Bytes fixture y diagnóstico |
| T03-N02 | Journal más corto que checkpoint | FAILED_CLOSED | Seq/hash esperado y EOF |
| T03-N03 | Secuencia duplicada/saltada o eventId conflictivo | FAILED_CLOSED | Registros conflictivos |
| T03-N04 | Snapshot no coincide con replay | FAILED_CLOSED/RECOVERY_REQUIRED | Snapshot vs estado derivado |
| T03-N05 | Lock ocupado por segundo Coordinator | Segundo proceso sin operaciones ni ACK | Proceso, lock result y gate cerrado |
| T03-N06 | `ATOMIC_MOVE` unsupported/falla | Sin fallback; sin ACK; Coordinator cerrado | Excepción y archivos observados |
| T03-N07 | I/O error/resultado ambiguo después de rename | Mantener cerrado y verificar por replay; no equiparar con unsupported | Estado de checkpoint, excepción, replay |
| T03-N08 | Same transitionKey con payload distinto | Rechazar; evento previo intacto | Clave y payloadDigest |
| T03-N09 | Tail de tipo sin política | RECOVERY_REQUIRED; no inferir | Tipo y motivo de detención |
| T03-N10 | Recovery key duplicada con IDs/payload incompatibles | FAILED_CLOSED | Todos los eventos conflictivos |
| T03-N11 | Permit expirado antes de START_INTENT y persistencia de rechazo falla | Admission sigue bloqueante; no handler | Estado de persistencia y ledger |
| T03-N12 | Permit expirado después de START_INTENT confirmado | No cancelar ni revalidar en HANDLER_ENTERED/body | Tiempos/orden bajo reloj de prueba aprobado |
| T03-N13 | `ADMISSION_REJECTED_EXPIRED` encontrado en tail no checkpointed | No derivar REJECTED_NOT_STARTED; RECOVERY_REQUIRED | Tail y status bloqueante |
| T03-N14 | REBIND sin evidencia Member | No rebind; RECOVERY_REQUIRED | Falta/invalidación de ledger/boot evidence |

## 7. Casos borde y crash matrix

La matriz se prueba solo después de las autorizaciones de implementación/experimento aplicables.

| ID | Corte/condición | Resultado esperado |
|---|---|---|
| T03-C01 | Antes de append | Evento ausente; no inferir transición |
| T03-C02 | Durante append | Tail parcial/corrupto; FAILED_CLOSED |
| T03-C03 | Después de journal force, antes de checkpoint temp | Tail válido; recovery según tipo, no ACK |
| T03-C04 | Durante temp write/force | Checkpoint anterior; temp no se acepta como checkpoint; clasificar tail |
| T03-C05 | Después de close temp, antes de atomic move | Checkpoint anterior sigue siendo autoridad; no promover temp residual automáticamente |
| T03-C06 | ATOMIC_MOVE unsupported/falla | Sin fallback; no ACK; cerrado |
| T03-C07 | Después de atomic move, antes de ACK | Inspeccionar checkpoint: si seq/hash nuevos verificables, retry devuelve resultado existente; no duplicar |
| T03-C08 | Directory force unsupported | Registrar capability no soportada; limitar claims al process termination bajo filesystem ensayado |
| T03-C09 | Directory force arroja error real/ambiguo después de rename | No clasificar como unsupported; mantener cerrado y verificar con replay |
| T03-C10 | ACK perdido después de checkpoint | STATUS por misma clave; una transición lógica |
| T03-C11 | Segundo crash durante recovery ADMIT/COMPLETE | Reutilizar mismo recoveryKey/ID o FAILED_CLOSED si parcial/conflictivo |
| T03-C12 | Dos Coordinators compiten por lock | Un único lock owner; contender permanece cerrado |
| T03-C13 | ADMIT concurrente con FENCE | Un orden total durable; ninguna admisión antigua confirmada después de FENCE |
| T03-C14 | FENCE tail no checkpointed | RECOVERY_REQUIRED; no asumir cierre confirmado ni abrir operación |
| T03-C15 | START_INTENT tail no checkpointed | RECOVERY_REQUIRED; no ACK/handler derivado del tail |
| T03-C16 | Rechazo expiry tail no checkpointed | No liberar admisión hasta política aprobada; no handler |
| T03-C17 | Crash antes/durante/después de epoch advance | No habilitar servicio hasta epoch durable; tail requiere regla aprobada |
| T03-C18 | Temporales residuales presentes | No promover/borrar sin política; inspección y estado cerrado si son ambiguos |
| T03-C19 | Dos eventId distintos para misma recoveryKey | FAILED_CLOSED |
| T03-C20 | Restart con checkpoint/journal copiados desde versión antigua íntegra | Limitación no detectable localmente; no afirmar anti-rollback |

## 8. Estrategia de testing

- Unit tests: canonicalización, codec, digest/hash, schema, secuencia, reducer, snapshot, claves de transición y recovery.
- File integration tests: append/force, checkpoint temp/force/atomic move, directory force y replay.
- Multi-process local: lock exclusivo y segundo Coordinator cerrado.
- Concurrency: ADMIT/FENCE con barreras deterministas.
- Restart: Coordinator hijo, cortes en fronteras y replay del mismo almacén.
- Fault injection: registrar antes/después de cada I/O y antes/después de ACK.
- Expiry: reloj controlado solo después de resolver/autorizar parámetros TTL/clock pendientes; D-01/D-VD03-5 se prueban en sus fronteras sin inventar duración.
- Windows filesystem: directorio temporal aislado, filesystem/provider registrado y experimento separado aprobado.
- Security: logs sin tokens, passwords, claves privadas, PSK/tickets ni secretos.
- No usar `sleep` para establecer orden. `ProcessBuilder`/terminación forzada demuestra terminación de proceso local, no energía/hardware ni muerte remota.

## 9. Riesgos técnicos y de seguridad

| Riesgo | Severidad | Control |
|---|---|---|
| ATOMIC_MOVE no soportado/semántica distinta | Crítico | Fail-closed; sin fallback; evidencia real antes de claims |
| Directory force unsupported | Alto | Registrar limitación; no claims de durabilidad eléctrica |
| I/O error después de rename | Crítico | Replay/validación; no clasificar como unsupported por inferencia |
| Tail no tipado | Crítico | RECOVERY_REQUIRED |
| Epoch ambiguo | Crítico | No habilitar Coordinator |
| Carrera ADMIT/FENCE | Crítico | Serializador lógico y pruebas con barreras |
| Estado memoria vs durable | Crítico | Publicar solo después de checkpoint confirmado |
| Recovery duplicado | Alto | recoveryKey/recoveryEventId estable; conflictos cerrados |
| Rebind sin ledger Member | Crítico | No ejecutar dentro de TASK-03 |
| Logs con secretos | Alto | Redacción y tests de logs |
| Rollback de copia íntegra | Limitación aceptada | Documentar ausencia de ancla externa |

## 10. Trazabilidad requisitos → escenarios → aceptación

| Requisito SPEC | Regla/invariante | Escenario principal | Evidencia/aceptación |
|---|---|---|---|
| T03-FR-001 | Schema y campos por tipo | P01, N01, N03 | Codec valida versión y campos obligatorios |
| T03-FR-002 | Canonicalización sin ambigüedad | P01, N01, N08 | Vectores deterministas y payloadDigest/recordHash |
| T03-FR-003 | Journal completo + force antes de checkpoint | P01, C01–C04 | bytes/force, secuencia y replay |
| T03-FR-004 | Checkpoint solo atomic move | N06–N07, C05–C09 | checkpoint/excepción; sin fallback |
| T03-FR-005 | Un solo Coordinator local | N05, C12 | lock owner y contender cerrado |
| T03-FR-006 | Replay del estado confirmado | P02, N02–N04 | snapshot y estado reconstruido coinciden |
| T03-FR-007 | Fail-closed de integridad | N01–N04, N09 | corrupción/contradicción no abre servicio |
| T03-FR-008 | Tail ADMIT/COMPLETE explícito | P05, C03, C11 | recovery solo ADMIT/COMPLETE una vez |
| T03-FR-009 | Otros tails sin política | N09, C14–C17 | RECOVERY_REQUIRED; sin inferencia |
| T03-FR-010 | Idempotencia same-key/same-payload | P03, P06, P08, N08, N10 | transitionKey/recoveryKey sin duplicación |
| T03-FR-011 | Orden total ADMIT/FENCE | P04, C13 | orden durable con barreras |
| T03-FR-012 | Startup/epoch durables | P02, C17 | no abrir antes de epoch confirmado |
| T03-FR-013 | D-01/D-02/D-03/D-VD03-5 | N11–N13, TL-05 | rechazo expiry/UNKNOWN/receipt local respetados |
| T03-FR-014 | Rebind requiere evidencia Member | N14 | no rebind sin ledger válido |
| NFR-001 | Coordinator único local | N05, C12, C20 | sin claims multi-host/consenso |
| NFR-002 | Sin dependencias externas nuevas | Cierre T03-12 | diff POM/dependency manifest no agrega dependencias |
| NFR-003 | Journal append-only sin compactación | P01, C01–C06 | bytes/eventos anteriores preservados; no truncate |
| NFR-004 | Serialización determinista/versionada | P01–P03, N01, N03 | vectores reproducibles y schema validado |
| NFR-005 | Fail-closed, sin fallback/soft-fail | N01–N14, C01–C20 | ningún caso ambiguo abre servicio |
| NFR-006 | Sin `sleep` como sincronización | C13 | barreras/latches verificables |
| NFR-007 | Logs sin secretos | Prueba de seguridad T03-11 | inspección/test de logs sanitizados |
| NFR-008 | Métricas p50/p95/p99 sin SLO inventado | Informe T03-12 | mediciones identificadas con entorno/muestra; no presupuesto asumido |
| NFR-009 | Terminación de proceso distinta de pérdida eléctrica | P07, C08–C09 | claims limitados al filesystem/escenario probado |
| NFR-010 | Límite anti-rollback explícito | C20 | no claim de detección de copia antigua íntegra |
| NFR-011 | Mantener exclusiones de alcance | Cierre T03-12 | diff de archivos confirma no cambios en artefactos excluidos |

## 11. Trazabilidad de tareas → entregables/evidencia

| Tarea | Entregable esperado | Evidencia de salida |
|---|---|---|
| T03-00 | Baseline read-only | Árbol, estado Git si existe, POM, inventario y hashes comparados con TASK-01 |
| T03-01 | Catálogo de eventos y decisiones | Matriz por tipo, política tail, decisiones cerradas/bloqueadas y aprobación registrada |
| T03-02 | Codec/schema | Vectores de canonicalización, round-trip, digest/hash y casos inválidos |
| T03-03 | Lock OS | Evidencia de un lock owner y contender cerrado en el filesystem ensayado |
| T03-04 | Journal writer | Bytes/seq/hash, resultados force, fallos parciales y ausencia de ACK prematuro |
| T03-05 | Checkpoint writer | Temporal, force, close, atomic move, directory force y clasificación de excepción |
| T03-06 | Reducer | Snapshots deterministas y transiciones inválidas rechazadas |
| T03-07 | Replay | Comparación journal/checkpoint/snapshot y diagnósticos fail-closed |
| T03-08 | Tail recovery | Casos ADMIT/COMPLETE recuperados una sola vez; otros tipos detenidos |
| T03-09 | Serializador | Historial ordenado ADMIT/FENCE bajo contención determinista |
| T03-10 | Índice idempotente | Retry same-key/same-payload y rechazo de payload conflictivo, incluso tras restart |
| T03-11 | Crash matrix | Registro de cada frontera, resultado durable, replay y ACK permitido |
| T03-12 | Cierre técnico | Informe, hashes, comandos, tests, Quality Gate, exclusiones y límites demostrados |

La evidencia de las tareas de implementación se generará únicamente después de las autorizaciones correspondientes. Esta tabla define qué deberá demostrarse; no afirma que ya se haya ejecutado.

## 12. Decisiones y bloqueos antes de aprobar implementación

1. Confirmar source set y hashes del workspace local 009A.1 (baseline T03-00).
2. Cerrar encoding, canonicalización, algoritmo de recordHash/payloadDigest y formato de checkpoint.
3. Aprobar política tail de cada evento no cubierto expresamente por R3.1 o mantenerlo como recovery manual/stop.
4. Definir evento, clave, primer arranque y crash recovery del avance `coordinatorEpoch`.
5. Clasificar directory force unsupported frente a error real/ambiguo después de rename.
6. Definir temporales residuales y transición RECOVERY_REQUIRED/FAILED_CLOSED.
7. Obtener autorización específica para ensayar FileLock/ATOMIC_MOVE/directory force en Windows.
8. Definir contrato y evidencia Member requeridos para REBIND; ejecución depende de TASK-04.
9. No congelar TTL/clock/skew/renewal por inferencia; decisiones temporales heredadas siguen pendientes.

## 13. Veredicto

**Diseño viable con condiciones para continuar la preparación documental; no listo para aprobación de implementación.** DRAFT-2 corrige numeración, prioridades, catálogo de eventos, startup/epoch, checkpoint/directory force y cobertura de escenarios. Los puntos del §11 deben cerrarse o aceptarse expresamente como bloqueos antes de congelar diseño.

No se ejecutaron Maven, tests, builds ni experimentos. No hubo modificaciones de código ni configuración. El gate de factibilidad CRL para producción permanece cerrado.
