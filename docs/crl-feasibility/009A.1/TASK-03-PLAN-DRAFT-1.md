# TASK-03 — Plan ejecutable de persistencia del Coordinator

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-PLAN-DRAFT-1`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Implementación:** no autorizada.

## 1. Objetivo

Planificar la entrega del subsistema de persistencia durable del Coordinator conforme a `TASK-03-SPEC-DRAFT-1.md` y el diseño R3.1. El plan es ejecutable solo después de aprobación humana de diseño e implementación.

## 2. Baseline y ubicación de trabajo

La inspección TASK-01 sitúa el arnés 009A.1 en:

`C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a1\project`

La fuente está bajo `src/test/java/lab/crl`, no en `src/main/java`. El repositorio público `iso8583-gateway` contiene documentación del Gateway y documentación archivada CRL; no contiene `pom.xml` ni fuentes del arnés 009A.1. No usar la raíz GitHub como workspace de implementación de TASK-03.

**Precondición de implementación:** antes de modificar archivos, volver a inspeccionar read-only el directorio local 009A.1, registrar commit/estado si está bajo Git, hash de todos los archivos candidatos y POM, y comparar con el informe TASK-01. El ZIP `CRL-FEASIBILITY-009A-evidence.zip` es antecedente 009A, no reemplaza la verificación del estado local 009A.1.

## 3. Candidatos de código (no autorizan modificaciones)

| Artefacto | Observación de TASK-01/evidencia | Tratamiento propuesto |
|---|---|---|
| `src/test/java/lab/crl/RotationCoordinator009A.java` | Coordinator actual con mapas en memoria, control TCP loopback y journal texto append + force | Candidato a fachada/adaptador; separar persistencia del estado de transición, evitando ampliar wire protocol |
| `src/test/java/lab/crl/DistributedRotation009ATest.java` | Harness de dos JVM y pruebas TLS existentes | Candidato a fixture de proceso; no ampliar integración full member/HTTP dentro de TASK-03 |
| `src/test/java/lab/crl/DistributedTlsInstance009A.java` | Miembro, comandos y persistencia simple generation/hash; fallback no atómico | Fuera de modificación TASK-03; TASK-04/05/06 |
| Clases nuevas | Aún no existen según inspección TASK-01 | Codec, journal store, checkpoint store, reducer/replay y recuperación; límites de clase se deciden tras inspección actual |

No fijar nombres ni crear clases hasta comprobar el workspace actual y la estructura real.

## 4. Estrategia por etapas

### T03-00 — Baseline read-only y congelamiento de alcance

- Inventariar `pom.xml`, árbol de fuentes, informes y archivos de evidencia actuales.
- Calcular hashes de archivos candidatos y verificar que no haya cambios inesperados respecto de la inspección TASK-01.
- Confirmar Java 21 y stack existente solo desde archivos/evidencia atribuible al proyecto; no asumir que el dependency-tree 008 prueba la resolución 009A.1.
- Identificar ruta de datos de ensayo y filesystem destino sin escribir ni probar.
- Detener si el directorio no es el workspace esperado, hay secretos o el estado difiere de forma material.

**Salida:** inventario y baseline de hashes; ningún archivo modificado; ninguna build/test.

### T03-01 — Congelar contrato del almacenamiento

- Definir envelope `JournalRecord`, campos opcionales por tipo y canonicalización versionada length-prefixed.
- Definir secuencia monotónica, `eventId`, `transitionKey`, payload digest, previous/record hash.
- Definir `Checkpoint` schema, secuencia/hash confirmados y snapshot determinista.
- Cerrar tabla de tipos de evento y política tail por tipo; no inventar recovery para eventos sin regla R3.1.
- Confirmar invariantes de ADMIT/FENCE y transición de autoridad.

**Salida:** decisión técnica del modelo/codec aprobada y casos de test derivados. No persistencia ejecutable todavía.

### T03-02 — Journal append-only y lock exclusivo

- Diseñar apertura/creación segura, `FileLock` exclusivo de proceso y ownership del recurso.
- Implementar/validar append de registro completo bajo lock y `force(true)`.
- Manejar write parcial/IOException como estado no confirmado; no emitir ACK.
- Prohibir truncamiento automático o recuperación por ignorar el último registro.

**Salida:** pruebas focalizadas de escritura completa, fallo/partial write y exclusión de segundo proceso.

### T03-03 — Checkpoint atómico

- Serializar snapshot determinista.
- Crear temporal en el mismo filesystem/directorio que checkpoint destino.
- Escribir todos los bytes, `force(true)`, cerrar y sustituir con `ATOMIC_MOVE`.
- No implementar fallback a move no atómico.
- Intentar force del directorio padre si soportado; registrar capability y clasificar excepciones.
- Definir cómo responde el store si el rename terminó pero la confirmación/dir force/ACK queda ambigua.

**Salida:** pruebas con filesystem Windows real y evidencia acotada. La prueba/experimento de filesystem requiere aprobación explícita separada si excede pruebas autorizadas en el plan definitivo.

### T03-04 — Replay y reconstrucción

- Adquirir lock antes de leer/reconstruir.
- Validar checkpoint, journal, secuencia, IDs, hash-chain y referencias.
- Reconstruir estado desde eventos checkpointed y comparar snapshot.
- Verificar restart con estado IDLE/GLOBAL_ACTIVE/FENCING/pending.
- Mantener servicio cerrado hasta finalizar.

**Salida:** replay determinista, tests positivos/negativos y diagnóstico fail-closed.

### T03-05 — Tail y recovery idempotente

- Implementar recuperación de ADMIT append+force sin checkpoint como `ADMIT_ABORTED_UNCOMMITTED` checkpointed, sin permiso.
- Implementar `RECOVERED_COMPLETE` para COMPLETE íntegro que refiere a ADMIT pendiente y cumple reglas R3.1.
- Derivar `recoveryKey` y `recoveryEventId` según R3.1 §3.17.
- En restart repetido, reutilizar recovery event existente; nunca añadir compensatorio duplicado.
- Para tipos no cubiertos, detener con `RECOVERY_REQUIRED` hasta política aprobada.

**Salida:** crash/restart repetido reproduce el mismo recovery event y resultado.

### T03-06 — Serialización de transiciones

- Construir una única frontera lógica Coordinator para check + estado + evento + append/force/checkpoint + publish + ACK.
- Serializar ADMIT y FENCE en el mismo orden total durable.
- No confiar en `synchronized record()` aislado.
- Publicar estado en memoria solo tras confirmación durable; error/ambiguo mantiene closed.
- Consultar idempotencia por la misma TransitionKey tras ACK perdido.

**Salida:** pruebas deterministas concurrentes ADMIT/FENCE y pérdida de respuesta.

### T03-07 — Validación focalizada y evidencia

- Ejecutar unit/integration tests autorizados del store y coordinator.
- Inyectar fallos en append, force, temp write/force, close, atomic move, directory force y antes/después de ACK.
- Reiniciar proceso real del Coordinator para verificar lock/replay.
- Ejecutar Quality Gate disponible únicamente si forma parte de la aprobación de implementación.
- Generar informe con comandos/resultados, hashes, límites demostrados, tareas pendientes y cambios exactos.

**Salida:** evidencia reproducible y dictamen TASK-03 para revisión técnica. No afirmar durabilidad eléctrica.

## 5. Dependencias y secuencia

| Orden | Entrega | Depende de | Bloquea |
|---|---|---|---|
| 0 | Baseline read-only | Paquete TASK-02 aprobado; workspace local disponible | Todo lo demás |
| 1 | Modelo/codec | SPEC y decisiones de schema | Journal/checkpoint/replay |
| 2 | Journal + lock | Codec y política de ownership | Checkpoint y replay |
| 3 | Checkpoint | Journal y política filesystem | Confirmación durable/replay |
| 4 | Replay/reducer | Journal + checkpoint | Startup y tail recovery |
| 5 | Tail recovery | Replay + recoveryKey | Idempotencia restart |
| 6 | Serializador ADMIT/FENCE | Reducer + store | Operación Coordinator segura |
| 7 | Verificación focalizada | Todos los anteriores | Cierre TASK-03 |

No se incluye implementación del Member ledger, transporte final, gate HTTP ni integración completa TLS/multi-JVM de las tareas posteriores.

## 6. Estrategia de pruebas

- Unitarias: canonicalización, codec, digest/hash, secuencia, reducer, checkpoint round-trip, transición válida/ilegal, dedupe.
- Integración de archivos: append/force, checkpoint temp/force/atomic move, lectura/replay y limpieza controlada de fixtures.
- Proceso: lock exclusivo con segundo proceso, terminación del Coordinator y restart/replay.
- Concurrencia: ADMIT vs FENCE con barreras/latches, idempotencia con llamadas simultáneas.
- Fault injection: límites precisos en el protocolo de persistencia; no `sleep` para sincronizar.
- Seguridad: no secretos en logs; archivos temporales con permisos/ubicación acotados según plataforma.
- Filesystem: separar resultados de terminación de proceso de resultados de pérdida de energía; esta última no se prueba ni se reclama.

## 7. Riesgos y mitigaciones

| Riesgo | Severidad | Mitigación / gate |
|---|---|---|
| `ATOMIC_MOVE` no soportado en filesystem destino | Crítico | Sin fallback; no ACK; permanecer cerrado |
| Directory force no soportado | Alto | Registrar limitación; claims solo hasta consistencia ante process termination probada |
| Evento append+force fuera del checkpoint | Alto | Tail tipado; ADMIT abort, COMPLETE recovered; otros tipos cerrados |
| Checkpoint y journal contradictorios | Crítico | `FAILED_CLOSED`, sin reparación automática |
| Doble Coordinator | Crítico | OS lock exclusivo; segundo proceso no opera |
| Carrera ADMIT/FENCE | Crítico | Serialización lógica end-to-end y tests deterministas |
| Recovery duplicado | Alto | recoveryKey/recoveryEventId estables; conflicto fail-closed |
| Rollback a copia íntegra antigua | Limitación aceptada | Documentar no detectable sin ancla externa; no añadir ancla |
| Confusión con documentos/producto Gateway | Medio | Mantener artefactos 009A.1 bajo carpeta docs separada; código queda en workspace de laboratorio aislado |

## 8. Gates de parada

Detenerse y solicitar decisión si:
- el workspace local 009A.1 no coincide con baseline TASK-01;
- se requiere tocar `DistributedTlsInstance009A`, Member ledger, HTTP handler, transporte final o archivos del ISO8583 Gateway;
- el filesystem no soporta `ATOMIC_MOVE` o su comportamiento es ambiguo;
- el diseño exige fallback no atómico, truncamiento, soft-fail, auto-repair o tolerar contradicciones;
- se requiere agregar dependencia, base de datos, servicios externos, HA/consenso o alcance financiero;
- se pretende fijar TTL/reloj/skew/renewal sin decisión aprobada.

## 9. Estado final esperado

TASK-03 quedará cerrada solo si el store/replay/lock/serialization cumplen los criterios de SPEC con evidencia reproducible, las limitaciones filesystem están registradas y no hay desviaciones de alcance. El cierre TASK-03 no autoriza TASK-04 ni tareas posteriores. El gate CRL para producción continúa cerrado.
