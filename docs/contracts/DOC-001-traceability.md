# DOC-001 — Matriz de trazabilidad SPEC → PLAN → TASKS

**Versión:** 2.0.1  
**Estado:** aprobado documentalmente por el usuario  
**Alcance:** trazabilidad del baseline v2.0; no constituye evidencia de pruebas ejecutadas.

## 1. Convenciones

- **Componente:** servicio o módulo responsable primario; las dependencias cruzadas se indican en PLAN.
- **Tarea:** ID del backlog que materializa el requisito. Una tarea puede cubrir varios requisitos relacionados.
- **Prueba/evidencia:** suite o tipo de prueba que debe producir evidencia al ejecutar la tarea. Los nombres son identificadores de prueba propuestos, no clases ya existentes.
- **N/A:** requisito no cubierto por código del baseline; su exclusión debe quedar verificada documentalmente.

## 2. Matriz RF

| Requisito | Componente principal | Tareas | Prueba/evidencia requerida |
|---|---|---|---|
| RF-001 | Gateway / Netty TLS | BOOT-006, ISO-001 | IT-TLS-001: handshake válido; certificado inválido rechazado |
| RF-002 | Gateway / FrameCodec | ISO-001 | PT-FRAME-001: fragmentación, concatenación, truncado y longitud inválida |
| RF-003 | Gateway / FrameCodec + perfil | DOC-002, ISO-001 | PT-FRAME-002: framing coincide con perfil `LAB-ATM-1`; sin atribución a redes |
| RF-004 | Gateway / jPOS | DOC-002, ISO-002 | PT-ISO-001: pack/unpack fixtures por MTI |
| RF-005 | Gateway / validador ISO | ISO-003 | UT-ISO-001: MTI, bitmap, obligatoriedad, longitud y tipo |
| RF-006 | Gateway / error mapping | ISO-003, FLOW-005 | UT-ERR-001: categorías de error no se confunden |
| RF-007 | Gateway / session-channel | ISO-006 | IT-TCP-001: respuesta en canal de request y desconexión |
| RF-010 | Gateway / Authorization use case | FLOW-001, FLOW-002 | IT-AUTH-001: autorización válida, inválida y duplicada |
| RF-011 | Gateway / Mongo + Crypto | DOC-006, DB-002, CRYPTO-005 | IT-DUR-001: no backend antes de aceptación durable |
| RF-012 | Gateway + Backend Simulator | DOC-004, BE-002, FLOW-002, FLOW-003 | CT-BE-001, IT-AUTH-002: aprobado/rechazado |
| RF-013 | Gateway / correlation | DOC-002, DOC-003, FLOW-003 | UT-CORR-001: token aleatorio y eco DE48; token inválido rechazado |
| RF-014 | Gateway / Financial use case | DOC-003, FLOW-004 | IT-FIN-001: origen, importe, moneda, autorización y referencia |
| RF-015 | Gateway + Backend Simulator | DOC-004, BE-003, FLOW-004, FLOW-005 | CT-BE-002, IT-FIN-002: respuesta y timeout ambiguo |
| RF-016 | Gateway / state machine | FLOW-001, FLOW-004 | UT-STATE-001: estados no elegibles no procesan financiero |
| RF-020 | Gateway / Reversal use case | DOC-002, DOC-003, REV-001, REV-002 | IT-REV-001: resolución exacta por DE90 y referencias |
| RF-021 | Gateway / idempotency + Mongo | DOC-003, DOC-008, REV-003 | IT-REV-002: dos reversos concurrentes, una reserva ganadora |
| RF-022 | Gateway + Backend Simulator | DOC-004, BE-004, REV-004 | CT-BE-003, IT-REV-003: reverse y status |
| RF-023 | Gateway / response builder | DOC-002, REV-005 | PT-ISO-REV-001: `0410` por resultado de backend |
| RF-024 | Gateway / state machine + backend status | DOC-003, DOC-004, REV-001, REC-001 | UT-REV-STATE-001: original inexistente/pending/processed/declined/indeterminate/reversed |
| RF-025 | Gateway / uncertainty policy | DOC-006, DOC-007, FLOW-005, REV-004 | IT-UNC-001: no rechazo artificial ni éxito falso |
| RF-026 | Gateway / idempotency | DOC-003, DOC-007, REV-003, REV-006 | IT-REV-004: retransmisión exacta y concurrente |
| RF-030 | MongoDB / repositories | DOC-008, DB-001..DB-006 | IT-MONGO-001: documentos operacionales y auditables |
| RF-031 | MongoDB replica set | DOC-008, DB-001, DB-007 | IT-MONGO-002: índices, atomicidad, rollback |
| RF-032 | Application orchestration | DOC-006, DB-007 | ARCH-001: ninguna transacción Mongo cruza llamada remota |
| RF-033 | Crypto Service + Mongo `messages` | DOC-005, DOC-006, CRYPTO-005, SEC-001 | IT-DOC-001: schema, saneamiento y persistencia |
| RF-034 | Crypto Service | DOC-005, CRYPTO-004 | UT-CRYPTO-001: AES-GCM nonce/tag/AAD |
| RF-035 | Crypto Service + HSM | DOC-005, CRYPTO-005 | IT-CRYPTO-002: wrapped DEK y metadata, sin DEK clara persistida |
| RF-036 | ISO sanitizer + Crypto Service | DOC-009, SEC-001, SEC-002 | SEC-LEAK-001: DE52/CVV ausentes de persistencia y telemetría |
| RF-037 | ISO sanitizer + document model | DOC-006, DOC-009, SEC-001, SEC-002 | SEC-LEAK-002: PAN completo ausente en parseo/logs |
| RF-038 | Crypto Service / digest | DOC-005, CRYPTO-005 | UT-DIGEST-001: hash de bytes definidos, no de trama excluida |
| RF-039 | Crypto Service + HSM | DOC-005, HSM-003, CRYPTO-006 | IT-CRYPTO-003: decrypt desde otra instancia autorizada |
| RF-040 | Recovery worker + Mongo | DOC-006, DOC-008, DB-002, CRYPTO-007, REC-002 | IT-REC-001: claims y reservas abandonadas |
| RF-050 | Cryptographic Service | DOC-005, BOOT-002, CRYPTO-001 | IT-DEPLOY-001: servicio separado y health |
| RF-051 | Crypto Service + Mongo | DOC-008, CRYPTO-002 | IT-BATCH-001: reserva atómica y estados |
| RF-052 | Crypto Service | DOC-005, CRYPTO-003 | UT-BATCH-001: límite 1.000 / 15 min |
| RF-053 | HSM Simulator | DOC-005, HSM-003 | IT-HSM-001: DEK por lote y KEK confinada |
| RF-054 | Crypto Service | DOC-005, CRYPTO-004 | SEC-MEM-001: DEK no persistida/logueada; limpieza best effort documentada |
| RF-055 | Crypto Service + Mongo | DOC-005, DOC-008, CRYPTO-004, CRYPTO-008 | IT-NONCE-001: unicidad por DEK bajo concurrencia |
| RF-056 | Service security | DOC-005, DOC-009, HSM-002, CRYPTO-001 | IT-MTLS-001: identidades permitidas/denegadas |
| RF-057 | HSM Simulator | DOC-005, HSM-001..HSM-004 | CT-HSM-001: generate/wrap/unwrap/version |
| RF-058 | HSM Simulator | DOC-005, HSM-005 | UT-PIN-SIM-001: fixtures sintéticos, sin PIN claro de salida |
| RF-059 | HSM Simulator | DOC-005, HSM-001 | ARCH-HSM-001: sin acceso a Mongo ni datos de transacción |
| RF-060 | Crypto Service + HSM | DOC-005, DOC-006, CRYPTO-001..005 | IT-CRYPTO-FAIL-001: fail closed, sin fallback |
| RF-070 | Gateway + Backend | DOC-003, DOC-004, DOC-007, DB-003, BE-002..004 | IT-IDEM-001: autorización/financiero/reverso idempotentes |
| RF-071 | Gateway / idempotency key builder | DOC-003, DOC-007, DB-003 | UT-IDEM-001: identidad compuesta y fingerprint |
| RF-072 | Backend Simulator | DOC-004, BE-002..006 | CT-BE-004: idempotency y consulta status |
| RF-073 | Gateway / uncertainty + recovery | DOC-004, DOC-006, DOC-007, REC-001 | IT-UNC-002: no retry ciego; consulta previa |
| RF-074 | Reconciliation worker | DOC-004, DOC-006, REC-001..005 | IT-REC-002: indeterminado resuelto con evidencia backend |
| RF-075 | Observability | DOC-009, OBS-001..004 | SEC-LEAK-003 + OBS-001: campos saneados/cardinalidad |
| RF-076 | Audit events + HSM | DOC-005, DOC-009, DB-006, HSM-004 | IT-AUDIT-001: eventos sin material secreto/payload |

## 3. Matriz NFR

| NFR | Componentes | Tareas | Evidencia |
|---|---|---|---|
| NFR-001 Rendimiento | Gateway, Backend, Crypto, Mongo, observabilidad | OBS-002, QA-007 | Benchmark reproducible por etapa; sin SLO bancario |
| NFR-002 Concurrencia | Mongo, Crypto, idempotencia, reversos | DOC-008, DB-003, CRYPTO-008, REV-003, QA-003 | Pruebas multi-instancia y carreras |
| NFR-003 Durabilidad | Mongo, Crypto, Gateway | DOC-006, DOC-008, DB-001, CRYPTO-005, QA-004 | Reinicio/fallo y write concern verificados |
| NFR-004 Resiliencia | Gateway, Backend, Crypto, HSM, recovery | DOC-004..DOC-007, REC-001..005 | Fault injection y estados coherentes |
| NFR-005 Seguridad | TLS/mTLS, secretos, autorización | DOC-005, DOC-009, BOOT-006, HSM-002, SEC-003 | Certificados válidos/revocados y acceso mínimo |
| NFR-006 Confidencialidad | Sanitizer, Crypto, observabilidad | DOC-009, SEC-001, SEC-002, OBS-001..003 | Escaneo de fuga automatizado |
| NFR-007 Recuperabilidad | Mongo + key store HSM | DOC-005, DOC-008, REC-006 | Restore conjunto de datos y claves |
| NFR-008 Mantenibilidad | Límites hexagonales y contratos | DOC-001..DOC-005, BOOT-002 | Revisión de dependencias/arquitectura |
| NFR-009 Calidad | Build/test/static analysis | DOC-001, BOOT-004, QA-001..QA-006 | Quality Gate con versiones y umbrales fijados |
| NFR-010 Operabilidad | Health, metrics, logs, traces | DOC-009, BOOT-007, OBS-001..004 | Smoke tests y revisión de telemetría |

## 4. Reglas de trazabilidad

1. Todo RF/NFR debe tener componente, tarea y evidencia asignados.
2. Si una tarea cambia de ID, actualizar esta matriz y TASKS en el mismo cambio documental.
3. La existencia de una prueba propuesta no implica que esté implementada ni ejecutada.
4. `0420/0430`, clearing/settlement, DE52 persistido y conectividad de redes reales quedan fuera del baseline y no deben aparecer como criterios de aceptación ejecutables.
5. QA-001 debe transformar esta matriz en trazabilidad requisito → test automatizado → evidencia real durante la implementación.

## 5. Resultado

La trazabilidad documental queda definida para el baseline RF-001..RF-076 y NFR-001..NFR-010. La evidencia de pruebas permanece pendiente hasta que las tareas de implementación correspondientes sean autorizadas y ejecutadas.
