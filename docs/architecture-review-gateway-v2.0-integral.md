# Revisión arquitectónica integral v2.0 — ISO8583 Gateway Lab

**Estado:** arquitectura v2.0 aprobada documentalmente por el usuario.  
**Versión:** 2.0-integral  
**Base:** decisiones arquitectónicas acordadas, SPEC/PLAN/TASKS v2.0 y contratos DOC-001..DOC-009 aprobados documentalmente.  
**Implementación:** autorizada exclusivamente para BOOT-001..BOOT-007, conforme a sus dependencias y criterios de salida. Las épicas funcionales posteriores requieren autorización independiente.  
**Propósito:** definir la arquitectura objetivo del laboratorio bancario moderno, operable y verificable, sin afirmar certificación ni compatibilidad con redes productivas.

---

## 1. Resumen ejecutivo

La v1.0 definía un laboratorio acotado para autorización y procesamiento financiero (`0100/0110`, `0200/0210`), con estado en memoria, backend REST simulado y reversos fuera de alcance. La evolución v2.0 busca representar un gateway transaccional moderno que procese autorizaciones, operaciones financieras y reversos, con persistencia durable, recuperación, seguridad de pagos, integración HSM y operación concurrente.

La arquitectura objetivo propuesta contiene cuatro responsabilidades principales:

1. **ISO8583 Gateway:** transporte TCP/TLS, framing, parseo/validación ISO 8583, correlación, idempotencia y orquestación de casos de uso.
2. **Cryptographic Service escalable:** cifrado/descifrado documental mediante Envelope Encryption, gestión de lotes criptográficos y persistencia del mensaje cifrado.
3. **HSM Simulator independiente:** servicio criptográfico externo que protege claves de datos y simula operaciones de PIN con material sintético. No almacena transacciones ni accede a MongoDB.
4. **Persistencia:** MongoDB será la única base de datos del gateway y almacenará tanto el estado operacional como los documentos de mensajes cifrados. El HSM Simulator conservará sus propias claves en un almacenamiento separado; no se considera una segunda base de datos transaccional del gateway.

La arquitectura se diseñará para múltiples instancias del Cryptographic Service. El laboratorio podrá ejecutar dos instancias en Docker Compose; no se requiere Kubernetes para demostrar concurrencia horizontal.

**Recomendación central:** mantener la complejidad en los límites que aportan valor demostrable —estado transaccional, reversos, cifrado, recuperación y concurrencia— y evitar construir infraestructura empresarial ajena al objetivo. El simulador no es un HSM físico ni una implementación certificada.

## 2. Objetivo y contexto

### 2.1 Objetivo

Construir un prototipo de gateway ISO 8583 moderno, defendible en una revisión técnica de arquitectura bancaria y útil para aprender y demostrar:

- procesamiento de mensajes ISO 8583;
- autorización, operación financiera y reverso;
- integración con backend bancario simulado;
- gestión criptográfica mediante HSM Simulator;
- cifrado de mensajes y persistencia durable;
- idempotencia y tratamiento de resultados indeterminados;
- concurrencia entre instancias;
- recuperación ante reinicios y fallos;
- observabilidad, seguridad y pruebas operativas.

### 2.2 Uso de la experiencia histórica

La experiencia anterior del usuario aporta escenarios funcionales reales: integración con Red Link, interacción con HSM, llamadas síncronas al core, mensajes `0400` por timeout, persistencia de mensajes y regularización bancaria externa. Esa experiencia se utiliza como conocimiento del dominio, no como restricción tecnológica.

No se reproducen Java 1.4, Apache Ant, el monolito histórico, la creación de un thread por transacción ni los circuitos batch/FTP internos del banco.

### 2.3 Requisitos derivados de la oportunidad laboral

La descripción del puesto menciona ISO 8583, MTI/bitmap/DE, autorización, reversos y conciliación, redes argentinas, baja latencia, resiliencia, alta disponibilidad, HSM, PIN block, PAN, PCI DSS, bases relacionales, Docker, Azure, pruebas y observabilidad. Estos puntos orientan el prototipo, pero no aportan las especificaciones privadas de Link o Prisma ni sus reglas operativas exactas.

## 3. Principios arquitectónicos

1. Java 21 y Spring Boot 3 como base de implementación.
2. Arquitectura hexagonal pragmática; DDD donde aclare el dominio, sin capas ceremoniales.
3. Separar protocolo, aplicación, dominio, persistencia y adaptadores externos.
4. No bloquear EventLoops de Netty con llamadas REST, HSM o base de datos.
5. Persistencia durable para operaciones y evidencia documental.
6. Idempotencia explícita para requests, operaciones financieras y reversos.
7. Un timeout no equivale a rechazo: puede producir resultado `INDETERMINATE`.
8. Ningún reintento automático de una operación financiera no idempotente sin contrato del backend.
9. Datos sensibles minimizados y nunca expuestos en logs, trazas o métricas.
10. HSM representado por un puerto; simulador inicial reemplazable por adaptador de fabricante.
11. No afirmar compatibilidad Link/Prisma sin perfil técnico oficial y pruebas de certificación.
12. Cada control de seguridad debe tener una amenaza, requisito o criterio de aceptación asociado.
13. La implementación requiere aprobación explícita del alcance y de las tareas. La autorización vigente cubre únicamente BOOT-001..BOOT-007; las épicas funcionales posteriores requieren autorización independiente.

## 4. Alcance funcional

### 4.1 Autorización: `0100/0110`

Flujo candidato:

1. Recibir trama y validar framing, tamaño, MTI y perfil.
2. Parsear y validar los campos obligatorios.
3. Identificar origen y construir clave de idempotencia.
4. Crear una operación durable en estado de autorización pendiente.
5. Enviar una solicitud al backend bancario simulado.
6. Interpretar resultado definitivo o indeterminado.
7. Construir `0110` según el perfil de laboratorio.
8. Persistir mensaje recibido, respuesta generada y transición de estado.

El token privado de correlación en DE48 de v1.0 puede mantenerse como convención del perfil de laboratorio si sigue siendo necesario para vincular autorización y operación financiera. No es una convención universal ISO ni se atribuye a Link.

### 4.2 Operación financiera: `0200/0210`

1. Recibir y validar `0200`.
2. Resolver la autorización/operación asociada mediante referencias explícitas del perfil.
3. Verificar origen, secuencia, importe, moneda y estado.
4. Reservar idempotentemente la ejecución financiera.
5. Invocar el backend simulado.
6. Persistir el resultado y producir `0210`.
7. Ante timeout con resultado desconocido, registrar `INDETERMINATE`; consultar/reconciliar antes de permitir una nueva ejecución.

El gateway no debe inferir una operación previa únicamente por importe, terminal o proximidad temporal.

### 4.3 Reverso request/response: `0400/0410`

El `0400` representa una solicitud de reverso en el perfil del laboratorio. Debe correlacionarse con la operación original mediante referencias explícitas definidas en el perfil, no mediante heurísticas.

El backend simulado debe distinguir al menos:

- operación original inexistente;
- operación original pendiente;
- operación original aprobada/procesada;
- operación ya revertida;
- operación rechazada;
- resultado original indeterminado;
- reverso ya solicitado o duplicado.

Reglas propuestas:

- Un reverso duplicado debe ser idempotente.
- No se debe revertir una operación distinta por coincidencia parcial.
- La reversa financiera efectiva corresponde al backend/core simulado, no al gateway.
- El `0410` se construye a partir del resultado del backend y del perfil ISO configurado.
- Si el resultado original es indeterminado, el gateway no debe inventar un rechazo ni afirmar que la reversa se completó; debe seguir una política explícita de consulta/estado pendiente.

El escenario histórico en que un `0400` sin `0210` se guardaba para regularización manual es un antecedente, no una regla automática del nuevo sistema. El laboratorio debe representar el estado indeterminado y un mecanismo de consulta/reconciliación, sin reproducir el proceso nocturno/FTP del banco.

### 4.4 Aviso de reverso: `0420/0430` — candidato

La descripción laboral menciona `0400/0420`; la experiencia histórica del usuario recuerda `0400/0410`. No deben tratarse como MTI intercambiables.

Se propone evaluar `0420/0430` como flujo separado de aviso de reverso, condicionado a definir su semántica exacta en el perfil de laboratorio. No se implementará como parte del mismo caso de uso que `0400/0410` por simple equivalencia numérica.

**Pendiente:** confirmar si el laboratorio debe soportar ambos pares en la primera versión funcional o si `0420/0430` queda como extensión posterior. Las reglas concretas de redes pueden variar.

### 4.5 Conciliación y clearing

Se incluye una capacidad acotada de consulta/reconciliación operacional para resolver operaciones indeterminadas y detectar discrepancias entre gateway y backend simulado.

Quedan fuera de alcance inicial:

- clearing/settlement bancario completo;
- cálculo de posiciones y liquidación entre participantes;
- archivos de intercambio propietarios de Link/Prisma;
- procesos batch/FTP históricos del banco;
- regularización contable real.

## 5. Arquitectura lógica objetivo

```text
                    Red / Cliente ISO de laboratorio
                                  |
                                TCP/TLS
                                  |
                         ISO8583 Gateway (N)
                    Netty + FrameCodec + jPOS
                                  |
                  Validación / Casos de uso / Correlación
                         /                 \
                        /                   \
              Backend Adapter          Crypto Client
                    |                       |
          Backend/Core Simulator     Cryptographic Service (N)
                    |                 Batch + Envelope Encryption
                    |                       |
                    |                  HSM Adapter (REST/mTLS)
                    |                       |
                    |                 HSM Simulator (1..N)
                    |                       |
                    +-----------+-----------+
                                |
                   MongoDB (única base del Gateway)
          estado operacional + lotes + mensajes cifrados
                                |
                    Logs / métricas / trazas
```

### 5.1 Componentes y responsabilidades

| Componente | Responsabilidad | No debe hacer |
|---|---|---|
| ISO8583 Gateway | TCP/TLS, framing, parseo, validación, correlación, orquestación y respuesta ISO | Cifrado documental interno ni custodia KEK |
| Backend/Core Simulator | Simular autorización, operación financiera, reverso y consulta de estado | Parsear TCP/ISO del emisor |
| Cryptographic Service | Gestionar lotes, cifrar/descifrar documentos, coordinar persistencia y HSM | Decidir autorizaciones financieras |
| HSM Simulator | Proteger claves, wrapping/unwrapping y simular operaciones PIN | Almacenar mensajes o acceder a MongoDB |
| MongoDB | Fuente de verdad del gateway: estado operacional, idempotencia, correlación, lotes, metadatos criptográficos, mensajes cifrados y auditoría operacional | Ser considerado un HSM o bóveda segura por sí mismo; sustituir controles de seguridad de aplicación |
| Observabilidad | Logs técnicos, métricas y trazas saneadas | Capturar payloads ISO o secretos |

### 5.2 Estilo de despliegue

Docker Compose reproducirá el entorno. La topología de laboratorio inicial podrá contener:

- una instancia del Gateway;
- dos instancias del Cryptographic Service;
- una instancia del HSM Simulator;
- MongoDB configurado como replica set de un nodo para probar transacciones multi-documento;
- backend/core simulado.

El diseño debe tolerar más de una instancia de Cryptographic Service. La ejecución inicial con una instancia no debe introducir supuestos incompatibles con concurrencia horizontal.

No se requiere Kubernetes para la primera implementación. Azure queda como etapa posterior de despliegue.

## 6. Persistencia y modelo de datos

### 6.1 MongoDB como única base de datos del Gateway

**Decisión del usuario: MongoDB gestionará todas las necesidades de persistencia del Gateway. PostgreSQL queda fuera de la arquitectura.**

MongoDB será la fuente de verdad para:

- estado de operaciones y sus transiciones;
- claves de idempotencia y deduplicación;
- correlación entre autorización, operación financiera y reverso;
- reservas, estado y ciclo de vida de lotes criptográficos;
- metadata de DEK protegidas (`wrappedDek`) y referencia a KEK/versiones;
- mensajes ISO8583 cifrados y sus metadatos;
- auditoría operacional de eventos relevantes.

El HSM Simulator mantiene su almacenamiento de claves independiente. Ese almacenamiento no contiene operaciones ni mensajes del Gateway y no convierte al HSM en una base de datos de transacciones.

**Topología requerida:** MongoDB se ejecutará como replica set, incluso en el laboratorio local, para habilitar transacciones multi-documento cuando sean necesarias y probar correctamente el comportamiento de concurrencia y recuperación. El despliegue inicial puede ser un replica set de un solo nodo; esto no representa alta disponibilidad. La evolución a varios nodos queda como etapa operativa posterior.

**Estrategia de consistencia:**

1. Preferir actualizaciones atómicas sobre un único documento cuando el agregado lo permita.
2. Utilizar índices únicos para claves de idempotencia, identificadores de mensaje, identificadores de lote y restricciones de nonce dentro del lote.
3. Utilizar transacciones MongoDB para cambios que deban confirmar conjuntamente varios documentos, por ejemplo reserva de capacidad y registro de una asignación cuando no puedan modelarse como una única actualización atómica.
4. No mantener transacciones abiertas durante llamadas al backend, Cryptographic Service o HSM.
5. Aplicar actualizaciones condicionales por estado/versión para evitar que instancias concurrentes sobrescriban transiciones válidas.
6. Definir política de write concern y read concern coherente con los requisitos de durabilidad; documentar el perfil usado en el laboratorio.
7. No introducir locks distribuidos propios si los mecanismos atómicos, índices y transacciones de MongoDB resuelven el caso.

La decisión reduce la cantidad de tecnologías y elimina la necesidad de consistencia distribuida entre PostgreSQL y MongoDB. A cambio, exige conocer los límites de las transacciones MongoDB, modelar correctamente los agregados y probar el comportamiento del replica set ante reinicios y fallos.

### 6.1.1 Coordinación y persistencia sin doble escritura

Como MongoDB será el único almacén del Gateway, no se necesita un patrón Outbox para sincronizar dos bases de datos. Los cambios de estado y los documentos relacionados podrán confirmarse en una misma transacción MongoDB cuando la atomicidad multi-documento sea necesaria.

Se evitará usar transacciones para operaciones largas o llamadas remotas. El flujo será: reservar/registrar estado de forma durable, ejecutar la operación externa fuera de la transacción, y confirmar el resultado mediante una nueva actualización condicional. Si el proceso cae entre etapas, el estado persistido permitirá recuperar o reconciliar la operación.

La respuesta financiera y el estado de archivo documental seguirán siendo conceptos distintos, aunque ambos estén en MongoDB. La política para responder al emisor deberá especificar qué evidencia debe estar durable antes de emitir la respuesta y qué hacer si el archivo cifrado no pudo persistirse.

### 6.2 Mensaje documental

Un documento por mensaje recibido o generado. Debe incluir como mínimo:

- `messageId`, `operationId` cuando aplique, `batchId`;
- dirección (`INBOUND`/`OUTBOUND`), MTI y perfil;
- timestamp de recepción/generación;
- trama original cifrada como bytes serializados en Base64;
- nonce, tag, algoritmo y versión de esquema criptográfico;
- referencia a DEK protegida del lote, sin clave en claro (la metadata y `wrappedDek` residen en MongoDB);
- metadatos parseados no sensibles;
- bitmap recibido y metadatos del parser;
- digest de evidencia segura cuando se justifique.

El JSON parseado no debe duplicar PAN o PIN block en claro. La trama original cifrada puede conservar los bytes originales únicamente si la política de retención de datos sensibles y la aplicabilidad de PCI DSS lo permiten.

### 6.3 PAN y PIN block

- PAN es CHD y debe protegerse de acuerdo con los requisitos aplicables, con minimización y controles de acceso.
- PIN block es SAD. No debe asumirse que cifrar toda la trama habilita su almacenamiento posterior a la autorización.
- El alcance v1.0 del HSM Simulator incluirá operaciones PIN simuladas con datos sintéticos, pero no expondrá PIN en claro.
- La conservación de PIN block será un perfil opcional, deshabilitado por defecto, condicionado a demostrar que el escenario es de emisor o entidad que soporta servicios de emisión, que existe necesidad legítima y documentada, y que QSA/marcas validan la aplicabilidad de la excepción PCI DSS 3.3.3.
- Si no se confirma esa aplicabilidad, DE52 se excluye de la persistencia y de toda telemetría. La trama persistida será saneada, no byte a byte idéntica a la recibida.
- No almacenar CVV/CVC después de la autorización.

El diseño del laboratorio no equivale a certificación PCI DSS ni PCI PIN Security. El cifrado no elimina por sí solo el alcance PCI del entorno.

## 7. Diseño criptográfico y HSM Simulator

### 7.1 Envelope Encryption

Se propone el siguiente esquema:

1. El Cryptographic Service solicita al HSM Simulator una DEK para un lote.
2. El HSM Simulator genera la DEK AES-256 y devuelve: DEK en claro para uso transitorio y copia protegida (`wrappedDek`). La KEK nunca sale del simulador.
3. El Cryptographic Service cifra cada mensaje con AES-256-GCM usando la DEK del lote.
4. MongoDB almacena ciphertext, nonce, tag y referencia a la DEK protegida del lote.
5. Para descifrar, un servicio autorizado recupera `wrappedDek`, solicita unwrap al HSM y usa la DEK temporal en memoria.
6. La DEK en claro no se persiste ni se registra; se mantiene solo el tiempo estrictamente necesario.

La KEK del HSM Simulator se mantiene en su almacenamiento de claves y protegida por un secreto raíz externo al volumen. Esto simula el límite de custodia, pero no aporta la protección física o certificación de un HSM comercial.

### 7.2 Algoritmos y parámetros candidatos

- Cifrado de documento: AES-256-GCM.
- Nonce GCM: 96 bits, único por cada cifrado bajo la misma DEK.
- Tag de autenticación: 128 bits.
- Protección de DEK: AES Key Wrap (RFC 3394) o mecanismo equivalente soportado por el adaptador real.
- Aleatoriedad: `SecureRandom` o generador criptográfico del HSM para las claves.
- AAD: incluir identificadores no secretos como `batchId`, `messageId`, tipo de documento y versión de esquema para vincular ciphertext con su contexto.

No se debe reutilizar nonce con una misma DEK. La política concreta de generación y detección de colisiones se especificará y probará antes de implementar.

### 7.3 HSM Simulator v1.0

Servicio independiente, Java 21 / Spring Boot 3, expuesto únicamente en red interna y autenticado mediante mTLS.

Responsabilidades:

- crear y versionar KEK;
- generar DEK por lote y devolver DEK temporal + DEK protegida;
- unwrap de DEK para clientes autorizados;
- metadatos y estado de claves;
- rotación y deshabilitación sin destruir prematuramente claves históricas;
- auditoría de operaciones sin registrar material secreto;
- operaciones PIN simuladas y acotadas a fixtures/datos sintéticos;
- modo de fallos y latencia de laboratorio.

No almacena mensajes, PAN, PIN block ni documentos y no accede a MongoDB.

### 7.4 API y adaptadores

El Cryptographic Service depende de un `HsmPort`. El adaptador inicial llama al HSM Simulator por REST/mTLS. El futuro adaptador de fabricante podrá utilizar SDK, PKCS#11 u otra interfaz según el dispositivo y las funciones disponibles.

La API del simulador no debe presentarse como API universal de HSM. Operaciones PIN requieren contratos específicos; no se modelarán como `decryptPin` genérico ni devolverán PIN en claro.

### 7.5 PIN simulado

Alcance funcional candidato:

- validar estructura/formato de PIN block de prueba;
- simular verificación PIN contra valores sintéticos;
- simular traducción de PIN block entre formatos de prueba;
- utilizar un dominio de claves PIN separado de las KEK documentales;
- rechazar entradas inválidas y no devolver PIN en claro.

Estas operaciones demuestran integración y contratos, no equivalen a comandos de un HSM certificado ni a cumplimiento PCI PIN.

## 8. Lotes criptográficos y concurrencia

### 8.1 Política acordada

| Parámetro | Valor |
|---|---:|
| Política | Híbrida |
| Máximo | 1.000 transacciones asignadas por lote |
| Duración máxima | 15 minutos |
| Cierre | El primer límite alcanzado |
| DEK | Una por lote |
| Instancias Crypto Service | Múltiples |

La ventana de 15 minutos comienza al abrir el lote. El máximo de 1.000 cuenta mensajes admitidos/reservados, no solamente documentos que ya terminaron de persistirse, para evitar sobreasignación bajo concurrencia.

### 8.2 Estados del lote

- `OPEN`: admite nuevas reservas.
- `CLOSING`: no admite nuevas reservas; se resuelven reservas ya admitidas.
- `CLOSED`: lote finalizado, solo lectura/descifrado.
- `FAILED`: fallo que requiere recuperación o intervención.

Un lote cerrado no se reabre. Un lote fallido no se utiliza para nuevos mensajes.

### 8.3 Coordinación multi-instancia

La coordinación debe residir en almacenamiento compartido con operaciones atómicas/transaccionales, no en locks locales de JVM.

Requisitos:

- reservar capacidad de lote de forma atómica;
- no superar 1.000 reservas;
- una sola asignación lógica por `messageId` idempotente;
- impedir nuevas reservas al pasar a `CLOSING`;
- registrar reserva y resultado de persistencia;
- recuperar reservas abandonadas tras reinicios;
- garantizar nonce único por DEK;
- permitir que cualquier instancia autorizada recupere `wrappedDek` y solicite unwrap al HSM.

MongoDB será el coordinador operacional y repositorio documental. Se requiere replica set, índices únicos, actualizaciones condicionales y transacciones multi-documento solo donde sean necesarias. No se deben usar locks distribuidos improvisados ni mantener transacciones abiertas durante llamadas al HSM.

### 8.4 Nonces y reintentos

Cada mensaje cifrado con la DEK del lote debe utilizar nonce GCM distinto. Un reintento de cifrado debe generar un nonce nuevo; no debe reutilizar el nonce de un intento cuyo resultado de persistencia sea incierto. Se recomienda restricción única por `(batchId, nonce)` en el repositorio que gestione los documentos, con reintento ante colisión.

La política final debe considerar límites de uso de GCM por clave, cantidad de mensajes, tamaño total y tiempo. El límite de 1.000 mensajes y 15 minutos es una política de laboratorio, no una demostración automática de cumplimiento de todos los límites criptográficos.

## 9. Estado transaccional, idempotencia y recuperación

### 9.1 Estados de operación

La máquina de estados v1.0 debe evolucionar para incluir reversos:

- `AUTHORIZATION_PENDING`
- `AUTHORIZED`
- `DECLINED`
- `FINANCIAL_PENDING`
- `COMPLETED`
- `FAILED`
- `INDETERMINATE`
- `REVERSAL_PENDING`
- `REVERSED`
- `REVERSAL_DECLINED` (si el contrato de negocio lo distingue)
- `INCOMPLETE`
- `EXPIRED`

Las transiciones exactas se definirán por caso de uso. No se debe permitir que un reverso duplique una reversa ya confirmada ni que una operación indeterminada se marque como fallida sin evidencia.

### 9.2 Idempotencia

Mantener la idea v1.0: no usar STAN por sí solo. La identidad debe combinar origen, MTI, referencias temporales/de red aplicables y una huella normalizada del contenido. El perfil debe definir la ventana de retención y cómo se trata la reutilización legítima de STAN.

El backend simulado debe aceptar una clave de idempotencia estable para autorización, operación financiera y reverso.

### 9.3 Recuperación

Escenarios obligatorios:

- Gateway reinicia después de enviar al backend y antes de responder al emisor.
- Backend procesa y la respuesta se pierde.
- Crypto Service reinicia después de reservar un lote.
- Crypto Service cifra y falla la persistencia.
- HSM Simulator se reinicia o queda indisponible.
- MongoDB queda temporalmente indisponible.
- Rotación de KEK mientras hay lotes abiertos y cerrados.
- Certificado mTLS expirado o revocado.

La recuperación no debe depender de memoria local de una instancia. Los resultados indeterminados requieren consulta al backend o proceso de reconciliación; no reejecutar a ciegas una operación financiera. MongoDB conservará los checkpoints y estados necesarios para que otra instancia continúe la recuperación.

## 10. Seguridad y límites de confianza

### 10.1 Controles mínimos implementables

- TLS en el canal ISO del laboratorio.
- mTLS entre Gateway/Cryptographic Service/HSM Simulator donde aplique.
- Red Docker privada para servicios internos.
- Secretos fuera del repositorio e imágenes.
- KEK no exportable en claro desde HSM Simulator.
- DEK solo transitoria en memoria de Crypto Service.
- Cifrado autenticado de documentos.
- Autorización por identidad de servicio para operaciones HSM.
- Logs y trazas sin PAN completo, PIN block, DEK, KEK ni payload ISO.
- Auditoría de uso de claves sin material secreto.
- Backups protegidos y prueba de restauración.

### 10.2 Límites explícitos

El HSM Simulator no ofrece:

- protección física contra manipulación;
- certificación FIPS 140-3;
- certificación PCI PIN Security;
- custodia de claves equivalente a hardware dedicado;
- separación física de operadores;
- cumplimiento PCI DSS del entorno.

El prototipo no debe etiquetarse como PCI compliant. La aplicabilidad de PCI DSS 3.3.3 para conservar SAD debe ser revisada por un QSA y las marcas. El cifrado de datos no elimina por sí solo el alcance PCI.

## 11. Resiliencia y operación

### 11.1 Timeouts y circuit breaker

Definir timeouts separados para:

- conexión y lectura TCP;
- llamada al backend;
- llamada al Cryptographic Service;
- llamada al HSM Simulator;
- operaciones de persistencia.

Circuit breaker puede utilizarse para dependencias remotas. No debe convertir un error temporal en rechazo financiero ni habilitar fallback criptográfico inseguro.

### 11.2 Política de fallos

- Si el HSM no está disponible, no se persiste un mensaje sensible como si hubiera sido cifrado correctamente.
- No hacer fallback a una KEK local o cifrado alternativo no aprobado.
- Las operaciones financieras ya enviadas al backend y con respuesta desconocida quedan indeterminadas.
- Las operaciones de cifrado fallidas se registran como fallidas sin payload sensible en logs.
- Las reservas de lote abandonadas se resuelven mediante recuperación explícita.

### 11.3 Backups y recuperación

- Backups de estado operacional y MongoDB.
- Backup del material de claves del simulador y de su secreto raíz mediante procedimiento separado y protegido.
- Pruebas de restauración periódicas dentro del laboratorio.
- Verificar que la restauración conjunta de datos y claves permita descifrar documentos históricos.
- Documentar que pérdida irreversible de KEK/raíz implica pérdida de acceso a ciphertext asociado.

## 12. Observabilidad

Implementar logs estructurados, métricas y trazas con cardinalidad controlada.

Métricas candidatas:

- mensajes recibidos/procesados por MTI;
- latencia por etapa;
- respuestas aprobadas/rechazadas/indeterminadas;
- reversos recibidos/completados/duplicados;
- timeouts y errores de backend;
- latencia y errores del HSM;
- tamaño/edad/estado de lotes;
- cantidad de reservas pendientes;
- errores de cifrado/descifrado;
- fallos de persistencia.

No utilizar PAN, PIN, STAN, RRN, tokens de correlación completos ni identificadores de alta cardinalidad como labels de métricas.

## 13. Estrategia de testing y criterios de aceptación

### 13.1 ISO y transporte

- framing fragmentado y frames concatenados;
- validación de MTI, bitmap, longitud y DE;
- TLS/mTLS válido e inválido;
- respuesta por canal correcto;
- concurrencia entre terminales y operaciones.

### 13.2 Flujos financieros

- autorización aprobada/rechazada;
- operación financiera válida/inválida;
- 0200 duplicado concurrente;
- 0400 sobre operación aprobada;
- 0400 duplicado;
- 0400 con operación inexistente, pendiente o indeterminada;
- timeout antes/después del commit del backend;
- 0410 según resultado del reverso;
- 0420/0430 si se aprueba su alcance.

### 13.3 Criptografía y HSM

- generar DEK y obtener wrapped DEK;
- cifrar/descifrar y comprobar igualdad de bytes;
- ciphertext/tag/AAD alterados deben fallar autenticación;
- wrapped DEK alterada o KEK incorrecta debe fallar;
- rotación y recuperación con versiones anteriores;
- HSM no disponible y timeout;
- reinicio del simulador y recuperación de KEK;
- verificación de que no se registran claves ni datos sensibles;
- operaciones PIN de prueba sin salida de PIN en claro.

### 13.4 Lotes y concurrencia

- cierre por 1.000 reservas;
- cierre por 15 minutos;
- cierre concurrente por ambas condiciones;
- dos instancias asignando al mismo lote;
- no superar capacidad;
- recuperación de reservas abandonadas;
- nonce único bajo concurrencia;
- lote cerrado no acepta nuevas reservas;
- cualquier instancia autorizada puede recuperar documentos históricos.

### 13.5 Persistencia y recuperación

- reinicio de cada servicio;
- caída e indisponibilidad de MongoDB;
- fallo entre reserva, cifrado y persistencia;
- restauración de backup de datos y claves;
- reconciliación de operaciones indeterminadas.

### 13.6 Criterios globales

El diseño se considerará validado cuando cada requisito tenga escenario de prueba, resultado esperado y evidencia definida. No se afirmará throughput bancario ni certificación sin pruebas y validaciones específicas.

## 14. Riesgos y decisiones abiertas

| ID | Tema | Recomendación | Estado |
|---|---|---|---|
| ADR-01 | Base de datos del Gateway | MongoDB como única base: estado operacional, idempotencia, lotes, metadata y mensajes cifrados; PostgreSQL excluido | Confirmada por el usuario |
| ADR-03 | `0420/0430` | Definir si entra en primera versión funcional | Pendiente |
| ADR-04 | Retención de DE52 | Deshabilitada por defecto; excepción 3.3.3 solo con validación formal | Pendiente de validación regulatoria |
| ADR-05 | Política de DEK por lote | 1.000 mensajes o 15 minutos, lo primero que ocurra | Confirmada por usuario |
| ADR-06 | Multi-instancia Crypto Service | Soportar múltiples instancias desde diseño | Confirmada por usuario |
| ADR-07 | mTLS | Obligatorio entre servicios internos indicados | Confirmada por usuario |
| ADR-08 | PIN simulator | Incluir operaciones simuladas con datos sintéticos | Confirmada en alcance |
| ADR-09 | HSM Simulator | Servicio independiente Docker, no almacena transacciones | Confirmada por usuario |
| ADR-10 | Azure | Fase posterior, no en el primer despliegue local | Propuesta |

### Riesgos principales

1. **Concurrencia y consistencia en MongoDB:** múltiples instancias pueden competir por reservas y transiciones. Mitigar con índices únicos, actualizaciones condicionales, transacciones breves y pruebas de replica set.
2. **DEK compartida por lote:** compromiso de una DEK afecta hasta 1.000 mensajes o 15 minutos. Mitigar con límites, nonces únicos, acceso mínimo y cierre estricto.
3. **Recuperación de claves:** perder KEK/raíz inutiliza ciphertext. Mitigar con backup/restauración probados.
4. **PIN/SAD:** almacenamiento no permitido salvo condiciones estrictas. Mitigar con exclusión por defecto y revisión formal.
5. **Perfil ISO incompleto:** no se conoce especificación de Link/Prisma. Mitigar con perfil de laboratorio explícito y no afirmar compatibilidad.
6. **Resultado indeterminado:** puede producir doble ejecución si se reintenta sin consulta. Mitigar con idempotencia backend y reconciliación.

## 15. Estado de aprobación e impacto sobre v1.0

La arquitectura integral v2.0 fue revisada y aprobada documentalmente por el usuario. La aprobación se materializa en los artefactos versionados `SPEC.md`, `PLAN.md`, `TASKS.md` y los contratos `DOC-001` a `DOC-009`.

Los documentos v1.0 aprobados permanecen intactos como baseline histórico. No se modifican ni se invalidan retroactivamente.

La aprobación documental no equivale a implementación ni habilita todo el backlog. El usuario autorizó explícitamente el Bootstrap limitado a:

- `BOOT-001`: fijar versiones compatibles.
- `BOOT-002`: establecer estructura de repositorios/proyectos.
- `BOOT-003`: configurar Maven Wrapper y builds.
- `BOOT-004`: configurar Quality Gate.
- `BOOT-005`: configurar Docker Compose y MongoDB replica set.
- `BOOT-006`: configurar secretos y certificados de laboratorio.
- `BOOT-007`: implementar health/readiness.

La ejecución debe respetar las dependencias y criterios de salida documentados en `TASKS.md`. Esta autorización no incluye épicas funcionales E2 en adelante ni tareas fuera de BOOT-001..BOOT-007.

## 16. Validación completada y siguiente paso

La validación arquitectónica y documental de v2.0 está completada. Las decisiones de alcance y los contratos correspondientes están reflejados en `SPEC.md`, `PLAN.md`, `TASKS.md` y `docs/contracts/DOC-001` a `DOC-009`.

La línea base aprobada queda definida así:

- Flujos incluidos: `0100/0110`, `0200/0210` y `0400/0410`.
- `0420/0430`: fuera del baseline; extensión futura sujeta a decisión específica.
- MongoDB: única base de datos del Gateway, desplegada como replica set; un nodo local habilita transacciones, pero no constituye alta disponibilidad.
- Persistencia de `DE52`/PIN block: excluida del baseline ejecutable. Las operaciones PIN del simulador utilizan exclusivamente fixtures sintéticos y nunca devuelven PIN en claro.
- Clearing/settlement completo: fuera de alcance. Se incluye reconciliación operacional acotada de operaciones indeterminadas.
- Azure/Kubernetes: fase posterior.
- No se declara compatibilidad Link/Prisma, certificación HSM/FIPS ni cumplimiento PCI DSS/PCI PIN.

**Siguiente paso autorizado:** ejecutar BOOT-001..BOOT-007 conforme al orden de dependencias y criterios de salida de `TASKS.md`. No iniciar E2 ni ninguna épica funcional sin autorización independiente.

**Gate obligatorio de CRL:** durante Bootstrap debe verificarse el soporte real de revocación mediante CRL en el stack TLS seleccionado y demostrarse su comportamiento fail-closed. Si el soporte no existe o no puede demostrarse, detener la tarea afectada, documentar evidencia y solicitar decisión. No sustituir CRL por OCSP, soft-fail ni otra política sin aprobación explícita.

Cada tarea debe producir evidencia verificable y el informe Markdown exigido por `AGENTS.md`. La finalización documental o de Bootstrap no debe presentarse como implementación funcional del Gateway.

**Estado final de este documento:** arquitectura v2.0 aprobada documentalmente. Bootstrap BOOT-001..BOOT-007 autorizado. Implementación funcional no iniciada ni autorizada.
