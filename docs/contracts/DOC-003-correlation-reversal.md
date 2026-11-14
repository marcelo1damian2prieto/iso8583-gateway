# DOC-003 — Correlación, idempotencia y referencias de reverso

**Versión:** 2.0.1  
**Perfil:** `LAB-ATM-1.0`  
**Estado:** aprobado documentalmente junto con SPEC/PLAN/TASKS v2.0.

## 1. Identificadores distintos

No deben confundirse los siguientes identificadores:

- `messageId`: UUID interno de cada mensaje recibido o emitido.
- `operationId`: UUID interno de la operación de negocio iniciada por `0100`.
- `financialOperationId`: identificador interno del procesamiento financiero asociado a la autorización.
- `reversalOperationId`: UUID interno de la solicitud lógica de reverso.
- `idempotencyKey`: clave estable para deduplicar un efecto de negocio en Gateway/Backend.
- `correlationToken`: token aleatorio de 128 bits (32 caracteres hex ASCII) emitido en DE48 de `0110` aprobado y ecoado en `0200`.
- `STAN`/`RRN`: referencias del mensaje/red del perfil; no reemplazan IDs internos ni idempotencyKey.

## 2. Origen lógico

La identidad de origen `OriginId` del perfil se construye como tupla exacta:

`OriginId = (DE32, DE33, DE41, DE42)`.

Los cuatro campos se normalizan según su formato del perfil (trim de padding definido por packager, sin alterar contenido semántico) y se comparan byte a byte después de normalización. No se aceptan comodines ni coincidencias parciales.

## 3. Correlación autorización → financiero

1. `0100` válido reserva `operationId` y `idempotencyKey`.
2. Tras resultado backend `APPROVED`, Gateway genera `correlationToken` con CSPRNG, 16 bytes, codificado como 32 caracteres hex en mayúsculas.
3. `0110` aprobado devuelve token en DE48. No se genera token en respuesta rechazada.
4. `0200` debe presentar DE48 con exactamente 32 caracteres hex y DE37 igual al RRN de autorización emitido por Backend/guardado en operación.
5. Gateway busca por hash del token (no almacenar el token en claro en índices/logs), `OriginId` y estado `AUTHORIZED`; verifica además DE3, DE4 y DE49 contra la autorización.
6. Token desconocido, expirado, consumido por operación distinta o de otro origen se rechaza sin invocar Backend.
7. La asociación se consume de forma atómica al reservar `FINANCIAL_PENDING`; una retransmisión exacta recupera la misma operación/resultado y no consume una segunda vez.

El token no es credencial ni autorización criptográfica; es un identificador opaco privado del laboratorio.

## 4. Idempotency key y fingerprint

Para cada request se calcula una clave lógica versionada. Forma conceptual:

`idempotencyKey = SHA-256(profileId || OriginId || MTI || STAN || DE7 || businessReferences || normalizedPayloadDigest)`.

`normalizedPayloadDigest` se calcula sobre representación canónica saneada y determinista del mensaje, excluyendo campos volátiles que el perfil identifique explícitamente. No se incluye DE52 ni bytes prohibidos. La serialización canónica y orden de campos se fija en el código de perfil y se prueba con fixtures.

- La clave se persiste como hash, no como concatenación sensible.
- Mismo key + mismo fingerprint: devolver/continuar el mismo resultado lógico.
- Mismo key + fingerprint distinto: conflicto `IDEMPOTENCY_CONFLICT`; no ejecutar Backend.
- STAN por sí solo nunca es clave.
- Una retransmisión con DE7 distinto se considera request distinto salvo regla específica futura aprobada.

## 5. Referencia inequívoca del reverso `0400`

`0400` solo puede referenciar un `0200` original mediante **todos** estos datos:

1. DE90 exacto de 42 caracteres: `0200(4) + originalSTAN(6) + originalDE7(10) + originalDE32(11) + originalDE33(11)`.
2. DE37 igual al RRN registrado para el `0200` original.
3. OriginId del reverso igual al del original: DE32, DE33, DE41 y DE42.
4. DE3, DE4 y DE49 iguales a los valores del original.

No se hace fallback a STAN solo, RRN solo, ventana temporal, importe o búsqueda aproximada. Si cualquiera de las referencias no coincide o devuelve más de un candidato, resultado interno `REVERSAL_REFERENCE_MISMATCH` y no se invoca Backend.

## 6. Reglas por estado original

| Estado de operación original | Acción para `0400` | Resultado interno |
|---|---|---|
| No existe | No invocar reverse | `ORIGINAL_NOT_FOUND` |
| `AUTHORIZATION_PENDING` / `FINANCIAL_PENDING` | Consultar estado backend; no ejecutar reverso hasta resolver | `REVERSAL_WAITING_ORIGINAL` |
| `INDETERMINATE` | Consultar estado backend; no inferir éxito ni rechazo | `REVERSAL_WAITING_RECONCILIATION` |
| `DECLINED` / `FAILED` sin efecto financiero | Rechazar reverso como no aplicable | `REVERSAL_NOT_APPLICABLE` |
| `COMPLETED` | Reservar reverso y solicitarlo a Backend | `REVERSAL_PENDING` |
| `REVERSAL_PENDING` | Recuperar misma solicitud; consultar estado, no crear otra | `REVERSAL_PENDING` |
| `REVERSED` | Responder resultado previo del mismo reverso si coincide; no ejecutar otra vez | `REVERSED` |
| `REVERSAL_DECLINED` | Repetición exacta devuelve resultado anterior; nuevo intento requiere política explícita | `REVERSAL_DECLINED` |

## 7. Idempotencia del reverso

La clave del reverso deriva de `profileId + OriginId + DE90 + DE37 + normalizedPayloadDigest`. MongoDB impone unicidad lógica para un único reverso activo/confirmado por operación original. La reserva usa actualización condicional/versionado. Dos `0400` concurrentes equivalentes deben converger en un único `reversalOperationId`.

Si el backend informa timeout tras posible commit, el reverso queda `INDETERMINATE`/`REVERSAL_PENDING` y se consulta por la misma idempotencyKey. No se emite un segundo comando con clave nueva.

## 8. Campos sensibles

- Token DE48: no loguear ni usar como label; almacenar hash para búsqueda y valor protegido solo si es necesario para construir/revalidar la respuesta.
- PAN: no formar parte de logs ni índices; si se requiere comparar, usar digest keyed/representación minimizada bajo decisión de seguridad.
- DE52: excluido.

## 9. Casos de aceptación

- Correlación exacta 0100→0110→0200.
- Token alterado, malformado, expirado o de otro OriginId rechazado.
- STAN repetido con DE7/origen distintos no colisiona indebidamente.
- DE90 con cualquier componente cambiado no resuelve operación original.
- Dos reversos concurrentes producen una única reserva y un efecto backend.
- Reverso repetido después de `REVERSED` devuelve resultado previo sin segundo efecto.
