# DOC-005 — Contratos Cryptographic Service y HSM Simulator

**Versión:** `crypto-api-v1` / `hsm-api-v1`  
**Estado:** contratos documentales aprobados por el usuario  
**Transporte:** HTTP JSON UTF-8 sobre TLS mutuo (mTLS).

## 1. Límites de responsabilidad

- **Cryptographic Service (CS):** coordina lotes, obtiene DEK temporal, cifra/descifra documentos, persiste documentos cifrados en MongoDB Gateway, administra reservas/checkpoints y recuperación.
- **HSM Simulator:** custodia KEK simulada, genera DEK, envuelve/desenvuelve DEK y ejecuta operaciones PIN exclusivamente con fixtures sintéticos. No almacena transacciones, PAN, mensajes ni accede a MongoDB.
- **Gateway:** solicita archivado de mensajes y no recibe material de clave.

## 2. Seguridad común

- mTLS obligatorio; identidades de servicio: `gateway`, `crypto-service`, `hsm-simulator`, `ops-readonly` (solo lectura operacional si se habilita).
- Autorización por identidad y operación: Gateway puede solicitar archive/status; CS puede pedir generate/unwrap; solo rol de recuperación autorizado puede decrypt; HSM no expone KEK.
- No se permite `GET` de secretos ni endpoint de exportación de KEK.
- Requests/responses nunca contienen PIN claro, PAN completo ni DE52.
- `X-Correlation-Id` UUID y `X-Request-Id` UUID. No loguear body.

## 3. HSM Simulator API

Base path `/api/v1/keys`.

### 3.1 Crear material de lote

`POST /api/v1/keys/data-keys`

Request: `{ "batchId":"uuid", "algorithm":"AES-256-GCM", "keyPurpose":"DOCUMENT_ENCRYPTION" }`

Response: `{ "batchId":"uuid", "dek":"base64-32-bytes", "wrappedDek":"base64", "keyId":"lab-kek-2026-01", "keyVersion":1, "algorithm":"AES-256-GCM" }`.

`batchId` es clave idempotente de esta operación: si se repite con el mismo algoritmo/purpose, HSM devuelve la misma DEK lógica y el mismo `wrappedDek` (reconstruyendo la DEK desde su copia envuelta); si el mismo batchId llega con parámetros distintos, responde `409 BATCH_KEY_CONFLICT`. `dek` se entrega al CS por canal mTLS y vive transitoriamente en memoria; nunca se persiste ni registra. `wrappedDek` es material opaco recuperable solo por HSM.

### 3.2 Desenvolver DEK

`POST /api/v1/keys/data-keys:unwrap`

Request: `{ "batchId":"uuid", "wrappedDek":"base64", "keyId":"lab-kek-2026-01", "keyVersion":1 }`.

Response: `{ "batchId":"uuid", "dek":"base64-32-bytes", "algorithm":"AES-256-GCM" }`.

Solo CS autenticado y autorizado. No exponer este endpoint a Gateway. El unwrap es idempotente para el par `(batchId, keyId, keyVersion)` mientras la versión esté habilitada; una versión revocada responde `KEY_REVOKED`.

### 3.3 Metadata

`GET /api/v1/keys/{keyId}/versions/{version}` devuelve algoritmo, estado (`ACTIVE`, `DECRYPT_ONLY`, `REVOKED`) y timestamps; nunca material de clave.

Rotación: nueva versión para nuevos lotes; versiones anteriores permanecen `DECRYPT_ONLY` mientras existan documentos retenidos. Revocación bloquea unwrap y requiere procedimiento explícito de recuperación/retención.

### 3.4 Operaciones PIN sintéticas

`POST /api/v1/pin/validate-fixture` acepta `fixtureId` de catálogo de pruebas, formato de bloque sintético y `operation` enumerada. Devuelve `valid`, `resultCode` y metadata no sensible. No acepta PIN claro, no devuelve PIN claro, no persiste PIN block y no implementa traducción a claves reales.

## 4. Cryptographic Service API

Base path `/api/v1/documents` y `/api/v1/batches`.

### 4.1 Archivar mensaje

`POST /api/v1/documents`

Request conceptual:

```json
{
  "messageId":"uuid",
  "operationId":"uuid",
  "direction":"INBOUND",
  "mti":"0100",
  "profileId":"LAB-ATM-1.0",
  "receivedAt":"2026-01-01T12:00:00Z",
  "sanitizedIsoBytes":"base64",
  "parsedSanitized": {"processingCode":"010000","amountMinor":250000,"currency":"032"},
  "digestAlgorithm":"SHA-256"
}
```

Response `201`:

```json
{
  "messageId":"uuid",
  "documentId":"uuid",
  "batchId":"uuid",
  "archiveStatus":"DURABLE",
  "digestAlgorithm":"SHA-256",
  "digest":"hex-sha256",
  "schemaVersion":1
}
```

CS determina/valida `batchId`; el caller no puede seleccionar un lote arbitrario. La operación es idempotente por `messageId`: mismo ID + mismo digest retorna documento existente; mismo ID + digest distinto responde `409 MESSAGE_ID_CONFLICT`.

### 4.2 Recuperar documento

`GET /api/v1/documents/{documentId}` retorna metadata saneada y ciphertext/nonce/tag solo a caller autorizado. No devuelve plaintext en endpoint general.

### 4.3 Descifrar documento

`POST /api/v1/documents/{documentId}:decrypt`

Solo identidad de servicio autorizada. Respuesta con bytes/parseo descifrado para consumo transitorio; nunca registrar body. El Gateway no tiene permiso de descifrado general en baseline; el flujo operativo normal no requiere volver a obtener PAN/DE52. Operación administrativa excepcional queda deshabilitada por defecto.

### 4.4 Estado de lote y recuperación

- `GET /api/v1/batches/{batchId}`: estado, contador reservado, openedAt, deadlineAt, cierre y checkpoint; nunca DEK.
- `POST /api/v1/batches/{batchId}:recover`: acción interna idempotente que resuelve reservas expiradas según lease/checkpoint; no reabre lote cerrado.

## 5. Cifrado documental

- AES-256-GCM, nonce de 96 bits, tag de 128 bits.
- AAD canónico incluye `profileId`, `messageId`, `batchId`, `direction`, `mti`, `schemaVersion`.
- Una DEK por lote; nonce único por DEK. Nonce generado con CSPRNG; índice único `(batchId, nonce)` en Mongo como defensa adicional.
- Si persistencia tiene resultado incierto, el reintento usa nonce nuevo y `attemptId` nuevo; nunca reutilizar nonce del intento anterior.
- SHA-256 se calcula sobre los bytes saneados exactos antes de cifrar. Campo `digestScope=SANITIZED_ISO_BYTES`; no calcular hash de la trama original que contenía DE52 excluido.
- Base64 representa `byte[]` en JSON; hexadecimal solo para digest/inspección. No transformar bytes con charset al calcular hash.
- `wrappedDek`, `keyId`, `keyVersion`, `algorithm`, `nonce`, `tag`, `ciphertext`, `aadVersion`, `schemaVersion` se guardan en documento `messages`.

## 6. Lotes

- Lote se abre al reservar la primera capacidad.
- Máximo 1.000 reservas admitidas o 15 minutos desde `openedAt`, lo que ocurra primero.
- Una reserva atómica incrementa `reservedCount` antes de cifrado.
- Estados `OPEN`, `CLOSING`, `CLOSED`, `FAILED`; no se reabre.
- Reserva con lease de 30 s; heartbeat cada 10 s mientras proceso activo. Recovery puede reclamar cuando `leaseUntil < now - 5s` (margen de reloj de laboratorio); toda transición usa CAS/versionado.
- Lote `CLOSING` no admite nuevas reservas; reservas ya admitidas deben completar o quedar `ABANDONED` para recovery.

## 7. Errores

| HTTP | Código | Semántica |
|---:|---|---|
| 400 | `INVALID_REQUEST` | Contrato inválido |
| 401/403 | `MTLS_IDENTITY_INVALID` / `NOT_AUTHORIZED` | Identidad o permiso incorrecto |
| 404 | `DOCUMENT_NOT_FOUND` / `BATCH_NOT_FOUND` | Recurso inexistente |
| 409 | `MESSAGE_ID_CONFLICT`, `BATCH_CLOSED`, `NONCE_CONFLICT` | Conflicto idempotente/estado |
| 422 | `UNSUPPORTED_PROFILE`, `SENSITIVE_FIELD_PRESENT` | Datos no permitidos |
| 429 | `BATCH_CAPACITY_EXCEEDED` | Capacidad agotada; obtener lote nuevo |
| 500 | `CRYPTO_FAILURE` | Error interno; no persistir como exitoso |
| 503 | `HSM_UNAVAILABLE`, `MONGO_UNAVAILABLE` | Fallo temporal; aplicar DOC-006, sin fallback |

## 8. Recuperación y no idempotencia

- `messageId` identifica un intento lógico de archivo; `attemptId` identifica cada cifrado físico.
- Si resultado del insert Mongo es incierto, consultar por `messageId` antes de decidir. Si no existe, reintentar con nuevo nonce/attemptId; si existe, comparar digest y devolver registro.
- Si lote queda sin DEK recuperable, no crear clave local alternativa. Marcar lote/documentos afectados y detener procesamiento; recuperación exige KEK/backup correcto.
- Nunca ejecutar dos veces el efecto de negocio por reintentar el archivado documental.

## 9. Criterios de aceptación

- Contract tests para todas las operaciones y roles mTLS.
- DEK solo transitoria y KEK no exportable.
- AES-GCM tampering en ciphertext/tag/AAD falla.
- Reintento de archivo no duplica documento y nunca reutiliza nonce.
- Dos instancias coordinan lote sin exceder 1.000/15 min.
- HSM/CS no reciben ni emiten PAN/PIN claro ni payload ISO original con DE52.
