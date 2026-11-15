# TASK-03 — Diseño formal del Reducer: decisiones aceptadas

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Tarea:** TASK-03 — Diseño formal del Reducer  
**Subtarea:** T03-06 — Contrato y reglas del reducer  
**Tipo:** Addendum documental de decisiones  
**Estado:** decisiones enumeradas aceptadas; diseño integral y contratos relacionados no congelados  
**Alcance de autorización:** documentación exclusivamente. Este documento no autoriza implementación, cambios funcionales, ejecución de Maven/build/tests ni experimentos.

## 1. Propósito y autoridad

Este documento consolida las decisiones aceptadas durante la validación de diseño T03-06. Se incorpora como documento independiente bajo `docs/crl-feasibility/` para mantener separada la documentación CRL de la documentación propia de ISO8583 Gateway Lab v2.0.

Las decisiones aquí registradas se limitan a los puntos expresamente aceptados. No completan huecos contractuales por inferencia, no sustituyen las fuentes de diseño anteriores y no implican aprobación integral de TASK-03.

Cuando un detalle se declara pendiente, deberá resolverse en el entregable responsable antes de implementar el comportamiento correspondiente.

## 2. Decisiones aceptadas

### VD-T06-01 — HANDLER_START_INTENT

1. `HANDLER_START_INTENT` no constituye un estado independiente de admisión.
2. Se representa como transición/evidencia central confirmada, mediante referencia al evento o atributo del snapshot según el contrato final.
3. No se agrega un estado `START_INTENT_CONFIRMED`.
4. Se mantienen diferenciados:
   - `HANDLER_START_INTENT`: intención durable y consumo del Permit.
   - `HANDLER_ENTERED`: evidencia local del Member.
   - `HANDLER_RETURNED`: evidencia local del Member.
   - `COMPLETE`: confirmación central de escritura HTTP.

La forma concreta de representación queda sujeta al contrato final de eventos y snapshot.

### VD-T06-02 — UNKNOWN

1. T03-06 no genera `UNKNOWN` por inferencia.
2. Un timeout o la ausencia de respuesta a STATUS no generan por sí mismos una transición de dominio.
3. Mientras no exista una transición aprobada que establezca `UNKNOWN`, la admisión permanece `PENDING` y bloquea `COMMIT_GLOBAL`.
4. `RECONCILE_UNKNOWN` solo puede aplicarse cuando `UNKNOWN` ya fue establecido por una fuente de evidencia/transición válida.
5. Queda como gap contractual reconciliar, con D-02 y la evidencia Member de TASK-04, quién establece UNKNOWN, qué evidencia lo habilita y cuál es la operación durable correspondiente.

### VD-T06-03 — Estado y outcome UNKNOWN

1. El Reducer distingue el estado `RECONCILED_UNKNOWN` del outcome `UNKNOWN`.
2. La reconciliación puede modificar la condición de bloqueo de `COMMIT_GLOBAL`, pero nunca convertir el resultado en `COMPLETE`.
3. Los campos y referencias concretos quedan sujetos al CODEC y a la evidencia definida en TASK-04.

### VD-T06-04 — COMMIT_GLOBAL

El Reducer debe preservar las siguientes invariantes:

1. Rechazar `COMMIT_GLOBAL` si existe alguna admisión `PENDING`.
2. Rechazar `COMMIT_GLOBAL` si existe algún `UNKNOWN` no reconciliado.
3. Permitir `RECONCILED_UNKNOWN` únicamente si la reconciliación está confirmada y sus referencias son válidas.
4. Nunca transformar un outcome `UNKNOWN` en `COMPLETE`.
5. Verificar el conjunto de Members y las referencias requeridas respecto del rollout.
6. Si una precondición no se cumple, rechazar sin mutación parcial.

La evidencia externa requerida para `RECONCILE_UNKNOWN` se valida fuera del Reducer. El Reducer consume la transición y sus referencias; no fabrica evidencia externa.

### VD-T06-05 — Atomicidad ante transiciones inválidas

1. El Reducer recibe un snapshot válido y un evento central previamente decodificado, validado y confirmado.
2. Comprueba precondiciones e invariantes de dominio.
3. Si la transición es válida, devuelve un nuevo snapshot lógico.
4. Si es inválida, devuelve un error tipado.
5. Ante error, el snapshot de entrada permanece inalterado.
6. No publica estados intermedios como resultado válido.
7. El Reducer es determinista y libre de efectos: no realiza I/O, no consulta relojes, no genera UUID y no modifica journal ni checkpoint.
8. Los códigos definitivos de error y su representación en el CODEC quedan pendientes.

### VD-T06-06 — Validación de referencias

El Reducer valida la consistencia de las referencias entre entidades antes de aceptar una transición. El alcance acordado incluye:

- `memberId`
- `processBootId`
- `admissionId`
- `requestId`
- `epoch` y rollout
- `eventId` y eventos previos
- claves de idempotencia
- correspondencia entre evento y transición

Una referencia inexistente, contradictoria o perteneciente a otro contexto provoca rechazo tipado sin mutación.

Esta validación se limita a la consistencia de referencias del estado durable. No valida evidencia externa ni reemplaza las responsabilidades de TASK-04.

### VD-T06-07 — Eventos desconocidos y versiones no soportadas

1. Se rechaza todo tipo de evento que no esté contemplado en el contrato soportado.
2. Se rechaza toda versión de schema que el componente no pueda interpretar.
3. No se ignoran campos obligatorios desconocidos ni se aplican defaults para completar datos ausentes.
4. El rechazo produce un error tipado y no modifica el snapshot.
5. Esta decisión no determina por sí misma si Replay debe clasificar el caso como `RECOVERY_REQUIRED` o `FAILED_CLOSED`; esa clasificación pertenece a la política de Replay/recuperación.
6. La compatibilidad futura debe definirse explícitamente en el CODEC; no se presume tolerancia implícita.

### VD-T06-08 — Límites entre Reducer y Replay

**Responsabilidades de T03-07 Replay:**

- Validar la secuencia física y lógica del journal.
- Validar integridad de registros, hashes y referencias de serialización.
- Determinar qué eventos confirmados pueden entregarse al Reducer.
- Comparar el estado reconstruido con el snapshot persistido.

**Responsabilidades de T03-06 Reducer:**

- Aplicar una transición de dominio por vez.
- Validar precondiciones, invariantes y referencias de dominio.
- Devolver un snapshot lógico nuevo o un error tipado.
- No validar offsets físicos, hash-chain, secuencias del journal ni checkpoint.
- No recibir eventos directamente desde el almacenamiento.

Esta separación no congela los contratos completos de Replay ni del CODEC.

### VD-T06-09 — Determinismo de colecciones del snapshot

1. Los mapas e índices se identifican por claves canónicas y su semántica no depende del orden de inserción.
2. Los conjuntos tienen orden determinista.
3. Las listas conservan el orden semántico definido por el dominio.
4. El Reducer no depende del orden accidental de iteración de colecciones.
5. Historias equivalentes que representan el mismo estado lógico deben producir snapshots lógicamente equivalentes.
6. La representación binaria exacta y el orden de serialización pertenecen al CODEC, no al Reducer.

Esta decisión no congela el CODEC.

### VD-T06-10 — Transiciones explícitas por tipo de evento

1. Cada tipo de evento define sus precondiciones, referencias requeridas y efectos sobre el snapshot.
2. Un evento conocido pero inválido para el estado actual se rechaza mediante error tipado.
3. No existe una transición genérica que acepte eventos por defecto.
4. No se infieren transiciones por similitud con otros tipos de evento.
5. La matriz de transiciones identifica los eventos válidos para cada estado y los rechazos correspondientes.
6. La incorporación de un nuevo evento requiere actualizar explícitamente el contrato, la matriz de transiciones y las pruebas.

### VD-T06-32 — Entidades canónicas frente a datos derivados

1. El snapshot contiene entidades canónicas que representan el estado del dominio.
2. Contadores e índices derivados no pueden contradecir las entidades canónicas.
3. Ante una contradicción, prevalecen las entidades canónicas.
4. El Reducer no repara silenciosamente datos derivados inconsistentes en el snapshot de entrada.
5. Una inconsistencia de entrada provoca rechazo tipado.
6. La definición exhaustiva de campos canónicos y derivados queda pendiente del modelo formal del snapshot.

### VD-T06-33 — Consistencia de relaciones entre entidades

1. Toda relación debe apuntar a entidades existentes.
2. Las relaciones deben respetar las cardinalidades y restricciones del modelo de dominio.
3. No se permiten referencias huérfanas.
4. Las relaciones entre Members, admisiones, requests y eventos deben conservar la correspondencia de contexto ya acordada.
5. El Reducer no crea relaciones implícitas para completar referencias faltantes.
6. Una relación inconsistente provoca rechazo tipado sin mutación.
7. El modelo formal debe documentar las relaciones y restricciones aplicables.


### VD-T06-11 — Validación del snapshot de entrada

1. Antes de aplicar cualquier evento, el Reducer comprueba que el snapshot de entrada cumple las invariantes estructurales y de dominio necesarias para procesar la transición.
2. El snapshot debe estar completo según el modelo de dominio soportado.
3. Debe respetar las relaciones y cardinalidades definidas.
4. No puede contener referencias huérfanas ni contradicciones entre entidades canónicas.
5. Los datos derivados deben ser consistentes con las entidades canónicas.
6. Si el snapshot es inválido, el Reducer rechaza la operación mediante un error tipado.
7. El Reducer no intenta reparar ni normalizar silenciosamente el snapshot.
8. El snapshot original permanece inalterado.

**Límite:** esta validación no sustituye la integridad física, los hashes ni la validación de serialización, que corresponden a Replay/CODEC.

## 3. Contratos y aspectos que permanecen abiertos

La aceptación de las decisiones anteriores no resuelve ni congela los siguientes puntos:

- Contrato completo del snapshot y enumeración exhaustiva de entidades/campos canónicos y derivados.
- Catálogo completo de eventos, payloads, precondiciones y efectos por transición.
- Matriz exhaustiva de transiciones y errores tipados/códigos definitivos.
- CODEC: tags, anchos, byte order, domain separators, nullability, límites, unknown fields y compatibilidad/versionado.
- Representación final de la confirmación de `HANDLER_START_INTENT` en evento/snapshot.
- Origen, evidencia y operación durable que establecen `UNKNOWN`.
- Contratos completos de Replay y recuperación.
- Evidencia Member y terminación que corresponden a TASK-04.
- Clasificación exhaustiva de estados de recuperación y fallos.

Los puntos abiertos deben resolverse en sus tareas/entregables responsables. No se deben completar por inferencia durante la implementación.

## 4. Límites de alcance y autorización

- Este addendum registra decisiones de diseño aceptadas para T03-06.
- No declara completada ni aprobada integralmente TASK-03.
- No autoriza implementación del Reducer, Replay, CODEC ni almacenamiento durable.
- No autoriza ejecutar Maven, builds, tests, benchmarks ni experimentos.
- No modifica la autorización previa relativa a otras tareas.
- Las decisiones de producción y el production gate CRL permanecen fuera del alcance de este documento y continúan sujetos a sus criterios y evidencias específicas.

## 5. Trazabilidad

Este documento consolida las decisiones VD-T06-01 a VD-T06-11, VD-T06-32 y VD-T06-33 aceptadas durante la conversación de validación de diseño de T03-06.

La numeración se conserva tal como fue aceptada; no se renumeran ni reinterpretan decisiones previas.

---

**Estado final:** addendum documental de decisiones aceptadas. No constituye autorización de implementación ni freeze integral del diseño.
