# TASK-03 — Especificación de persistencia durable del Coordinator

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-SPEC-DRAFT-4`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Autorización vigente:** preparación y validación del diseño. No autoriza implementación, modificaciones funcionales, Maven, builds, tests ni experimentos.

## 1. Propósito

Definir el diseño de persistencia durable local del Coordinator del laboratorio CRL-FEASIBILITY-009A.1: journal append-only, checkpoint, replay, reconciliación tipada, exclusión de escritores y serialización lógica de ADMIT/FENCE. El diseño debe reconstruir de manera determinista el último estado confirmado, preservar idempotencia y permanecer fail-closed ante corrupción, contradicción o resultado ambiguo.

Este documento especifica el alcance de TASK-03. No convierte el laboratorio en un servicio productivo ni demuestra durabilidad frente a pérdida de energía, fallas de kernel, disco o hardware.

## 2. Fuentes normativas y precedencia

1. `CRL-FEASIBILITY-009A.1-DESIGN(9).md`, revisión R3.1 del 2026-11-14, especialmente §§3.2, 3.11, 3.15, 3.17, 3.18 y 3.20.
2. Paquete TASK-02-DRAFT-2 cerrado y aprobado por el usuario: D-01, D-02, D-03 y D-VD03-5.
3. Informe de inspección TASK-01 compartido anteriormente, dictamen `VIABLE WITH CONDITIONS`; inspeccionó 009A (`%TEMP%\crl-feasibility-009a\project`), no el workspace aislado 009A.1. Es evidencia estática anterior y no acredita la provisión de 009A.1.
4. Evidencia histórica 009A, únicamente como antecedente; no acredita identidad del workspace actual 009A.1.

Ante conflicto prevalecen las decisiones humanas aprobadas en TASK-02 y la revisión fuente R3.1. Los estados históricos contradictorios de documentos TASK-02 quedan subordinados al addendum de cierre aprobado.

## 3. Baseline y limitaciones conocidas

### 3.1 Evidencia histórica TASK-01

El informe TASK-01 identifica como laboratorio inspeccionado:

`C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a\project`

Ese informe describe fuentes candidatas bajo `src/test/java/lab/crl`, un POM y evidencia histórica de 009A. No informa que haya inspeccionado ni provisionado `crl-feasibility-009a1\project`. Sus observaciones no deben atribuirse al workspace 009A.1.

### 3.2 Workspace aislado propuesto por R3.1

El diseño fuente R3.1 propone un subdirectorio nuevo `%TEMP%\crl-feasibility-009a1` y exige conservar 009A como evidencia inmutable. La ruta hija `%TEMP%\crl-feasibility-009a1\project` fue adoptada como ubicación esperada del arnés aislado, pero no existe evidencia TASK-01 que certifique que esa copia se haya creado o inspeccionado.

### 3.3 Estado observado el 2026-11-15

La captura de Explorador de archivos proporcionada por el usuario muestra la raíz:

`C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a1`

El usuario indicó que ese directorio contiene únicamente los ocho documentos Markdown visibles. La captura no muestra un subdirectorio `project`, un `pom.xml` ni un source set Java. Esta evidencia visual no equivale a un inventario filesystem completo ni permite afirmar el estado Git, hashes, atributos ocultos o contenido de archivos no visibles.

En consecuencia, con la evidencia disponible:
- la raíz documental 009A.1 existe;
- el subdirectorio normativo `project` no está provisionado;
- no se ha verificado un POM ni fuentes Java de 009A.1;
- T03-00 permanece `BLOCKED — WORKSPACE NOT PROVISIONED`.

### 3.4 Límite de baseline y gate de provisión

El ZIP `project.zip` y el directorio histórico 009A son antecedentes de 009A. No sustituyen ni acreditan identidad del workspace 009A.1, aun cuando contengan fuentes, POM o evidencia reutilizable.

La provisión de `009a1\project` es una operación separada de T03-00, que es exclusivamente read-only. Debe tener un plan de copia, manifiesto de inclusión/exclusión, tratamiento de `.git`, artefactos generados y material criptográfico local, verificación SHA-256 y autorización humana independiente. No se copiará, moverá, reconstruirá ni modificará el proyecto 009A como parte de T03-00.

Una vez autorizada y completada la provisión, T03-00 podrá inspeccionar el nuevo workspace en modo read-only y producir su baseline real.

### 4.1 Incluido en TASK-03

- Modelo de eventos del Coordinator y codec versionado.
- Journal append-only con secuencia monotónica, identidad de evento, referencias, digest y hash-chain.
- Checkpoint versionado con secuencia/hash confirmados y snapshot determinista.
- Escritura completa, `force(true)`, temporal, `force(true)`, cierre y sustitución exclusivamente mediante `ATOMIC_MOVE`.
- Lock exclusivo de sistema operativo para un Coordinator local escritor.
- Replay del estado checkpointed y validación cruzada entre checkpoint, journal y snapshot.
- Reconciliación idempotente de los dos tipos de tail explícitamente cubiertos por R3.1: ADMIT y COMPLETE.
- Indexación de claves de transición y recovery para evitar duplicaciones.
- Serialización integral de ADMIT/FENCE.
- Protocolo de arranque cerrado hasta completar lock, replay, recovery permitido y avance durable del epoch.
- Pruebas focalizadas del almacén del Coordinator después de la autorización correspondiente.

### 4.2 Excluido de TASK-03

- Ledger durable local del Member y receipt `PERMIT_RECEIVED`; D-03 permanece vigente y corresponde a TASK-04.
- Ejecución efectiva de `REBIND_UNSTARTED_ADMISSION`, porque requiere evidencia íntegra del ledger local y de terminación/no continuidad del proceso Member. TASK-03 podrá definir el contrato/referencias del evento Coordinator, pero no producir por sí sola la evidencia Member ni autorizar el rebind.
- Transporte/wire protocol final, framing, autenticación nueva y cambios del servidor TCP (TASK-05).
- Wrapper/gate HTTP y ledger de `HANDLER_ENTERED` del Member (TASK-06).
- Integración completa TLS, dos Members, conexiones persistentes y rotación (TASK-07).
- Matriz global de fault injection y paquete reproducible (TASK-08/09), sin perjuicio de pruebas focalizadas de TASK-03.
- Consenso, quorum, HA, multi-host, multirregión, replicación, base de datos externa, Kafka, Redis o servicios nuevos.
- Ancla anti-rollback externa. La sustitución conjunta por una copia antigua íntegra de journal y checkpoint no es detectable usando solo esos archivos locales.
- Cualquier afirmación PCI, FIPS, HSM certificado, compatibilidad Link/Prisma o uso productivo.

## 5. Decisiones heredadas y reglas de negocio

- **D-01:** para el GET sintético, el Permit se consume al confirmar durablemente `HANDLER_START_INTENT`. La validación de vigencia y la persistencia forman una operación lógica serializada con ADMIT/FENCE. Expiración anterior al commit rechaza; expiración posterior no cancela ni provoca una segunda validación en `HANDLER_ENTERED` o en el body.
- **D-02:** `PENDING` y `UNKNOWN` no reconciliado bloquean `COMMIT_GLOBAL`. `RECONCILED_UNKNOWN` conserva el outcome desconocido; puede dejar de bloquear solo con evidencia durable suficiente y nunca equivale a `COMPLETE`.
- **D-03:** el Member fuerza `PERMIT_RECEIVED` en su ledger local antes de enviar `PERMIT_RECEIVED_ACK` al Coordinator. No se añade una escritura central redundante.
- **D-VD03-5:** si el Permit expiró antes del commit durable de `HANDLER_START_INTENT`, Coordinator persiste `ADMISSION_REJECTED_EXPIRED`; confirma el rechazo solo después de persistencia durable. Entonces deriva a `REJECTED_NOT_STARTED` y deja de bloquear. Si la persistencia es fallida o ambigua, continúa bloqueando. ACK perdido se consulta por STATUS con la misma clave. No hay `HANDLER_ENTERED`, GET ni COMPLETE después de rechazo confirmado.
- TTL, instante de referencia, fuente de reloj, precisión/tolerancia y renovación pre-consumo siguen pendientes. TASK-03 no puede inventarlos ni derivarlos de `createdAt`.
- Timeout o ACK perdido no equivalen a éxito ni a fallo terminal.

## 6. Modelo de persistencia

### 6.1 JournalRecord

Cada registro deberá transportar, según aplique al tipo:

- `schemaVersion`;
- secuencia monotónica;
- `eventId`;
- `rolloutId`;
- `coordinatorEpoch`;
- `eventType`;
- campos del dominio y estado necesarios para reconstruir la transición (`memberId`, `requestId`, `admissionId`, `processBootId`, generation/hash y referencias);
- `transitionKey`;
- `previousEventId` o referencia al evento de origen cuando corresponda;
- `payloadDigest`;
- `previousHash` y `recordHash`.

La canonicalización de claves/hash debe ser versionada, length-prefixed y no ambigua. Esta revisión no inventa encoding JSON/binario, algoritmo de `recordHash`, algoritmo de `payloadDigest`, canonicalización de todos los campos ni representación de null/ausencia. Esos puntos permanecen como decisiones de diseño bloqueantes antes de congelar el codec. La regla fuente para `recoveryEventId` sí es fija: `SHA-256(canonical(recoveryKey))`.

### 6.2 Checkpoint

El checkpoint deberá contener como mínimo:

- `schemaVersion`;
- última secuencia confirmada;
- hash exacto del último registro confirmado;
- snapshot determinista del estado derivado del Coordinator;
- metadata suficiente para validar correspondencia entre snapshot, journal y schema.

El journal no se compacta ni trunca en TASK-03. El formato canónico del checkpoint y la política de temporales residuales deben cerrarse antes de implementación.

### 6.3 Lock y ownership

Se requiere un lock exclusivo del sistema operativo sobre el almacén local. Se adquiere durante startup, antes de inspeccionar/servir el almacén, y se mantiene mientras el proceso Coordinator sea propietario operativo del recurso; se libera al cerrar ordenadamente y el sistema operativo lo libera al terminar el proceso. Si no se obtiene, el estado del lock es incierto o la propiedad del recurso no puede verificarse, Coordinator no opera. Este lock evita un segundo Coordinator escritor local en el filesystem realmente ensayado; no representa consenso ni fencing entre hosts.

Este **OS lock de vida del proceso** no sustituye la **frontera de serialización lógica de operaciones** de T03-09. El primero excluye otro proceso Coordinator sobre el mismo almacén. La segunda ordena dentro del Coordinator las operaciones concurrentes que compiten por cambiar el estado durable (ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED). Puede implementarse con lock/actor/cola u otro mecanismo, pero debe preservar el orden lógico descrito en §11. No se presupone que ambos mecanismos sean el mismo lock ni que uno resuelva el cometido del otro.

## 7. Catálogo de eventos y política de tail

Esta matriz distingue el tratamiento normal de un evento checkpointed de un evento íntegro posterior al checkpoint. La presencia de bytes completos en el journal no autoriza por sí sola la transición.

| Tipo lógico | Tratamiento normal cuando está checkpointed | Tail íntegro posterior al checkpoint en TASK-03-DRAFT-2 |
|---|---|---|
| `ADMIT` | Crea una admisión durable antes de emitir Permit | Aplicar únicamente la regla R3.1: `ADMIT_ABORTED_UNCOMMITTED`, con recoveryKey estable, checkpoint y sin permiso/handler. Misma transitionKey devuelve `NOT_ADMITTED/ABORTED_UNCOMMITTED`. |
| `COMPLETE` | Cierra una admisión después del write HTTP completado en servidor | Aplicar únicamente la regla R3.1 si referencia un ADMIT pendiente del mismo Member/boot y el resultado es permitido: `RECOVERED_COMPLETE` una sola vez. No prueba recepción del cliente. |
| `FENCE` | Cierra nuevas admisiones del epoch anterior | Sin política tail explícita aprobada: `RECOVERY_REQUIRED`; no se infiere FENCE confirmado ni se reabre servicio. |
| `HANDLER_START_INTENT` | Consume Permit en el punto D-01; ACK tras confirmación durable | Sin política tail de Coordinator aprobada: `RECOVERY_REQUIRED`; no generar ACK, permiso ni ejecución a partir del tail. |
| `ADMISSION_REJECTED_EXPIRED` | Rechazo pre-consumo conforme D-VD03-5, solo tras confirmación durable | Política de tail pendiente. No declarar `REJECTED_NOT_STARTED` ni liberar bloqueo por el solo hecho de hallar el registro en tail. Mantener `RECOVERY_REQUIRED` hasta decisión explícita. |
| `RECONCILE_UNKNOWN` | Conserva outcome UNKNOWN; su efecto bloqueante depende de evidencia y decisión explícita durable | `RECOVERY_REQUIRED`; no liberar admisión ni convertir outcome en COMPLETE por tail no checkpointed. |
| `COMMIT_GLOBAL` | Publica commit solo tras comprobar cero admissions bloqueantes y miembros exactos verificados | `RECOVERY_REQUIRED`; no publicar estado global activo por un commit no checkpointed. |
| Avance de `coordinatorEpoch` | Debe persistirse antes de habilitar operaciones tras reinicio, conforme R3.1 §3.2 | La identidad del evento, la clave de startup y su política de tail requieren decisión explícita. No habilitar operaciones hasta resolver. |
| Recovery event | Acción explícita, correlacionada y deduplicada mediante recoveryKey/recoveryEventId | Reutilizar un recovery event válido para las acciones ADMIT/COMPLETE definidas por R3.1. Para otras acciones, validar el contrato aprobado; sin regla, detener. |

**Límite:** R3.1 especifica recuperación automática de tail para ADMIT y COMPLETE. No se extenderá esa política por analogía a FENCE, START_INTENT, rechazo por expiración, RECONCILE_UNKNOWN, COMMIT_GLOBAL o avance de epoch.

### 7.1 Recovery de ADMIT

Si un ADMIT íntegro aparece en tail sin checkpoint, el protocolo R3.1 asegura que no se envió ACK/permiso. Se escribe `ADMIT_ABORTED_UNCOMMITTED` y se checkpointa como acción de recovery. No se crea otro admissionId para el mismo request/transitionKey. La misma clave consulta el resultado abortado.

### 7.2 Recovery de COMPLETE

Si COMPLETE está íntegro en tail, se valida su referencia a un ADMIT pendiente del mismo Member/boot y el formato del resultado. Se registra `RECOVERED_COMPLETE` una sola vez y se checkpointa antes de responder al retry/STATUS. Evento parcial, referencia inválida, estado incompatible o payload conflictivo dejan el sistema cerrado.

### 7.3 Rebind

`REBIND_UNSTARTED_ADMISSION(targetProcessBootId)` existe en R3.1 §3.17, con el boot destino incluido en la acción canonicalizada. Su ejecución requiere evidencia del Member: ledger íntegro, receipt previo, ausencia de START_INTENT, proceso/boot anterior terminado y ausencia de tail/contradicción. Por exclusión de TASK-04, TASK-03 no puede fabricar ni verificar esa evidencia. Sin ella, permanece `RECOVERY_REQUIRED`; no se rebind ni se emite un permiso al nuevo boot.

## 8. Protocolo de confirmación durable

La transición del Coordinator es una unidad lógica serializada. Conforme R3.1 §3.15, la secuencia es:

1. Tomar la frontera lógica/lock y validar precondiciones contra el estado confirmado.
2. Canonicalizar el evento y asignar sus IDs/secuencia sin publicar todavía el nuevo estado.
3. Append de todos los bytes del registro.
4. `force(true)` del journal.
5. Escribir completamente el checkpoint temporal en el mismo filesystem/directorio que el destino.
6. `force(true)` del temporal.
7. Cerrar el temporal.
8. Reemplazar el checkpoint vigente exclusivamente con `ATOMIC_MOVE`.
9. Intentar `force(true)` del directorio padre cuando el filesystem/JDK lo soporte y registrar la capacidad/resultado.
10. Publicar el estado nuevo en memoria y emitir ACK solo después de completar el punto de confirmación y la política de error acordada.

No hay fallback silencioso a movimiento no atómico. Si `ATOMIC_MOVE` no está soportado o falla, no se confirma ni se envía ACK; Coordinator permanece cerrado. Si el directory force no está soportado, se documenta esa capacidad ausente y los claims quedan limitados a terminación de proceso bajo el filesystem ensayado; no se reclama durabilidad ante pérdida de energía/kernel/disco/hardware.

Un error real de I/O o resultado ambiguo después de que el movimiento pudo ocurrir no se clasificará automáticamente como “unsupported”. Se mantiene el servicio cerrado hasta verificar journal/checkpoint mediante replay. La clasificación exacta de excepciones y proveedores Windows es decisión pendiente y requiere evidencia autorizada.

El checkpoint (secuencia/hash) es el punto de transición confirmada por el protocolo. Si el checkpoint nuevo es verificable después de una caída anterior al ACK, el retry consulta la misma transitionKey y devuelve el resultado existente, sin duplicarlo. La forma de gestionar un fallo real de directory force después de rename debe respetar este punto de confirmación y quedar explícita.

## 9. Startup, epoch y disponibilidad

### 9.1 Secuencia de arranque requerida

1. Iniciar en `RECOVERING`; no aceptar comandos de operación ni solicitudes que dependan del Coordinator.
2. Adquirir lock exclusivo de sistema operativo.
3. Validar checkpoint, schema, secuencia/hash y snapshot.
4. Recorrer journal y validar secuencia contigua, IDs, referencias, payload digest y hash-chain.
5. Reconstruir el estado checkpointed y compararlo con el snapshot.
6. Clasificar todo tail por tipo. Ejecutar únicamente las políticas explícitas de ADMIT/COMPLETE; cualquier otro tipo sin política deja `RECOVERY_REQUIRED`.
7. Resolver los temporales residuales conforme a una política de diseño aprobada; nunca promoverlos ni eliminarlos por inferencia.
8. Persistir el avance de `coordinatorEpoch` requerido para este arranque, antes de habilitar operaciones.
9. Solo después de completar todos los pasos y las decisiones pendientes aplicables puede publicarse el estado operativo.

### 9.2 Avance durable de epoch

R3.1 §3.2 exige que cada reinicio incremente y persista `coordinatorEpoch` antes de habilitar operaciones. La SPEC incorpora ese requisito; no fija todavía el nombre final del evento, startup key, tratamiento del primer arranque, interacción con tail de epoch, ni recuperación de caída durante append/force/checkpoint del incremento. Esas decisiones son bloqueantes. No se reutilizará un epoch anterior por conveniencia ni se abrirá el Coordinator mientras el avance no esté confirmado.

El epoch es local al Coordinator único de laboratorio. No previene rollback de la copia completa de journal+checkpoint y no constituye consenso entre hosts.

## 10. Replay, integridad y estados de arranque

- `RECOVERING`: fase transitoria de startup; no hay servicio disponible.
- `RECOVERY_REQUIRED`: el estado no puede continuar automáticamente con las reglas aprobadas; requiere decisión/evidencia/reconciliación, y permanece cerrado.
- `FAILED_CLOSED`: se detectó corrupción, contradicción, lock no confiable o condición incompatible con recuperación segura; permanece cerrado y requiere intervención definida antes de reintentar.

La transición entre los dos estados cerrados y la condición exacta de retorno a `RECOVERING`/operativo se documentarán en la máquina de estado del Coordinator. No se equiparan silenciosamente.

Condiciones de cierre obligatorio: journal más corto que checkpoint; registro parcial; schema/encoding/hash inválido; secuencia repetida/saltada; identidad duplicada/conflictiva; referencia ausente; snapshot discordante; lock ocupado/incierto; tail sin política; temporal residual no clasificable; epoch no confirmado.

No se trunca, repara, compacta ni selecciona “el archivo más nuevo” automáticamente.

## 11. Serialización lógica de ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED

No basta con `synchronized(record())` ni con serializar únicamente ADMIT/FENCE. Una sola frontera lógica de operación, dentro del Coordinator que posee el OS lock, debe abarcar de extremo a extremo:

- lectura del estado durable confirmado y de las admisiones pendientes;
- validación de Member, epoch, estado, FENCE y correspondencia del Permit;
- determinación de si el intento de consumo produce `HANDLER_START_INTENT` o `ADMISSION_REJECTED_EXPIRED` conforme D-01/D-VD03-5;
- asignación de orden y clave idempotente para ADMIT, FENCE y el resultado del intento de START_INTENT;
- append completo, journal `force(true)`, checkpoint temporal, force, close y `ATOMIC_MOVE`;
- publicación del estado derivado y decisión de ACK/permiso/rechazo, siempre después del punto de confirmación durable.

La frontera debe imponer un **orden lógico total** entre las cuatro transiciones. No puede existir una carrera en la que la validación de vigencia observe un estado, otra operación confirme FENCE/ADMIT y la persistencia de START_INTENT o del rechazo se ordene como si la validación hubiese ocurrido sobre el estado anterior. La validación de D-01 y la persistencia del resultado no son dos operaciones independientes.

Reglas de orden que debe preservar el serializador:

1. **ADMIT confirmado antes de FENCE:** la admisión queda como trabajo previo/in-flight. FENCE cierra nuevas admisiones, pero no revoca el derecho de esa admisión ya confirmada a intentar completar su ciclo.
2. **START_INTENT posterior a FENCE para una admisión previa:** se permite confirmar `HANDLER_START_INTENT` después de FENCE si el ADMIT de referencia fue confirmado antes de FENCE y el Permit sigue vigente en el punto de commit durable de START_INTENT. Esta transición no crea una admisión nueva ni reabre ADMIT.
3. **Expiración antes del commit de START_INTENT:** en lugar de START_INTENT, Coordinator persiste `ADMISSION_REJECTED_EXPIRED` para la admisión original. Solo después de confirmar durablemente ese rechazo puede derivar `REJECTED_NOT_STARTED`, liberar el bloqueo asociado y confirmar el rechazo. Si la escritura/checkpoint falla o su resultado es ambiguo, la admisión continúa bloqueando; no se permite HANDLER_ENTERED, GET ni COMPLETE.
4. **FENCE antes de un ADMIT nuevo:** la nueva admisión se rechaza; no se emite Permit ni se crea una admisión autorizada después de FENCE.
5. **START_INTENT confirmado antes de FENCE:** permanece como trabajo ya iniciado/intención consumida; FENCE no lo cancela ni provoca una segunda validación de vigencia. La semántica de UNKNOWN y no-retry permanece intacta.

El orden observable es el orden lógico/durable de las operaciones, no el orden de llegada de threads ni un timestamp de log. Si dos o más operaciones compiten, el mecanismo debe producir una única secuencia serializable que respete las reglas anteriores. Un timeout, ACK perdido o resultado de I/O ambiguo no habilita una transición alternativa ni libera una admisión.

El punto de commit durable sigue siendo el checkpoint confirmado conforme §8. La evaluación de vigencia debe estar ligada a ese commit como una sola operación lógica; no se permite validar anticipadamente y asumir que el Permit seguirá vigente al confirmar. Esta corrección no fija TTL, fuente de reloj, precisión, skew ni renovación pre-consumo: esos parámetros permanecen pendientes y no deben inferirse.

El tail de `HANDLER_START_INTENT` y `ADMISSION_REJECTED_EXPIRED` continúa sin política automática aprobada. Esta sección define la serialización de operaciones vivas; no cierra D03-OPEN-02 ni autoriza recovery automático de esos tipos. Si cualquiera aparece en tail sin checkpoint, se mantiene el tratamiento fail-closed ya especificado: `RECOVERY_REQUIRED` hasta decisión explícita.

El mecanismo interno (lock, actor, cola u otro) queda abierto. Debe demostrar, con barreras/latches deterministas y sin `sleep`, el orden total y todos los resultados de los casos positivos, negativos y de carrera de la validación de diseño.

## 12. Requisitos funcionales

| ID | Requisito | Criterio verificable |
|---|---|---|
| T03-FR-001 | Registro versionado | Cada record incluye schema, seq, eventId, tipo y campos/ref obligatorios por tipo |
| T03-FR-002 | Canonicalización e integridad | Codec rechaza encoding ambiguo/inválido; payloadDigest y recordHash se validan según contrato aprobado |
| T03-FR-003 | Journal append-only | Write completo y `force(true)` antes de checkpoint; no truncamiento automático |
| T03-FR-004 | Checkpoint | Temporal completo, force, close y `ATOMIC_MOVE`; sin fallback no atómico |
| T03-FR-005 | Lock exclusivo | Un solo Coordinator local obtiene lock; contender no opera |
| T03-FR-006 | Replay | Estado reconstruido coincide con snapshot y secuencia/hash del checkpoint |
| T03-FR-007 | Fail-closed de integridad | Corrupción, truncamiento, referencia o snapshot contradictorio no habilitan servicio |
| T03-FR-008 | Tail ADMIT/COMPLETE | Aplicación idempotente exclusivamente conforme reglas R3.1 |
| T03-FR-009 | Tail de otros tipos | Tipos sin política explícita mantienen `RECOVERY_REQUIRED`; no se infiere estado |
| T03-FR-010 | Idempotencia | Misma clave/mismo payload reutiliza resultado; misma clave/payload distinto se rechaza |
| T03-FR-011 | Serialización Coordinator | Una frontera lógica serializa ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED; conserva el derecho de completar admisiones pre-FENCE y el rechazo por expiry durable |
| T03-FR-012 | Startup y epoch | No habilitar Coordinator antes de lock, replay, recovery permitido y epoch durable |
| T03-FR-013 | Decisiones TASK-02 | D-01/D-02/D-03/D-VD03-5 se preservan sin reinterpretación |
| T03-FR-014 | Límite Member | No se ejecuta rebind sin evidencia Member íntegra; implementación de ledger queda fuera de TASK-03 |

## 13. Requisitos no funcionales

- NFR-001: Coordinator único local; sin HA, consenso ni multi-host.
- NFR-002: no agregar dependencias externas ni actualizar versiones preventivamente.
- NFR-003: journal append-only sin compactación en esta fase.
- NFR-004: serialización determinista, versionada y no ambigua.
- NFR-005: fail-closed ante error, corrupción o ambigüedad; sin soft-fail/fallback no atómico.
- NFR-006: pruebas deterministas con hooks/barreras/latches; `sleep` no establece sincronización.
- NFR-007: logs sin tokens, claves, PSK/tickets, secretos, certificados privados ni material sensible.
- NFR-008: medir p50/p95/p99 del camino de persistencia solo cuando la implementación y medición estén autorizadas; no fijar SLO sin evidencia.
- NFR-009: separar terminación de proceso de pérdida de energía/kernel/disco/hardware.
- NFR-010: documentar límite anti-rollback de copia íntegra antigua.
- NFR-011: no modificar artefactos principales del ISO8583 Gateway ni componentes Member/HTTP/wire fuera del alcance.

## 14. Criterios de aceptación

1. Codec y checkpoint tienen round-trip determinista y validación de schema/canonicalización.
2. Replay desde almacenamiento íntegro reproduce exactamente el estado confirmado.
3. Journal/checkpoint truncado, parcial, corrupto o contradictorio no habilita servicio.
4. El segundo proceso no obtiene lock ni puede operar/escribir.
5. `ATOMIC_MOVE` unsupported/fallido no provoca fallback ni ACK.
6. Directory force unsupported se informa por separado de un error real; los claims se limitan a evidencia del filesystem ensayado.
7. Tail ADMIT produce abort explícito sin Permit; tail COMPLETE válido se recupera una sola vez.
8. Tail de FENCE, START_INTENT, rechazo expiry, RECONCILE_UNKNOWN, COMMIT_GLOBAL o epoch no recibe política automática no aprobada.
9. Recovery repetido conserva el mismo recoveryKey/recoveryEventId y no duplica transiciones.
10. Retry de misma transitionKey recupera resultado existente; payload conflictivo se rechaza.
11. ADMIT, FENCE, HANDLER_START_INTENT y ADMISSION_REJECTED_EXPIRED comparten un orden lógico total verificable: una admisión confirmada antes de FENCE puede confirmar START_INTENT después de FENCE si el Permit sigue vigente en el commit durable; expiry anterior produce rechazo durable; un nuevo ADMIT después de FENCE se rechaza; una persistencia fallida/ambigua del rechazo mantiene el bloqueo.
12. El Coordinator no sirve durante RECOVERING, RECOVERY_REQUIRED o FAILED_CLOSED.
13. Coordinator no habilita operaciones hasta registrar durably el epoch del arranque según política aprobada.
14. D-01/D-02/D-03/D-VD03-5 permanecen intactas; TTL/reloj/skew/renewal no se inventan.
15. REBIND no se ejecuta sin evidencia del Member; no se implementa Member ledger dentro de TASK-03.
16. Las afirmaciones sobre Windows filesystem se limitan a resultados observados y autorizados.
17. No se modifica código/configuración fuera de la autorización posterior; gate CRL productivo permanece cerrado.

## 15. Decisiones de diseño que bloquean el freeze

| ID | Decisión pendiente | Por qué bloquea | Tratamiento provisional seguro |
|---|---|---|---|
| D03-OPEN-01 | Encoding exacto, canonicalización, `recordHash`, `payloadDigest`, null/ausencia y checkpoint schema | No se puede congelar el codec ni validar interoperabilidad | No implementar codec final |
| D03-OPEN-02 | Política tail para FENCE, HANDLER_START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN y COMMIT_GLOBAL | R3.1 no ofrece regla automática explícita para estos tipos | `RECOVERY_REQUIRED`; no inferir ni ACKear |
| D03-OPEN-03 | Evento/clave/semántica de avance de `coordinatorEpoch`, primer arranque y crash durante incremento | Requerido por R3.1 §3.2 antes de servicio | Coordinator permanece cerrado hasta decisión y commit durable |
| D03-OPEN-04 | Clasificación de directory force unsupported frente a error real/ambiguo después del rename | Puede afectar claim y resultado del checkpoint | Unsupported documentado; error real/ambiguo => replay cerrado |
| D03-OPEN-05 | Formato, ubicación y manejo de temporales residuales | No promover ni borrar archivos cuya autoridad no está definida | Detectar y cerrar; no limpiar/promover automáticamente |
| D03-OPEN-06 | Semántica de transición entre RECOVERY_REQUIRED y FAILED_CLOSED | Afecta operador/recuperación y reintento | Permanecer cerrado; no auto-reset |
| D03-OPEN-07 | Evidencia Windows real para FileLock, ATOMIC_MOVE y directory force | TASK-01 no probó el filesystem destino | No afirmar compatibilidad antes de experimento autorizado |
| D03-OPEN-08 | Contrato y evidencia para REBIND desde Member | Member ledger está fuera de TASK-03 | No ejecutar rebind; depender de TASK-04 |

TTL, reloj, skew y renovación pre-consumo son decisiones heredadas pendientes, no se incorporan ni congelan por esta tabla.

## 16. Estado de esta revisión

`TASK-03-SPEC-DRAFT-4` conserva la corrección H-01 aceptada y rectifica la trazabilidad del baseline: preserva la serialización lógica de T03-09 y la separación entre el OS lock de vida del Coordinator y la frontera intra-proceso de operaciones; corrige la atribución del informe TASK-01 a 009A y registra que el workspace aislado 009A.1 aún no está provisionado. No modifica el catálogo de eventos ni decide la política tail de `HANDLER_START_INTENT` o `ADMISSION_REJECTED_EXPIRED`; D03-OPEN-02 permanece abierta. Las decisiones D03-OPEN-01..08 siguen sin cerrarse. Este documento continúa `DRAFT — PENDING HUMAN REVIEW` y no autoriza implementación, experimentos, Maven, builds o tests. La aprobación del diseño y la autorización humana independiente para implementar siguen siendo gates distintos.
