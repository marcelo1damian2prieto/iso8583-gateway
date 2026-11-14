# DOC-004 — Contrato REST del Backend/Core Simulator

**Versión:** `backend-api-v1`  
**Estado:** contrato de laboratorio aprobado documentalmente por el usuario  
**Transporte:** HTTP sobre TLS; mTLS desde Gateway. JSON UTF-8.

## 1. Responsabilidad y límites

Backend Simulator representa el core que decide autorización, aplica el efecto financiero simulado, ejecuta reversos y permite consultar estado. No implementa clearing/settlement ni se presenta como core bancario real.

Para mantener el laboratorio acotado, el simulador conserva estado/idempotencia en memoria durante la vida del proceso. Reiniciar Backend Simulator reinicia su estado. Las pruebas de reinicio de Gateway/Crypto asumen que Backend permanece activo; persistencia durable del simulador requeriría decisión y tarea separadas.

## 2. Convenciones comunes

- Base path: `/api/v1`.
- `Content-Type: application/json`; `Accept: application/json`.
- `X-Correlation-Id`: UUID de traza no sensible.
- `Idempotency-Key`: obligatorio para POST con efecto; valor opaco hash de 64 hex minúsculas.
- No incluir PAN completo, PIN, PIN block ni payload ISO en requests/responses.
- Dinero: `amountMinor` entero positivo (unidad menor); `currency` 3 dígitos.
- Timestamps: RFC 3339 UTC.
- Todos los IDs internos son UUID.

## 3. Endpoints

### 3.1 Autorizar

`POST /api/v1/authorizations`

Request:

```json
{
  "operationId": "uuid",
  "originId": {"acquirerId": "12345678901", "forwarderId": "12345678901", "terminalId": "ATM00001", "acceptorId": "ATM-LAB-0000001"},
  "stan": "123456",
  "transmissionDateTime": "MMDDhhmmss",
  "processingCode": "010000",
  "amountMinor": 250000,
  "currency": "032",
  "rrn": null,
  "profileId": "LAB-ATM-1.0"
}
```

Response `200`:

```json
{
  "operationId": "uuid",
  "backendOperationId": "uuid",
  "status": "APPROVED",
  "responseCode": "00",
  "rrn": "123456789012",
  "decidedAt": "2026-01-01T12:00:00Z",
  "resultVersion": 1
}
```

Estados posibles: `APPROVED`, `DECLINED`. Rechazos de negocio usan `200` con `status=DECLINED` y código configurado; errores de contrato usan 4xx.

### 3.2 Procesamiento financiero

`POST /api/v1/financial-transactions`

Request:

```json
{
  "operationId": "uuid",
  "authorizationOperationId": "uuid",
  "originId": {"acquirerId": "12345678901", "forwarderId": "12345678901", "terminalId": "ATM00001", "acceptorId": "ATM-LAB-0000001"},
  "stan": "123457",
  "transmissionDateTime": "MMDDhhmmss",
  "processingCode": "010000",
  "amountMinor": 250000,
  "currency": "032",
  "rrn": "123456789012",
  "profileId": "LAB-ATM-1.0"
}
```

Response `200`:

```json
{
  "operationId": "uuid",
  "backendOperationId": "uuid",
  "status": "COMPLETED",
  "responseCode": "00",
  "rrn": "123456789012",
  "committedAt": "2026-01-01T12:00:01Z",
  "resultVersion": 1
}
```

Estados posibles: `COMPLETED`, `DECLINED`, `INDETERMINATE`. El simulador no devuelve `INDETERMINATE` como resultado normal salvo fault injection; un timeout de transporte puede dejar el Gateway sin conocer el resultado.

### 3.3 Reversar

`POST /api/v1/reversals`

Request:

```json
{
  "reversalOperationId": "uuid",
  "originalOperationId": "uuid",
  "originalMti": "0200",
  "originalStan": "123457",
  "originalTransmissionDateTime": "MMDDhhmmss",
  "originalRrn": "123456789012",
  "amountMinor": 250000,
  "currency": "032",
  "originId": {"acquirerId": "12345678901", "forwarderId": "12345678901", "terminalId": "ATM00001", "acceptorId": "ATM-LAB-0000001"},
  "profileId": "LAB-ATM-1.0"
}
```

Response `200`:

```json
{
  "reversalOperationId": "uuid",
  "originalOperationId": "uuid",
  "status": "REVERSED",
  "responseCode": "00",
  "reversedAt": "2026-01-01T12:00:02Z",
  "resultVersion": 1
}
```

Estados: `REVERSED`, `REVERSAL_DECLINED`, `INDETERMINATE`, `NOT_APPLICABLE`. Reverso solo aplica a original `COMPLETED`.

### 3.4 Consultar estado

`GET /api/v1/operations/{operationId}`

Response `200` incluye `operationId`, `backendOperationId`, `operationType`, `status`, `responseCode` si existe, `rrn` si existe, `resultVersion`, `createdAt`, `updatedAt`. `404` si no existe. No devuelve payloads sensibles.

### 3.5 Consultar por idempotencia

`GET /api/v1/idempotency/{idempotencyKey}`

Devuelve el resultado previamente registrado o `404`. Requiere identidad de servicio autorizada. La clave se trata como dato sensible operacional y no se loguea completa.

## 4. Semántica idempotente

- Cada POST de efecto exige `Idempotency-Key`.
- Primera solicitud reserva la clave y aplica una única transición lógica.
- Misma clave + mismo fingerprint retorna el resultado original.
- Misma clave + payload semántico distinto retorna `409 IDEMPOTENCY_CONFLICT`.
- Dos requests concurrentes con misma clave convergen a un solo resultado.
- Una respuesta perdida después de commit puede recuperarse mediante GET status/idempotency.
- El simulador no crea un segundo efecto al repetir la misma clave.

## 5. Errores HTTP

| HTTP | `code` | Significado | Reintento |
|---:|---|---|---|
| 400 | `VALIDATION_ERROR` | JSON/valor inválido | No sin corregir |
| 401 | `UNAUTHENTICATED` | identidad ausente/inválida | No; revisar mTLS |
| 403 | `FORBIDDEN` | identidad sin permiso | No |
| 404 | `OPERATION_NOT_FOUND` | operación inexistente | Consulta posterior solo si procedente |
| 409 | `IDEMPOTENCY_CONFLICT` | clave repetida con contenido distinto o estado incompatible | No |
| 422 | `BUSINESS_RULE_VIOLATION` | regla de negocio no satisfecha | No sin cambio de estado |
| 429 | `RATE_LIMITED` | límite de simulador | Reintento solo con misma clave y backoff |
| 500 | `INTERNAL_ERROR` | fallo interno | Consultar status antes de repetir |
| 503 | `DEPENDENCY_UNAVAILABLE` | simulador no disponible | Consultar status; misma clave |

Formato de error:

```json
{"code":"VALIDATION_ERROR","message":"field validation failed","correlationId":"uuid","retryable":false}
```

No incluir stack traces, PAN, payload ISO ni secretos.

## 6. Fault injection

Configurable solo en perfil de laboratorio:

- `delayBeforeCommitMs`.
- `delayAfterCommitMs`.
- `dropResponseAfterCommit`.
- `forceDecline`.
- `forceUnavailable`.

La fault injection no cambia idempotency semantics. Debe poder distinguir timeout pre-commit (sin efecto) y post-commit (efecto durable en proceso, respuesta perdida).

## 7. Timeouts cliente propuestos

Ver DOC-007. Backend connect timeout 1 s, response timeout 3 s. Timeout después de enviar request se trata como resultado desconocido; Gateway consulta por la misma idempotencyKey/status antes de decidir cualquier acción.

## 8. Criterios de contrato

- OpenAPI 3.1 versionado como artefacto de implementación posterior.
- Contract tests para éxito, rechazo, errores, idempotencia, concurrencia y respuesta perdida.
- Campos de dinero, ID, estados y errores coinciden exactamente con este documento.
- La semántica de memoria volátil del simulador queda reflejada en README y pruebas.
