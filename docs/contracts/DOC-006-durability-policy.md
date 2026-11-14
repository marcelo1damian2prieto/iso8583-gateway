# DOC-006 — Política de durabilidad documental y orden transaccional

**Versión:** 2.0.1  
**Decisión de laboratorio:** `DURABLE-BEFORE-EFFECT-AND-RESPONSE`.

## 1. Objetivo

Definir qué debe estar durable antes de invocar un efecto financiero y antes de emitir una respuesta ISO. La persistencia documental no debe producir doble efecto ni falsear un resultado conocido.

## 2. Invariantes

1. Ninguna llamada de autorización/financiera/reverso al Backend se inicia hasta que el request validado tenga operación/idempotencia durable en Mongo y mensaje inbound archivado por Crypto Service.
2. Mongo write acknowledgement debe cumplir DOC-008 (`w: majority`, journaling habilitado, read concern majority para lectura crítica).
3. El archivado exige cifrado AES-GCM y persistencia confirmada. No existe fallback a plaintext, clave local ni cola solo en memoria.
4. No se mantiene una transacción Mongo abierta durante llamada a Backend, Crypto o HSM.
5. El estado `INDETERMINATE` nunca se traduce automáticamente en rechazo o éxito.
6. La respuesta ISO solo se envía después de que su resultado y mensaje outbound estén persistidos durablemente.

## 3. Secuencia obligatoria por request

### 3.1 Ingreso y validación

1. Recibir frame TLS y asignar `messageId`.
2. Parsear y validar framing, MTI, bitmap y campos del perfil.
3. Sanear campos prohibidos (DE52, CVV/CVC) y minimizar PAN.
4. Calcular fingerprint/idempotency key sobre representación canónica saneada.
5. Si request inválido, no se invoca Backend; el error de protocolo se responde solo si el canal/protocolo permite respuesta segura y su mensaje outbound se archiva según política. Si no puede archivarse, se cierra el canal sin respuesta de negocio.

### 3.2 Reserva durable

1. Crear/recuperar `idempotency_record` con índice único y estado `RESERVED`.
2. Crear/actualizar operación con versión condicional.
3. Solicitar a Crypto Service archivo inbound idempotente por `messageId`.
4. Confirmar `archiveStatus=DURABLE`.
5. Marcar checkpoint `REQUEST_DURABLE`.

Solo después del paso 5 puede comenzar una llamada Backend. Si el registro Mongo o archivo Crypto no se confirma, no hay efecto financiero.

### 3.3 Invocación Backend

1. Invocar Backend fuera de transacción Mongo con idempotencyKey estable.
2. Ante respuesta concluyente, guardar resultado con CAS/versionado.
3. Ante timeout/desconexión después de enviar request, persistir `INDETERMINATE` si Mongo está disponible; si Mongo también está caído, el estado previo `REQUEST_DURABLE` permite recuperación.
4. No generar nueva idempotencyKey ni reejecutar ciegamente.

### 3.4 Respuesta

1. Construir respuesta ISO desde resultado Backend conocido.
2. Archivar respuesta outbound en Crypto Service y esperar `DURABLE`.
3. Guardar checkpoint `RESPONSE_DURABLE` y estado terminal correspondiente.
4. Escribir bytes en canal TCP.
5. Registrar `RESPONSE_SENT` de forma best effort/condicional; la desconexión puede impedir saber si el peer recibió respuesta.

La entrega TCP no es exactamente-once. Si el Gateway cae entre persistencia y escritura, el cliente puede retransmitir; idempotencia devuelve el mismo resultado lógico. Si cae después de escribir pero antes del checkpoint, también se devuelve el mismo resultado.

## 4. Fallos por punto de control

| Punto del fallo | Estado durable esperado | Acción | ¿Backend puede haber aplicado efecto? |
|---|---|---|---|
| Antes de validación | Ninguno | Cerrar/rechazar framing | No |
| Mongo indisponible al reservar | Ninguno/previo | Fail closed; no invocar dependencias | No |
| Crypto/HSM indisponible antes de archivo inbound | Reserva provisional o ninguna | Cancelar/expirar reserva; no invocar Backend | No |
| Archivo inbound confirmado, antes de Backend | `REQUEST_DURABLE` | Recovery puede continuar de forma idempotente | No todavía |
| Timeout antes de enviar request Backend | `REQUEST_DURABLE` | Error conocido de conexión; no efecto si se demuestra no envío | No |
| Timeout después de enviar request | `REQUEST_DURABLE` o `INDETERMINATE` | Consultar status/idempotency; nunca nueva clave | Sí, desconocido |
| Backend respondió, Gateway cae antes de guardar | `REQUEST_DURABLE` | Consultar Backend por idempotencyKey | Sí, conocido por Backend |
| Mongo cae después de commit Backend | `REQUEST_DURABLE` | No responder; recuperar por Backend status | Sí |
| Resultado guardado, Crypto outbound falla | Resultado Backend durable + `RESPONSE_PENDING` | No enviar; recovery archiva respuesta | Sí, resultado conocido |
| Outbound durable, Gateway cae antes de write TCP | `RESPONSE_DURABLE` | Retransmisión devuelve mismo resultado | Sí |
| Cliente se desconecta durante write | `RESPONSE_DURABLE` | No repetir efecto; esperar retransmisión/consulta | Sí |

## 5. Estados/checkpoints adicionales

Se añaden a la máquina de proceso (no sustituyen los estados de negocio):

- `RECEIVED`
- `VALIDATED`
- `RESERVED`
- `REQUEST_DURABLE`
- `BACKEND_IN_FLIGHT`
- `RESULT_DURABLE`
- `RESPONSE_PENDING`
- `RESPONSE_DURABLE`
- `RESPONSE_SENT` (observación, no garantía de recepción)
- `INDETERMINATE`
- `ARCHIVE_FAILED`

Cada checkpoint tiene `updatedAt`, `version`, `attemptId` y referencias a `messageId`/`operationId`. Actualizaciones condicionales evitan regresiones. La máquina de estados de negocio sigue siendo independiente.

## 6. Reintentos permitidos

- Retry de lectura de status: permitido con backoff.
- Retry de archivado documental: solo con mismo `messageId`, mismo digest y nuevo nonce si requiere nuevo cifrado; consultar primero si insert previo quedó incierto.
- Retry de POST Backend: solo misma idempotencyKey y solo según DOC-004; preferir consulta status antes.
- Retry de efecto financiero con nueva key: prohibido automáticamente.
- Retry de escritura TCP: no existe retry transparente; nueva solicitud del cliente se deduplica.

## 7. Disponibilidad vs seguridad

El baseline prioriza no ejecutar un efecto no auditable y no afirmar un resultado no durable. Por tanto, indisponibilidad Mongo/Crypto/HSM puede rechazar temporalmente nuevos requests aunque Backend esté disponible. Es una decisión explícita de laboratorio, no una recomendación universal para redes de pago.

## 8. Criterios de aceptación

- Fault injection en cada checkpoint de §4.
- Ningún Backend POST ocurre antes de `REQUEST_DURABLE`.
- Ninguna respuesta ISO de negocio sale antes de `RESPONSE_DURABLE`.
- Timeout post-send recupera resultado por status/idempotency sin doble efecto.
- Reinicio Gateway conserva checkpoints y reanuda o resuelve estado.
