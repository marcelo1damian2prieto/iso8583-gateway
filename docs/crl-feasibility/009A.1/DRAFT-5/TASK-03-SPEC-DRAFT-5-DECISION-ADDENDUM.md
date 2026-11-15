# TASK-03 — Enmienda normativa DRAFT-5 a SPEC-DRAFT-4

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-SPEC-DRAFT-5-DECISION-ADDENDUM`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Documento base:** `TASK-03-SPEC-DRAFT-4.md`  
**Base documental:** DRAFT-4 en `main` (`bef7aa7c615745e19f548b47854fb4e37d543b18`), T03-00 aceptada, `TASK-03-01-DOMAIN-EVENT-CATALOG-DRAFT-1.md`, `TASK-03-01-HUMAN-ACCEPTANCE-ADDENDUM-1.md` y `TASK-03-01-COLLISION-ANALYSIS-AND-REFACTOR-PLANNING-CLARIFICATION-1.md`.

## 1. Efecto y precedencia

Esta enmienda es aditiva y debe leerse junto con SPEC-DRAFT-4. No elimina ni reescribe su contenido histórico. Para los temas enumerados aquí, **esta enmienda prevalece sobre cualquier estado, precondición o conclusión de DRAFT-4 que resulte anterior o contradictorio**. El resto de SPEC-DRAFT-4 permanece vigente.

La revisión documental combinada se denomina `SPEC-DRAFT-5`. No equivale a aprobación integral del diseño TASK-03 ni a autorización de implementación.

## 2. Baseline T03-00 — estado actualizado de §3

Reemplazar el estado histórico de §3.2–§3.4 que indicaba ausencia del workspace y `T03-00 BLOCKED — WORKSPACE NOT PROVISIONED` por el siguiente baseline:

- Workspace provisionado: `%TEMP%\<redacted>sibility-009a1\project`.
- Origen de la copia: `%TEMP%\<redacted>sibility-009a\project`; el origen se preservó y no se movió.
- Manifiesto: `WORKSPACE-PROVISIONING-MANIFEST.json`.
- Inventario: 31 archivos incluidos (1 POM, 21 archivos Java bajo `src/test/java/lab/crl`, 9 fixtures públicos PEM); 70 rutas/carpetas excluidas.
- Git: no se detectó metadata `.git`; branch/HEAD/status no están disponibles en el workspace provisionado.
- POM: `artifactId=crl-feasibility-009a`, `version=1.0.0`, Java 21, Spring Boot 3.5.16 declarado. No se resolvió el árbol Maven efectivo porque no se ejecutó Maven.
- Source set: no se encontró `src/main/java` en el inventario; los 21 Java incluidos están en `src/test/java/lab/crl`.
- Inspección: estática y read-only. Se identificaron 23 métodos anotados `@Test` por lectura de fuentes; esto no es resultado de ejecución.

La verificación independiente disponible contrastó los 31 registros del manifiesto con las entradas correspondientes de `project.zip`: 31/31 coincidieron en tamaño y SHA-256, sin faltantes ni discrepancias. El manifiesto declara además que los hashes copiados coinciden con el origen, que el origen no cambió durante la copia y que no fue movido/eliminado. No se recalcularon directamente los hashes leyendo el filesystem local `C:` desde esta sesión. La limitación debe permanecer explícita.

El informe T03-00 `T03-00-BASELINE-INSPECTION.md` fue inspeccionado y aceptado por el usuario. La comparación con TASK-01 es estructural; TASK-01 no contiene hashes históricos por archivo y, por tanto, no permite declarar equivalencia hash-by-hash. T03-00 queda `COMPLETED — STATIC READ-ONLY BASELINE WITH ACCESS LIMITATION`.

Los hallazgos de base continúan: el Coordinator existente es un harness de pruebas con mapas en memoria y journal de texto append-only; el `synchronized` de `record()` no proporciona la serialización de negocio H-01; no se identificó checkpoint/replay/hash-chain/FileLock/epoch de Coordinator. La persistencia del Member es distinta y no debe reutilizarse como checkpoint de Coordinator. Ninguna de estas brechas se resolvió por el aprovisionamiento.

## 3. Registro normativo de decisiones D03-OPEN-01..08

La aprobación humana del usuario acepta las direcciones conservadoras descritas en el catálogo T03-01, con los estados y reservas siguientes. La aceptación **no** cierra los detalles técnicos que se indican como abiertos.

| ID | Estado humano efectivo | Baseline normativo aceptado | Detalles que siguen abiertos |
|---|---|---|---|
| D03-OPEN-01 | `DIRECTION APPROVED — DETAIL SPECIFICATION OPEN` | Codec binario tipado, versionado, determinista y length-prefixed; versiones separadas para record, checkpoint y canonicalización; presencia/null/valor explícitos; SHA-256 con domain separation para payload y record; `recoveryEventId` conserva fórmula R3.1. | Tags, anchos/byte order, separadores exactos, nullability por campo, límites, unknown fields, genesis hash y compatibilidad/versionado. No congelar el codec hasta definirlos. |
| D03-OPEN-02 | `CLOSED — TAIL POLICY APPROVED` | Recuperación automática de tail únicamente para ADMIT y COMPLETE, con todas las precondiciones R3.1. Tails de FENCE, HANDLER_START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL y epoch advance permanecen `RECOVERY_REQUIRED`. No inferir confirmación, enviar ACK ni producir evento compensatorio por analogía. | Cualquier resolución manual futura requiere contrato independiente de evidencia, autoridad, evento durable, idempotencia, STATUS y salida del estado cerrado. No hay autorización de recovery automático adicional. |
| D03-OPEN-03 | `DIRECTION APPROVED — STARTUP CONTRACT DETAIL OPEN` | Evento durable explícito de avance de epoch, nombre provisional `COORDINATOR_EPOCH_ADVANCED`; `newEpoch = previousEpoch + 1` estricto, sin wrap/reutilización; almacén genuinamente vacío puede iniciar en epoch 1 solo si journal, checkpoint y temporales están todos ausentes; no operar antes del checkpoint del epoch de arranque. Tail de epoch => `RECOVERY_REQUIRED`; no ignorar ni duplicar. | Schema/nombre final, `startupOperationId`, definición operativa de reinicio, overflow, genesis exacto y cortes/crash recovery. |
| D03-OPEN-04 | `POLICY APPROVED — PLATFORM EVIDENCE OPEN` | Clasificar resultados como `SUPPORTED_AND_SUCCEEDED`, `UNSUPPORTED` explícito o `FAILED_OR_AMBIGUOUS`. Solo declarar `UNSUPPORTED` ante indicación explícita de API/proveedor. I/O, acceso denegado y ambigüedad no son unsupported por heurística. Sin fallback no atómico; error real/ambiguo mantiene cierre y requiere replay/verificación. | Clasificación concreta de excepciones/proveedores en el JDK/Windows/filesystem objetivo. Directory force no soportado limita claims a lo realmente ensayado; no prueba durabilidad ante pérdida de energía/kernel/disco/hardware. |
| D03-OPEN-05 | `CLOSED — RESIDUAL TEMP POLICY APPROVED` | Crear temporales con `CREATE_NEW` en el mismo directorio/filesystem y nombre reservado correlacionado con la operación/evento. Startup detecta residuales; no los promueve, reemplaza ni elimina automáticamente. | Prefijo/naming final y límites de lectura. Limpieza manual, si se plantea, exige decisión, autoridad, resguardo/hash previo y registro; no es parte del replay automático. |
| D03-OPEN-06 | `DIRECTION APPROVED — CLASSIFICATION DETAIL OPEN` | Distinguir `RECOVERING` (fase transitoria), `RECOVERY_REQUIRED` (datos íntegros con semántica/evidencia no resuelta) y `FAILED_CLOSED` (corrupción, contradicción o propiedad del almacén/lock no confiable). No auto-reset. Retorno exige acción humana explícita y documentada; `FAILED_CLOSED` requiere reparación/restauración controlada y baseline verificable. | Tabla exhaustiva condición→estado→acción→evidencia→autoridad; clasificar lock ocupado por otro Coordinator legítimo frente a lock incierto/no confiable. |
| D03-OPEN-07 | `GATE APPROVED — EXPERIMENT NOT AUTHORIZED` | FileLock, ATOMIC_MOVE y directory force en Windows siguen `UNVERIFIED` hasta experimento aislado, con fixtures sintéticos y registro de JDK/OS/proveedor/filesystem/excepciones/hashes. Cada mecanismo se prueba por separado. | **El experimento no está autorizado.** Requiere autorización humana específica, directorio aislado y plan de rollback antes de cualquier escritura o prueba en Windows. |
| D03-OPEN-08 | `BOUNDARY AND PRECONDITIONS APPROVED — MEMBER EVIDENCE REMAINS TASK-04` | TASK-03 define contrato/referencia del recovery event pero no ejecuta REBIND. Se preservan requestId/admissionId; targetProcessBootId forma parte de la acción canonicalizada; se usa recoveryKey/recoveryEventId R3.1. Se requieren ledger Member íntegro, receipt durable, ausencia demostrable de START_INTENT/ENTERED, boot previo terminado y ausencia de tail/corrupción/contradicción. Si falta cualquier evidencia, no REBIND. | Ledger/evidencia local y demostración de terminación corresponden a TASK-04. TASK-04 no está autorizada. UNKNOWN y RECONCILED_UNKNOWN nunca equivalen a NOT_STARTED. |

## 4. Reglas de detalle derivadas de las decisiones aceptadas

### 4.1 Codec — complemento a §§6 y 15

El codec final deberá cumplir el baseline binario tipado, versionado y length-prefixed de D03-OPEN-01. Los campos de presencia deberán distinguir `ABSENT`, `NULL` y `VALUE` cuando corresponda al tipo. Los digests/hashes deberán usar SHA-256 con domain separation y no autorreferenciar el campo `recordHash`. La fórmula R3.1 `recoveryEventId = SHA-256(canonical(recoveryKey))` se preserva.

Esta sección no inventa tags, byte order, anchos, nombres de domain separator, genesis hash o límites. Son detalles bloqueantes para congelar SPEC/codec y requieren especificación posterior y revisión humana.

### 4.2 Tail — reemplazo de estados “pendientes” en §§7, 9, 10, 14 y 15

La política se considera decidida: ADMIT/COMPLETE son los únicos tipos con recuperación automática definida. Para FENCE, HANDLER_START_INTENT, ADMISSION_REJECTED_EXPIRED, RECONCILE_UNKNOWN, COMMIT_GLOBAL y epoch advance, la presencia como tail íntegro no confirma la operación. Coordinator inicia/quedará `RECOVERY_REQUIRED`, cerrado, sin ACK, sin permiso, sin estado derivado nuevo y sin evento compensatorio. Se requiere decisión/procedimiento manual separado si se pretende resolver uno de esos tipos.

### 4.3 Epoch — complemento a §9

Se adopta el evento provisional `COORDINATOR_EPOCH_ADVANCED`, que transporta como mínimo epoch anterior, nuevo epoch, `startupOperationId` y correlación del Coordinator. El incremento confirmado debe ser exactamente uno. El primer epoch 1 se permite solo para un almacén absolutamente vacío: journal ausente, checkpoint ausente y ningún temporal reservado residual. Cualquier combinación parcial o epoch en tail impide arrancar y queda `RECOVERY_REQUIRED`. El Coordinator no habilita operaciones antes de checkpoint confirmado del nuevo epoch.

La semántica de reinicio, schema final, startup key exacta, overflow y recuperación de cortes se mantienen como detalles abiertos. No se puede inferir que un evento de epoch en tail fue confirmado.

### 4.4 Directory force y atomic move — complemento a §8

Registrar el resultado de directory force usando tres clases: `SUPPORTED_AND_SUCCEEDED`, `UNSUPPORTED` explícito, `FAILED_OR_AMBIGUOUS`. No usar heurísticas para convertir I/O/access denied/ambiguous en unsupported. No fallback a rename/move no atómico. `ATOMIC_MOVE` unsupported o fallido impide confirmar y ACKear; error real o ambiguo después de rename mantiene Coordinator cerrado y exige verificar journal/checkpoint por replay. Directory force unsupported, por sí solo, limita los claims al entorno/escenario efectivamente ensayado y nunca permite afirmar durabilidad frente a pérdida de energía/kernel/disco/hardware.

### 4.5 Temporales residuales — complemento a §§6, 8, 9 y 10

El temporal se crea con `CREATE_NEW` dentro del mismo directorio/filesystem que el checkpoint, bajo un prefijo reservado y correlación de operación/evento. Durante startup, después de obtener el OS lock, todo temporal reservado residual se detecta y registra. No se promueve, reemplaza ni elimina automáticamente; el estado es `RECOVERY_REQUIRED` hasta el procedimiento aprobado. El patrón y los límites finales se fijan en el detalle de formato, sin asumirlos por inferencia.

### 4.6 Estados de recuperación — complemento a §10

- `RECOVERING`: fase transitoria mientras se valida/recupera startup; sin servicio.
- `RECOVERY_REQUIRED`: estructura internamente íntegra, pero existe tail, temporal, decisión, evidencia o semántica no resoluble por políticas aprobadas; sin servicio.
- `FAILED_CLOSED`: corrupción, contradicción, secuencia/hash/schema inválidos o propiedad del almacén/lock no confiable; sin servicio y requiere intervención controlada.

No hay auto-reset. `RECOVERY_REQUIRED → RECOVERING` requiere acción humana explícita documentada y evidencia. `FAILED_CLOSED → RECOVERING` requiere reparación/restauración controlada autorizada y baseline/hash verificable. La clasificación final de lock ocupado legítimamente vs propiedad incierta permanece como detalle por cerrar.

### 4.7 REBIND — complemento a §§4.2 y 7.3

TASK-03 no realiza REBIND. El futuro contrato deberá incluir el boot destino en la acción canonicalizada y conservar requestId/admissionId original. Ledger Member íntegro y durable, receipt durable, ausencia comprobada de START_INTENT/ENTERED en las evidencias pertinentes, boot anterior terminado, y ausencia de tails/corrupción/contradicción son precondiciones acumulativas. Cualquier evidencia ausente, START_INTENT, UNKNOWN o RECONCILED_UNKNOWN impide REBIND. La evidencia Member corresponde a TASK-04.

## 5. Gobernanza de incompatibilidades y refactor

Se incorpora como procedimiento normativo el siguiente flujo, que reemplaza cualquier lectura del addendum de aceptación que pudiera implicar propuesta automática de un refactor mínimo por cada colisión:

1. Registrar la incompatibilidad y su fuente concreta de evidencia.
2. Determinar si es colisión demostrada, riesgo hipotético, falta de información o diferencia estructural no incompatible.
3. Analizar individualmente requisitos, decisiones aprobadas, invariantes, contratos, estados, datos persistidos, journal/checkpoint, replay, idempotencia, concurrencia, seguridad y escenarios afectados.
4. Evaluar alternativas: conservar diseño y adaptar integración; añadir capa de compatibilidad; modificar diseño; refactor; u otra opción justificada. Registrar coste, riesgo, compatibilidad, migración y rollback cuando aplique.
5. Mantener la colisión abierta hasta que exista decisión humana o resolución aprobada. La decisión aceptada no cambia silenciosamente.
6. Tras analizar las incompatibilidades relevantes, elaborar un plan de refactor consolidado con inventario/evidencia, alcance, trazabilidad, alternativas, secuencia/dependencias, compatibilidad y migración/replay/rollback, testing/aceptación, seguridad, riesgos residuales y gates.
7. Someter el plan a revisión/aprobación humana. La autorización para implementar el refactor es un gate separado.

No se presume que todas las colisiones requieran refactor, ni que la solución sea siempre el refactor mínimo. La decisión de no refactor también debe justificarse. La detección de una colisión no revoca automáticamente las ocho decisiones aceptadas.

## 6. Decisiones heredadas que siguen pendientes

La aceptación D03-OPEN-01..08 no resuelve D-VD03-1..4 (TTL, punto temporal, reloj, precisión/tolerancia y renovación pre-consumo), D-ID-1 (encoding/entropía de IDs), D-EVIDENCE-1 (evidencia de muerte de proceso remoto), D-FS-1 (capacidades reales del filesystem) ni D-WIRE-1 (framing/encoding/autenticación final). D-01/D-02/D-03/D-VD03-5 y H-01 se preservan sin reinterpretación. H-01 trata orden vivo; no habilita recovery de tail.

## 7. Criterios actualizados y gates

- T03-00 está completada y aceptada como baseline estático, con limitación de acceso directo al filesystem documentada.
- T03-01 tiene aceptadas las decisiones a nivel de dirección y políticas de tail/temporales; los detalles indicados en §3 y §4 siguen requiriendo especificación y revisión antes de freeze.
- No congelar codec, epoch startup contract ni clasificación exhaustiva de estados/locks hasta completar los detalles abiertos aplicables.
- No ejecutar experimento Windows FileLock/ATOMIC_MOVE/directory force sin autorización separada.
- No implementar tareas T03-02..T03-12 sin aprobación integral del diseño y autorización humana independiente.
- No modificar Member ledger, wire, HTTP gate, WebClient, TLS/CRL ni componentes excluidos.
- No aprobar TASK-04, tareas posteriores ni el gate de factibilidad CRL de producción.

## 8. Estado de la revisión combinada

**DRAFT-5: `VIABLE WITH CONDITIONS — PENDING HUMAN REVIEW`.** T03-00 completada; ocho propuestas aceptadas en los niveles expresados; tail policy y residual temp policy cerradas; detalle de codec/epoch/classification remain open; experimento Windows no autorizado; análisis de incompatibilidades deberá preceder cualquier plan consolidado de refactor. La implementación permanece `NOT AUTHORIZED`.

No se ejecutaron Maven, builds, tests ni experimentos. No se modificó código, POM ni configuración.