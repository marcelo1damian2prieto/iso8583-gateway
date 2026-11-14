# Validación del diseño — ISO8583 Gateway Lab v2.0

**Estado:** DOC-001..DOC-009, SPEC v2.0, PLAN v2.0 y TASKS v2.0 aprobados documentalmente por el usuario. No hay implementación ni evidencia de tests ejecutados.  
**Base:** arquitectura integral v2.0 aprobada y borradores SPEC/PLAN/TASKS v2.0.

## 1. Requisitos funcionales

Cubiertos en SPEC: transporte/framing, perfil ISO, autorización `0100/0110`, financiero `0200/0210`, reverso `0400/0410`, MongoDB durable, idempotencia, mensajes cifrados, lotes, HSM Simulator, reconciliación y observabilidad.

## 2. Requisitos no funcionales

Cubiertos: no bloqueo de EventLoops, concurrencia multi-instancia, durabilidad, resiliencia, seguridad de canales/secretos, confidencialidad, recuperabilidad, mantenibilidad, calidad y operabilidad. Rendimiento queda como medición del laboratorio, sin SLO ni claims bancarios no sustentados.

## 3. Reglas de negocio e invariantes

- `0200` solo continúa una autorización válida y definitiva.
- `0400` identifica la operación por referencias explícitas del perfil; no se usan heurísticas.
- Un reverso confirmado no se ejecuta por segunda vez.
- Duplicados concurrentes producen un único efecto lógico.
- Timeout ambiguo no equivale a rechazo ni autoriza reejecución ciega.
- Operación `INDETERMINATE` requiere consulta/reconciliación.
- Lote acepta como máximo 1.000 reservas o 15 minutos, lo que ocurra primero.
- Una DEK por lote; nonce GCM único por cifrado bajo esa DEK.
- Lote cerrado/fallido no acepta nuevas reservas.
- No se persiste DE52/PIN block en el baseline ejecutable.
- KEK no sale en claro del HSM Simulator; DEK no se persiste en claro.
- MongoDB es la única base de datos del Gateway; no se añade Outbox entre bases.

## 4. Escenarios positivos, negativos y de borde

| Dominio | Positivos | Negativos/borde obligatorios |
|---|---|---|
| TCP/framing | frame único, fragmentado, frames concatenados | longitud inválida, truncado, exceso de tamaño, desconexión |
| ISO | MTI/campos válidos | bitmap inconsistente, campo requerido ausente, longitud/tipo inválido |
| Autorización | aprobado, rechazado | duplicado concurrente, backend timeout pre/post commit |
| Financiero | autorizado válido | token desconocido, importe distinto, autorización indeterminada, duplicado |
| Reverso | reverso exitoso | original inexistente/pending/indeterminado, duplicado concurrente, ya revertido |
| Mongo | insert/update normal | caída, write timeout, conflicto de versión, transacción abortada |
| Crypto | cifrar/descifrar | tampering ciphertext/tag/AAD, wrapped key incorrecta, HSM no disponible |
| Lotes | cierre por tiempo/cantidad | dos instancias compiten, límite simultáneo, reserva abandonada, nonce colisionado |
| Recuperación | reinicio y continuación | caída entre cada checkpoint, respuesta backend perdida |
| Seguridad | mTLS válido, descifrado autorizado | certificado inválido/revocado, acceso no autorizado, fuga en logs/traces |

## 5. Estrategia de testing

Unit tests de dominio y transiciones; contract tests de APIs; pruebas de protocolo con fixtures; integración con Mongo replica set y servicios Docker; concurrencia con dos instancias Crypto; fault injection; pruebas de recuperación y backup/restore; pruebas de seguridad y escaneo de telemetría. Cada RF/NFR debe vincularse con prueba y evidencia.

## 6. Riesgos y mitigaciones

| Riesgo | Mitigación verificable |
|---|---|
| Ambigüedad de perfil ISO/red | perfil de laboratorio explícito; no afirmar Link/Prisma |
| Doble efecto por timeout | idempotencia backend + consulta de estado; sin retry ciego |
| Carrera en Mongo/lotes | índices únicos, actualizaciones condicionales, transacciones breves, tests multi-instancia |
| Reutilización nonce GCM | nonce único por DEK, restricción única y tests concurrentes |
| Pérdida KEK/raíz | backup protegido y restore conjunto ensayado |
| Exposición PAN/SAD | saneamiento, exclusión DE52, pruebas de leakage |
| Confusión simulador/compliance | rotular límites; no claims PCI/FIPS |

## 7. Criterios de aceptación verificables

Se consideran satisfechos únicamente con evidencia de tests/build: invariantes financieros, límites de lote, no doble ejecución, persistencia durable, recuperación, cifrado autenticado, ausencia de secretos en telemetría, mTLS, y restauración de datos+claves. No se considera evidencia suficiente una inspección visual del código.

## 8. Resultado de DOC-001..DOC-009 y gates restantes

| Gate | Documento | Estado documental | Condición restante |
|---|---|---|---|
| Trazabilidad RF/NFR | DOC-001 | Elaborado | Verificar cobertura al ejecutar QA-001 |
| Perfil ISO y framing | DOC-002 | Definido como perfil privado `LAB-ATM-1.0` | Implementar packager/fixtures y validar con tests |
| Correlación/reverso | DOC-003 | Definido sin heurísticas | Validar concurrencia e invariantes en código |
| Backend REST | DOC-004 | Contrato `backend-api-v1` definido | Generar OpenAPI/contract tests en implementación |
| Crypto/HSM | DOC-005 | Contratos `crypto-api-v1`/`hsm-api-v1` definidos | Verificar implementación mTLS y key lifecycle |
| Durabilidad | DOC-006 | Política fail-closed definida | Fault injection por checkpoint |
| Timeouts/retención | DOC-007 | Defaults configurables definidos | Validar comportamiento temporal con tests |
| Mongo consistency | DOC-008 | Concerns, colecciones, índices y leases definidos | Testcontainers replica set y concurrencia |
| Exclusiones/seguridad | DOC-009 | Baseline y ciclo mTLS definidos | Verificar soporte CRL en stack TLS; si no, detenerse y solicitar decisión |
| Quality Gate | BOOT-004 | No definido aún | Seleccionar herramientas/umbrales durante BOOT-004 autorizado |
| Versiones dependencias | BOOT-001 | No fijadas | Verificar compatibilidad y fijar versiones durante BOOT-001 autorizado |

`0420/0430` y persistencia DE52 permanecen fuera del baseline. La excepción PCI DSS 3.3.3 no está habilitada. Los contratos son decisiones de laboratorio, no certificación ni evidencia de ejecución.

## 9. Resultado

El bloque documental DOC-001..DOC-009 y SPEC/PLAN/TASKS v2.0 fueron aprobados por el usuario. Bootstrap aún requiere autorización explícita de BOOT-*; durante BOOT-001/004 se fijarán versiones y Quality Gate, y la compatibilidad CRL debe verificarse antes de declarar mTLS listo. Esta validación no es una auditoría PCI ni certificación de seguridad. No autoriza implementación.
