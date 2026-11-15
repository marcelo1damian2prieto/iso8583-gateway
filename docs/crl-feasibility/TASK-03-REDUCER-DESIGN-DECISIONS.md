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

### VD-T06-12 — Validación del payload del evento

1. CODEC valida la estructura binaria, la versión y la decodificación del evento.
2. El Reducer valida que el payload decodificado contenga los datos de dominio requeridos por el tipo de evento.
3. El Reducer verifica las restricciones semánticas y la consistencia entre campos.
4. Si el payload es semánticamente inválido, rechaza la transición mediante un error tipado y sin modificar el snapshot.
5. El Reducer no completa campos ausentes mediante valores por defecto ni infiere información faltante.

**Límite:** la validación estructural/binaria y la decodificación corresponden al CODEC; la validación semántica del contenido de dominio corresponde al Reducer.

### VD-T06-13 — Idempotencia de aplicación del evento

1. El Reducer no administra por sí mismo la deduplicación del journal.
2. La identidad del evento (`eventId`) debe estar validada antes de invocar el Reducer.
3. El Reducer recibe un evento que Replay determinó como aplicable.
4. El contrato de invocación debe impedir que una repetición accidental de la misma invocación aplique dos veces una transición.
5. La responsabilidad de detectar eventos ya aplicados y controlar el avance corresponde a Replay/checkpoint.
6. Si el contrato no permite determinar si el evento ya fue aplicado, la operación debe detenerse fuera del Reducer, sin inferir un resultado.

**Límite:** se distingue la idempotencia del procesamiento del journal, responsabilidad de Replay/checkpoint, de la aplicación de una transición de dominio. La garantía concreta frente a invocaciones repetidas debe quedar definida en el contrato de integración Reducer/Replay.

### VD-T06-14 — Validación del snapshot resultante

1. Toda transición válida debe producir un snapshot consistente con el modelo de dominio.
2. Deben conservarse las invariantes globales, incluidas las relaciones entre entidades.
3. Los contadores e índices derivados deben seguir siendo consistentes con las entidades canónicas.
4. No pueden quedar referencias huérfanas ni entidades en estados imposibles según el contrato.
5. Si la transición no puede producir un snapshot válido, debe rechazarse mediante error tipado.
6. El snapshot de entrada debe permanecer inalterado ante el rechazo.
7. El Reducer no debe corregir inconsistencias mediante reparaciones implícitas.

**Límite:** el Reducer garantiza la consistencia lógica del resultado. La persistencia durable y la confirmación del nuevo snapshot corresponden a la capa responsable del almacenamiento.

### VD-T06-15 — Inmutabilidad del snapshot

1. El Reducer trata el snapshot de entrada como inmutable.
2. Una transición válida produce un nuevo snapshot lógico.
3. No modifica estructuras compartidas del snapshot original.
4. No conserva referencias mutables que permitan alterar posteriormente el snapshot de entrada.
5. Una transición rechazada no deja modificaciones parciales.
6. La estrategia concreta de implementación (copias, estructuras persistentes u otra alternativa) queda para el diseño técnico.

### VD-T06-16 — Determinismo del Reducer

1. El mismo snapshot lógico y el mismo evento producen el mismo resultado lógico.
2. El resultado no depende del horario, la zona horaria ni el reloj del sistema.
3. No depende de UUID, números aleatorios ni identificadores generados internamente.
4. No depende del orden accidental de iteración de colecciones.
5. No depende de estado global, variables estáticas mutables ni configuración externa cambiante.
6. No realiza I/O ni invoca servicios externos.
7. Cualquier dato externo necesario debe llegar explícitamente en el evento o snapshot.

Esta decisión complementa VD-T06-05 y VD-T06-09 y establece el determinismo como propiedad contractual verificable.

### VD-T06-17 — Política de errores del Reducer

1. Todo rechazo debe expresarse mediante un error tipado.
2. Los errores deben distinguir, como mínimo:
   - Snapshot inválido.
   - Evento o payload inválido.
   - Referencia inconsistente.
   - Transición no permitida para el estado actual.
   - Invariante de dominio incumplida.
   - Versión o tipo de evento no soportado.
3. Un error no debe modificar el snapshot de entrada.
4. El Reducer no debe convertir errores en estados de dominio.
5. No debe capturar errores para continuar con una transición parcial.
6. Los códigos definitivos y su representación serializada quedan pendientes del contrato de errores y del CODEC.

**Límite:** Replay decide cómo clasificar y gestionar los errores durante la recuperación; el Reducer solo informa el resultado de aplicar la transición.

### VD-T06-18 — Separación entre errores recuperables y no recuperables

1. El Reducer informa el tipo de error y el contexto de dominio necesario.
2. No determina si el error es recuperable, reintentable o terminal.
3. No decide si debe repetirse la aplicación del evento.
4. No modifica el estado para facilitar un retry.
5. Replay y la política de recuperación determinan la acción posterior según el tipo de error y la evidencia disponible.
6. Un retry nunca debe implicar aceptar una transición inválida ni omitir validaciones.

**Motivo:** mantener el Reducer como componente determinista de dominio y evitar que incorpore decisiones operativas de recuperación.

### VD-T06-19 — Trazabilidad de la transición aplicada

1. La transición debe quedar asociada al `eventId` que la originó.
2. Debe conservarse la referencia al tipo de evento aplicado.
3. La trazabilidad no debe generar identificadores nuevos ni depender del reloj.
4. No debe duplicar ni sustituir el registro durable del journal.
5. La información de trazabilidad debe ser determinista y coherente con el evento de entrada.
6. La representación concreta en el snapshot queda pendiente del modelo formal y del CODEC.

**Límite:** el Reducer mantiene la relación lógica entre evento y transición; Replay conserva la trazabilidad física y durable del journal.

### VD-T06-20 — Separación entre estado canónico y datos derivados

1. El snapshot debe distinguir entidades y atributos canónicos de índices o datos derivados.
2. Los datos derivados no constituyen una fuente independiente de verdad.
3. Toda inconsistencia entre datos derivados y entidades canónicas debe provocar rechazo tipado.
4. El Reducer debe mantener los datos derivados coherentes después de cada transición válida.
5. No debe reconstruir ni reparar silenciosamente un snapshot de entrada inconsistente.
6. La estrategia concreta de almacenamiento y reconstrucción de índices queda pendiente del diseño técnico.

**Límite:** el Reducer garantiza consistencia lógica del snapshot; Replay y CODEC mantienen sus responsabilidades sobre recuperación y representación durable.

### VD-T06-21 — Alcance de las invariantes del Reducer

1. Las invariantes del Reducer deben estar identificadas explícitamente en el contrato de dominio.
2. Deben distinguirse invariantes estructurales, referenciales y de negocio.
3. Cada transición debe declarar qué invariantes verifica y cuáles preserva.
4. El Reducer debe verificar las invariantes globales aplicables antes y después de cada transición.
5. No debe incorporar reglas de negocio implícitas ni derivadas de convenciones de implementación.
6. Las invariantes aún no definidas deben registrarse como pendientes; no deben completarse por inferencia.

**Límite:** esta decisión define cómo se documentan y verifican las invariantes, no congela todavía el catálogo completo de invariantes del dominio.

### VD-T06-22 — Tratamiento de transiciones sin cambio de estado

1. Una transición válida puede producir un snapshot lógicamente equivalente al de entrada, únicamente cuando el contrato del evento permita explícitamente ese resultado.
2. El Reducer no debe interpretar automáticamente una transición sin cambio como duplicado o evento ya procesado.
3. Si el evento exige modificar el estado y no lo hace, debe rechazarse mediante error tipado.
4. La equivalencia se evalúa sobre el estado lógico, no sobre identidad de objetos ni representación binaria.
5. La detección de duplicados y la decisión de re-aplicar eventos corresponden a Replay/checkpoint.
6. Cada tipo de evento debe declarar explícitamente si admite una transición sin cambio de estado.

**Límite:** esto no habilita idempotencia implícita ni reemplaza la política de deduplicación del journal.

### VD-T06-23 — Orden de aplicación de transiciones

1. El Reducer aplica un único evento por invocación.
2. No recibe ni procesa lotes de eventos.
3. El orden de aplicación lo determina Replay según la secuencia validada del journal.
4. El Reducer no reordena eventos ni intenta resolver dependencias temporales entre ellos.
5. Si un evento no es válido para el snapshot actual, lo rechaza; no busca eventos posteriores para corregir el estado.
6. La aplicación secuencial debe preservar las invariantes definidas para cada transición.

**Límite:** Replay controla el orden y la secuencia; Reducer aplica la transición individual. La concurrencia y sincronización de invocaciones quedan fuera de este contrato.

### VD-T06-24 — Contrato de salida del Reducer

1. El Reducer devuelve un resultado explícito y tipado.
2. El resultado distingue entre transición aplicada y rechazo.
3. En caso de transición aplicada, devuelve el nuevo snapshot lógico.
4. En caso de rechazo, devuelve un error tipado y no expone un snapshot parcialmente modificado.
5. El resultado identifica el `eventId` asociado a la invocación.
6. No incluye confirmación de persistencia durable ni afirma que el evento quedó confirmado en journal.
7. La representación concreta de tipos y estructuras queda pendiente del diseño técnico y del CODEC.

**Límite:** el Reducer informa el resultado lógico de la aplicación; Replay y almacenamiento determinan persistencia, checkpoint y confirmación durable.

### VD-T06-25 — Validación de pertenencia del evento al contexto de operación

1. El Reducer verifica que el evento corresponda al contexto de operación representado por el snapshot.
2. Debe validar las referencias de contexto requeridas, incluyendo `rolloutId`, `epoch` y las identificaciones de operación que el modelo defina.
3. Un evento de otro rollout, epoch u operación se rechaza mediante error tipado.
4. No debe adaptar el snapshot para aceptar un evento de contexto ajeno.
5. No debe inferir pertenencia por similitud de identificadores ni por orden temporal.
6. El conjunto exacto de campos de contexto obligatorios queda sujeto al modelo formal de eventos y snapshot.

**Límite:** Reducer valida consistencia contextual de dominio. Replay conserva la validación de secuencia y procedencia física del registro.

### VD-T06-26 — Consistencia temporal de las transiciones

1. Reducer no utiliza reloj local ni obtiene timestamps durante la aplicación.
2. Las restricciones temporales relevantes deben estar expresadas mediante datos explícitos del evento y/o snapshot.
3. Cuando el dominio requiera orden temporal, Reducer valida los datos temporales disponibles contra las reglas explícitas del contrato.
4. No infiere timestamps faltantes ni corrige secuencias temporales.
5. Una contradicción temporal definida por el contrato produce rechazo tipado.
6. La validación de orden físico del journal y de secuencias de persistencia corresponde a Replay, no al Reducer.

**Límite:** esta decisión no define todavía qué timestamps forman parte de cada evento ni sus reglas de comparación; eso queda pendiente del modelo formal.

### VD-T06-27 — Límites de responsabilidad sobre datos derivados

1. El Reducer puede actualizar índices y datos derivados como consecuencia de una transición válida.
2. Los datos derivados deben poder reconstruirse desde las entidades canónicas, cuando el modelo los defina como reconstruibles.
3. El Reducer no debe introducir datos derivados que no estén especificados por el modelo.
4. Si un dato derivado es obligatorio para mantener una invariante, su actualización forma parte de la transición.
5. La reconstrucción de índices durante recuperación o carga inicial corresponde a la estrategia de Replay/snapshot, no a una reparación implícita del Reducer.
6. Los datos derivados no pueden alterar el significado de las entidades canónicas.

**Límite:** la decisión no define todavía el catálogo de índices, su algoritmo de reconstrucción ni su representación serializada.

### VD-T06-28 — Validación de cardinalidades y unicidad

1. El Reducer valida las cardinalidades definidas por el modelo de dominio.
2. Las relaciones que deban ser uno a uno, uno a muchos o muchos a muchos deben respetar esas restricciones.
3. Las claves que el modelo declare únicas no pueden duplicarse.
4. Una transición que viole cardinalidad o unicidad se rechaza mediante error tipado.
5. No se resuelven duplicados eligiendo arbitrariamente una entidad ni sobrescribiendo datos existentes.
6. Las cardinalidades y claves únicas deben estar explícitamente documentadas; las que no estén definidas quedan pendientes.

**Límite:** la decisión no establece todavía cuáles son las cardinalidades y claves únicas concretas del modelo CRL.

### VD-T06-29 — Validación de invariantes entre Members

1. El Reducer valida las invariantes que relacionen distintos Members dentro del mismo rollout.
2. Debe comprobar que las referencias entre Members correspondan a entidades existentes y compatibles.
3. Las restricciones globales del rollout deben verificarse cuando una transición pueda afectarlas.
4. No debe asumir independencia entre Members si el modelo define relaciones o restricciones compartidas.
5. Una transición que viole una invariante entre Members se rechaza sin mutación parcial.
6. Las reglas concretas de dependencia y coordinación entre Members deben estar explícitas en el modelo de dominio.

**Límite:** Reducer verifica consistencia lógica del snapshot completo; no coordina ejecución concurrente, locks ni persistencia entre Members.

### VD-T06-30 — Alcance de las transiciones que afectan varios Members

1. Una transición individual puede afectar más de un Member únicamente si el contrato del evento lo define explícitamente.
2. El Reducer aplica todos los efectos de esa transición como una única operación lógica.
3. Si alguno de los efectos incumple una precondición o invariante, se rechaza la transición completa.
4. No se admiten actualizaciones parciales de Members.
5. El Reducer no divide una transición multientidad en operaciones independientes ni establece coordinación concurrente.
6. Los eventos que afectan varios Members deben identificar explícitamente las entidades involucradas.

**Límite:** esto define atomicidad lógica dentro del resultado del Reducer; no garantiza atomicidad física ni persistencia durable, responsabilidades de almacenamiento.

### VD-T06-31 — Consistencia de los identificadores y claves de negocio

1. Los identificadores de entidades y las claves de negocio deben respetar las reglas de formato y semántica definidas por el modelo.
2. El Reducer valida que los identificadores obligatorios estén presentes y sean coherentes con el contexto.
3. No genera, normaliza ni reemplaza identificadores.
4. No considera equivalentes identificadores distintos salvo que el contrato defina explícitamente esa equivalencia.
5. Las colisiones de claves únicas se rechazan mediante error tipado.
6. Los formatos, ámbitos de unicidad y reglas de normalización quedan pendientes cuando no estén definidos por el modelo.

**Límite:** Reducer valida consistencia de dominio; CODEC valida la representación y decodificación, y Replay controla la identidad y deduplicación de eventos del journal.

### VD-T06-32 — Entidades canónicas frente a datos derivados

1. El snapshot contiene entidades canónicas que representan el estado del dominio.
2. Contadores e índices derivados no pueden contradecir las entidades canónicas.
3. Ante una contradicción, las entidades canónicas prevalecen como fuente de verdad.
4. Esa precedencia no autoriza al Reducer a descartar la inconsistencia ni a continuar con la transición.
5. El Reducer no repara silenciosamente datos derivados inconsistentes en el snapshot de entrada.
6. Una inconsistencia de entrada provoca rechazo tipado y el snapshot original permanece inalterado.
7. La definición exhaustiva de campos canónicos y derivados queda pendiente del modelo formal del snapshot.

### VD-T06-33 — Consistencia de relaciones entre entidades

1. Toda relación debe apuntar a entidades existentes.
2. Las relaciones deben respetar las cardinalidades y restricciones del modelo de dominio.
3. No se permiten referencias huérfanas.
4. Las relaciones entre Members, admisiones, requests y eventos deben conservar la correspondencia de contexto ya acordada.
5. El Reducer no crea relaciones implícitas para completar referencias faltantes.
6. Una relación inconsistente provoca rechazo tipado sin mutación.
7. El modelo formal debe documentar las relaciones y restricciones aplicables.
8. Esta decisión consolida las reglas de consistencia referencial entre entidades y debe interpretarse junto con VD-T06-06, VD-T06-11, VD-T06-28 y VD-T06-29, respetando el alcance específico de cada una.
9. Su carácter integrador no duplica responsabilidades ni introduce restricciones adicionales no documentadas en el modelo formal.

### VD-T06-34 — Frontera de validación del evento

1. CODEC valida estructura binaria, versión y decodificación.
2. Replay valida integridad del journal, secuencia y condición de evento confirmado.
3. Reducer valida payload semántico, referencias de dominio, precondiciones e invariantes.
4. La expresión «evento previamente validado» en VD-T06-05 se refiere exclusivamente a las validaciones correspondientes a CODEC y Replay.
5. Reducer no delega ni omite sus propias validaciones semánticas por el hecho de recibir un evento confirmado.
6. Ningún componente debe asumir que las validaciones de otro componente sustituyen las propias.

### VD-T06-35 — Validación defensiva del snapshot de entrada

1. Replay entrega al Reducer un snapshot reconstruido según su contrato.
2. Reducer verifica defensivamente las invariantes estructurales y de dominio necesarias antes de aplicar cada evento.
3. La responsabilidad de Replay sobre la reconstrucción correcta no sustituye la validación defensiva del Reducer.
4. Si el snapshot incumple una invariante necesaria, Reducer rechaza la operación mediante error tipado.
5. Reducer no repara ni normaliza el snapshot recibido.
6. El snapshot original permanece inalterado ante el rechazo.
7. Esta validación no transfiere al Reducer las responsabilidades de integridad física, hash-chain, checkpoint o serialización.

**Motivo:** establecer una frontera defensiva entre reconstrucción y aplicación de dominio, sin duplicar las responsabilidades de Replay.

### VD-T06-36 — Alcance de la validación de invariantes globales

1. El contrato identifica explícitamente el conjunto completo de invariantes globales del snapshot.
2. Cada transición declara qué invariantes puede afectar directa o indirectamente.
3. Reducer puede utilizar validación selectiva durante una transición únicamente cuando el contrato permita demostrar que las invariantes omitidas no pueden resultar afectadas.
4. La validación selectiva no reduce ni modifica la garantía contractual de consistencia global.
5. Toda invariante potencialmente afectada debe verificarse antes de aceptar el nuevo snapshot.
6. Si no puede determinarse con certeza qué invariantes pueden verse afectadas, se verifican todas las invariantes globales aplicables.
7. La optimización concreta de estas verificaciones queda para el diseño técnico y no puede introducir falsos negativos.

**Motivo:** permitir optimizaciones sin debilitar las garantías de consistencia del dominio.

### VD-T06-37 — Trazabilidad cuando una transición no modifica el snapshot

1. El resultado del Reducer conserva el `eventId` y el tipo de evento de la invocación.
2. Esta información se conserva incluso cuando la transición válida produce un snapshot lógicamente equivalente al de entrada.
3. La trazabilidad de la invocación no se utiliza como mecanismo de deduplicación.
4. Reducer no mantiene historial de eventos aplicados ni modifica el snapshot para registrar artificialmente una transición sin cambios.
5. Replay conserva la trazabilidad durable y determina si el evento ya fue aplicado.
6. La representación concreta de esta información en el resultado queda sujeta al contrato de salida definido en VD-T06-24.

**Motivo:** separar la trazabilidad de una invocación de la mutación del estado de dominio y de la deduplicación del journal.

### VD-T06-38 — Frontera de validación de referencias a eventos

1. Reducer valida la presencia y consistencia contextual del `eventId` asociado al evento que recibe.
2. Reducer solo valida referencias a eventos previos cuando esas referencias formen parte explícita del snapshot lógico y estén definidas por el modelo de dominio.
3. Reducer no consulta, recorre ni reconstruye el journal para validar referencias o determinar el historial de aplicación.
4. Reducer no determina si un evento ya fue aplicado ni administra la deduplicación de eventos.
5. Replay conserva la responsabilidad de validar el historial, la secuencia del journal y la condición de evento ya aplicado.
6. La validación de referencias de dominio por Reducer no sustituye ni duplica las validaciones de historial y deduplicación correspondientes a Replay.

**Motivo:** delimitar la validación de referencias de dominio y evitar trasladar al Reducer responsabilidades de historial y deduplicación del journal.

### VD-T06-39 — Validez del snapshot de entrada y frontera con Replay

1. Replay entrega al Reducer un snapshot reconstruido según su contrato.
2. La reconstrucción realizada por Replay no implica que el snapshot haya superado las validaciones estructurales y de dominio que corresponden al Reducer.
3. Antes de aplicar cada evento, Reducer valida las invariantes estructurales y de dominio necesarias para procesar la transición.
4. Si el snapshot incumple una invariante necesaria, Reducer rechaza la operación mediante error tipado.
5. Reducer no repara ni normaliza el snapshot recibido.
6. El snapshot original permanece inalterado ante el rechazo.
7. Se mantienen en Replay/CODEC las responsabilidades sobre integridad física, hash-chain, checkpoint y serialización.

**Aclaración de VD-T06-05:** la expresión «recibe un snapshot válido» debe entenderse como un snapshot reconstruido y entregado por Replay, no como uno cuyas invariantes estructurales y de dominio ya hayan sido certificadas para la aplicación de la transición. El Reducer conserva la responsabilidad de realizar esas validaciones antes de aplicar el evento.

**Motivo:** eliminar la ambigüedad sobre la validez de entrada y establecer que la reconstrucción de Replay no sustituye las validaciones defensivas del Reducer.

### VD-T06-40 — Garantía global y estrategia de validación selectiva de invariantes

1. El contrato define explícitamente el conjunto completo de invariantes globales del snapshot.
2. La garantía del Reducer consiste en preservar todas las invariantes globales aplicables.
3. Cada transición identifica las invariantes que puede afectar directa o indirectamente.
4. Reducer puede aplicar validación selectiva únicamente cuando el contrato permita demostrar que las invariantes omitidas no pueden verse afectadas por la transición.
5. Toda invariante potencialmente afectada debe verificarse antes de aceptar el resultado.
6. Ante incertidumbre sobre el alcance de los efectos de una transición, se verifican todas las invariantes globales aplicables.
7. La validación selectiva no reduce la garantía de consistencia global ni puede introducir falsos negativos.

**Aclaración de VD-T06-21 y VD-T06-36:** la obligación contractual es preservar el conjunto completo de invariantes globales aplicables. La estrategia de comprobación puede ser selectiva solo bajo las condiciones demostrables establecidas en esta decisión.

**Motivo:** armonizar la garantía global de consistencia con la posibilidad de optimizar la ejecución de las verificaciones, sin debilitar las propiedades del dominio.

### VD-T06-41 — Metadatos de trazabilidad en el resultado del Reducer

1. `eventId` y tipo de evento son metadatos del resultado de la invocación del Reducer.
2. Estos metadatos se conservan tanto si la transición modifica el snapshot como si produce un resultado lógicamente equivalente al snapshot de entrada.
3. No se incorporan artificialmente al snapshot cuando no corresponden a un cambio del estado de dominio.
4. Replay conserva la trazabilidad durable en el journal.
5. Reducer no administra historial de eventos ni deduplicación.

**Aclaración de VD-T06-19, VD-T06-24 y VD-T06-37:** la asociación lógica entre invocación y evento se expresa en el resultado del Reducer; la trazabilidad durable permanece en el journal bajo responsabilidad de Replay. La equivalencia lógica del snapshot no elimina los metadatos de la invocación.

**Motivo:** precisar la ubicación de la trazabilidad sin introducir mutaciones artificiales en el snapshot ni duplicar responsabilidades de Replay.

### VD-T06-42 — Frontera de soporte de versiones y tipos de evento

1. CODEC reconoce y decodifica únicamente versiones soportadas.
2. Un evento cuya versión no pueda decodificarse no llega al Reducer como evento válido.
3. Reducer valida que el tipo de evento decodificado esté contemplado en su contrato de dominio.
4. Un tipo conocido y decodificable por CODEC, pero no soportado por Reducer, se rechaza mediante error tipado.
5. Replay gestiona el rechazo según su política de recuperación.
6. Ningún componente ignora versiones o tipos desconocidos ni aplica defaults implícitos para aceptarlos.

**Aclaración de VD-T06-07 y VD-T06-34:** CODEC es responsable de la compatibilidad de versión y decodificación; Reducer, de la admisibilidad del tipo de evento en el contrato de dominio; Replay, de gestionar el rechazo durante recuperación. Estas responsabilidades no se sustituyen entre sí.

**Motivo:** evitar que un evento no interpretable o no contemplado por el dominio sea aceptado por tolerancia implícita o por defaults.

### VD-T06-43 — Consolidación de entidades canónicas y datos derivados

1. Las entidades y atributos canónicos constituyen la fuente de verdad del dominio.
2. Los datos derivados deben ser coherentes con las entidades canónicas.
3. Una inconsistencia entre datos canónicos y derivados en el snapshot de entrada provoca rechazo tipado; Reducer no la repara silenciosamente.
4. Tras una transición válida, Reducer actualiza los datos derivados que el modelo defina como necesarios para preservar las invariantes.
5. Los datos derivados reconstruibles no constituyen una fuente independiente de verdad.
6. La reconstrucción de datos derivados durante la carga inicial o recuperación corresponde a Replay/snapshot, no a una reparación implícita del Reducer.
7. El catálogo de campos canónicos y derivados, así como las reglas de reconstrucción, queda pendiente del modelo formal.

**Consolidación de VD-T06-11, VD-T06-20, VD-T06-27 y VD-T06-32:** esta decisión reúne las reglas comunes sobre fuente de verdad, consistencia y actualización de datos derivados. No introduce nuevas restricciones de dominio ni modifica las responsabilidades específicas establecidas en aquellas decisiones.

**Motivo:** evitar interpretaciones divergentes sobre la autoridad de los datos canónicos y los límites del Reducer frente a índices y datos derivados.

### VD-T06-44 — Organización y orden documental de las decisiones

1. Las decisiones aceptadas se presentan en orden numérico ascendente, desde VD-T06-01 hasta VD-T06-43.
2. Se conserva íntegramente el contenido de cada decisión.
3. No se renumeran identificadores ni se alteran referencias cruzadas.
4. Las decisiones aceptadas permanecen separadas de los contratos y aspectos pendientes.
5. La trazabilidad del documento se actualiza para incluir VD-T06-44.
6. La reorganización no modifica el significado ni el estado de aceptación de ninguna decisión anterior.

**Motivo:** facilitar la revisión, cobertura y mantenimiento del documento sin cambiar el contrato acordado.


### VD-T06-45 — Frontera de validación de claves de idempotencia

1. Reducer valida una clave de idempotencia únicamente si forma parte explícita del modelo de dominio.
2. Valida su presencia, formato y consistencia contextual según las reglas definidas por el modelo.
3. Reducer no consulta registros de deduplicación ni historiales de claves utilizadas.
4. No determina si una clave ya fue utilizada para procesar otro evento.
5. Replay conserva la responsabilidad de controlar la deduplicación y determinar la aplicabilidad del evento.
6. La validación de dominio de Reducer no sustituye los controles de idempotencia de Replay.
7. El formato, ámbito de unicidad y reglas de equivalencia de las claves quedan pendientes del modelo formal.

**Aclaración de VD-T06-06, VD-T06-13 y VD-T06-38:** Reducer valida únicamente la consistencia de dominio de las claves que el modelo incorpore explícitamente; Replay controla su uso histórico, deduplicación y aplicabilidad del evento.

**Motivo:** precisar la frontera de responsabilidades sin trasladar al Reducer consultas de historial ni controles de deduplicación.


### VD-T06-46 — Ubicación contractual de la trazabilidad

1. `eventId` y tipo de evento son metadatos del resultado de la invocación del Reducer.
2. Se conservan tanto en transiciones con cambios como en transiciones sin cambios lógicos.
3. No es obligatorio incorporarlos al snapshot como atributos de dominio.
4. El snapshot solo contiene información de trazabilidad si el modelo formal define explícitamente que pertenece a su estado lógico.
5. Replay conserva la trazabilidad durable en el journal.
6. Esta decisión aclara VD-T06-19, VD-T06-24, VD-T06-37 y VD-T06-41, sin modificar sus restantes criterios.

**Motivo:** eliminar la ambigüedad sobre la ubicación de la trazabilidad y evitar que se introduzcan mutaciones artificiales en el estado de dominio.


### VD-T06-47 — Responsabilidad de Replay sobre la aplicación única de eventos

1. Reducer no mantiene estado interno de eventos aplicados ni administra deduplicación.
2. Replay determina si un evento del journal está habilitado para ser aplicado al snapshot reconstruido.
3. Replay debe impedir que un evento ya aplicado vuelva a incorporarse como una nueva transición dentro de la misma progresión de replay.
4. La responsabilidad de controlar el avance de aplicación y su correspondencia con el estado reconstruido pertenece a Replay.
5. Reducer aplica exclusivamente la transición solicitada; no verifica por sí mismo si el evento ya fue aplicado anteriormente.
6. Una invocación repetida del Reducer no constituye un mecanismo válido de deduplicación.
7. El mecanismo técnico para garantizar esta coordinación entre Replay, checkpoint y Reducer queda pendiente del diseño de integración.
8. Esta decisión aclara VD-T06-13, VD-T06-24, VD-T06-37, VD-T06-38 y VD-T06-45.

**Motivo:** establecer quién garantiza la aplicabilidad de un evento sin atribuir al Reducer responsabilidades de persistencia, historial o deduplicación.

### VD-T06-48 — Alcance lógico del snapshot

1. El snapshot representa el estado lógico durable administrado por el Coordinator y reconstruible mediante Replay.
2. Su contenido comprende las entidades y relaciones que el Coordinator necesita para continuar aplicando transiciones válidas: estado del rollout; membership y enrollment asociados al rollout; estado y observaciones de Members conocidas por el Coordinator; `coordinatorEpoch`; y admisiones y estados de admisión bajo responsabilidad durable del Coordinator.
3. No incorpora el journal ni sus metadatos físicos, como secuencias, offsets, hash-chain o posiciones de lectura.
4. No incorpora el historial de eventos como mecanismo de trazabilidad. Esa responsabilidad permanece en Replay y el journal, conforme a VD-T06-46 y VD-T06-47.
5. No incorpora el ledger local del Member ni convierte `PERMIT_RECEIVED` en una escritura central, respetando D-03.
6. El envelope del checkpoint conserva por separado `schemaVersion`, última secuencia confirmada, hash del último registro y metadata de correspondencia.
7. La estructura exacta de entidades, campos, cardinalidades y datos derivados se define en decisiones posteriores.

**Motivo:** separar el estado de dominio del Coordinator de la infraestructura de persistencia y de la evidencia que pertenece al Member.


### VD-T06-49 — Inventario inicial de entidades canónicas del snapshot

Se reconoce el siguiente inventario inicial de entidades canónicas:

| Entidad | Contenido de referencia |
|---|---|
| `CoordinatorEpoch` | Epoch vigente del Coordinator. |
| `Rollout` | Identidad, fase y estado lógico del rollout. |
| `MembershipManifest` | Membership exacto asociado al rollout. |
| `MemberObservation` | Observaciones conocidas por el Coordinator para cada Member del rollout. |
| `Admission` | Admisión identificada por `admissionId`, con su contexto y estado lógico. |

Criterios:

1. Las entidades canónicas son la fuente de verdad del snapshot.
2. Las relaciones entre ellas deben respetar las referencias y cardinalidades del modelo de dominio aprobado en TASK-02.
3. `Admission` representa el estado durable conocido por el Coordinator; no absorbe el ledger local del Member.
4. `TransitionKey` y los índices de idempotencia no se incorporan todavía a este inventario: su representación y reconstrucción deben analizarse junto con T03-10.
5. Este inventario no fija cardinalidades entre rollouts, campos internos ni estructura Java.

**Motivo:** identificar las unidades de estado que el snapshot debe representar sin anticipar detalles que todavía no están definidos en el modelo formal.


### VD-T06-50 — Identidad y referencias entre entidades del snapshot

Se fijan las siguientes identidades y referencias iniciales, conforme al modelo de dominio aprobado en TASK-02:

1. `CoordinatorEpoch` representa el fencing epoch monotónico del Coordinator único. Es estado del Coordinator y se asocia a eventos y permisos; no se modela como una entidad subordinada a cada Member.
2. `Rollout` se identifica por `rolloutId`.
3. `MembershipManifest` se identifica por `(membershipVersion, digest)` y se asocia a `rolloutId`.
4. `MemberObservation` representa la observación de un Member dentro del rollout. La identidad del Member es `(membershipVersion, memberId, processBootId)`. El nombre `MemberObservation` es una unidad lógica de representación del snapshot; no agrega una nueva entidad de dominio al catálogo de TASK-02.
5. `Admission` se identifica por `admissionId`. Conserva las referencias de contexto definidas para la admisión: `requestId`, `memberId`, `processBootId`, epoch, generación/hash, `rolloutId` y transición ADMIT, según el contrato aplicable.
6. Las referencias deben apuntar a entidades existentes y compatibles. No se crean relaciones implícitas ni se infiere identidad por similitud.
7. Esta decisión no fija cardinalidades globales entre rollouts, manifests, Members y admissions, más allá de las relaciones expresamente definidas en TASK-02.

**Motivo:** establecer identidades y referencias para el snapshot sin ampliar el modelo de dominio aprobado ni inventar multiplicidades no definidas.


### VD-T06-51 — Separación entre estado canónico y referencias

Se establecen los siguientes criterios para el estado lógico del snapshot:

1. Las entidades canónicas contienen el estado lógico necesario para reconstruir el dominio conforme al modelo formal.
2. Las referencias entre entidades identifican la entidad referenciada, pero no duplican su contenido como una segunda fuente de verdad.
3. Los identificadores usados como referencias deben coincidir con la identidad canónica definida para la entidad correspondiente.
4. Los índices de consulta y búsqueda son datos derivados; no constituyen fuentes independientes de verdad.
5. Una referencia ausente, inexistente o contradictoria produce un rechazo tipado. El Reducer no repara, completa ni infiere datos faltantes.
6. La decisión no fija la representación concreta de referencias, índices ni el algoritmo de reconstrucción.

**Motivo:** mantener una única fuente de verdad por entidad y evitar divergencias entre estado canónico y estructuras auxiliares, en continuidad con VD-T06-20, VD-T06-32 y VD-T06-43.


### VD-T06-52 — Cardinalidad de las observaciones Member

Se establecen los siguientes criterios para las observaciones de Members en el snapshot:

1. Cada observación corresponde a un Member identificado por `(membershipVersion, memberId, processBootId)`.
2. Las observaciones pertenecen al contexto de un `Rollout`.
3. El `MembershipManifest` determina el conjunto esperado de Members para ese rollout.
4. No se presume que todos los Members tengan una observación disponible en todo momento.
5. Una observación de un Member que no pertenece al manifest correspondiente es inconsistente y debe rechazarse mediante error tipado.
6. No se define todavía si se conserva una única observación vigente o un historial de observaciones. Esa decisión requiere precisar qué estados y evidencias necesita el Coordinator.

**Motivo:** evitar confundir el membership esperado con las observaciones efectivamente recibidas, sin introducir una cardinalidad temporal todavía no definida.


### VD-T06-53 — Relación entre Admission y Rollout

Se establecen los siguientes criterios para la relación entre admisiones y rollouts:

1. Toda `Admission` pertenece a un único `Rollout`, identificado por `rolloutId`.
2. La `Admission` conserva la identidad del Member y su `processBootId`.
3. El Member de la admisión debe pertenecer al `MembershipManifest` asociado al rollout.
4. La admisión conserva el `coordinatorEpoch` bajo el cual fue concedida.
5. Una admisión cuyo rollout, Member, boot o epoch no coincida con el contexto validado se rechaza mediante error tipado.
6. Esta decisión no establece un límite de cantidad de admissions por Member ni por rollout.

**Motivo:** mantener la correlación de la admisión con el contexto que autorizó el permiso, sin introducir límites de concurrencia no definidos.


### VD-T06-54 — Relación entre Admission y sus eventos

Se establecen los siguientes criterios para la relación entre admisiones y eventos:

1. Los eventos asociados a una `Admission` se identifican mediante `EventId`.
2. La relación `AdmissionId 1—N EventId` respeta el modelo de dominio aprobado en TASK-02.
3. Cada evento debe conservar una referencia coherente con la admisión a la que afecta.
4. Un evento que referencia una admisión inexistente o incompatible se rechaza mediante error tipado.
5. El Reducer valida las referencias de dominio disponibles en el snapshot, pero no consulta el journal ni reconstruye el historial de eventos.
6. La secuencia, unicidad y aplicación previa de eventos siguen siendo responsabilidad de Replay.

**Motivo:** mantener la frontera entre la validación de dominio del Reducer y el control del historial durable de Replay.


### VD-T06-55 — Alcance de EventId dentro del snapshot

Se establecen los siguientes criterios respecto de `EventId` y el snapshot:

1. `EventId` no se incorpora automáticamente como colección histórica dentro del snapshot.
2. La referencia lógica al evento que se está aplicando se conserva en el resultado de la invocación del Reducer, conforme VD-T06-46.
3. El snapshot solo contiene referencias a eventos cuando el modelo de dominio las define expresamente como parte del estado lógico.
4. El Reducer no mantiene un registro histórico de eventos aplicados.
5. Replay conserva el journal y determina qué eventos ya fueron aplicados.

**Motivo:** evitar duplicar en el snapshot el historial que pertenece al journal y mantener coherencia con las decisiones de trazabilidad ya aceptadas.


### VD-T06-56 — Consistencia entre Admission y estado de Rollout

Se establecen los siguientes criterios de consistencia para las admisiones:

1. Una `Admission` solo puede pertenecer a un `Rollout` existente en el snapshot.
2. El `rolloutId` de la admisión debe coincidir con el del `MembershipManifest` asociado.
3. El Member de la admisión debe estar incluido en ese manifest.
4. El `coordinatorEpoch` de la admisión debe ser coherente con el contexto de epoch que el contrato de dominio permita para esa transición.
5. El Reducer rechaza referencias o contextos contradictorios mediante error tipado; no corrige la admisión ni cambia su rollout.
6. Las reglas que determinan qué epochs siguen siendo válidos para cada estado de admisión quedan pendientes del contrato de transición correspondiente.

**Motivo:** asegurar consistencia referencial sin introducir una regla de vigencia de epochs que todavía no está definida.


### VD-T06-57 — Conservación de admisiones terminales

Se establecen los siguientes criterios para la representación de admisiones terminales en el snapshot:

1. Una admisión que alcanza `COMPLETE` permanece representada en el estado lógico durable del Coordinator.
2. Una admisión que alcanza `REJECTED_NOT_STARTED` también permanece representada.
3. Los estados terminales no se eliminan automáticamente del snapshot.
4. El Reducer no aplica políticas de retención ni compactación.
5. La eliminación o el archivado de admisiones, si llegara a ser necesario, requiere un contrato explícito fuera de estas decisiones del Reducer.

**Motivo:** evitar que una transición terminal implique pérdida implícita de estado durable o de correlación de admisiones.


### VD-T06-58 — Representación de estados de Admission

Se establecen los siguientes criterios para el estado de cada admisión en el snapshot:

1. El snapshot conserva el estado lógico actual de cada `Admission`.
2. El estado debe pertenecer al catálogo definido por TASK-02 y las decisiones de dominio aprobadas.
3. El Reducer no infiere estados intermedios a partir de ausencia de eventos, timeouts o falta de observaciones.
4. Una transición solo cambia el estado cuando existe un evento explícito y válido que la autoriza.
5. Un estado desconocido o no contemplado se rechaza mediante error tipado.
6. La matriz exacta de transiciones permitidas y sus precondiciones sigue siendo parte del contrato formal pendiente.

**Motivo:** mantener el estado de admisión gobernado por eventos explícitos, sin inferencias temporales ni transiciones implícitas.


### VD-T06-59 — Tratamiento de PENDING, UNKNOWN y RECONCILED_UNKNOWN

Se establecen los siguientes criterios para representar estos estados de admisión:

1. `PENDING` representa una admisión cuyo resultado todavía no está resuelto y que continúa bloqueando `COMMIT_GLOBAL`.
2. `UNKNOWN` representa un resultado desconocido aún no reconciliado y también bloquea `COMMIT_GLOBAL`.
3. `RECONCILED_UNKNOWN` representa un UNKNOWN reconciliado con evidencia suficiente para dejar de bloquear `COMMIT_GLOBAL`, pero su outcome continúa siendo `UNKNOWN`.
4. El Reducer no convierte `UNKNOWN` en `COMPLETE`, ni interpreta `RECONCILED_UNKNOWN` como éxito.
5. La transición a `RECONCILED_UNKNOWN` requiere el evento explícito y las referencias de reconciliación válidas.
6. Se mantienen las responsabilidades de D-02 y las fronteras de evidencia establecidas para Replay.

**Motivo:** reflejar en el snapshot la diferencia entre estado de reconciliación y resultado de negocio, sin alterar las garantías de D-02.


### VD-T06-60 — Consistencia entre estado y outcome de Admission

Se establecen los siguientes criterios para separar el estado de admisión de su outcome:

1. El estado de `Admission` representa su situación dentro del protocolo de coordinación.
2. El outcome representa el resultado conocido de la ejecución asociada.
3. `COMPLETE` es un estado de admisión y outcome confirmado, conforme al contrato aprobado.
4. `RECONCILED_UNKNOWN` es un estado de reconciliación cuyo outcome permanece `UNKNOWN`.
5. El Reducer no deriva el outcome exclusivamente del estado ni modifica uno para hacerlo coincidir con el otro.
6. Las combinaciones válidas entre estados y outcomes deben quedar definidas por la matriz formal de transiciones.

**Motivo:** impedir que la representación del estado de coordinación o reconciliación se confunda con el resultado de ejecución.


### VD-T06-61 — Representación de la evidencia de reconciliación

Se establecen los siguientes criterios para la evidencia asociada a `RECONCILED_UNKNOWN`:

1. `RECONCILED_UNKNOWN` debe conservar una referencia a la evidencia de reconciliación que justificó el cambio de estado.
2. Esa referencia debe identificar el `RecoveryRecord` correspondiente, conforme al modelo de TASK-02.
3. El Reducer valida que la referencia exista en el snapshot y sea compatible con la admisión y el evento de reconciliación.
4. El Reducer no verifica por sí mismo la evidencia externa ni reconstruye el historial durable.
5. La validación de secuencia, unicidad y persistencia del `RecoveryRecord` corresponde a Replay.
6. La forma exacta de la referencia y sus campos queda pendiente del contrato formal.

**Motivo:** evitar que `RECONCILED_UNKNOWN` aparezca como un estado sin justificación trazable, respetando las responsabilidades de Reducer y Replay.


### VD-T06-62 — Cardinalidad de RecoveryRecord y Admission

Se establecen los siguientes criterios para la relación entre registros de recuperación y admisiones:

1. Cada `RecoveryRecord` referencia un único evento original mediante `originalEventId` y conserva la correlación con la admisión definida en TASK-02.
2. Una `Admission` puede tener cero o más `RecoveryRecord` asociados.
3. Cada registro debe corresponder a una acción de recuperación explícita y válida.
4. El Reducer no deduce cardinalidad a partir de eventos ausentes ni crea registros de recuperación.
5. Replay conserva la responsabilidad de validar la unicidad de la clave de recuperación y la secuencia de registros.
6. La relación exacta entre `RecoveryRecord`, `Admission` y `EventId` debe respetar el modelo de TASK-02; esta decisión no agrega una relación que lo contradiga.

**Motivo:** explicitar la multiplicidad sin confundir la referencia lógica que valida Reducer con el control de unicidad durable de Replay.


### VD-T06-63 — Identidad y consistencia de RecoveryRecord

Se establecen los siguientes criterios para la identidad y consistencia de los registros de recuperación:

1. `RecoveryRecord` se identifica por la clave lógica `(originalEventId, recoveryAction)`, conforme TASK-02.
2. `recoveryEventId` identifica el evento durable que registra esa acción de recuperación.
3. La referencia a `originalEventId` debe corresponder al evento original asociado a la admisión.
4. El `recoveryEventId` y el contenido del registro deben ser coherentes con la acción declarada.
5. El Reducer valida las referencias disponibles en el snapshot, pero no calcula ni verifica la clave durable de deduplicación.
6. Replay valida la unicidad, canonicalización, digest y correspondencia durable del registro.

**Motivo:** separar la identidad lógica de la recuperación de la identidad del evento durable que la registra.


### VD-T06-64 — Compatibilidad entre RecoveryRecord y RECONCILED_UNKNOWN

Se establecen los siguientes criterios para la compatibilidad de la evidencia de recuperación:

1. Una transición a `RECONCILED_UNKNOWN` debe referenciar un `RecoveryRecord` cuya `recoveryAction` sea compatible con la reconciliación aplicada.
2. El `originalEventId` del registro debe corresponder al evento que originó el estado `UNKNOWN` de esa admisión.
3. Un `RecoveryRecord` asociado a otra admisión o a otro evento original no puede justificar esa transición.
4. El Reducer valida esas correspondencias mediante las referencias presentes en el snapshot.
5. Replay conserva la validación de unicidad, secuencia y persistencia del registro.
6. La enumeración exacta de acciones de recuperación compatibles queda sujeta al catálogo formal de acciones.

**Motivo:** impedir que una reconciliación válida para una admisión o evento se reutilice para justificar otra.


### VD-T06-65 — Inmutabilidad de RecoveryRecord

Se establecen los siguientes criterios para la inmutabilidad de los registros de recuperación:

1. Un `RecoveryRecord` confirmado es inmutable dentro del estado lógico del snapshot.
2. Una acción posterior de recuperación no modifica ni reemplaza un registro anterior.
3. Una acción adicional válida se representa mediante otro `RecoveryRecord`, con su propia identidad lógica.
4. Dos registros con la misma clave `(originalEventId, recoveryAction)` no pueden coexistir como registros distintos.
5. El Reducer no resuelve conflictos de identidad ni sobrescribe registros; rechaza inconsistencias mediante error tipado.
6. Replay controla la unicidad durable y la detección de duplicados o conflictos.

**Motivo:** preservar la trazabilidad de cada acción de recuperación y evitar que una reconciliación posterior altere silenciosamente evidencia previa.


### VD-T06-66 — Retención de RecoveryRecord en el snapshot

Se establecen los siguientes criterios para la retención de registros de recuperación:

1. Los `RecoveryRecord` confirmados permanecen representados en el estado lógico durable del Coordinator.
2. El Reducer no elimina registros de recuperación por antigüedad, cantidad ni estado de la admisión.
3. La compactación del journal no implica automáticamente eliminar sus referencias del snapshot.
4. El Reducer no implementa políticas de retención ni archivado.
5. Cualquier eliminación futura requiere un contrato explícito que preserve las garantías de reconciliación, unicidad y trazabilidad.

**Motivo:** evitar que la compactación o la finalización de una admisión provoquen pérdida implícita de evidencia de recuperación.


### VD-T06-67 — Consistencia global de las referencias de RecoveryRecord

Se establecen los siguientes criterios para la consistencia referencial de los registros de recuperación:

1. Todo `RecoveryRecord` presente en el snapshot debe referenciar un `originalEventId` existente en el modelo lógico.
2. Su `recoveryEventId` debe ser coherente con el registro que representa.
3. La admisión asociada debe existir y mantener correspondencia con el evento original.
4. No se admiten referencias huérfanas ni asociaciones contradictorias.
5. El Reducer valida la consistencia referencial del snapshot y del registro involucrado en la transición.
6. Replay conserva la validación de integridad histórica del journal, orden, unicidad y persistencia.

**Motivo:** asegurar que la conservación de registros de recuperación no introduzca referencias huérfanas o inconsistencias en el estado lógico.


### VD-T06-68 — Integridad de referencias ante transiciones multiMember

Se establecen los siguientes criterios para transiciones que afectan varios Members:

1. Una transición multiMember valida todas las referencias de los Members afectados antes de producir el nuevo snapshot.
2. Todas las referencias deben corresponder al mismo `Rollout` y contexto de `MembershipManifest`.
3. Si una referencia es inexistente, incompatible o contradictoria, se rechaza la transición completa.
4. No se admite actualizar parcialmente las observaciones de algunos Members.
5. El Reducer devuelve un único resultado lógico: transición aplicada o rechazo tipado.
6. La atomicidad aquí es lógica; no implica persistencia durable ni coordinación concurrente. Esas responsabilidades permanecen fuera del Reducer.

**Motivo:** mantener la garantía de atomicidad lógica multiMember ya aceptada en VD-T06-29 y VD-T06-30, extendiéndola a la validación de referencias.


### VD-T06-69 — Consistencia de MembershipManifest

Se establecen los siguientes criterios para el `MembershipManifest`:

1. El `MembershipManifest` asociado a un `Rollout` es la referencia canónica del conjunto esperado de Members.
2. Su identidad se mantiene mediante `(membershipVersion, digest)`.
3. El manifest es inmutable durante el rollout.
4. Las observaciones y admisiones deben referenciar Members pertenecientes a ese manifest.
5. Un manifest vacío, con IDs duplicados o incompatible con el contexto del rollout se rechaza.
6. El Reducer valida la coherencia lógica; la verificación criptográfica del digest y la integridad de su representación corresponden a CODEC/Replay según sus respectivas fronteras.

**Motivo:** consolidar la función del manifest como referencia canónica de membership y evitar que las transiciones alteren implícitamente el conjunto esperado.


### VD-T06-70 — Inmutabilidad de membership durante el rollout

Se establecen los siguientes criterios para la membership de un rollout existente:

1. El Reducer rechaza cualquier transición que intente modificar `membershipVersion`, `digest` o el conjunto de Members del `MembershipManifest` asociado al rollout.
2. La membership permanece inmutable durante la vida del rollout.
3. Un cambio de membership requiere un contrato explícito que defina su alcance y sus efectos.
4. Ninguna transición puede introducir un cambio de membership como efecto lateral de otra operación.

**Motivo:** hacer efectiva la inmutabilidad establecida en VD-T06-69 y evitar cambios implícitos en el conjunto de Members esperado.


### VD-T06-71 — Identidad de Member dentro del snapshot

Se establecen los siguientes criterios para la identidad de las observaciones Member:

1. La identidad lógica de una observación Member se compone de `(membershipVersion, memberId, processBootId)`.
2. No pueden coexistir dos observaciones con esa misma identidad dentro del mismo rollout.
3. Un `processBootId` diferente representa una identidad de proceso distinta, no una actualización de la observación anterior.
4. Una observación debe corresponder a un Member incluido en el `MembershipManifest`.
5. El Reducer rechaza identidades duplicadas o incompatibles, sin fusionarlas ni reemplazarlas implícitamente.

**Motivo:** preservar la identidad de cada proceso Member y evitar colisiones o sustituciones implícitas de observaciones.

La cardinalidad histórica de observaciones por Member permanece abierta y no queda determinada por esta decisión.


### VD-T06-72 — Reemplazo de observaciones ante reinicio de Member

Se establecen los siguientes criterios para observaciones Member asociadas a distintos procesos:

1. Una observación con un nuevo `processBootId` no reemplaza automáticamente la observación anterior.
2. La transición que incorpora la nueva observación debe estar explícitamente definida en el catálogo de eventos.
3. El Reducer conserva las observaciones anteriores mientras formen parte del estado lógico definido por el modelo.
4. El Reducer no infiere que un proceso anterior terminó únicamente porque aparece otro `processBootId`.
5. La política de retención o eliminación de observaciones anteriores requiere un contrato explícito.

**Motivo:** evitar confundir la identidad de proceso con el ciclo de vida del Member.


### VD-T06-73 — Estado de ciclo de vida de las observaciones Member

Se establecen los siguientes criterios para interpretar y actualizar observaciones Member:

1. Una observación Member no constituye por sí sola prueba automática de que el proceso esté activo o haya finalizado.
2. El Reducer modifica el estado de una observación únicamente ante eventos explícitos definidos por el contrato.
3. El Reducer no deduce la terminación de un proceso por ausencia de mensajes, aparición de otro `processBootId` ni transcurso del tiempo.
4. La identidad del proceso, la observación conocida por Coordinator y el estado real del proceso son conceptos diferenciados.
5. La semántica y el catálogo de estados de observación quedan sujetos al modelo formal y al contrato de eventos.

**Motivo:** evitar inferencias de ciclo de vida no respaldadas por evidencia explícita y mantener la separación entre identidad, observación y estado real del proceso.


### VD-T06-74 — Evidencia para actualizar observaciones Member

Se establecen los siguientes criterios para eventos que actualizan observaciones Member:

1. El evento identifica explícitamente el `rolloutId`.
2. El evento identifica la identidad del Member mediante `membershipVersion`, `memberId` y `processBootId`.
3. El evento identifica el tipo de observación que actualiza.
4. El evento contiene los datos necesarios para aplicar la transición definida por el contrato.
5. El Reducer rechaza eventos cuya identidad no coincida con una observación compatible del snapshot.
6. El Reducer no crea observaciones ni completa campos faltantes por inferencia, salvo que el contrato del evento defina expresamente una operación de alta.

**Motivo:** exigir evidencia explícita y contexto suficiente para actualizar observaciones, evitando altas o modificaciones implícitas.


### VD-T06-75 — Alta explícita de observaciones Member

Se establecen los siguientes criterios para crear observaciones Member:

1. Una observación nueva solo puede crearse mediante un tipo de evento que defina explícitamente la operación de alta.
2. El evento debe incluir la identidad completa del Member y el contexto del rollout.
3. El Member debe pertenecer al `MembershipManifest` asociado al rollout.
4. No puede existir ya una observación con la misma identidad lógica.
5. El alta no implica que el proceso esté activo ni que haya completado ninguna etapa operativa.
6. El Reducer rechaza altas duplicadas o incompatibles sin modificar el snapshot.

**Motivo:** definir las condiciones generales para incorporar observaciones sin inferir estados operativos ni anticipar el nombre o payload del evento.


### VD-T06-76 — Eliminación de observaciones Member

Se establecen los siguientes criterios para eliminar observaciones Member:

1. El Reducer no elimina observaciones por ausencia de actividad, reinicio de proceso, finalización del rollout ni antigüedad.
2. La eliminación de una observación solo puede producirse mediante un evento explícito y un contrato que defina sus precondiciones y efectos.
3. El contrato de eliminación debe preservar las referencias necesarias para mantener la consistencia del snapshot.
4. El Reducer rechaza una eliminación que deje referencias huérfanas o contradiga invariantes del modelo.
5. Esta decisión no establece una retención ilimitada: cualquier política de eliminación futura requiere un contrato explícito.

**Motivo:** evitar eliminaciones implícitas y preservar la consistencia referencial del estado lógico.


### VD-T06-77 — Consistencia de referencias al eliminar una observación

Se establecen los siguientes criterios para eliminar una observación Member:

1. Antes de aceptar una eliminación, el Reducer verifica que ninguna entidad canónica del snapshot conserve referencias obligatorias a la observación eliminada.
2. Si existen referencias obligatorias, la transición se rechaza, salvo que el mismo evento defina explícitamente las modificaciones necesarias para mantener la integridad referencial.
3. Cuando esas modificaciones estén definidas, se aplican como una única transición lógica.
4. No se admiten eliminaciones parciales ni referencias huérfanas.
5. El Reducer rechaza la transición completa si no puede preservar las invariantes y relaciones del modelo.

**Motivo:** complementar VD-T06-76 con una regla explícita de integridad referencial y mantener la atomicidad lógica establecida en VD-T06-30.


## 3. Contratos y aspectos que permanecen abiertos

La aceptación de las decisiones anteriores no resuelve ni congela los siguientes puntos:

- Estructura exacta del snapshot: campos, cardinalidades globales y relaciones adicionales; VD-T06-49 fija un inventario inicial y VD-T06-50 identidades/referencias, sin establecer un esquema exhaustivo.
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

Este documento consolida las decisiones VD-T06-01 a VD-T06-77 aceptadas durante la conversación de validación de diseño de T03-06.

La numeración se conserva tal como fue aceptada; no se renumeran ni reinterpretan decisiones previas.

---

**Estado final:** addendum documental de decisiones aceptadas. No constituye autorización de implementación ni freeze integral del diseño.
