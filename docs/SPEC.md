# SPEC — ISO8583 Gateway Lab

**Versión:** 2.0  
**Estado:** v2.0 aprobado documentalmente por el usuario, incluidos los contratos DOC-001..DOC-009  
**Base:** arquitectura integral v2.0 aprobada por el usuario; SPEC/PLAN/TASKS v1.0 como baseline histórico.  
**Código/Bootstrap:** requieren autorización explícita de los IDs BOOT-* correspondientes; la aprobación documental no autoriza su ejecución.  
**Perfil:** laboratorio ISO 8583:1987; no certificado ni compatible por defecto con Link/Prisma.

## 1. Objetivo

Definir un gateway transaccional ISO8583 moderno que procese autorización, operación financiera y reverso; conserve estado durable; gestione mensajes documentales cifrados; integre un HSM Simulator independiente; y soporte recuperación, concurrencia y observabilidad. La experiencia histórica sirve como conocimiento del dominio, no como restricción tecnológica.

## 2. Alcance funcional

### Incluido en la línea base v2.0

- `0100/0110`: autorización.
- `0200/0210`: operación financiera.
- `0400/0410`: solicitud/respuesta de reverso.
- Consulta/reconciliación operacional de operaciones indeterminadas contra el backend simulado.
- Persistencia MongoDB durable del estado operacional y mensajes cifrados/saneados.
- Servicio criptográfico independiente, lotes híbridos y Envelope Encryption.
- HSM Simulator independiente, con operaciones de PIN únicamente sobre fixtures sintéticos.
- Recuperación tras reinicios/fallos, múltiples instancias del Cryptographic Service, mTLS y observabilidad.

### Fuera de alcance de la línea base

- `0420/0430`: aviso de reverso. Se conserva como extensión identificada; no se asume equivalente a `0400/0410` ni se crean tareas de implementación para este par sin decisión de alcance específica.
- `0800/0810` y administración de red ISO.
- Clearing, settlement, liquidación, conciliación contable y archivos propietarios de redes.
- Integración real/certificada con Red Link o Prisma.
- HSM físico, certificación FIPS, PCI PIN o declaración de cumplimiento PCI DSS.
- Despliegue Azure/Kubernetes en la primera entrega; se deja como fase posterior.
- Retención de DE52/SAD en la configuración ejecutable inicial. La excepción PCI DSS 3.3.3 queda documentada como asunto condicionado a validación externa formal; no se activa por el mero hecho de cifrar la trama.

## 3. Stack y restricciones técnicas

- Java 21, Spring Boot 3.x, Maven.
- Netty para TCP/TLS y framing; jPOS para pack/unpack ISO8583.
- Spring WebClient para llamadas HTTP no bloqueantes.
- MongoDB como única base de datos del Gateway; replica set requerido incluso en laboratorio (un nodo local no implica HA).
- HSM Simulator separado, Java 21/Spring Boot 3.x, API interna REST/mTLS.
- Docker Compose para entorno local.
- JUnit 5, Mockito y Testcontainers donde aporten valor.
- Arquitectura hexagonal pragmática; dominio sin dependencias de Spring, Netty, jPOS, Mongo driver ni DTO HTTP.
- No bloquear EventLoops con I/O de backend, criptografía, HSM o persistencia.
- No agregar PostgreSQL ni otra base transaccional al Gateway.

Las versiones concretas de dependencias se fijarán en Bootstrap autorizado, comprobando compatibilidad y evitando versiones flotantes.

## 4. Requisitos funcionales

### Transporte y protocolo

- **RF-001:** aceptar conexiones TCP protegidas por TLS con configuración externa de certificados y límites.
- **RF-002:** reconstruir frames desde TCP stream soportando fragmentación, concatenación, frames incompletos y longitudes inválidas.
- **RF-003:** encapsular framing detrás de `FrameCodec`. El prefijo de 2 bytes big-endian de v1.0 solo se conserva si se confirma en el perfil de laboratorio; no atribuirlo a Link/Prisma.
- **RF-004:** parsear y empaquetar el perfil ISO 8583:1987 de laboratorio con jPOS.
- **RF-005:** validar MTI, bitmap, presencia, longitud, tipo y reglas de campos por MTI.
- **RF-006:** separar errores de framing, parseo, validación, negocio, backend, persistencia y criptografía.
- **RF-007:** responder por el canal que recibió el request correspondiente; no acoplar la operación a la conexión TCP original.

### Autorización y operación financiera

- **RF-010:** recibir `0100`, validar perfil y campos, identificar origen y reservar idempotentemente la solicitud.
- **RF-011:** persistir operación/estado antes de invocar dependencias externas, según el orden transaccional definido en PLAN.
- **RF-012:** invocar backend simulado para autorización y producir `0110` a partir de su resultado.
- **RF-013:** si el perfil de laboratorio requiere correlación entre autorización y `0200`, emitir token aleatorio de 128 bits en DE48 de `0110` aprobado y validarlo en `0200`. DE48 es convención privada de laboratorio, no universal.
- **RF-014:** recibir `0200`, resolver operación asociada y validar origen, importe, moneda, estado y referencias del perfil.
- **RF-015:** invocar backend simulado de procesamiento financiero y producir `0210`.
- **RF-016:** impedir ejecución financiera cuando autorización esté rechazada, expirada, indeterminada o no exista.

### Reversos

- **RF-020:** recibir `0400`, validar referencias explícitas del perfil y localizar la operación original sin heurísticas.
- **RF-021:** reservar idempotentemente el reverso y evitar doble reversión bajo concurrencia.
- **RF-022:** invocar al backend simulado para ejecutar/consultar la reversa; la reversa financiera efectiva pertenece al backend/core simulado.
- **RF-023:** generar `0410` conforme al resultado y perfil configurado.
- **RF-024:** distinguir operación original inexistente, pendiente, aprobada/procesada, rechazada, ya revertida e indeterminada.
- **RF-025:** no afirmar reversa completada ni generar rechazo artificial cuando el resultado original o el reverso sean indeterminados.
- **RF-026:** tratar retransmisiones de `0400` idempotentemente.

### Persistencia, cifrado y recuperación

- **RF-030:** usar MongoDB como fuente de verdad única del Gateway para operaciones, idempotencia, correlación, lotes, metadatos criptográficos, mensajes y auditoría operacional.
- **RF-031:** desplegar MongoDB como replica set; preferir actualizaciones atómicas, índices únicos y transacciones breves solo cuando se requiera atomicidad multi-documento.
- **RF-032:** no mantener transacciones MongoDB abiertas durante llamadas remotas.
- **RF-033:** persistir un documento por mensaje con identificadores, dirección, MTI, perfil, timestamps, estado de archivo y representación parseada saneada; los campos PAN del parseo deben omitirse o protegerse con cifrado de campo, nunca quedar en claro.
- **RF-034:** cifrar documentalmente los bytes retenibles mediante AES-256-GCM (nonce 96 bits, tag 128 bits) y AAD contextual; los bytes se serializan en Base64 en JSON.
- **RF-035:** persistir `ciphertext`, nonce, tag, `batchId`, `wrappedDek`, `keyId`/versión y metadata requerida; nunca persistir DEK/KEK en claro.
- **RF-036:** no persistir DE52 ni PIN block en la configuración inicial. Trama retenida debe estar saneada si contiene SAD excluido. No persistir CVV/CVC después de autorización.
- **RF-037:** proteger PAN como CHD: la representación parseada no puede contener PAN en claro (omitirlo o cifrarlo a nivel de campo); no exponer PAN completo en logs, métricas, trazas ni excepciones.
- **RF-038:** calcular SHA-256 únicamente sobre los bytes exactos de la representación saneada/cifrada que el contrato documental identifique; no hashear la trama original que contenga DE52 excluido. Registrar qué bytes fueron hasheados.
- **RF-039:** permitir recuperar documentos desde cualquier instancia autorizada mediante `wrappedDek` y HSM Simulator.
- **RF-040:** recuperar reservas abandonadas y operaciones pendientes a partir de estado durable, sin depender de memoria local.

### Cryptographic Service y HSM Simulator

- **RF-050:** ejecutar Cryptographic Service como servicio independiente, escalable horizontalmente.
- **RF-051:** coordinar lotes en MongoDB con reserva atómica de capacidad y ciclo de vida `OPEN`, `CLOSING`, `CLOSED`, `FAILED`.
- **RF-052:** usar política híbrida: cerrar lote al alcanzar 1.000 mensajes reservados o 15 minutos desde apertura, lo que ocurra primero.
- **RF-053:** usar una DEK por lote; el HSM Simulator genera y protege DEK; la KEK permanece bajo custodia del simulador.
- **RF-054:** mantener DEK en claro solo transitoriamente en memoria del Cryptographic Service; eliminar referencias/bytes de uso al terminar cuando sea practicable en Java.
- **RF-055:** garantizar nonce GCM único por DEK; reintentos de cifrado usan nonce nuevo.
- **RF-056:** autenticar llamadas internas indicadas mediante mTLS y autorizar por identidad de servicio.
- **RF-057:** HSM Simulator ofrece generación/wrapping/unwrapping de DEK, metadata/versionado/rotación y auditoría sin secretos.
- **RF-058:** incluir operaciones PIN simuladas limitadas a fixtures sintéticos: validación de formato, verificación o traducción de prueba; nunca devolver PIN en claro.
- **RF-059:** HSM Simulator no almacena mensajes, PAN, PIN block ni accede a MongoDB.
- **RF-060:** fallos de HSM/cifrado no permiten fallback a KEK local ni persistir contenido sensible como si estuviera cifrado.

### Idempotencia, reconciliación y observabilidad

- **RF-070:** usar idempotencia para autorización, operación financiera y reverso; STAN solo nunca es suficiente.
- **RF-071:** construir identidad de retransmisión con origen, MTI, STAN, DE7/referencias aplicables y huella del contenido normalizado, según perfil versionado.
- **RF-072:** backend simulado debe aceptar claves idempotentes estables y exponer consulta de estado para operaciones indeterminadas.
- **RF-073:** timeout con resultado desconocido produce `INDETERMINATE`; no reintentar automáticamente operaciones financieras sin contrato idempotente y consulta previa.
- **RF-074:** proveer reconciliación operacional acotada para resolver discrepancias/indeterminados contra backend simulado; no realizar clearing/settlement.
- **RF-075:** emitir logs estructurados, métricas y trazas con cardinalidad controlada y datos saneados.
- **RF-076:** auditar operaciones de clave y transiciones sensibles sin material secreto ni payload ISO.

## 5. Requisitos no funcionales

- **NFR-001 Rendimiento:** medir latencia por etapa y throughput del laboratorio; no afirmar capacidad bancaria sin benchmark representativo.
- **NFR-002 Concurrencia:** soportar varias operaciones por origen y varias instancias de Cryptographic Service sin duplicar reservas ni exceder límites.
- **NFR-003 Durabilidad:** no confirmar como durable un mensaje/estado antes del write concern definido; documentar el perfil de consistencia.
- **NFR-004 Resiliencia:** timeouts independientes por dependencia, circuit breaker cuando aporte valor y tratamiento explícito de indeterminados.
- **NFR-005 Seguridad:** TLS, mTLS interno, secretos externos al repositorio, mínimo privilegio y separación de responsabilidades.
- **NFR-006 Confidencialidad:** PAN minimizado/protegido; PIN block, DEK, KEK y payload sensible ausentes de telemetría.
- **NFR-007 Recuperabilidad:** backups de MongoDB y claves del simulador por procedimientos separados; probar restauración conjunta.
- **NFR-008 Mantenibilidad:** límites hexagonales y contratos explícitos; evitar abstracciones no justificadas.
- **NFR-009 Calidad:** pruebas automatizadas de dominio, integración, concurrencia, fallos y seguridad; Quality Gate acordado.
- **NFR-010 Operabilidad:** health/readiness, métricas de lotes, crypto, backend, Mongo y transporte sin exponer datos sensibles.

## 6. Modelo de dominio y estados

### Estados de operación

`AUTHORIZATION_PENDING`, `AUTHORIZED`, `DECLINED`, `FINANCIAL_PENDING`, `COMPLETED`, `FAILED`, `INDETERMINATE`, `REVERSAL_PENDING`, `REVERSED`, `REVERSAL_DECLINED`, `INCOMPLETE`, `EXPIRED`.

Las transiciones permitidas deben modelarse explícitamente por caso de uso. `REVERSAL_DECLINED` se usa solo si el contrato del backend/perfil distingue el rechazo de reverso del estado original. No permitir regresiones ni transiciones basadas solo en timeout.

### Estados de lote

`OPEN` → `CLOSING` → `CLOSED`; fallo controlado a `FAILED`. `CLOSED` y `FAILED` no aceptan reservas nuevas. Las reservas admitidas antes de `CLOSING` deben resolverse o marcarse recuperables.

## 7. Contratos y perfil ISO

Los contratos de laboratorio se definen en `docs/contracts/`:

- DOC-002: perfil `LAB-ATM-1.0`, framing, bitmap, codificación, DEs, MTIs y DE39.
- DOC-003: OriginId, token DE48, fingerprint idempotente y referencia exacta de reverso por DE90 + DE37 + origen + importe/moneda/proceso.
- DOC-004: Backend Simulator REST `backend-api-v1`.
- DOC-005: Cryptographic Service `crypto-api-v1` y HSM Simulator `hsm-api-v1`.
- DOC-006: punto de durabilidad antes de efectos y respuestas.
- DOC-007: timeouts, ventana idempotente y retenciones de laboratorio.
- DOC-008: concerns, colecciones, índices, CAS, leases y recuperación Mongo.
- DOC-009: exclusiones, clasificación, mTLS y límites de seguridad.

Todos son contratos privados del laboratorio. No inferir framing, packager, codificación ni reglas de Link/Prisma.

## 8. Seguridad y cumplimiento

El laboratorio no se declara PCI DSS ni PCI PIN compliant. MongoDB no es un HSM ni un vault por sí mismo. La custodia KEK pertenece al HSM Simulator; las claves de raíz del simulador requieren protección externa al volumen de datos. La retención de SAD solo podría evaluarse bajo la excepción de emisor/servicio de emisión y con necesidad legítima documentada y validación QSA/marcas. Esta evaluación no habilita su persistencia en el baseline ejecutable.

## 9. Criterios globales de aceptación

1. Flujos `0100/0110`, `0200/0210`, `0400/0410` pasan pruebas positivas, negativas y concurrentes.
2. Frames fragmentados/concatenados se reconstruyen correctamente; entradas inválidas no degradan el servicio.
3. Duplicados concurrentes no generan dobles efectos financieros ni reversos.
4. Estado sobrevive reinicio de Gateway/Crypto Service y permite recuperación controlada.
5. MongoDB replica set soporta índices/atomicidad/actualizaciones condicionales verificadas.
6. Ningún EventLoop ejecuta I/O bloqueante.
7. Cifrado/descifrado autentica correctamente; modificación de ciphertext, tag o AAD falla.
8. No se persisten DE52/PIN block en baseline; escaneo de logs/trazas/errores no encuentra PAN completo, PIN block ni secretos.
9. Lotes nunca exceden 1.000 reservas ni 15 minutos; concurrencia multi-instancia no duplica asignaciones ni reutiliza nonce.
10. HSM Simulator no expone KEK/DEK persistente en claro ni PIN en claro.
11. Recuperación de backup de datos + claves permite descifrar documentos de prueba.
12. Reconciliación resuelve estados indeterminados sin reejecución ciega.
13. Quality Gate y pruebas críticas pasan con evidencia registrada.

## 10. Decisiones de baseline y gates restantes

- `0420/0430` queda fuera del baseline inicial y reservado como extensión; incorporarlo requiere nueva decisión de alcance.
- Retención de DE52 permanece deshabilitada; la excepción PCI DSS 3.3.3 no está habilitada y requiere validación externa formal.
- El perfil `LAB-ATM-1.0` es deliberadamente privado y no afirma compatibilidad de red.
- Los defaults de timeout/retención son configurables de laboratorio, no valores productivos.
- La semántica de durabilidad, idempotencia y MongoDB se detalla en DOC-006..DOC-008.
- Gate técnico pendiente: comprobar durante Bootstrap que el stack TLS elegido soporta CRL/revocación de certificados conforme DOC-009; si no, detenerse y solicitar decisión.
- Gate pendiente de Bootstrap: fijar versiones exactas de dependencias, Quality Gate y umbrales en BOOT-001/BOOT-004.
- Aprobación documental de SPEC/PLAN/TASKS y contratos DOC-001..DOC-009 recibida. Antes de ejecutar Bootstrap, verificar CRL en el stack TLS y obtener autorización explícita para los IDs BOOT-* concretos.
