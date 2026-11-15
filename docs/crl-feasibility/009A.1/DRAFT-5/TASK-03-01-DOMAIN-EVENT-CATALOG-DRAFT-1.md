# TASK-03-01 — Catálogo de dominio, eventos y decisiones de persistencia

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Base de trabajo:** T03-00 aceptada; diseño fuente R3.1; TASK-02-DRAFT-2 `CLOSED — HUMAN APPROVED`; H-01 `ACCEPTED — HUMAN APPROVED`; TASK-03-DRAFT-4.  
**Autorización vigente:** preparación/validación documental solamente. No autoriza implementación, cambios de código/configuración, Maven, builds, tests ni experimentos de filesystem.

---

## 1. Propósito y resultado

Esta entrega establece el catálogo de eventos persistidos por el Coordinator, separa eventos centrales de evidencias que pertenecen al Member y presenta una propuesta técnica revisable para D03-OPEN-01..08.

El análisis es documental. No congela el codec, no aprueba políticas nuevas de tail, no define parámetros temporales pendientes, no cierra el diseño integral TASK-03 y no habilita implementación.

**Resultado principal:** el catálogo funcional puede quedar descrito sin ambigüedad a nivel de dominio. Sin embargo, las ocho decisiones D03-OPEN siguen siendo bloqueantes para el freeze del diseño según TASK-03-SPEC-DRAFT-4 §15. Se proponen alternativas y recomendaciones para revisión humana; ninguna se considera aprobada por estar escrita aquí.

## 2. Precedencia y fuentes

1. `CRL-FEASIBILITY-009A.1-DESIGN(9).md`, revisión R3.1, especialmente §§3.2, 3.11, 3.12, 3.13, 3.15, 3.17 y 3.18.
2. Paquete TASK-02-DRAFT-2, estado `CLOSED — HUMAN APPROVED`, en especial Domain Model, Protocol Contracts, State Machines, Scenario Matrix y Design Validation. Se conservan D-01, D-02, D-03 y D-VD03-5.
3. Aceptación humana H-01: serialización lógica integral de ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED. H-01 no cierra política tail.
4. TASK-03-SPEC/PLAN/TASKS/DESIGN-VALIDATION-DRAFT-4, commit `bef7aa7c615745e19f548b47854fb4e37d543b18`.
5. Informe T03-00 aceptado por el usuario el 2026-11-15. La inspección fue estática y read-only; no se ejecutaron builds ni pruebas.

En caso de conflicto, se preservan las decisiones TASK-02 aprobadas y R3.1. H-01 prevalece para el orden lógico vivo de las cuatro operaciones que cubre, pero no puede extrapolarse a la recuperación de tails.

## 3. Límites del dominio

### 3.1 Autoridad y alcance

- Existe un único Coordinator local de laboratorio. El OS lock excluye otro escritor local sobre el mismo almacén cuando el filesystem ensayado lo soporta; no proporciona consenso, quorum, HA ni fencing multi-host.
- El experimento usa GET sintético sin efectos financieros. No se extrapolan sus semánticas a transacciones de negocio.
- El servicio opera fail-closed ante corrupción, contradicción, ambigüedad o evidencia insuficiente.
- Journal y checkpoint locales no detectan la sustitución conjunta por una copia antigua íntegra sin ancla anti-rollback externa. Esa ancla permanece fuera de alcance.
- No se declaran claims PCI, FIPS, HSM, producción ni compatibilidad Link/Prisma.

### 3.2 Separación Coordinator / Member

El Coordinator es dueño del journal y checkpoint centrales: admisiones, fencing, resultado de START_INTENT, cierre COMPLETE, reconciliación central, commit global, epoch y recovery events.

El Member es dueño de su ledger local. `PERMIT_RECEIVED`, `HANDLER_ENTERED` y `HANDLER_RETURNED` son evidencias locales del Member. D-03 exige forzar `PERMIT_RECEIVED` localmente antes de enviar `PERMIT_RECEIVED_ACK` al Coordinator. No se agrega una escritura central redundante para duplicar esa evidencia.

La evidencia Member necesaria para distinguir NOT_STARTED de UNKNOWN, o para ejecutar REBIND, no puede ser fabricada por replay del Coordinator. Su formato/propiedad corresponde a TASK-04; el wrapper `HANDLER_ENTERED` corresponde al alcance posterior definido en TASK-06.

### 3.3 Punto de confirmación

La transición central queda confirmada cuando el checkpoint que referencia la secuencia/hash del evento nuevo ha sido sustituido mediante `ATOMIC_MOVE`, después del append completo y `force(true)` del journal, y después del write/force/close del temporal. Se intenta directory force cuando el filesystem/JDK lo soporta. El ACK, Permit o rechazo no se publica antes de ese punto y de aplicar la clasificación de error acordada.

Un registro íntegro en el journal pero posterior al checkpoint es tail: evidencia de una operación incierta, no autorización por sí sola.

## 4. Catálogo de eventos y operaciones

La tabla separa transiciones centrales durables, eventos de recovery y evidencias locales. Los nombres propuestos para eventos nuevos (en particular epoch advance) son nombres de trabajo, no contratos congelados.

| Evento / operación | Dueño | Propósito y efecto de dominio | Clave / correlación | Reglas esenciales |
|---|---|---|---|---|
| `ENROLLMENT_SET` | Coordinator | Registra el conjunto exacto e inmutable de Members de la corrida, versión y digest de IDs ordenados | `membershipVersion`, digest, `rolloutId` cuando aplique | El conjunto no es inferido por cantidad de ACKs; vacío, duplicados, omitidos, adicionales o mismatch se rechazan. Cambio solo como transición explícita permitida en IDLE, según R3.1. |
| `ADMIT` | Coordinator | Registra una admisión antes de emitir Permit y antes de permitir que el Member continúe | `(memberId, processBootId, requestId, ADMIT)` | Permit solo después del checkpoint. Misma clave/mismo payload devuelve la admisión previa; payload conflictivo se rechaza. |
| `FENCE` | Coordinator | Cierra nuevas admisiones para el epoch/rollout anterior | `commandId`/`transitionKey`, `rolloutId`, epoch afectado | Se serializa con ADMIT. No revoca el derecho de completar una admisión confirmada antes del FENCE. No permite un ADMIT nuevo posterior. |
| `HANDLER_START_INTENT` | Coordinator | Consume el Permit en el punto D-01, haciendo durable la intención antes del ACK al Member | `(admissionId, HANDLER_START_INTENT)` | Validación de vigencia y persistencia son una sola operación lógica serializada con ADMIT/FENCE. START_INTENT no prueba que el método haya comenzado. |
| `ADMISSION_REJECTED_EXPIRED` | Coordinator | Registra el rechazo pre-consumo por expiración antes del commit de START_INTENT | clave estable de la transición, ADMIT original, `admissionId`/Member/boot/request | Solo después de checkpoint confirmado deriva `REJECTED_NOT_STARTED` y libera el bloqueo. Fallo/ambigüedad conserva el bloqueo; ACK perdido consulta STATUS con la misma clave. |
| `COMPLETE` | Coordinator | Cierra admisión después de que Reactor Netty confirme la escritura HTTP en el servidor | `(admissionId, COMPLETE)` | Acredita write completion del servidor; no prueba recepción por el cliente ni resultado de negocio. Se persiste antes de retirar la admisión del conjunto pendiente. |
| `RECONCILE_UNKNOWN` | Coordinator | Registra reconciliación explícita con evidencia suficiente para sacar un UNKNOWN del conjunto bloqueante | `admissionId`, Member/boot, evidencia, operador/testId, recovery key/event | El outcome permanece `UNKNOWN`; nunca se convierte en COMPLETE. Sin evidencia íntegra y correlacionada continúa bloqueando. |
| `COMMIT_GLOBAL` | Coordinator | Publica `GLOBAL_ACTIVE` solo tras verificar el conjunto exacto de Members y cero admissions bloqueantes | `rolloutId`, membership version/digest, referencias a verificaciones | Requiere journal íntegro, Members exactos verificados, y ausencia de PENDING/UNKNOWN no reconciliado. `RECONCILED_UNKNOWN` solo puede dejar de bloquear con todos los requisitos aprobados, sin cambiar el outcome. |
| Epoch advance — nombre por decidir | Coordinator | Incrementa y persiste el `coordinatorEpoch` de cada arranque antes de habilitar operaciones | startup operation key / identidad de arranque, epoch previo y siguiente | No se habilita servicio antes del checkpoint. Evento, clave, primer arranque y crash recovery son D03-OPEN-03. |
| Recovery event | Coordinator | Registra una acción de recuperación derivada de un evento original y la deduplica entre reinicios | `recoveryKey=(originalEventId,recoveryAction)`; `recoveryEventId=SHA-256(canonical(recoveryKey))` | Valida original, acción, estado esperado/resultante y digest. No crea eventos compensatorios duplicados. Solo acciones expresamente definidas pueden recuperarse automáticamente. |
| `PERMIT_RECEIVED` | Member | Receipt durable del Permit en ledger local | `admissionId`, `grantDigest`, `processBootId` | `force(true)` local antes de continuar y antes de `PERMIT_RECEIVED_ACK`; no se replica como escritura central. |
| `HANDLER_ENTERED` | Member / wrapper | Evidencia durable de que se alcanzó la primera instrucción del wrapper protegido | `admissionId`, boot y referencia a START_INTENT | Se fuerza antes del cuerpo GET. Su ausencia después de START_INTENT no prueba que el handler no comenzó. |
| `HANDLER_RETURNED` | Member / wrapper | Evidencia durable del retorno del cuerpo GET | `admissionId`, boot y referencia a ENTERED | No equivale a HTTP write completion. |
| `STATUS` | Coordinator/Member | Consulta observacional de una clave/estado existente | clave original y correlación de identidad/boot | No muta ni confirma una transición. Timeout conserva estado no resuelto. |

### 4.1 Estados de admisión que no deben colapsarse

| Estado | Significado | ¿Bloquea `COMMIT_GLOBAL`? |
|---|---|---|
| `PENDING` | La ejecución aún puede comenzar/continuar o la evidencia no descarta esa posibilidad | Sí |
| `UNKNOWN` no reconciliado | Resultado incierto; no se demostró que la ejecución dejó de ser posible | Sí |
| `RECONCILED_UNKNOWN` | Evidencia durable demuestra muerte/no continuidad para esa admisión; el resultado de negocio sigue desconocido | Puede dejar de bloquear solo si cumple íntegramente D-02; nunca se convierte en COMPLETE |
| `REJECTED_NOT_STARTED` | Expiración anterior al consumo, después de confirmar durablemente `ADMISSION_REJECTED_EXPIRED` | No |
| `COMPLETE` | HTTP response write completado en el servidor y evento central confirmado | No |

`PERMIT_RECEIVED`, `HANDLER_START_INTENT`, `HANDLER_ENTERED`, `HANDLER_RETURNED` y `COMPLETE` representan fronteras distintas. No inferir una etapa posterior por la ausencia de un evento.

## 5. Política de tail — matriz del catálogo

La política vigente se limita a lo que R3.1 especifica expresamente. El hecho de que un tail esté completo y su hash sea válido no lo convierte en checkpoint confirmado.

| Tipo en tail íntegro posterior al checkpoint | Acción permitida por la base actual | Tratamiento seguro si no existe política aprobada |
|---|---|---|
| `ADMIT` | Recovery explícito `ADMIT_ABORTED_UNCOMMITTED`; no se envió Permit/ACK antes del checkpoint de ADMIT. Reutilizar recoveryKey y checkpointear el resultado una sola vez | No crear otro admissionId ni emitir Permit; si hay contradicción o evidencia de ACK/permiso, cerrar sin inferir |
| `COMPLETE` | `RECOVERED_COMPLETE` una sola vez, solo si referencia un ADMIT pendiente del mismo Member/boot y el resultado es permitido conforme R3.1 | Si referencia/resultado no valida, `RECOVERY_REQUIRED`/fail-closed; no inferir cierre |
| `FENCE` | Ninguna recuperación automática aprobada en TASK-03-DRAFT-4 | `RECOVERY_REQUIRED`; no suponer ni negar FENCE confirmado; no reabrir ADMIT ni emitir permisos |
| `HANDLER_START_INTENT` | Ninguna recuperación automática aprobada | `RECOVERY_REQUIRED`; no enviar ACK ni reejecutar handler. Mantener estado incierto/UNKNOWN y no asumir NOT_STARTED |
| `ADMISSION_REJECTED_EXPIRED` | Ninguna recuperación automática aprobada | `RECOVERY_REQUIRED`; no marcar `REJECTED_NOT_STARTED` ni liberar bloqueo por tail sin checkpoint |
| `RECONCILE_UNKNOWN` | Ninguna recuperación automática aprobada | `RECOVERY_REQUIRED`; conservar UNKNOWN y bloqueo hasta decisión/evidencia válida |
| `COMMIT_GLOBAL` | Ninguna recuperación automática aprobada | `RECOVERY_REQUIRED`; no publicar `GLOBAL_ACTIVE` desde tail no checkpointed |
| Epoch advance | Sin política tail aprobada | No servir ni incrementar ignorando el tail; `RECOVERY_REQUIRED` hasta decisión explícita |
| Recovery event | Reutilizar solo un recovery event válido para una acción ADMIT/COMPLETE expresamente permitida por R3.1 | Acción desconocida, incompatible o conflictiva: `RECOVERY_REQUIRED`/`FAILED_CLOSED` según integridad; no derivar una acción nueva por analogía |

**Conclusión del catálogo:** no recomiendo ampliar la recuperación automática más allá de ADMIT/COMPLETE en esta fase. Es el límite explícito de R3.1 y evita inferir seguridad de handler o resultado desconocido a partir de bytes de journal no checkpointed.

## 6. Propuestas para D03-OPEN-01..08

Cada propuesta contiene una recomendación técnica para revisión. Estado de las ocho decisiones: `OPEN — HUMAN DECISION REQUIRED`.

### D03-OPEN-01 — Encoding, canonicalización, hashes/digests y checkpoint schema

**Problema:** no se puede congelar el codec sin una representación canónica inequívoca, fórmulas de hash no recursivas, distinción entre null y ausencia y un schema de checkpoint reproducible.

**Alternativas:**
- A. JSON canónico con reglas externas estrictas de canonicalización.
- B. Formato binario tipado propio, versionado, con campos identificados y longitudes explícitas.

**Recomendación no aprobada:** B, un encoding binario pequeño, versionado y length-prefixed, sin agregar dependencias. Motivo: el laboratorio necesita un contrato estable para hashing/replay y no necesita interoperabilidad JSON/wire.

Propuesta de reglas v1 para discutir:

1. Separar `recordSchemaVersion`, `checkpointSchemaVersion` y `canonicalizationVersion`.
2. Cada campo se codifica con `fieldTag` estable, estado de presencia (`ABSENT`, `NULL`, `VALUE` cuando el campo admita esas variantes), longitud explícita y bytes de valor. Campos serializados en orden ascendente de tag definido por schema; tags duplicados, campos desconocidos en schema v1, truncamiento y bytes sobrantes se rechazan.
3. Enteros y secuencias usan ancho fijo y byte order definido por contrato; strings usan UTF-8 estricto. No aplicar trim, case-folding ni normalización implícita. Los identificadores mantienen las restricciones léxicas que se decidan en D-ID-1; esta propuesta no cierra su entropía/gramática.
4. Listas mantienen orden semántico; conjuntos se canonicalizan ordenando por representación canónica del identificador antes de serializar. Nulos y campos ausentes nunca se colapsan.
5. `payloadDigest = SHA-256(domain-separator-payload || canonical(typeSpecificPayload))`.
6. `recordHash = SHA-256(domain-separator-record || canonical(recordEnvelopeWithoutRecordHash))`. El envelope incluye `previousHash`, `payloadDigest`, secuencia, eventId, epoch y metadatos del tipo. Se evita la autorreferencia excluyendo `recordHash` de la entrada.
7. `recoveryEventId` conserva la fórmula normativa R3.1 `SHA-256(canonical(recoveryKey))`; canonicalización debe ser la misma versión persistida en el evento de recovery.
8. Checkpoint contiene, al menos, checkpoint schema version, canonicalization version, última secuencia confirmada, `lastRecordHash`, snapshot schema version, snapshot determinista y `snapshotDigest`. Puede incorporar un `checkpointDigest` calculado sobre el checkpoint sin ese campo.
9. Hashes se almacenan como bytes de longitud fija en el formato binario; una representación hexadecimal es solo para logs/manifiestos, no para el input canónico.
10. Snapshot: mapas/sets ordenados por clave canónica; listas conservan el orden definido por el dominio. No incluir timestamps de generación que cambien el digest sin cambiar el estado.

**Decisiones específicas aún necesarias:** tamaño/ancho de los enteros, tags de cada campo, domain separators exactos, representación de `previousHash` genesis, campos nullable por tipo, límites máximos de longitud/registro, política de unknown fields y versionado compatible/incompatible.

**Aceptación propuesta:** vectores golden byte-for-byte para cada tipo; digest/hash esperado conocido; round-trip; rechazo de null/absent ambiguo, duplicate tag, unknown tag, UTF-8 inválido, length overflow, truncamiento, bytes sobrantes y payload conflictivo; mismo estado lógico produce checkpoint idéntico.

**Bloquea:** T03-02 Codec y la congelación de journal/checkpoint.

### D03-OPEN-02 — Política tail de eventos no cubiertos por R3.1

**Problema:** un evento íntegro posterior al checkpoint puede reflejar una operación que pudo tener efecto, pero no está confirmado por el protocolo. Extender la regla ADMIT/COMPLETE por analogía puede autorizar handler, liberar bloqueo o publicar GLOBAL_ACTIVE incorrectamente.

**Recomendación no aprobada:** mantener exactamente el límite actual: recuperación automática exclusiva para ADMIT y COMPLETE, conforme a sus precondiciones R3.1. Para FENCE, START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL y epoch, dejar `RECOVERY_REQUIRED`, servicio cerrado y sin ACK, sin emitir eventos compensatorios y sin cambiar el estado derivado confirmado.

Para cada tipo pendiente, una eventual política manual debe especificar: evidencia requerida, autoridad de decisión, evento de resolución durable, idempotency/recovery key, respuesta STATUS, condición de no duplicación y criterio de salida del estado bloqueado. Esta entrega no redacta una política de recuperación automática nueva.

**Aceptación propuesta:** crash matrix que demuestre ADMIT/COMPLETE recuperados una vez y cada otro tipo de tail detenido sin publicar estado/permiso ni invocar handler.

**Bloquea:** T03-08 Tail recovery y parte de T03-07 Replay. D03-OPEN-02 no se resuelve por aceptación de H-01.

### D03-OPEN-03 — Epoch: evento, clave, primer arranque y crash durante incremento

**Problema:** R3.1 requiere incrementar y persistir `coordinatorEpoch` en cada reinicio antes de habilitar operaciones, pero no fija el evento, startup key, valor inicial ni qué hacer si la caída ocurre entre append y checkpoint.

**Recomendación no aprobada:** modelar una transición central explícita (nombre provisional `COORDINATOR_EPOCH_ADVANCED`) que registre `previousEpoch`, `newEpoch`, `startupOperationId` y correlación del Coordinator. `newEpoch` debe ser estrictamente `previousEpoch + 1`, sin wrap ni reutilización. Para almacén realmente nuevo, proponer genesis controlado con epoch 1, solo si journal, checkpoint y temporales están todos ausentes; combinación parcial de archivos no equivale a almacén nuevo.

Secuencia propuesta:

1. Adquirir OS lock.
2. Validar/replay del estado y resolver únicamente tails con política ya aprobada.
3. Determinar el último epoch confirmado.
4. Generar una clave única de intento de arranque y persistir el evento de advance siguiendo el protocolo journal/checkpoint.
5. Habilitar operaciones solo después del checkpoint de ese nuevo epoch.

Caída antes del append confirmado: no se sirve; el siguiente arranque vuelve a inspeccionar el almacenamiento. Si existe un evento de epoch en tail, no se ignora ni se escribe un segundo incremento: queda `RECOVERY_REQUIRED` hasta una política humana explícita. No se considera el epoch del journal tail como confirmado.

**Decisiones específicas aún necesarias:** nombre/versión de evento, genesis 0→1 o alternativa, forma de `startupOperationId`, definición de “cada reinicio” para aborts muy tempranos, reglas para overflow, tail de epoch y proceso de recuperación tras corte en cada frontera.

**Aceptación propuesta:** escenarios de primer arranque, restart limpio, crash antes de append, durante append, después de journal force, durante checkpoint, después de checkpoint/antes de servicio; epoch confirmado nunca se reutiliza y Coordinator nunca opera sin epoch confirmado para ese arranque.

**Bloquea:** T03-01 cierre del catálogo de startup, T03-07 Replay y T03-11 crash matrix.

### D03-OPEN-04 — Directory force: unsupported frente a error real/ambiguo

**Problema:** después de `ATOMIC_MOVE`, el intento de `force(true)` sobre el directorio padre puede no estar soportado, o puede fallar por I/O/seguridad/estado ambiguo. No deben tratarse como equivalentes.

**Recomendación no aprobada:** usar tres resultados explícitos y no inferidos:

- `SUPPORTED_AND_SUCCEEDED`: el proveedor aceptó la operación y no devolvió error.
- `UNSUPPORTED`: solo cuando la API/proveedor informa explícitamente que esa capacidad no está disponible. Registrar excepción/clase y proveedor.
- `FAILED_OR_AMBIGUOUS`: errores I/O, acceso denegado u otro resultado que no pruebe ausencia de soporte. Mantener Coordinator cerrado y verificar journal/checkpoint mediante replay; no rebajar a `UNSUPPORTED` por heurística.

`UNSUPPORTED` no convierte en fallido el `ATOMIC_MOVE` ya verificado ni autoriza fallback; limita los claims al escenario de terminación de proceso y al filesystem realmente ensayado, sin claim de pérdida de energía/kernel/disco/hardware. Un fallo/ambigüedad real tras rename no se resuelve automáticamente por esta clasificación.

**Aceptación propuesta:** probar separadamente atomic move y directory force; registrar JDK, OS, proveedor, tipo de filesystem cuando sea identificable, clase de excepción y existencia/contenido del checkpoint antes/después. No ejecutar un ensayo de pérdida eléctrica.

**Bloquea:** clasificación de fallos T03-05/T03-11 y los claims de durabilidad.

### D03-OPEN-05 — Temporales residuales

**Problema:** después de un corte puede quedar un checkpoint temporal parcial o completo, pero no existe regla aprobada que le otorgue autoridad frente al checkpoint vigente.

**Recomendación no aprobada:** crear temporales mediante `CREATE_NEW` en el mismo directorio/filesystem del checkpoint, con un prefijo reservado y un identificador de operación único, por ejemplo `checkpoint.tmp.<startupOperationId>.<eventId>`. No incluir secretos en el nombre.

En startup, después de adquirir el OS lock, detectar cualquier temporal reservado. No promoverlo, no reemplazar el checkpoint vigente y no borrarlo automáticamente. Clasificarlo `RECOVERY_REQUIRED` y registrar nombre, tamaño y digest si puede leerse sin mutarlo. La limpieza manual, si se aprueba después, debe generar evidencia del archivo, decisión/autoridad, copia de resguardo o hash previo y resultado; no forma parte automática del replay.

**Aceptación propuesta:** temporales parciales/completos, uno o varios temporales, temporal más checkpoint anterior y temporal corrupto nunca alteran el checkpoint seleccionado ni abren el Coordinator por inferencia.

**Bloquea:** T03-05 Checkpoint writer y T03-07 Replay.

### D03-OPEN-06 — Transición RECOVERY_REQUIRED / FAILED_CLOSED

**Problema:** ambos estados mantienen cerrado el Coordinator, pero representan problemas diferentes y no deben ser aliases ni tener un reset automático.

**Recomendación no aprobada:** definir una distinción por clase de evidencia:

- `RECOVERING`: estado transitorio de startup; no sirve.
- `RECOVERY_REQUIRED`: datos internamente íntegros, pero existe una operación/tail/temporal/decisión cuya semántica o resultado no se puede resolver automáticamente con las reglas aprobadas; no sirve y exige decisión/evidencia adicional.
- `FAILED_CLOSED`: violación de integridad o contradicción (registro parcial/corrupto, hash inválido, secuencia rota, checkpoint incompatible, referencia imposible, snapshot discordante), o propiedad del lock/almacén no confiable; no sirve y requiere intervención controlada.

Transiciones propuestas:

- `RECOVERING → OPERATIONAL`: solo si lock, replay, snapshot, tails permitidos, temporales y epoch están resueltos.
- `RECOVERING → RECOVERY_REQUIRED`: ambigüedad íntegra y no resuelta.
- `RECOVERING → FAILED_CLOSED`: corrupción/contradicción o exclusión de escritor no confiable.
- `RECOVERY_REQUIRED → RECOVERING`: solo por acción humana explícita, documentada, con evidencia y procedimiento aprobado; nunca por timeout, nuevo proceso o retry automático.
- `FAILED_CLOSED → RECOVERING`: solo tras reparación/restauración controlada autorizada y un baseline/hash verificable; no se hace repair in-place por heurística.

**Caso a decidir:** lock ocupado por otro Coordinator legítimo puede tratarse como `RECOVERY_REQUIRED` temporal; lock con propiedad/estado indeterminado puede tratarse como `FAILED_CLOSED`. TASK-03-DRAFT-4 agrupa algunas condiciones de lock como cierre estricto; requiere confirmar clasificación.

**Aceptación propuesta:** tabla exhaustiva condición→estado→acción permitida→evidencia→autoridad; ningún camino automático lleva de estado cerrado a operativo.

**Bloquea:** startup state machine, replay y operación de diagnóstico.

### D03-OPEN-07 — Evidencia Windows para FileLock, ATOMIC_MOVE y directory force

**Problema:** el diseño no puede declarar las capacidades del filesystem Windows por API/JDK teórica o por resultado en otro entorno. La inspección T03-00 fue read-only y no ensayó estas operaciones.

**Recomendación no aprobada:** mantener claims `UNVERIFIED` hasta ejecutar un experimento separado, con aprobación explícita, en un directorio temporal aislado y fixtures sintéticos. Los tres mecanismos deben probarse individualmente:

1. **FileLock:** proceso A adquiere; proceso B intenta adquirir y no puede operar; liberar por cierre ordenado y por terminación anormal de A; documentar proveedor/volumen.
2. **ATOMIC_MOVE:** archivo temporal y destino en el mismo directorio; probar creación/reemplazo según el caso de uso; capturar excepción y comprobar nombres/bytes visibles; sin fallback.
3. **Directory force:** intentar abrir/fuerza del directorio padre; registrar éxito explícito, unsupported explícito o error real/ambiguo.

Guardar comando/código exacto, JDK, versión Windows, proveedor, filesystem/volumen, excepciones y hashes antes/después. No usar datos reales, no escribir en el workspace de código, no simular pérdida eléctrica ni reclamar durabilidad de hardware.

**Estado:** requiere autorización humana específica del experimento; no incluido en T03-01 ni autorizado por la aceptación de T03-00.

### D03-OPEN-08 — Contrato/evidencia para REBIND_UNSTARTED_ADMISSION

**Problema:** REBIND depende de demostrar NOT_STARTED para el boot anterior. El Coordinator por sí solo no puede probarlo; el Member ledger y evidencia de terminación pertenecen a otra tarea.

**Base normativa R3.1:** `recoveryKey=(originalEventId,recoveryAction)` y `recoveryEventId=SHA-256(canonical(recoveryKey))`; la acción puede ser `REBIND_UNSTARTED_ADMISSION(targetProcessBootId)`. El boot destino forma parte de la acción canonicalizada. Un rebind conserva el `requestId/admissionId` original y no crea un ADMIT nuevo.

**Recomendación no aprobada:** TASK-03 define únicamente la referencia/contrato del recovery event y sus invariantes. TASK-04 debe aportar y validar el ledger local Member. No ejecutar rebind dentro de TASK-03.

Precondiciones acumulativas a exigir antes de un futuro REBIND:

1. ADMIT original existe y está checkpointed, con Member/boot/request/admission correlacionados.
2. Ledger local del Member íntegro, completo y vinculado al `admissionId`, `grantDigest` y boot anterior.
3. `PERMIT_RECEIVED` está durablemente registrado en el boot anterior.
4. No existe `HANDLER_START_INTENT` ni en la historia confirmada del Coordinator ni en un tail/estado ambiguo; el ledger local tampoco contiene START_INTENT ni HANDLER_ENTERED.
5. Evidencia suficiente y correlacionada demuestra que el proceso/boot anterior terminó y no puede seguir ejecutando esa admisión. PID o ausencia de heartbeat por sí solos no bastan.
6. Journal/checkpoint y Member ledger no tienen parcialidad, corrupción, tail no resuelto ni contradicción.
7. No existe otro boot/proceso ejecutable que reclame la admisión.
8. `targetProcessBootId` nuevo está identificado; la acción y recoveryKey incluyen ese boot destino.
9. Recovery event se persiste/checkpointed idempotentemente antes de comunicar rebind; mismo target boot reutiliza mismo recoveryEventId.

Si falta cualquiera de estas condiciones, existe START_INTENT, o el estado es UNKNOWN/RECONCILED_UNKNOWN, no se permite REBIND. `RECONCILED_UNKNOWN` conserva outcome UNKNOWN y no es NOT_STARTED.

**Bloquea:** cierre del contrato de recovery en T03-01; implementación y evidencia real dependen de TASK-04. La política de evidencia para proceso remoto no se inventa aquí.

## 7. Decisiones heredadas fuera de D03-OPEN-01..08

Estas decisiones siguen pendientes y no deben desaparecer al cerrar el catálogo:

- **D-VD03-1:** duración/TTL y punto temporal de referencia del Permit.
- **D-VD03-2:** fuente de reloj, precisión y tolerancia de desfase.
- **D-VD03-3:** si hay renovación pre-consumo y cómo se representa/persiste.
- **D-VD03-4:** evidencia necesaria para replay/restart al determinar vigencia.
- **D-ID-1:** encoding/entropía de `processBootId`, `requestId` y `commandId`.
- **D-EVIDENCE-1:** alcance de la prueba de proceso muerto, en particular evidencia remota.
- **D-FS-1:** capacidad real del filesystem para operaciones de persistencia.
- **D-WIRE-1:** framing/encoding/autenticación final del protocolo.

D-01 ya fijó el punto de consumo: la vigencia se evalúa al commit durable de START_INTENT. Estos pendientes no alteran ese punto y no se deben resolver inventando TTL, reloj, skew o renovación.

## 8. Validación del catálogo y cobertura de escenarios

### 8.1 Casos positivos

| ID | Caso | Resultado esperado | Evidencia futura |
|---|---|---|---|
| CAT-P01 | ADMIT normal | ADMIT checkpointed antes del Permit; retry same-key/same-payload recupera mismo resultado | seq/hash, transitionKey, admissionId |
| CAT-P02 | ADMIT previo y FENCE posterior | admisión previa se mantiene in-flight; se permite START_INTENT posterior si Permit vigente al commit | orden durable ADMIT→FENCE→START_INTENT |
| CAT-P03 | Permit expirado antes del commit | `ADMISSION_REJECTED_EXPIRED` checkpointed antes de `REJECTED_NOT_STARTED`/liberación de bloqueo | seq/hash, STATUS misma clave, ausencia de handler |
| CAT-P04 | START_INTENT antes de FENCE | FENCE no cancela ni revalida el intent ya consumido | orden START_INTENT→FENCE |
| CAT-P05 | COMPLETE válido en tail | recuperación única `RECOVERED_COMPLETE` si las referencias y resultado cumplen R3.1 | recoveryKey/eventId, checkpoint |
| CAT-P06 | Recovery de ADMIT repetido | mismo recoveryEventId tras restart, sin duplicado | journal antes/después, key e ID |
| CAT-P07 | RECONCILE_UNKNOWN con evidencia | outcome sigue UNKNOWN; solo sale del conjunto bloqueante con todas las condiciones D-02 | RecoveryRecord y evidencia asociada |
| CAT-P08 | Retry tras ACK perdido | STATUS por misma clave; no se crea transición paralela | transitionKey, eventId y resultado único |

### 8.2 Casos negativos y bordes

| ID | Caso | Resultado esperado |
|---|---|---|
| CAT-N01 | mismo transitionKey con payload distinto | rechazo auditable; estado previo intacto |
| CAT-N02 | ADMIT nuevo después de FENCE | rechazado; no admissionId/Permit |
| CAT-N03 | tail FENCE | `RECOVERY_REQUIRED`; no asumir FENCE confirmado ni reabrir admisiones |
| CAT-N04 | tail START_INTENT | `RECOVERY_REQUIRED`; no handler/retry/ACK inferido |
| CAT-N05 | tail ADMISSION_REJECTED_EXPIRED | no liberar bloqueo ni derivar REJECTED_NOT_STARTED |
| CAT-N06 | tail RECONCILE_UNKNOWN | conserva UNKNOWN; no liberar bloqueo por inferencia |
| CAT-N07 | tail COMMIT_GLOBAL | no publicar GLOBAL_ACTIVE |
| CAT-N08 | tail epoch advance | no servir ni crear un segundo incremento por ignorar el tail |
| CAT-N09 | registro parcial, hash/schema inválido o secuencia rota | `FAILED_CLOSED`; no truncate/repair automático |
| CAT-N10 | checkpoint y snapshot discordantes | `FAILED_CLOSED` o estado estricto conforme matriz aprobada; nunca operar |
| CAT-N11 | temporal residual | no promover ni borrar automáticamente; `RECOVERY_REQUIRED` |
| CAT-N12 | directory force unsupported explícito | registrar capacidad ausente; claim limitado al entorno ensayado |
| CAT-N13 | IOException/resultado ambiguo tras rename | no clasificar como unsupported; cerrar y verificar por replay |
| CAT-N14 | rebind sin ledger íntegro/evidencia de proceso muerto | no rebind; sigue recovery required |
| CAT-N15 | rebind cuando existe START_INTENT/UNKNOWN | no rebind ni reejecución automática |
| CAT-N16 | lock ocupado/propiedad incierta | ningún servicio ni escritura; estado final según D03-OPEN-06 |

### 8.3 Riesgos principales

| Riesgo | Severidad | Mitigación propuesta |
|---|---|---|
| Canonicalización ambigua | Crítica | schema versionado, length-prefix, golden vectors, rechazo de bytes extra/duplicados |
| Tail clasificado por analogía | Crítica | tabla allowlist; solo ADMIT/COMPLETE automáticos hasta decisión distinta |
| Epoch repetido o no confirmado | Crítica | evento explícito y checkpoint antes de servicio; tail de epoch no se ignora |
| Promoción/borrado de temporal no autorizado | Alta | detectar y detener; intervención manual documentada |
| Confundir UNKNOWN con NOT_STARTED/COMPLETE | Crítica | estados separados, no retry, evidencia Member íntegra |
| Claims excesivos de filesystem | Alta | prueba Windows aprobada y claim limitado a los resultados observados |
| REBIND sin evidencia local Member | Crítica | responsabilidad TASK-04; ninguna inferencia del Coordinator |
| Cierre de estado ambiguo por timeout | Crítica | timeout solo limita espera; no es transición de dominio |

## 9. Criterios de aceptación de T03-01

T03-01 podrá considerarse documentalmente completa cuando se cumplan todos estos puntos:

1. Catálogo enumera tipos centrales, recovery events, evidencias locales Member y operaciones observacionales sin duplicar responsabilidades.
2. Para cada evento central hay propósito, dueño, correlación/idempotency key, precondiciones, efecto durable y resultado observable.
3. Tabla de tail distingue explícitamente recovery permitido por R3.1 de tipos que permanecen `RECOVERY_REQUIRED`.
4. D-01, D-02, D-03, D-VD03-5 y H-01 se preservan textualmente en semántica; ningún escenario los contradice.
5. Cada D03-OPEN-01..08 tiene alternativas, recomendación, criterios verificables, dependencias y estado humano explícito.
6. Pendientes temporales D-VD03 e identificadores/protocolo fuera de alcance continúan visibles y no se rellenan por inferencia.
7. Requisitos funcionales/no funcionales y escenarios positivos, negativos y borde trazan hacia tareas de T03-02..T03-11.
8. No se afirma validación de Windows, durabilidad de energía/hardware, ejecución de tests ni compatibilidad productiva.
9. D03-OPEN-01..08 solo pasan a `CLOSED` mediante decisión humana explícita con alcance y texto aceptado; la recomendación del documento no es aprobación.
10. Hasta cerrar las decisiones bloqueantes o acordar un alcance bloqueado, no se congela el diseño de implementación.

## 10. Veredicto y estado

**Dictamen:** `CATÁLOGO DOCUMENTAL PREPARADO — D03-OPEN-01..08 PENDIENTES DE DECISIÓN HUMANA`.

La matriz actual permite preservar con seguridad el catálogo de dominio y las reglas ya aprobadas. No existe base para resolver automáticamente las ocho decisiones. D03-OPEN-02 es especialmente crítica: H-01 cubre la serialización viva, pero no define recovery de tails START_INTENT o ADMISSION_REJECTED_EXPIRED.

**Siguiente gate:** revisión humana de las propuestas D03-OPEN-01..08. Las respuestas deben registrarse por ID y con el alcance exacto aceptado/rechazado. Después se prepara la actualización coherente de SPEC, PLAN, TASKS y DESIGN-VALIDATION. La prueba de filesystem Windows requiere autorización separada y no queda habilitada por este documento.

No se modificó código, POM ni configuración. No se ejecutaron Maven, Java, tests, builds ni experimentos. No se publicó este borrador en GitHub. La implementación TASK-03 sigue `NOT AUTHORIZED`; TASK-04 y tareas posteriores no autorizadas; gate CRL de producción `CLOSED`.