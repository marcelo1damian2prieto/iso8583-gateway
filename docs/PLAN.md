# PLAN — ISO8583 Gateway Lab

**Versión:** 2.0  
**Estado:** borrador para validación de diseño  
**Código/Bootstrap:** requieren autorización explícita para los IDs BOOT-* concretos; la aprobación documental no habilita ejecución.  
**Arquitectura base:** `architecture-review-gateway-v2.0-integral.md` aprobada por el usuario.

## 1. Objetivo del plan

Descomponer la SPEC v2.0 en componentes, contratos, decisiones de implementación y fases verificables. El plan no autoriza por sí mismo la ejecución.

## 2. Arquitectura objetivo

### 2.1 Servicios

1. **ISO8583 Gateway**: TCP/TLS Netty, `FrameCodec`, jPOS, validación del perfil, casos de uso, correlación, idempotencia, orquestación y respuesta.
2. **Backend/Core Simulator**: API para autorización, operación financiera, reverso, consulta de estado e idempotencia.
3. **Cryptographic Service**: lotes, reserva de capacidad, cifrado/descifrado documental, persistencia de mensajes y recuperación.
4. **HSM Simulator**: servicio separado con API REST/mTLS; custodia simulada de KEK, generación/wrapping/unwrapping de DEK, operaciones PIN sintéticas.
5. **MongoDB Replica Set**: única base de datos del Gateway y repositorio documental/operacional. Replica set local de un nodo para pruebas; no representa HA.
6. **Observabilidad**: logs JSON, métricas y trazas saneadas.

### 2.2 Stack

Java 21, Spring Boot 3.x, Maven, Netty, jPOS, WebClient, MongoDB, Docker Compose, JUnit 5, Mockito y Testcontainers. Versiones concretas fijadas en Bootstrap autorizado, sin versiones flotantes.

### 2.3 Estructura lógica del Gateway

Proyecto Maven único, paquetes por responsabilidad:

```text
gateway/
  domain/
    operation/ reversal/ message/ identity/
  application/
    port/ service/ correlation/ idempotency/ recovery/
  infrastructure/
    inbound/tcp/ iso8583/ outbound/backend/
    persistence/mongodb/ crypto-client/ observability/
  configuration/
```

Cryptographic Service y HSM Simulator son aplicaciones desplegables separadas; no módulos/capas internas del Gateway. Mantener DDD pragmático y dominio independiente de frameworks.

## 3. Modelo de persistencia MongoDB

Colecciones propuestas, sujetas a validación de índices y agregados:

- `operations`: estado actual, referencias, versión optimista, timestamps y relaciones funcionales.
- `idempotency_records`: clave normalizada/huella, `messageId`, resultado asociado, expiración/retención.
- `messages`: documento por mensaje, metadata, representación parseada saneada (PAN omitido o cifrado a nivel de campo), ciphertext, nonce, tag, `batchId`, `wrappedDek` o referencia al lote, key metadata y digest definido.
- `crypto_batches`: estado, apertura/cierre, contador reservado, key metadata, reservas y checkpoints necesarios.
- `audit_events`: eventos operacionales de seguridad/transiciones sin secretos ni payloads.

No duplicar `wrappedDek` por mensaje si una referencia estable al lote resuelve la recuperación. Elegir un modelo consistente antes de Bootstrap. Índices únicos/compuestos: idempotencia, `messageId`, reserva de mensaje por lote, `(batchId, nonce)`, claves de correlación y referencias de operación/reverso. Las restricciones exactas se derivan de la clave de negocio y deben probarse.

### Consistencia

- Atomicidad de documento como primera opción.
- Actualizaciones condicionales por `status`/`version`.
- Transacciones multi-documento breves cuando la operación de negocio lo requiera.
- Nunca mantener transacción abierta sobre backend/Crypto/HSM.
- Configurar y documentar `writeConcern`/`readConcern` para el objetivo de durabilidad.
- Sin Outbox entre bases: MongoDB es el único almacén del Gateway. La recuperación usa checkpoints y estados persistidos.

## 4. Diseño criptográfico

### 4.1 Envelope Encryption

- HSM Simulator crea DEK AES-256 por lote y devuelve DEK temporal + `wrappedDek`.
- KEK nunca se expone en claro desde el simulador.
- Cryptographic Service cifra cada documento con AES-256-GCM, nonce de 96 bits, tag de 128 bits y AAD que vincula `batchId`, `messageId`, tipo y versión de esquema.
- MongoDB conserva ciphertext, nonce, tag y metadata de clave protegida.
- Descifrado requiere autorización, recuperación de `wrappedDek`, unwrap por HSM y uso transitorio de DEK.
- Nunca fallback a clave local ni log de material secreto.

### 4.2 Lotes

Política híbrida acordada: cerrar al primer límite alcanzado, 1.000 mensajes reservados o 15 minutos desde apertura. La reserva debe ser atómica y contar antes de cifrado/persistencia para evitar sobreasignación. Estados `OPEN`, `CLOSING`, `CLOSED`, `FAILED`. No reabrir lote cerrado.

### 4.3 Nonce y fallo parcial

Cada cifrado usa nonce único por DEK. Ante resultado incierto de persistencia, reintentar con nonce nuevo y registrar la relación de intento sin exponer secretos. Una restricción única por `(batchId, nonce)` detecta colisiones. Definir límites de uso de GCM por DEK como control explícito; 1.000/15 min es política de laboratorio, no certificación.

## 5. Contratos de servicio

Los contratos versionados y sus payloads/errores están definidos en `docs/contracts/`:

- DOC-002: perfil ISO `LAB-ATM-1.0` y framing.
- DOC-003: correlación y reverso inequívoco.
- DOC-004: Backend Simulator `backend-api-v1`.
- DOC-005: Cryptographic Service `crypto-api-v1` y HSM Simulator `hsm-api-v1`.
- DOC-006: orden transaccional y durabilidad.
- DOC-007: timeouts, idempotency window y retención.
- DOC-008: consistencia/índices/recuperación MongoDB.
- DOC-009: mTLS, clasificación y exclusiones.

OpenAPI 3.1, DTOs y pruebas de contrato se crearán en implementación, derivados de estos documentos; no se afirma que existan todavía.

## 6. Flujos transaccionales

### 6.1 Autorización

1. Recibir frame y validar transporte/protocolo.
2. Parsear/validar mensaje; derivar origen e idempotency key.
3. Registrar mensaje/operación y reserva idempotente con consistencia durable.
4. Invocar backend fuera de transacción Mongo.
5. Persistir resultado con actualización condicional.
6. Construir respuesta `0110`; persistir documento saliente cifrado/saneado conforme política.
7. Escribir respuesta por el canal del request.

### 6.2 Operación financiera

1. Recibir/validar `0200` y resolver autorización mediante referencias explícitas.
2. Validar estado, importe, moneda, origen y duplicidad.
3. Reservar `FINANCIAL_PENDING` idempotentemente.
4. Invocar backend con idempotency key estable.
5. Persistir resultado o `INDETERMINATE` ante timeout ambiguo.
6. Construir/persistir `0210` y responder por canal.

### 6.3 Reverso

1. Recibir/validar `0400` y resolver operación original por referencias del perfil.
2. Reservar reverso idempotentemente; no permitir doble reversa.
3. Si la operación original es indeterminada, consultar backend; no inferir estado.
4. Invocar backend `reverse`/consulta según contrato.
5. Persistir resultado y generar `0410`.
6. Ante incertidumbre, mantener estado indeterminado/reversal pending y reconciliar.

### 6.4 Mensajería documental

La respuesta ISO no debe depender de una transacción remota larga. PLAN debe fijar el punto de durabilidad exigido antes de responder. Si no puede persistirse evidencia cifrada requerida, aplicar política fail-closed para archivo documental y no simular éxito de cifrado. No conservar DE52 en baseline; trama persistida saneada.

## 7. Recuperación y resiliencia

- Timeouts separados por TCP, backend, Crypto, HSM y Mongo.
- Circuit breaker solo donde sea semánticamente seguro.
- Estados durables y checkpoints; workers de recuperación con claims/lease o actualización condicional en Mongo, evitando locks JVM.
- Backend expone consulta de estado para resolver `INDETERMINATE`.
- No reejecutar ciegamente una operación financiera o reverso.
- Recuperar reservas de lote expiradas/abandonadas de forma determinista.
- Pruebas de reinicio, indisponibilidad y recuperación de backup + claves.

## 8. Seguridad

- TLS inbound y mTLS entre servicios internos acordados.
- Certificados y secretos fuera del repositorio/imágenes.
- MongoDB con autenticación, roles mínimos y red privada.
- PAN minimizado/protegido; no PAN completo en telemetría.
- DE52/SAD excluido de persistencia ejecutable inicial.
- DEK solo transitoria; KEK dentro del simulador; secreto raíz protegido fuera del volumen de datos.
- HSM Simulator no se presenta como HSM certificado.
- Auditoría de uso de claves y accesos a descifrado.
- Backups separados de datos y claves; restauración coordinada probada.

## 9. Observabilidad

Logs estructurados con `operationId`/`messageId` enmascarados o IDs internos no sensibles; métricas por MTI/resultado/etapa y latencia; trazas sin payload. No usar PAN, PIN, STAN, RRN, token completo ni identificadores de alta cardinalidad como labels.

## 10. Estrategia de testing

- Unit: estados, reglas, idempotencia, correlación, reversos, validadores, política de lotes.
- Protocol: pack/unpack, bitmap, DE, framing fragmentado/concatenado y límites.
- Integration: Mongo replica set, backend, Crypto Service y HSM Simulator.
- Concurrency: dos o más instancias Crypto, carreras de reserva, duplicados de operación/reverso y nonce.
- Failure: timeouts, respuesta perdida, reinicios, caída Mongo/HSM, fallos entre reserva/cifrado/persistencia.
- Security: mTLS válido/inválido, autorización de servicios, secretos ausentes en logs, tampering GCM/AAD, acceso no autorizado a descifrado.
- Recovery: restauración de datos y claves, operaciones indeterminadas, reservas huérfanas.

## 11. Fases y gates

| Fase | Resultado | Gate de salida |
|---|---|---|
| 0. Validación documental | SPEC/PLAN/TASKS alineados, decisiones abiertas etiquetadas | aprobación documental explícita |
| 1. Bootstrap técnico | repositorios, Maven, versiones, quality gate, Compose mínimo | builds reproducibles y health checks |
| 2. Fundaciones protocolo | FrameCodec, perfil jPOS, validación, simulador backend | pruebas protocolo y contratos |
| 3. Persistencia e idempotencia | Mongo replica set, repositorios, índices, estados | concurrencia/recuperación base |
| 4. Flujos financieros | 0100/0110, 0200/0210, 0400/0410 | aceptación funcional completa |
| 5. Seguridad criptográfica | HSM Simulator, Crypto Service, envelope, mTLS | pruebas crypto y seguridad |
| 6. Lotes y recuperación | 1.000/15 min, multi-instancia, reconciliación | pruebas concurrentes y fallos |
| 7. Observabilidad y endurecimiento | métricas, trazas, backups, quality gate final | evidencia de VERIFY |
| 8. Azure (posterior) | despliegue cloud | nueva aprobación de alcance |

## 12. Gates restantes antes de Bootstrap

Los gates de perfil, correlación/reverso, contratos backend/crypto/HSM, durabilidad, timeouts/retención, MongoDB y exclusiones fueron definidos documentalmente en DOC-001..DOC-009. No son evidencia de implementación ni pruebas ejecutadas.

La revisión y aprobación formal de SPEC/PLAN/TASKS y contratos DOC-001..DOC-009 fue recibida. Antes de declarar listo el canal mTLS debe verificarse la compatibilidad de revocación CRL en el stack TLS elegido; si no se soporta, detenerse y solicitar decisión. Las versiones exactas se fijan en BOOT-001 y las herramientas/umbrales del Quality Gate en BOOT-004. La ejecución de cada BOOT-* requiere autorización explícita por ID o rango. `0420/0430` y DE52 permanecen excluidos.

## 13. Restricciones de ejecución

No implementar, crear código de producción, generar scaffolding ni iniciar Bootstrap hasta aprobación explícita de SPEC/PLAN/TASKS y autorización explícita de tareas concretas. Una aprobación documental posterior tampoco equivale a autorización general para implementar todo el backlog.
