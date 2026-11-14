# DOC-007 — Timeouts, idempotencia, retención y expiración

**Versión:** 2.0.1  
**Naturaleza:** valores iniciales configurables del laboratorio `LAB-ATM-1`; no son parámetros de Link/Prisma ni SLO productivos.

## 1. Timeouts propuestos

| Límite | Valor inicial | Comportamiento |
|---|---:|---|
| TCP idle timeout | 60 s | Cerrar conexión inactiva |
| Frame assembly timeout | 5 s | Descartar frame incompleto y cerrar canal |
| Frame máximo | 4.096 bytes | Rechazar mayor tamaño |
| Mongo server selection | 2 s | Fallo temporal de persistencia |
| Mongo connect timeout | 2 s | Fallo temporal de persistencia |
| Mongo socket timeout | 3 s | Resultado de write puede ser incierto; consultar por ID antes de reintentar |
| Mongo write concern timeout | 5 s | No tratar como durable; reconciliar insert/update por ID/version |
| Backend connect timeout | 1 s | Si no se envió request, fallo conocido |
| Backend response timeout | 3 s | Tras envío, tratar como indeterminado hasta consultar |
| Crypto connect timeout | 1 s | No invocar Backend sin archivo durable |
| Crypto response timeout | 3 s | Consultar archivo por `messageId` antes de reintentar |
| HSM connect timeout | 1 s | Fallo de crypto; sin fallback |
| HSM response timeout | 2 s | Consultar estado del batch/key si operación pudo completarse |
| Recovery lease | 30 s | Claim de worker expira si no heartbeat |
| Recovery heartbeat | 10 s | Renovación condicional del lease |
| Clock-skew grace | 5 s | Margen antes de reclamar lease vencido |

Todos los valores son configuración externa con límites validados; no deben alterarse silenciosamente en el código.

## 2. Política de timeout

- Connect timeout antes de enviar request: clasificar como no enviado solo si el cliente puede demostrarlo.
- Read/response timeout después de enviar: resultado desconocido; estado `INDETERMINATE` o checkpoint recuperable.
- Mongo write concern timeout: resultado de escritura incierto; consultar documento/versión por clave natural antes de reinsertar.
- Crypto timeout: consultar `messageId` y digest antes de repetir archivo.
- HSM timeout al crear DEK: no crear otra DEK para el mismo lote hasta consultar estado; si no es recuperable, marcar lote `FAILED` y abrir lote nuevo, sin usar claves locales.
- No usar retry automático genérico para operaciones financieras/reversos.

## 3. Idempotency window

- Ventana de detección de duplicados exactos: **24 horas** desde primer request aceptado.
- Retener el registro de idempotencia al menos **7 días después de estado terminal**; si operación está `INDETERMINATE`, `PENDING` o `REVERSAL_PENDING`, no expira por TTL.
- Una clave no puede reutilizarse con payload distinto durante retención: conflicto 409.
- Para una operación todavía activa, el registro permanece sin expiración.
- La expiración se ejecuta con worker lógico; Mongo TTL puede limpiar solo registros elegibles con `expiresAt` ya asignado a terminales. No usar TTL como mecanismo de transición de estado.
- Una retransmisión posterior a 24 h pero antes de que el registro se elimine sigue deduplicándose. La ventana de 24 h es el mínimo de garantía; retención 7 días extiende la protección.

## 4. Retención documental y operacional

| Clase | Retención de laboratorio | Condición |
|---|---:|---|
| Mensajes ISO cifrados/saneados | 30 días | Solo bytes saneados; DE52 excluido |
| Operaciones y referencias | 90 días | Retener mientras exista indeterminado/pending; no TTL prematuro |
| Registros idempotencia | 7 días tras terminal | No expirar estados no terminales |
| Eventos audit operacionales | 180 días | Sin payload, PAN completo ni secretos |
| Lotes crypto cerrados | 90 días después de último documento | Metadata mínima necesaria para decrypt durante retención documental |
| Reservas/leases completadas | 7 días | Datos de diagnóstico saneados |

Estos valores son defaults de laboratorio para demostrar expiración y recuperación; no son política bancaria/legal. La retención final se configura por propiedad y se documenta en README de despliegue.

## 5. Expiración de autorización

- Duración de autorización pendiente: 60 s para esperar Backend; al vencer sin respuesta conocida, `INDETERMINATE`, no `DECLINED`.
- Validez de autorización aprobada para recibir `0200`: 15 minutos desde `approvedAt` como regla privada de laboratorio.
- Autorización expirada no puede ejecutar nuevo financiero.
- Una operación `INDETERMINATE` no expira automáticamente a `FAILED`; requiere status/reconciliation.

## 6. Lotes y expiración

- Máximo 1.000 reservas o 15 minutos desde apertura.
- `leaseUntil` 30 s, heartbeat 10 s, grace 5 s (DOC-005).
- Mensaje reservado antes del cierre debe completar o ser recuperado; no se libera contador sin transición durable de la reserva.
- Lote cerrado/fallido no admite nuevas reservas.

## 7. Circuit breaker y backoff

- No introducir circuit breaker en primera implementación salvo que las pruebas demuestren valor. Si se configura, debe aislar por dependencia.
- Para consultas status: backoff exponencial 250 ms, 500 ms, 1 s, 2 s, máximo 5 s entre intentos; jitter ±20%; límite de intentos por worker configurable.
- POST de efecto no se reintenta con nueva clave. Repetición con misma clave solo conforme contrato Backend y tras consultar status.

## 8. Criterios de aceptación

- Pruebas de boundary exactas para cada timeout.
- Simulación de respuesta perdida pre/post commit.
- Pruebas TTL no eliminan operaciones activas/indeterminadas.
- Idempotency key repetida con mismo payload converge; distinta payload conflicto.
- Autorización de 14:59 es elegible; de 15:00 o más no es elegible según reloj controlado.
