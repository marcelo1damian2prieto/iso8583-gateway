# DOC-008 — Modelo MongoDB, consistencia, índices y recuperación

**Versión:** 2.0.1  
**Base:** MongoDB única base de datos del Gateway, replica set.  
**Estado:** modelo aprobado documentalmente; no contiene comandos de despliegue ejecutados.

## 1. Topología

- MongoDB replica set `rs0`.
- Compose inicial: un solo nodo con replica set habilitado para transacciones/Change Streams que el driver requiera; no representa HA ni tolerancia a pérdida de nodo.
- Base de datos del Gateway: `iso8583_gateway`.
- HSM Simulator conserva key store separado (volumen/archivo protegido); no es base de datos transaccional del Gateway.
- Backend Simulator mantiene estado en memoria conforme DOC-004.

## 2. Consistencia por defecto

- Lecturas críticas de operaciones/idempotencia/correlación: `readPreference=primary`, `readConcern=majority`.
- Escrituras críticas: `writeConcern={w:"majority", j:true, wtimeout:5000}`.
- Escrituras no críticas de telemetría/audit secundario pueden usar política menos estricta solo si se declara y no se usa como evidencia de negocio.
- Transacciones multi-documento: `readConcern=snapshot`, `writeConcern={w:"majority", j:true}`; transacciones breves con `maxCommitTimeMS=5000` como default de laboratorio.
- Retryable writes habilitadas en driver para operaciones compatibles. Una excepción de timeout no se interpreta como rollback seguro: se consulta por ID/versión antes de reintentar.
- Sin transacción Mongo abierta durante llamadas HTTP, HSM, Crypto o TCP.

## 3. Colecciones y agregados

### `operations`

Fuente de verdad del ciclo de vida de autorización/financiero/reverso. Campos: `_id=operationId`, `profileId`, `originId`, `authorization`, `financial`, `reversal`, `businessStatus`, `processingCheckpoint`, `version`, `createdAt`, `updatedAt`, `expiresAt` solo cuando terminal y elegible.

### `idempotency_records`

`_id`/unique key hash, `operationId`, `messageId`, `requestFingerprint`, `operationType`, `status`, `resultRef`, `createdAt`, `terminalAt`, `expiresAt`. No almacenar PAN ni token DE48 en claro.

### `correlations`

`tokenHash`, `operationId`, `originIdHash`, `rrn`, `status`, `createdAt`, `expiresAt`, `version`. Token DE48 no se indexa ni registra en claro.

### `messages`

Un documento por mensaje: `messageId`, `operationId`, `direction`, `mti`, `profileId`, `receivedAt`, `archiveStatus`, `sanitizedParsed`, `digestAlgorithm`, `digestScope`, `digest`, `ciphertext`, `nonce`, `tag`, `batchId`, `wrappedDek` o referencia de batch, `keyId`, `keyVersion`, `algorithm`, `aadVersion`, `schemaVersion`, `attemptId`. No raw frame con DE52; no PAN claro.

### `crypto_batches`

`batchId`, `status`, `openedAt`, `deadlineAt`, `reservedCount`, `version`, `keyId`, `keyVersion`, `wrappedDek`, `lease/recovery metadata`, `closedAt`, `failureCode`. Nunca DEK/KEK en claro.

### `crypto_reservations`

`reservationId`, `batchId`, `messageId`, `attemptId`, `status`, `leaseOwner`, `leaseUntil`, `heartbeatAt`, `nonce`, `createdAt`, `updatedAt`. Reserva durable antes de cifrado. Si se demuestra que esta colección puede integrarse atómicamente en `messages` sin perder recuperación, se puede simplificar en implementación, pero el criterio de unicidad debe conservarse.

### `audit_events`

Eventos append-only lógicos: `eventId`, `aggregateId`, `eventType`, `actorService`, `occurredAt`, `correlationId`, `safeAttributes`, `schemaVersion`. Sin payload, secretos, PAN completo ni DE52.

## 4. Índices únicos/compuestos

1. `idempotency_records`: unique `{keyHash:1}`.
2. `operations`: unique `{_id:1}` implícito; unique sparse para combinación de referencia de negocio solo si la cardinalidad del perfil lo justifica.
3. `correlations`: unique `{tokenHash:1}` y búsqueda `{operationId:1,status:1}`.
4. `messages`: unique `{messageId:1}`; unique `{batchId:1,nonce:1}` cuando nonce se asigna; índice `{operationId:1,receivedAt:1}`.
5. `crypto_batches`: unique `{batchId:1}`; índice `{status:1,deadlineAt:1}`.
6. `crypto_reservations`: unique `{messageId:1}`; índice `{batchId:1,status:1,leaseUntil:1}`; unique `{batchId:1,nonce:1}` para nonce no nulo.
7. `audit_events`: unique `{eventId:1}`; índice `{aggregateId:1,occurredAt:1}`.
8. TTL solo en `idempotency_records.expiresAt`, `messages.expiresAt` y colecciones elegibles después de terminalidad; no usar TTL para `operations` pendientes ni para `crypto_batches` que soportan decrypt.

Los índices parciales/sparse y las restricciones exactas deben verificarse con el modelo de documento final y Testcontainers; no confiar en TTL para sincronización inmediata.

## 5. Operaciones atómicas

- Reservar idempotencia mediante insert con unique key; duplicate-key significa leer y comparar fingerprint.
- Transición de estado con filtro `{_id, version, expectedStatus}` y `$inc version`.
- Reserva de lote con `findOneAndUpdate` condicionado a `status=OPEN`, `reservedCount < 1000`, `deadlineAt > now`; incrementar contador y crear reservation en transacción breve si no puede resolverse con un documento agregado.
- Cierre de lote por CAS cuando `reservedCount=1000` o `deadlineAt<=now`; después no se admiten reservas.
- Reclamar lease con filtro `leaseUntil < now-5s` y update atómico del owner/lease/version.
- Nunca decremento de contador por timeout ambiguo sin comprobar estado de reservation.

## 6. Política de nonce

- Nonce 12 bytes generado con CSPRNG por cada intento de cifrado.
- Restricción unique `(batchId, nonce)` como detección de colisión, no como generador.
- Si el insert de mensaje queda incierto, primero leer por `messageId`; si no existe, reintentar con nonce nuevo.
- Si colisiona índice, generar nonce nuevo y registrar el intento sin guardar secretos.
- No reutilizar nonce bajo la misma DEK, aunque el primer insert haya fallado aparentemente.

## 7. Recuperación

- Worker obtiene claim por actualización condicional y lease.
- Si reserva `RESERVED` sin documento, consulta `messageId` y `attemptId`; retoma o marca `ABANDONED` conforme checkpoint.
- Si `crypto_batch` `OPEN` supera deadline, pasa a `CLOSING`; resuelve reservations; luego `CLOSED` o `FAILED`.
- Operaciones Gateway se recuperan por `processingCheckpoint` y consulta Backend status según DOC-004/DOC-006.
- Una operación indeterminada no se elimina por TTL.
- Si HSM key version/wrappedDek no puede recuperarse, bloquear descifrado y generar incidente; nunca fallback.

## 8. Transacciones multi-documento

Uso permitido solo para consistencia local corta, por ejemplo: reservar capacidad + crear reservation, o actualizar operation + correlation si ambas deben quedar atómicas. Si el modelo permite un único documento con CAS, preferirlo. No envolver llamadas externas ni operaciones largas. Configurar retry de transacción según driver, releyendo estado antes de repetir efectos externos.

## 9. Criterios de aceptación

- Testcontainers con replica set y concerns documentados.
- Dos instancias compiten por última capacidad: solo una reserva gana.
- Índices únicos evitan idempotency/message/nonce duplicates.
- Tests de write timeout validan lectura de reconciliación antes de retry.
- Tests de lease expiry/heartbeat evitan doble claim.
- Tests TTL verifican que no se eliminan estados no terminales.
- Failover/restart de nodo único prueba recuperación del proceso, no HA.
