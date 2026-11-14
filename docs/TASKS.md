# TASKS — ISO8583 Gateway Lab

**Versión:** 2.0  
**Estado:** backlog v2.0 y contratos DOC-001..DOC-009 aprobados documentalmente por el usuario  
**Autorización vigente:** DOC-001..DOC-009 aprobadas documentalmente. BOOT-* e implementación requieren autorización explícita por IDs/rango.  
**Regla:** ninguna tarea se ejecuta sin autorización explícita del usuario; respetar dependencias y gates.

## Convenciones

- `P0`: bloqueante/fundacional; `P1`: necesario para alcance funcional; `P2`: endurecimiento/operación.
- `S`: pequeña; `M`: media; `L`: grande. Estimaciones relativas, no horas.
- Las tareas de documentación/contratos preceden a Bootstrap.
- `0420/0430` y persistencia DE52 no forman parte del backlog inicial.

## Épica E0 — Validación y contratos

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida | Estado |
|---|---|---:|---|---|---|
| DOC-001 | P0 | M | Consolidar matriz SPEC→PLAN→TASKS y trazabilidad RF/NFR | — | Cada requisito tiene componente, tarea y prueba asociada  | Aprobado documentalmente |
| DOC-002 | P0 | M | Definir perfil ISO de laboratorio por MTI `0100/0110`, `0200/0210`, `0400/0410` | DOC-001 | Campos, tipos, longitudes, obligatoriedad, bitmap y códigos documentados  | Aprobado documentalmente |
| DOC-003 | P0 | M | Definir referencias de correlación y reverso sin heurísticas | DOC-002 | Casos de correlación/reverso inequívocos documentados  | Aprobado documentalmente |
| DOC-004 | P0 | M | Definir contrato REST del Backend Simulator | DOC-001 | authorize/process/reverse/status/idempotency y errores versionados  | Aprobado documentalmente |
| DOC-005 | P0 | M | Definir contrato Crypto Service y HSM Simulator | DOC-001 | APIs, errores, mTLS, límites y semántica idempotente documentados  | Aprobado documentalmente |
| DOC-006 | P0 | M | Definir política de durabilidad documental antes de responder | DOC-002,DOC-005 | Comportamiento ante fallo Mongo/Crypto/HSM sin ambigüedad  | Aprobado documentalmente |
| DOC-007 | P0 | M | Definir timeouts, idempotency window, retención y política de expiración | DOC-003,DOC-004 | Valores configurables y justificados en perfil de laboratorio  | Aprobado documentalmente |
| DOC-008 | P0 | M | Validar índices, agregados, transacciones y recuperación Mongo | DOC-001 | Modelo Mongo y pruebas de concurrencia propuestos  | Aprobado documentalmente |
| DOC-009 | P0 | S | Documentar exclusiones `0420/0430`, DE52 y PCI | DOC-001 | Límites y gates externos explícitos  | Aprobado documentalmente |

## Épica E1 — Bootstrap técnico (solo tras aprobación y autorización)

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| BOOT-001 | P0 | S | Verificar versiones compatibles Java 21/Spring Boot 3/Maven/Netty/jPOS/Mongo | DOC-001..DOC-009 | Matriz versionada sin dependencias flotantes |
| BOOT-002 | P0 | M | Crear estructura de repositorios/proyectos para Gateway, Backend, Crypto y HSM Simulator | BOOT-001 | Límites de despliegue claros; sin módulos por capa innecesarios |
| BOOT-003 | P0 | M | Configurar Maven Wrapper, compilación reproducible y perfiles | BOOT-002 | Build reproducible desde entorno limpio |
| BOOT-004 | P0 | M | Configurar Quality Gate y análisis estático/dependencias | BOOT-003 | Comandos y umbrales documentados y ejecutables |
| BOOT-005 | P0 | M | Crear Docker Compose mínimo con Mongo replica set y servicios vacíos | BOOT-002 | Servicios healthy; replica set inicializado |
| BOOT-006 | P1 | S | Configurar gestión externa de secretos/certificados de laboratorio | BOOT-005 | No hay secretos en Git ni imágenes |
| BOOT-007 | P1 | S | Crear health/readiness y configuración externa por servicio | BOOT-002 | Health checks verificables |

## Épica E2 — Transporte y protocolo ISO

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| ISO-001 | P0 | M | Implementar `FrameCodec` configurable | BOOT-003,DOC-002 | Fragmentación/concatenación/longitudes probadas |
| ISO-002 | P0 | M | Configurar packager jPOS del perfil de laboratorio | ISO-001,DOC-002 | Pack/unpack de fixtures válidos |
| ISO-003 | P0 | M | Implementar validación MTI/bitmap/DE | ISO-002 | Rechaza campos faltantes, tipos y longitudes inválidas |
| ISO-004 | P1 | M | Implementar servidor TCP/TLS Netty | BOOT-007,ISO-001 | TLS válido/inválido y límites probados |
| ISO-005 | P1 | M | Asegurar no bloqueo de EventLoops y gestión de canales | ISO-004 | Test de concurrencia y revisión de threads |
| ISO-006 | P1 | S | Implementar mapeo ISO↔commands/responses de aplicación | ISO-002,ISO-003 | Dominio no depende de jPOS |
| ISO-007 | P1 | S | Implementar respuesta por canal request y cierre seguro | ISO-004,ISO-006 | Pruebas de canal correcto/desconexión |

## Épica E3 — Backend/Core Simulator

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| BE-001 | P0 | M | Implementar modelo de operación y estados backend | BOOT-002,DOC-004 | Transiciones backend explícitas y probadas |
| BE-002 | P0 | M | Implementar autorización idempotente | BE-001,DOC-004 | Duplicados retornan resultado original |
| BE-003 | P0 | M | Implementar procesamiento financiero idempotente | BE-001,DOC-004 | Sin doble efecto ante retransmisión |
| BE-004 | P0 | M | Implementar reverso idempotente | BE-001,DOC-004 | Reversa única y estados de error diferenciados |
| BE-005 | P1 | M | Implementar consulta de estado/reconciliación | BE-002,BE-003,BE-004 | Resuelve estados conocidos/desconocidos sin reejecución ciega |
| BE-006 | P1 | S | Agregar fixtures y fault injection configurable | BE-002,BE-003,BE-004 | Simula timeout, respuesta perdida y latencia |

## Épica E4 — MongoDB, dominio e idempotencia Gateway

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| DB-001 | P0 | M | Configurar cliente Mongo, replica set y política read/write concern | BOOT-005,DOC-008 | Pruebas de conexión, fail/retry y consistencia |
| DB-002 | P0 | M | Implementar repositorio de operaciones y versionado optimista | DB-001 | Actualizaciones condicionales evitan lost update |
| DB-003 | P0 | M | Implementar repositorio de idempotencia e índices únicos | DB-001,DOC-007 | Reservas concurrentes producen una sola clave ganadora |
| DB-004 | P0 | M | Implementar repositorio de correlación | DB-001,DOC-003 | Lookup exacto y expiración probados |
| DB-005 | P1 | M | Implementar modelo y repositorio de mensajes documentales | DB-001,DOC-006 | Schema versionado; datos sensibles saneados |
| DB-006 | P1 | M | Implementar auditoría operacional | DB-001 | Sin secretos/payload; eventos correlacionables |
| DB-007 | P1 | M | Probar transacciones breves multi-documento | DB-002,DB-003 | Atomicidad y rollback demostrados |

## Épica E5 — Flujos de autorización y operación financiera

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| FLOW-001 | P0 | M | Implementar máquina de estados Gateway | DB-002,DOC-003 | Transiciones válidas/ilegales cubiertas |
| FLOW-002 | P0 | L | Implementar caso de uso `0100` autorización | ISO-006,DB-003,BE-002,FLOW-001 | Aprobación/rechazo/timeout y duplicado probados |
| FLOW-003 | P0 | M | Construir `0110` según perfil y token DE48 | FLOW-002,DOC-002 | Respuesta correcta; token solo en perfil aprobado |
| FLOW-004 | P0 | L | Implementar caso de uso `0200` financiero | DB-004,BE-003,FLOW-001,FLOW-003 | Validación de correlación/estado/idempotencia probada |
| FLOW-005 | P0 | M | Construir `0210` y manejar indeterminados | FLOW-004,DOC-002 | Timeout ambiguo no se transforma en rechazo |
| FLOW-006 | P1 | M | Implementar persistencia de mensajes inbound/outbound | DB-005,FLOW-002,FLOW-004 | Mensajes persistidos conforme política documental |

## Épica E6 — Reversos `0400/0410`

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| REV-001 | P0 | M | Definir y probar máquina de estados de reverso | FLOW-001,DOC-003 | Transiciones por estado original documentadas |
| REV-002 | P0 | L | Implementar correlación `0400` con operación original | DB-004,REV-001 | No usa heurísticas ni correlación ambigua |
| REV-003 | P0 | M | Implementar reserva idempotente del reverso | DB-003,REV-002 | Dos reversos concurrentes no producen doble efecto |
| REV-004 | P0 | L | Implementar orquestación backend de reverso | BE-004,REV-003 | Resultado confirmado/denegado/indeterminado |
| REV-005 | P0 | M | Construir `0410` según resultado y perfil | REV-004,DOC-002 | Códigos/estado reflejan resultado real |
| REV-006 | P1 | M | Implementar reverso duplicado y casos borde | REV-003,REV-004 | Repetición segura; operación inexistente/pending/indeterminate probada |

## Épica E7 — HSM Simulator y Cryptographic Service

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| HSM-001 | P0 | M | Definir almacén de claves del simulador y protección del secreto raíz | BOOT-006,DOC-005 | KEK no se guarda en claro junto a datos transaccionales |
| HSM-002 | P0 | M | Implementar autenticación mTLS e identidad de cliente | BOOT-006,HSM-001 | Certificados válidos/inválidos/revocados probados |
| HSM-003 | P0 | M | Implementar generación DEK y wrapping/unwrapping | HSM-001,HSM-002 | DEK temporal y wrapped DEK funcionan; KEK nunca sale |
| HSM-004 | P1 | M | Implementar metadata/versionado/rotación de KEK | HSM-003 | Datos históricos descifrables con versión correcta |
| HSM-005 | P1 | M | Implementar operaciones PIN de fixtures sintéticos | HSM-002 | Sin PIN en claro de salida/logs; no procesa datos reales |
| CRYPTO-001 | P0 | M | Implementar cliente HSM port/adaptador REST | HSM-002 | Timeouts, errores y mTLS mapeados |
| CRYPTO-002 | P0 | M | Implementar coordinador de lotes Mongo | DB-001,CRYPTO-001 | Reserva atómica y límites configurables |
| CRYPTO-003 | P0 | M | Implementar cierre híbrido 1.000/15 min | CRYPTO-002 | Primer límite alcanzado cierra lote |
| CRYPTO-004 | P0 | M | Implementar AES-256-GCM con AAD y nonce único | HSM-003,CRYPTO-002 | Tampering/tag/AAD inválido falla autenticación |
| CRYPTO-005 | P0 | M | Implementar persistencia documental cifrada | DB-005,CRYPTO-004 | Ciphertext + nonce/tag + metadata; nunca DEK en claro |
| CRYPTO-006 | P1 | M | Implementar recuperación/descifrado autorizado | CRYPTO-005,HSM-003 | Cualquier instancia autorizada recupera documento |
| CRYPTO-007 | P1 | M | Implementar recuperación de reservas abandonadas | CRYPTO-002,DB-007 | No quedan reservas bloqueadas indefinidamente |
| CRYPTO-008 | P1 | M | Escalar a dos instancias y probar coordinación | CRYPTO-002,CRYPTO-003,CRYPTO-004 | Sin sobreasignación, doble asignación ni nonce duplicado |

## Épica E8 — Reconciliación y recuperación

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| REC-001 | P0 | M | Implementar consulta de estado para operaciones indeterminadas | BE-005,FLOW-001 | Estado actualizado solo con evidencia backend |
| REC-002 | P0 | M | Implementar worker de recuperación con claim/lease Mongo | DB-002,REC-001 | Dos instancias no reclaman el mismo trabajo simultáneamente |
| REC-003 | P1 | M | Recuperar flujo tras reinicio de Gateway | FLOW-002,FLOW-004,REC-001 | Reinicio no pierde operación ni duplica efectos |
| REC-004 | P1 | M | Recuperar Crypto Service tras reserva/cifrado/persistencia parcial | CRYPTO-007 | Reanudación o cierre controlado verificable |
| REC-005 | P1 | M | Probar caída Mongo/HSM/backend y recuperación | REC-001,REC-003,REC-004 | Estados y resultados coherentes tras recuperación |
| REC-006 | P2 | M | Implementar backup/restore Mongo + claves simulador | DB-001,HSM-001 | Restauración permite descifrar documentos de prueba |

## Épica E9 — Seguridad y observabilidad

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| SEC-001 | P0 | M | Revisar clasificación y saneamiento de datos ISO | DOC-002,DB-005 | Matriz DE/CHD/SAD y política por campo |
| SEC-002 | P0 | M | Asegurar ausencia de PAN completo/PIN/DE52/secrets en logs | SEC-001 | Tests automáticos de redacción/escaneo pasan |
| SEC-003 | P1 | M | Configurar autorización de acceso a descifrado y auditoría | CRYPTO-006,HSM-002 | Identidad no autorizada no puede descifrar |
| OBS-001 | P1 | M | Implementar logs estructurados saneados | BOOT-007 | Campos correlacionables, sin payload sensible |
| OBS-002 | P1 | M | Instrumentar métricas de MTI, latencia, estados, lotes y dependencias | OBS-001 | Métricas sin labels de alta cardinalidad |
| OBS-003 | P2 | M | Instrumentar trazas distribuidas saneadas | OBS-001,OBS-002 | No contienen payload/secretos |
| OBS-004 | P2 | M | Dashboards y alertas básicas de laboratorio | OBS-002,OBS-003 | Fallos/latencias/lotes visibles |

## Épica E10 — Verificación final

| ID | Pri. | Tamaño | Tarea | Dependencias | Criterio de salida |
|---|---|---:|---|---|---|
| QA-001 | P0 | M | Matriz trazabilidad requisito→test→evidencia | DOC-001 | Cobertura de RF/NFR y huecos explícitos |
| QA-002 | P0 | L | Suite end-to-end autorización/financiero/reverso | FLOW-*,REV-* | Escenarios positivos, negativos, duplicados y bordes pasan |
| QA-003 | P0 | L | Suite de concurrencia multi-instancia | CRYPTO-008,REV-003,DB-007 | Límites e invariantes pasan bajo carga concurrente |
| QA-004 | P0 | L | Suite de fallos y recuperación | REC-005,REC-006 | Resultados indeterminados y restauración probados |
| QA-005 | P0 | M | Revisión de seguridad y leakage | SEC-001,SEC-002,SEC-003 | Sin hallazgos críticos abiertos en alcance de laboratorio |
| QA-006 | P0 | M | Ejecutar Quality Gate completo y registrar evidencia | BOOT-004,QA-002,QA-003,QA-004,QA-005 | Build/tests/análisis pasan; resultados adjuntos |
| QA-007 | P1 | M | Benchmark reproducible y reporte de límites | QA-002,OBS-002 | Latencia/throughput medidos, sin claims no sustentados |

## 12. Exclusiones explícitas del backlog inicial

- No hay tareas `0420/0430` hasta aprobación específica de alcance.
- No hay tareas para persistir DE52/PIN block; operaciones PIN solo fixtures sintéticos en HSM Simulator.
- No hay clearing/settlement ni batch/FTP bancario.
- No hay Azure/Kubernetes en el primer despliegue.
- No hay conexión a redes productivas ni certificación.

## Gate de ejecución

DOC-001..DOC-009 fueron elaboradas y aprobadas documentalmente junto con SPEC/PLAN/TASKS v2.0. BOOT-001..BOOT-007 requieren autorización explícita por IDs/rango; la aprobación documental no habilita Bootstrap ni implementación general.
