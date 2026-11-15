# TASK-03 — Validación del diseño

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-DESIGN-VALIDATION-DRAFT-1`  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Validación:** análisis documental; no se ejecutaron builds, tests ni experimentos.

## 1. Requisitos funcionales y no funcionales

La matriz de requisitos está en `TASK-03-SPEC-DRAFT-1.md` §§10–11. Los ejes obligatorios son: journal versionado append-only, checkpoint atómico, lock exclusivo, replay determinista, tail recovery tipado, idempotencia, serialización ADMIT/FENCE y fail-closed.

## 2. Reglas de negocio e invariantes

1. Solo el estado checkpointed habilita ACK/permiso; journal tail durable pero no checkpointed no es una autorización.
2. No se habilita Coordinator antes de lock + replay + reconciliación completa.
3. No se reordena ADMIT confirmado detrás de FENCE.
4. No hay ADMIT G1 confirmado después de FENCE G1.
5. Timeout o ACK perdido no equivale a fallo ni éxito; consultar la misma transitionKey.
6. Idempotencia exige misma clave y mismo payload; misma clave con payload distinto se rechaza.
7. ADMIT tail sin checkpoint se aborta explícitamente; COMPLETE tail válido se recupera una vez; nunca se descarta el tail silenciosamente.
8. `PENDING` y `UNKNOWN` no reconciliado bloquean `COMMIT_GLOBAL`; `RECONCILED_UNKNOWN` nunca es `COMPLETE`.
9. Rechazo expiry confirmado por `ADMISSION_REJECTED_EXPIRED` durable deriva a `REJECTED_NOT_STARTED`; fallo/ambigüedad conserva bloqueo.
10. Hash-chain/checkpoint local no detectan reemplazo conjunto por una copia antigua íntegra; esta limitación no se oculta ni se resuelve con un servicio externo.
11. No afirmar durabilidad frente a pérdida de energía/hardware sin prueba explícita.

## 3. Escenarios positivos

| ID | Escenario | Resultado esperado | Evidencia futura |
|---|---|---|---|
| T03-S01 | Append event + force + checkpoint + atomic move | transición checkpointed; ACK permitido después del punto de confirmación | journal, checkpoint, seq/hash y ACK correlacionados |
| T03-S02 | Restart desde journal/checkpoint íntegros | estado reconstruido idéntico; solo después abrir Coordinator | snapshot replay y estado de inicio |
| T03-S03 | Retry same transitionKey y mismo payload | devuelve el mismo eventId/admissionId/outcome, sin nuevo evento lógico | journal + respuesta query |
| T03-S04 | ADMIT confirmado antes de FENCE | ADMIT queda in-flight/pending y se conserva al FENCE | orden secuencial durable y ledger |
| T03-S05 | COMPLETE válido en tail posterior a checkpoint | `RECOVERED_COMPLETE` una sola vez | originalEventId, recoveryKey, recoveryEventId, nuevo checkpoint |
| T03-S06 | Recovery reiniciado repetidamente | mismo recoveryEventId reutilizado; no se duplica | historial y checkpoint tras cada reinicio |

## 4. Escenarios negativos

| ID | Escenario | Resultado esperado | Evidencia futura |
|---|---|---|---|
| T03-N01 | Journal con registro parcial/hash inválido | FAILED_CLOSED; no truncar ni continuar | bytes fixture y diagnóstico |
| T03-N02 | Journal más corto que checkpoint | FAILED_CLOSED | seq/hash checkpoint y EOF |
| T03-N03 | Secuencia duplicada/saltada o eventId conflictivo | FAILED_CLOSED | registro conflictivo y error |
| T03-N04 | Snapshot checkpoint discordante con replay | FAILED_CLOSED/RECOVERY_REQUIRED | snapshot vs estado reconstruido |
| T03-N05 | Lock ya adquirido por otro Coordinator | segundo proceso sin operaciones ni ACK | PID, lock result y gate cerrado |
| T03-N06 | `ATOMIC_MOVE` unsupported/fail | no fallback; sin ACK; RECOVERY_REQUIRED | excepción, archivos before/after |
| T03-N07 | TransitionKey reutilizada con payload distinto | conflicto, evento previo intacto | claves y payload digests |
| T03-N08 | Tipo de tail sin regla R3.1 aprobada | RECOVERY_REQUIRED; no inferir | evento y razón de stop |

## 5. Casos borde y crash matrix

La implementación futura debe instrumentar cortes deterministas en cada frontera. Los resultados de las fronteras deben coincidir con R3.1 §3.15:

| ID | Corte | Resultado esperado |
|---|---|---|
| T03-C01 | Antes de append | evento ausente; ninguna transición inferida |
| T03-C02 | Durante append | tail parcial/corrupto; FAILED_CLOSED |
| T03-C03 | Después de append, antes de force | estado no confirmado; si tail parcial/invalid, FAILED_CLOSED |
| T03-C04 | Después de force journal, antes de checkpoint temp | tail válido; recovery por tipo, no ACK |
| T03-C05 | Durante temp write/force | checkpoint viejo permanece; RECOVERY_REQUIRED y tail tipado |
| T03-C06 | Después de close temp, antes de atomic move | checkpoint viejo; temp no aceptado como checkpoint |
| T03-C07 | `ATOMIC_MOVE` falla | sin fallback no atómico; no ACK |
| T03-C08 | Después de atomic move, antes de directory force/ACK | estado potencialmente ambiguo; gate cerrado hasta STATUS/replay y clasificación de force |
| T03-C09 | Después de checkpoint confirmado, antes del ACK | outcome duradero recuperable por misma transitionKey; ACK reintentado sin duplicar |
| T03-C10 | Segundo crash durante replay/recovery | reutiliza recoveryKey/eventId existente o falla cerrado; nunca compensa dos veces |
| T03-C11 | ADMIT concurrente con FENCE | un orden total durable, conforme a la operación lógica que confirma primero |
| T03-C12 | FENCE confirmado antes de ADMIT | ADMIT antiguo rechazado; no nuevo admissionId/permiso |

Los tests usan hooks/latches/barreras, no `sleep` como sincronización. `ProcessBuilder`/terminación forzada puede demostrar muerte de proceso en el host de laboratorio; no es prueba de pérdida eléctrica.

## 6. Estrategia de testing

- Unit tests para canonicalización, codec, integridad, reducer, transitionKey/recoveryKey y snapshot.
- File integration tests para append/force/checkpoint/atomic move y replay.
- Multi-process test de exclusión de lock del Coordinator en filesystem local.
- Concurrency test ADMIT/FENCE con una barrera determinista y dos contendientes.
- Restart tests con JVM Coordinator hija y cortes en fronteras journal/checkpoint/ACK.
- Tests de Windows filesystem exclusivamente en directorio temporal aislado, cuando estén autorizados; registrar filesystem/provider/resultado real.
- Verificación de logs para ausencia de tokens, passwords, claves privadas, PSK/tickets y secretos.

Las pruebas completas de dos Members, gate HTTP, sockets TLS persistentes y CRL end-to-end corresponden a tareas posteriores y no deben colarse en TASK-03.

## 7. Riesgos técnicos y de seguridad

| Riesgo | Severidad | Control |
|---|---|---|
| `ATOMIC_MOVE` no soportado o comportamiento distinto en filesystem Windows | Crítico | Fail-closed, sin fallback; evidencia real antes de claims |
| `force(true)` de directorio no soportado | Alto | Registrar limitación; no reclamar durabilidad eléctrica |
| File lock con semántica distinta en FS/red | Crítico | Restringir laboratorio a filesystem local probado; no claims cross-host |
| Estado en memoria divergente del durable | Crítico | Publicar nuevo estado solo tras commit confirmado; replay al restart |
| Dos operaciones críticas intercaladas | Crítico | Serializador lógico completo ADMIT/FENCE |
| Replay procesa evento incompleto | Crítico | Validación de esquema/seq/hash/referencias y políticas tail por tipo |
| Logs revelan bearer/secretos | Alto | redaction tests, no logging de payload/token |
| Rollback a copia íntegra antigua | Limitación aceptada | documentar; no ancla externa autorizada |

## 8. Criterios de aceptación del diseño

- SPEC/PLAN/TASKS contienen los mismos límites e invariantes.
- Cada requisito tiene escenario y evidencia futura asociada.
- La crash matrix contempla append, force, temp, atomic move, directory force y ACK.
- La política de tail recovery es explícita por tipo; ningún tipo desconocido recibe regla implícita.
- `ADMIT` y `FENCE` comparten la misma frontera lógica de serialización.
- Windows filesystem es una condición de evidencia abierta, no una garantía asumida.
- No se introducen dependencias, HA, transportes o tareas posteriores.
- El gate de producción CRL permanece cerrado.

## 9. Bloqueos/decisiones antes de aprobar implementación

1. Confirmar el source set y hashes del workspace local actual 009A.1; el GitHub repo no contiene ese código.
2. Cerrar recordHash/canonicalization exacta de journal y checkpoint a partir de R3.1; el recoveryEventId sí se fija como SHA-256(canonical(recoveryKey)).
3. Definir reglas para tipos tail que R3.1 no describe explícitamente.
4. Clasificar `force(directory)` unsupported versus error real después de rename.
5. Verificar semántica de lock y `ATOMIC_MOVE` en el filesystem local Windows mediante prueba controlada futura, con autorización.
6. Confirmar formato de checkpoint y manejo de temporales residuales.
7. Asegurar que no se cierre por timeout ni se trunque/repare automáticamente ningún journal.

## 10. Veredicto preliminar

**Diseño viable con condiciones para continuar la preparación documental.** No es aprobación de implementación. Los puntos 1–6 deben resolverse/documentarse antes de aprobar TASK-03 para código. El presente documento no ejecuta ni autoriza experimentos, Maven, tests, builds ni cambios de código.
