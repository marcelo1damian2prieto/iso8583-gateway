# CRL-FEASIBILITY-009A.1 — T03-00 Baseline de inspección read-only

**Fecha:** 2026-11-15  
**Estado:** `INSPECCIÓN ESTÁTICA COMPLETADA — CON LIMITACIÓN DE ACCESO DIRECTO AL FILESYSTEM`  
**Workspace esperado:** `%TEMP%\crl-feasibility-009a1\project`  
**Origen declarado:** `%TEMP%\crl-feasibility-009a\project`  
**Alcance:** baseline, estructura, POM, inventario, hashes y brechas para TASK-03. Sin cambios de código/configuración y sin ejecución.

## 1. Dictamen ejecutivo

T03-00 queda **completada sobre la evidencia disponible**. El workspace aislado fue provisionado fuera de T03-00 y cuenta con un manifiesto de provisión. Contrasté independientemente el `project.zip` subido en la conversación contra los 31 registros incluidos en `WORKSPACE-PROVISIONING-MANIFEST.json`: **31/31 archivos coinciden exactamente en tamaño y SHA-256; 0 faltantes; 0 discrepancias**.

La inspección estática confirma que el contenido es una copia aislada del arnés histórico `crl-feasibility-009a`, no una implementación nueva de TASK-03. El POM conserva `artifactId=crl-feasibility-009a`, `version=1.0.0`, Java 21 y Spring Boot 3.5.16. El código Java está íntegramente bajo `src/test/java/lab/crl`; no existe `src/main/java` en el inventario provisionado.

**Conclusión de ingeniería:** el Coordinator de 009A es reutilizable como antecedente/harness de control loopback y journal básico, pero no implementa el almacén durable ni las garantías de TASK-03. La persistencia, replay, checkpoint, idempotencia durable, OS lock, epoch y serialización H-01 siguen siendo trabajo nuevo de diseño/implementación.

## 2. Evidencia y método

### 2.1 Evidencia usada

- `WORKSPACE-PROVISIONING-MANIFEST.json`, recibido del usuario. Declara source/destination, POM, ausencia de `.git`, inventario incluido/excluido y validaciones de copia.
- `project.zip`, subido por el usuario como origen del proyecto.
- `Markdown pegado.md`, informe histórico TASK-01, que identifica que la inspección anterior correspondía a `009a\project`, no a `009a1\project`.
- Paquete documental TASK-03-DRAFT-4 en GitHub, commit `bef7aa7c615745e19f548b47854fb4e37d543b18`, usado para respetar el alcance de T03-00.
- Captura del usuario de la raíz local `009a1\project`.

### 2.2 Verificación de identidad del conjunto provisionado

Extraje en un directorio temporal de análisis **solo** los paths listados en el manifiesto. Para cada ruta del manifiesto, comparé contra la entrada `project/<ruta>` de `project.zip`, comprobando tamaño en bytes y SHA-256.

| Control | Resultado |
|---|---:|
| Entradas del manifiesto | 31 |
| Coincidencias tamaño + SHA-256 | **31** |
| Faltantes | **0** |
| Discrepancias | **0** |
| Bytes totales de los archivos incluidos | 158,663 |
| Archivos Java | 21 |
| Fixtures PEM públicos | 9 |
| POM | 1 |

Esto enlaza el inventario aprobado con el archivo ZIP origen. El manifiesto declara además `sourceFilesUnchangedDuringCopy=true`, `allCopiedSha256MatchSource=true` y `sourceWasNotMovedOrDeleted=true`. No recalculé directamente los 31 hashes leyendo tu disco `C:`: la verificación independiente se hizo contra el ZIP subido y el manifiesto; la validación del destino local queda respaldada por lo declarado en el manifiesto y la captura.

### 2.3 Git y exclusiones

- Git: `repositoryDetected=false`; no se detectó metadata `.git` en la raíz del proyecto. No hay branch, HEAD ni status que informar.
- Se excluyeron logs históricos, `target/` y artefactos generados/sensibles según el manifiesto.
- No se copió `.git`, en línea con el plan de provisión aprobado.
- Los fixtures PEM incluidos son los nueve que el manifiesto autoriza bajo `fixtures/public/`; el inventario no incluye PKCS12 ni claves privadas.

## 3. Inventario del workspace

| Grupo | Cantidad | Rutas principales |
|---|---:|---|
| Configuración Maven | 1 | `pom.xml` |
| Código Java | 21 | `src/test/java/lab/crl/*.java` |
| Certificados/CRL PEM | 9 | `fixtures/public/*.pem` |
| **Total** | **31** | |

No se encontró ningún source set `src/main/java` en el conjunto provisionado. Los 21 archivos Java son clases de soporte y pruebas del laboratorio.

## 4. POM y stack declarado

| Elemento | Declaración observada |
|---|---|
| Maven coordinates | `lab.crl:crl-feasibility-009a:1.0.0` |
| Parent | `org.springframework.boot:spring-boot-starter-parent:3.5.16` |
| Java | `21` |
| Aplicación | `spring-boot-starter-webflux` |
| Red | `reactor-netty-http` |
| Testing | `spring-boot-starter-test` (`test`) |
| PKI sintética | `bcpkix-jdk18on:1.81` (`test`) |

El POM no fija versiones explícitas para Reactor Netty ni para plugins Maven. No ejecuté `mvn`, por lo que no afirmo un árbol efectivo de dependencias ni versiones resueltas del entorno actual.

## 5. Componentes relevantes y reutilización

| Archivo | Responsabilidad observada | Relación con TASK-03 |
|---|---|---|
| `RotationCoordinator009A.java` | Coordinador dentro de la JVM JUnit; mantiene mapas locales de rollout; TCP loopback a miembros; journal texto | Único punto de partida directo para el Coordinator, pero requiere rediseño del almacenamiento y del modelo de estado |
| `DistributedTlsInstance009A.java` | Proceso hijo; `STATUS/PREPARE/ACTIVATE/STOP`; TLS/Netty; estado local generation/hash | Referencia de lifecycle del Member 009A; TASKS-DRAFT-4 excluye modificarlo salvo lectura |
| `DistributedRotation009ATest.java` | Arranca dos JVM hijas y prueba rotación G1→G2 | Reutilizable como patrón futuro de harness multiproceso, no como prueba del Coordinator durable |
| `ControlledCrlManager.java`, `CrlSnapshot.java` | Descarga, valida y publica snapshots CRL/PKIX con digest | Infraestructura CRL; no es persistencia de Coordinator |
| `FreshnessTrustManager.java` | Wrapper de trust manager y registro de callbacks | Observabilidad PKIX/TLS; no provee el gate de admission |
| `SyntheticTransactionDrain008Test.java` | Simulador in-memory de operaciones in-flight, UNKNOWN e idempotencia con mapa concurrente | Antecedente conceptual; no proporciona ledger durable ni índice idempotente persistente |
| `RestartJvmTest.java`, `RestartProbeMain.java` | Reinicio de JVM hija y revalidación CRL antes de escuchar | No reconstruye el journal ni el estado del Coordinator |
| `CrlHttpFixture.java`, `GateATest.java`, `FreshnessHardeningTest.java`, `Session*`, `RevocationResumption008Test.java`, `ControlledLoadTest.java`, `SpringBootWebClientTest.java` | Fixtures y pruebas TLS/CRL, sesiones, frescura y carga | Harness heredado; no cubre el store de TASK-03 |

## 6. Hallazgos técnicos

### F-01 — No existe el Coordinator durable de TASK-03

**Severidad: crítica para el alcance funcional de TASK-03.**

`RotationCoordinator009A` modela `CREATED`, `PREPARING`, `PREPARED`, `ACTIVATING`, `ACTIVE` y `FAILED`. Mantiene endpoints y observaciones en mapas en memoria. Su journal escribe líneas de texto con timestamp y evento mediante `FileChannel` en `APPEND`, seguido de `force(true)`.

No se observa implementación de:

- schema versionado, secuencia monotónica ni envelope de eventos;
- `eventId`, `transitionKey`, `payloadDigest`, `previousHash` o `recordHash`;
- checkpoint versionado y snapshot determinista;
- replay, validación cruzada o recuperación de tail;
- OS lock exclusivo (`FileLock`) ni ownership del almacén;
- `coordinatorEpoch` durable;
- ADMIT, COMPLETE, FENCE, `HANDLER_START_INTENT`, `ADMISSION_REJECTED_EXPIRED`, `RECONCILE_UNKNOWN` o `COMMIT_GLOBAL`.

El `synchronized` de `record()` serializa exclusivamente la escritura individual del log. No envuelve de extremo a extremo la lectura del estado, validación de precondiciones, decisión de transición, append, checkpoint y publicación requerida por SPEC §11. Por lo tanto, no satisface H-01.

### F-02 — Persistencia local del Member es un mecanismo distinto y no conforme al Coordinator

**Severidad: alta como riesgo de reutilización incorrecta; componente fuera de alcance de cambios T03.**

`DistributedTlsInstance009A` almacena únicamente `generationId|crlHash` en un archivo local. Escribe un temporal `.new`, ejecuta `force(true)`, cierra el archivo e intenta `ATOMIC_MOVE`; si recibe `AtomicMoveNotSupportedException`, cae a `Files.move` no atómico.

Ese comportamiento no satisface el diseño R3.1 para el checkpoint del Coordinator, que prohíbe fallback no atómico. Tampoco incorpora journal append-only, checkpoint/snapshot, hash-chain, OS lock ni replay. No se debe reutilizar este método como implementación del checkpoint de TASK-03. TASKS-DRAFT-4 excluye modificar este Member en esta etapa.

### F-03 — El source set es de pruebas, no una aplicación de producción

**Severidad: alta para el diseño de integración.**

Todo el código Java está en `src/test/java/lab/crl`. El POM incluye dependencias WebFlux/Netty, pero no hay código bajo `src/main/java` en el conjunto incluido. El Coordinator actual es un helper de test-JVM; no existe un proceso Coordinator de producción o una aplicación separada en este baseline.

### F-04 — Idempotencia/UNKNOWN existente es in-memory

**Severidad: alta para cualquier inferencia sobre TASK-03.**

`SyntheticTransactionDrain008Test` usa `ConcurrentHashMap` y `computeIfAbsent` para ilustrar idempotencia, y estados locales para `UNKNOWN`. No hay persistencia del mapa ni recuperación después de reinicio. Es un simulador de lifecycle, no el índice durable transitionKey/recoveryKey exigido por T03-10.

### F-05 — Suite existente no cubre los invariantes de TASK-03

**Severidad: alta como brecha de verificación.**

Identifiqué 23 métodos anotados `@Test` mediante inspección estática. Sus escenarios se orientan a mTLS, TLS 1.2/1.3, revocación CRL, resumption, rotación G1→G2, frescura y drenaje sintético. No encontré pruebas para journal/checkpoint Coordinator, replay, hash-chain, OS lock, tail recovery por tipo, epoch, idempotencia durable ni serialización total H-01.

El recuento es una inspección de anotaciones en el código; no es un resultado de ejecución. No se ejecutaron pruebas.

## 7. Comparación con el informe TASK-01

El informe TASK-01 identificó explícitamente `009a\project` como el workspace inspeccionado, y señaló que el Coordinator solo tenía mapas locales y journal textual append+force; el Member persistía generation/hash con fallback no atómico; y no existían replay, checkpoint, hash-chain ni fencing por request.

La inspección actual **corrobora estructuralmente** esas observaciones sobre el conjunto provisionado. Se corrige el estado de baseline: ahora existe la copia aislada `009a1\project` documentada por el manifiesto, y los 31 archivos seleccionados coinciden con el ZIP origen.

No es posible declarar igualdad byte a byte contra el estado exacto examinado en TASK-01 porque ese informe histórico no contiene hashes por archivo. Por eso la comparación con TASK-01 es de ruta, estructura y contenido técnico observado, no un diff criptográfico contra el baseline histórico.

## 8. Riesgos y límites que continúan abiertos

1. Windows `FileLock`, `ATOMIC_MOVE` y directory force no fueron ensayados. La prueba real requiere autorización específica de experimento, aislamiento y rollback.
2. El manifiesto no demuestra durabilidad ante pérdida de energía, kernel, disco o hardware.
3. No hay `.git`; el versionado del workspace local no puede determinarse desde branch/HEAD/status.
4. La hash-chain local futura no detectará por sí sola la sustitución conjunta del journal y checkpoint por una copia antigua íntegra, conforme al límite ya aceptado en el diseño.
5. D03-OPEN-01..08 permanecen abiertas. El baseline no las resuelve ni habilita congelar codec, política tail, epoch, temporales, clasificación de errores o rebind.
6. No se ejecutó Maven ni se verificó el árbol de dependencias efectivo.

## 9. Estado T03-00 y recomendación

**T03-00: COMPLETADA — BASELINE ESTÁTICO CON LIMITACIÓN EXPLÍCITA DE ACCESO.**

- Inventario y hashes: 31/31 vinculados por SHA-256 al ZIP origen y al manifiesto.
- POM: inspeccionado estáticamente.
- Source tree: 21 Java bajo `src/test/java`; sin `src/main/java`.
- Git: no detectado; `.git` excluido en el aprovisionamiento.
- Comparación TASK-01: corroborada estructuralmente; no existe baseline histórico hash-by-hash para diff exacto.
- Archivos modificados: ninguno.
- Maven/build/tests/experimentos ejecutados: ninguno.

**Siguiente paso del plan:** T03-01, catálogo de dominio/eventos y tratamiento de D03-OPEN-01..08. T03-01 sigue siendo preparación/decisión de diseño; no autoriza implementar T03-02..T03-12. La aprobación integral de TASK-03, autorización separada de implementación y autorización específica para experimentos Windows siguen siendo gates independientes.

## 10. Inventario SHA-256 por archivo

Los hashes siguientes son los valores del manifiesto y fueron contrastados byte a byte con el archivo `project.zip` subido.

| Ruta relativa | Bytes | SHA-256 |
|---|---:|---|
| `pom.xml` | 1,113 | `4317f7b1adc42d04f4038860c1d14188a67a2b6d8863949051ede747f4c497d0` |
| `src/test/java/lab/crl/BootMtlsApplication.java` | 3,728 | `a7f9b97973de099499751171d1c2211fef0393b3249c1b829dcabd327142900d` |
| `src/test/java/lab/crl/ControlledCrlManager.java` | 10,143 | `ebcbdde8866a8f85f6fb2c5bd2bee02eac73fec7728e84f112d87ceece43e177` |
| `src/test/java/lab/crl/ControlledLoadTest.java` | 3,743 | `b8701f4ff61e62a0510995829f29e442f746c9efb08c844fd6903ddbb3346b54` |
| `src/test/java/lab/crl/CrlHttpFixture.java` | 1,669 | `1a10cd49bba87bb049d3fee113386d48e882f89d1fd61d3923b85e30f5a45263` |
| `src/test/java/lab/crl/CrlSnapshot.java` | 622 | `0a77ec3115eb01c04bff1870e2988a6004d292d1aadaad7f0a07af9df0cf2b12` |
| `src/test/java/lab/crl/DistributedRotation009ATest.java` | 17,960 | `da60f81854cbc939c295a1bb7a8a940c2d7bda861c95a8262c04e8b4c0b842ab` |
| `src/test/java/lab/crl/DistributedTlsInstance009A.java` | 13,151 | `06158f3102b0c0bde82b6fd2b5ccc49595d7ad278382f06773ec880b5bf8cfa7` |
| `src/test/java/lab/crl/FreshnessHardeningTest.java` | 19,891 | `c764b32af155a8b823b28f568ed1001c74cda47f48deba8d610bb81bd57c2481` |
| `src/test/java/lab/crl/FreshnessTrustManager.java` | 4,291 | `56a643924eb106b0e8c20ccc89fbd154807ffdc17fadff0d1080e2caa21c2e93` |
| `src/test/java/lab/crl/GateATest.java` | 9,198 | `0912bc8173716856a3d349cf6d9ef0dd60c879a81bab47a48c16ed39664d65a6` |
| `src/test/java/lab/crl/MultiInstance008Test.java` | 5,767 | `1e988f492fee1e764aeab8e5fce36c3ceba8c0b570e88fea6dc0dc7fe038a6b7` |
| `src/test/java/lab/crl/PersistentMtlServer.java` | 1,731 | `1d481e3d52dc972b66c1534e64fe14e8a888114b9c1e0623077f5d87ced9d86b` |
| `src/test/java/lab/crl/ReactorHandshake.java` | 2,144 | `e44e0de29bb9d863a4c198a51e2c207127facbac892f4c011773dd4f70c103e6` |
| `src/test/java/lab/crl/RestartJvmTest.java` | 2,381 | `25927bd856d66ea6e31ce8be769dc4a7178c872cf435d05e810550ff02be56b9` |
| `src/test/java/lab/crl/RestartProbeMain.java` | 1,230 | `656e5142bef2e797f1d86c6ab770da9f3d1de3268339f4bcc46b2ed24d50158c` |
| `src/test/java/lab/crl/RevocationResumption008Test.java` | 7,134 | `6bbe8e61141f905c650f271fe7102518c0f8aade40c3015d58d55700927bab21` |
| `src/test/java/lab/crl/RotationCoordinator009A.java` | 5,101 | `2e1270f0c52e1c2f41bc391d03541368210d3ca48a6b422ea7f995f83f31f795` |
| `src/test/java/lab/crl/SessionInvalidation008Test.java` | 10,976 | `240f874d6f6c9e34c0df4a6b66c00f0a544367eca849b606bd61470e10fd62c3` |
| `src/test/java/lab/crl/SessionResumptionProbeTest.java` | 6,875 | `143378072c182af219054d86d23f47982e7322c14767ad00026535c60fc4af60` |
| `src/test/java/lab/crl/SpringBootWebClientTest.java` | 14,586 | `521c662fca8840c87eb68bd2cf3b8b2eecc9cab82c83cbc3e5dfe9ac29666b8a` |
| `src/test/java/lab/crl/SyntheticTransactionDrain008Test.java` | 6,556 | `2ac7f015687b3013bc279af05fed927a41fac7c708729515af950f85d301e236` |
| `fixtures/public/008-client.pem` | 1,155 | `e5b4c98046d353871bce72d082378ef30c10882af553c0c6ec95fec18e5df482` |
| `fixtures/public/008-crl-client-revoked.pem` | 629 | `8ebf7e1be85ae1baddcf2b90d2475bbbfc77a0ce1bc8defdf4e57ffe1ac998e4` |
| `fixtures/public/008-crl-server-revoked.pem` | 629 | `d86996b8e8ef7694054a08ce1ba29dff5a352183945bee8b8f2001bd31b780cc` |
| `fixtures/public/008-root-ca.pem` | 1,066 | `f1d6916bb2b1fe22d0a8c58bcddf76f7784c85ec590794df02836b74f87b33f6` |
| `fixtures/public/008-server.pem` | 1,180 | `9be7353e9dbc39fbcda688b7d78170d9d79d3dec0445d5226904add5f2390b85` |
| `fixtures/public/lab-client.pem` | 1,147 | `7ebf871fdbfb896dd23fb958b7146ebc002b8247ce5a5f5fafe240f4830c2641` |
| `fixtures/public/lab-root-ca.pem` | 1,066 | `59d3818feff87f98423d7c2ad593c95a76e4e95fe54e6b1851e657d2a1e8f129` |
| `fixtures/public/lab-server-revoked.crl.pem` | 629 | `38553b13a0e6620ab3e938d935c365671720348984416e670873ffdf09108c37` |
| `fixtures/public/lab-server.pem` | 1,172 | `8251ed269306205b018022a7bb49f513f7f463be31e7ab7842b825c0f148c22d` |

---

**Fuente normativa de alcance:** `TASK-03-TASKS-DRAFT-4.md` y documentos relacionados, en commit `bef7aa7c615745e19f548b47854fb4e37d543b18` del repositorio `marcelo1damian2prieto/iso8583-gateway`.  
**No constituye:** aprobación de diseño completo, autorización de implementación, validación Windows de durabilidad, ni aprobación para producción.