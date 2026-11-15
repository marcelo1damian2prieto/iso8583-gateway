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


### VD-T06-78 — Cierre de consistencia del ciclo de vida de observaciones

Se consolidan los siguientes criterios para toda transición de alta, actualización o eliminación de observaciones Member:

1. La transición respeta la identidad lógica `(membershipVersion, memberId, processBootId)`.
2. La observación corresponde a un Member perteneciente al `MembershipManifest` asociado al rollout.
3. La transición preserva las referencias obligatorias del snapshot.
4. La transición cumple las invariantes globales aplicables, conforme a VD-T06-36 y VD-T06-40.
5. El Reducer produce un nuevo snapshot lógico completo o un rechazo tipado, sin estados parciales.
6. Esta decisión consolida las reglas anteriores; no cierra el catálogo de eventos ni la cardinalidad histórica de observaciones.

**Motivo:** establecer una garantía común de consistencia para el ciclo de vida de observaciones Member, sin anticipar detalles todavía abiertos del modelo formal.


### VD-T06-79 — Representación de RecoveryRecord en el snapshot y referencias EventId

Se establecen los siguientes criterios para representar la evidencia de recuperación en el snapshot:

1. El snapshot no almacena el historial completo de eventos ni duplica el contenido íntegro del journal.
2. Cuando el estado de una `Admission` requiere conservar evidencia de reconciliación, el snapshot mantiene una referencia lógica al `RecoveryRecord` correspondiente.
3. Esa referencia conserva únicamente la identidad y los datos de correlación necesarios para validar la relación con la admisión y el evento original, conforme al modelo de TASK-02.
4. La referencia identifica `originalEventId`, `recoveryEventId` y `recoveryAction`, respetando la identidad `(originalEventId, recoveryAction)` definida en VD-T06-63.
5. El Reducer valida la coherencia de esas referencias con las entidades y el estado lógico del snapshot. No consulta el journal ni reconstruye su historial.
6. Replay conserva la responsabilidad de verificar que los eventos existen realmente en el journal, así como su secuencia, unicidad, canonicalización, digest y persistencia.
7. No se introduce una cardinalidad independiente entre `Admission` y `RecoveryRecord`: las relaciones deben respetar TASK-02, especialmente `AdmissionId 1—N EventId` y `EventId 1—0..N RecoveryRecord`.
8. Esta representación no autoriza compactación del journal, eliminación de eventos ni políticas de retención.

**Motivo:** conservar la evidencia lógica necesaria para el estado reconciliado sin duplicar el historial durable ni trasladar al Reducer responsabilidades de Replay.

La estructura concreta de la referencia dentro de `Admission`, los datos adicionales que eventualmente requiera la validación de dominio, la representación Java, la serialización CODEC y la reconstrucción durante Replay permanecen abiertos.


### VD-T06-80 — Alcance de las referencias EventId en el snapshot y responsabilidad de Replay

Se establecen los siguientes criterios para delimitar las referencias a eventos en el snapshot:

1. `EventId` como identidad de un evento durable pertenece al journal; la conservación y validación de su historial corresponden a Replay.
2. El snapshot no considera `EventId` una entidad canónica por sí misma.
3. El snapshot no mantiene una colección histórica genérica de todos los `EventId` asociados a una `Admission`.
4. El snapshot conserva referencias a identificadores de eventos únicamente cuando sean necesarias para representar o justificar un estado lógico durable definido por el modelo.
5. Para la evidencia de reconciliación, se conservan las referencias `originalEventId` y `recoveryEventId`, junto con `recoveryAction`, conforme a VD-T06-79.
6. La relación `AdmissionId 1—N EventId` definida en TASK-02 se mantiene como relación del modelo durable. Su historial completo y su correspondencia efectiva con los eventos del journal son responsabilidad de Replay; no se reproduce automáticamente como una colección en el snapshot.
7. El Reducer valida la presencia y coherencia contextual de las referencias de eventos que formen parte del estado lógico recibido. No consulta el journal ni determina si un evento existió históricamente.
8. Replay valida la existencia efectiva de los eventos, sus asociaciones con Admission, la secuencia, unicidad, integridad y aplicación previa.
9. El Reducer no genera, sustituye ni elimina identificadores de eventos por inferencia.
10. Esta decisión no modifica la identidad ni la cardinalidad de `RecoveryRecord` definidas en TASK-02 y VD-T06-63.

**Motivo:** distinguir la referencia lógica necesaria para representar el estado durable del historial completo de eventos, evitando convertir el snapshot en un segundo journal o acoplar el Reducer al almacenamiento.

La estructura Java de las referencias, la determinación de si otros estados de Admission requieren referencias adicionales, la representación CODEC y la reconstrucción durante Replay permanecen abiertas. También queda pendiente la matriz definitiva de referencias obligatorias por tipo de evento.


### VD-T06-81 — Distinción semántica entre ADMITTED y PENDING

Se acepta que `ADMITTED` y `PENDING` representan situaciones distintas de una `Admission`:

1. `ADMITTED` representa la autorización durable concedida por el Coordinator para esa admisión.
2. `PENDING` representa una admisión cuyo resultado de ejecución todavía no está resuelto y que, conforme a D-02 y VD-T06-59, bloquea `COMMIT_GLOBAL`.
3. No se consideran sinónimos ni se colapsan en un único estado.
4. La distinción semántica no establece por sí sola que la transición entre ambos sea inmediata, automática ni causada por un evento específico.
5. El evento o condición explícita que establece `PENDING`, su relación temporal con `HANDLER_START_INTENT` y las precondiciones de esa transición permanecen pendientes de definición en la matriz formal.
6. Esta decisión no altera la secuencia de evidencias locales del Member ni convierte `PERMIT_RECEIVED`, `HANDLER_ENTERED` o `HANDLER_RETURNED` en estados centrales de Admission.

**Motivo:** distinguir la concesión durable de autorización del estado de resolución del resultado, sin inferir transiciones que las fuentes aún no especifican.



### VD-T06-82 — Transición de ADMITTED a PENDING al confirmar HANDLER_START_INTENT

Se acepta que la confirmación durable de `HANDLER_START_INTENT` establece la transición lógica de `ADMITTED` a `PENDING` para la Admission correspondiente:

1. `ADMITTED` representa la autorización durable concedida por Coordinator.
2. Cuando `HANDLER_START_INTENT` queda confirmado durablemente conforme a D-01 y al protocolo de persistencia de TASK-03, la Admission pasa a `PENDING`.
3. `PENDING` representa que existe una ejecución autorizada cuyo resultado todavía no está resuelto y, conforme a D-02, bloquea `COMMIT_GLOBAL`.
4. Esta transición no afirma que el handler haya comenzado efectivamente a ejecutarse. `HANDLER_ENTERED` permanece como evidencia local del Member y no se convierte en evento ni estado central.
5. La transición no depende de recibir `PERMIT_RECEIVED`, `HANDLER_ENTERED` ni otra evidencia local adicional para establecer `PENDING`.
6. La confirmación durable de `HANDLER_START_INTENT` sigue siendo el punto de consumo del Permit. La validación de vigencia y la persistencia permanecen dentro de la operación lógica serializada definida por D-01 y §11 de TASK-03 SPEC.
7. Si el intento vence antes del commit durable, no se establece `PENDING`: se aplica la rama `ADMISSION_REJECTED_EXPIRED` conforme a D-VD03-5 y sus condiciones de confirmación.
8. Si la persistencia de `HANDLER_START_INTENT` falla o queda ambigua, no se deriva `PENDING` como transición confirmada ni se habilita una rama alternativa; se conserva el tratamiento bloqueante y fail-closed definido por TASK-03.
9. Esta decisión no define el origen ni la transición que establece `UNKNOWN`, ni la matriz completa de estados y eventos. Tampoco modifica la política de tail: un `HANDLER_START_INTENT` encontrado en tail sin checkpoint continúa en `RECOVERY_REQUIRED` hasta decisión explícita.

**Motivo:** al confirmarse durablemente el consumo del Permit existe una admisión autorizada cuyo resultado aún no se conoce. Establecer `PENDING` en ese punto preserva D-02 sin confundir la intención durable con evidencia de entrada efectiva al handler.




### VD-T06-83 — Establecimiento explícito de UNKNOWN por el Coordinator

Se acepta que el estado `UNKNOWN` se establece mediante una transición explícita y durable del Coordinator, después de evaluar evidencia Member íntegra y correlacionada que, aun siendo válida, no permite determinar un resultado definitivo de ejecución:

1. La autoridad para establecer la transición central a `UNKNOWN` corresponde al Coordinator. La evidencia que permite evaluarla pertenece al Member y no puede ser fabricada por Replay del Coordinator.
2. El Coordinator solo puede confirmar `UNKNOWN` después de validar la integridad y correlación de la evidencia Member disponible con la Admission y el `processBootId` correspondientes, y de persistir durablemente la transición.
3. El nombre de trabajo propuesto para el evento central es `ADMISSION_OUTCOME_UNKNOWN`. Su nombre contractual definitivo, payload, referencias de evidencia, precondiciones exhaustivas y ubicación en el catálogo de eventos quedan pendientes de especificación.
4. La transición no se produce por timeout, falta de respuesta, ausencia de mensajes, vencimiento de un plazo ni resultado de `STATUS`. Tampoco puede inferirse por la mera ausencia de `HANDLER_ENTERED` o `HANDLER_RETURNED`.
5. La transición a `UNKNOWN` no equivale a `COMPLETE`, no modifica el outcome a éxito y mantiene el bloqueo de `COMMIT_GLOBAL` conforme a D-02.
6. La transición no autoriza reintentar ni volver a ejecutar el handler para esa Admission.
7. `RECONCILED_UNKNOWN` continúa siendo un estado distinto: requiere la evidencia durable de muerte o no continuidad y las condiciones de reconciliación aprobadas. La transición a `UNKNOWN` no satisface por sí sola esos requisitos ni libera el bloqueo.
8. Si la evidencia necesaria no está disponible, no es íntegra, no está correlacionada o resulta contradictoria, el Coordinator no confirma esta transición; conserva el tratamiento cerrado que corresponda según la clasificación de recuperación/fallo aprobada.
9. La decisión no define cómo el Member produce, fuerza ni expone la evidencia; tampoco congela el contrato de Replay, la matriz completa de transiciones ni el schema del evento. Esos detalles deben resolverse en las tareas y contratos responsables, en particular TASK-04 para la evidencia Member.

**Motivo:** mantener separadas la producción de evidencia local y la autoridad central de transición, permitiendo representar explícitamente un resultado indeterminado sin convertir timeout o ausencia de observaciones en hechos de dominio, ni relajar el bloqueo establecido por D-02.



### VD-T06-84 — UNKNOWN originado por declaración durable del Member y confirmado por el Coordinator

Se acepta la opción A como dirección de diseño para establecer `UNKNOWN`: el Member registra durablemente una declaración local de resultado indeterminado y el Coordinator, como autoridad de las transiciones centrales, valida esa evidencia y confirma durablemente la transición central correspondiente.

1. La declaración de resultado indeterminado es evidencia local producida y persistida por el Member. No constituye por sí misma una transición central de Admission.
2. El Coordinator valida la integridad y correlación de la evidencia Member con la Admission correspondiente antes de confirmar durablemente `UNKNOWN`.
3. La transición central a `UNKNOWN` se registra mediante un evento explícito del Coordinator. El nombre `ADMISSION_OUTCOME_UNKNOWN` permanece como denominación de trabajo, no como nombre contractual congelado.
4. La declaración local del Member expresa que el resultado de ejecución es indeterminado; no requiere demostrar que el proceso murió ni que la ejecución dejó de ser posible.
5. `UNKNOWN` conserva el bloqueo de `COMMIT_GLOBAL` conforme a D-02. No equivale a `COMPLETE`, no establece éxito y no habilita reintento ni nueva ejecución del handler.
6. `RECONCILED_UNKNOWN` sigue siendo una transición posterior y distinta, sujeta a evidencia durable de muerte o no continuidad y a las condiciones de reconciliación aprobadas. La declaración local que habilita `UNKNOWN` no satisface por sí sola esos requisitos ni libera el bloqueo.
7. El contrato detallado de la declaración local —tipo/nombre definitivo, payload, identidad, referencias, secuencia, integridad, garantías de persistencia y condiciones en que el Member puede emitirla— queda para TASK-04. También quedan pendientes el payload y las precondiciones contractuales completas del evento central y la validación correspondiente en Replay.
8. Esta decisión no autoriza inferir `UNKNOWN` por timeout, ausencia de mensajes o ausencia de `HANDLER_ENTERED`/`HANDLER_RETURNED`; tampoco modifica las responsabilidades de Member, Coordinator y Replay establecidas en las decisiones anteriores.

**Motivo:** hacer explícita la frontera de responsabilidades: el Member aporta evidencia durable sobre su propia ejecución y el Coordinator decide y persiste el estado central. Separar `UNKNOWN` de `RECONCILED_UNKNOWN` evita exigir prueba de muerte del proceso para declarar un resultado indeterminado y evita liberar el bloqueo sin la reconciliación requerida.



### VD-T06-85 — Fallo al producir o persistir la declaración Member de UNKNOWN

Se acepta la combinación B + A + C como dirección de diseño para tratar la imposibilidad del Member de producir o confirmar durablemente su declaración local de resultado indeterminado:

1. **Reintento acotado de la operación de evidencia:** cuando resulte seguro, el Member puede reintentar exclusivamente la producción/persistencia de la declaración local, usando la misma identidad lógica y garantías de idempotencia. Este reintento no es un reintento del handler ni autoriza una nueva ejecución de negocio.
2. **Persistencia no confirmada o ambigua:** mientras no exista confirmación durable verificable de la declaración local, el Coordinator no confirma `ADMISSION_OUTCOME_UNKNOWN` ni deriva `UNKNOWN`. No se interpreta el fallo o la ambigüedad de persistencia como una declaración válida.
3. **Fail-closed:** si no puede confirmarse la evidencia, la Admission conserva el tratamiento bloqueante y se deriva a `RECOVERY_REQUIRED` cuando corresponda conforme a la clasificación de recuperación aprobada. No se habilita `COMMIT_GLOBAL`, retry del handler ni una ejecución alternativa.
4. **Recovery posterior:** si el mecanismo automático no puede resolver el caso, Recovery puede examinar la evidencia disponible y aplicar el procedimiento explícito que corresponda. Solo puede confirmar una resolución si existe evidencia suficiente conforme al contrato aplicable; no puede fabricar la declaración Member ni inferir `UNKNOWN` por ausencia de evidencia. Si la evidencia sigue siendo insuficiente, inaccesible, ambigua o contradictoria, el caso permanece sin resolver y bloqueante.
5. La secuencia B → A → C expresa el orden de tratamiento preferido: reintento idempotente de la operación de evidencia, fail-closed ante falta de confirmación y Recovery si persiste el problema. No implica que todo fallo admita reintento ni fija cantidad, intervalo o política de backoff.
6. El contrato de idempotencia e identidad de la declaración, resolución de escrituras ambiguas, condiciones que habilitan el reintento, clasificación exacta `RECOVERY_REQUIRED`/fallo y procedimiento/autorizaciones de Recovery permanecen pendientes de definición en sus tareas responsables, en particular TASK-04 para la evidencia Member.

**Motivo:** una declaración local solo puede habilitar la transición central si su persistencia es verificable. Reintentar la operación de evidencia puede resolver fallas transitorias sin volver a ejecutar el handler; si no hay confirmación, el tratamiento fail-closed preserva la incertidumbre y Recovery ofrece una vía explícita sin convertir la ausencia de evidencia en un hecho de dominio.



### VD-T06-86 — Resolución de escritura ambigua de la declaración Member de UNKNOWN

Se acepta la combinación A + B + C como dirección de diseño para resolver una escritura ambigua de la declaración local Member de resultado indeterminado:

1. **Identidad lógica estable:** la declaración tiene una identidad lógica estable. Ante una escritura ambigua, cualquier consulta, verificación o reintento se realiza usando esa misma identidad; no se genera una identidad nueva para resolver la ambigüedad.
2. **Consulta por identidad:** como primer paso, el Member consulta el ledger por la identidad de la declaración y verifica el registro encontrado. La mera presencia de bytes o de un registro parcial no constituye confirmación durable.
3. **Verificación del prefijo durable:** la confirmación requiere comprobar que el registro íntegro forma parte del prefijo durable confirmado del ledger, de acuerdo con el contrato de integridad y secuencia que se defina para TASK-04.
4. **Reintento solo ante ausencia demostrada:** si se demuestra que la declaración no forma parte del prefijo durable y que no queda una escritura pendiente o ambigua que pueda incorporarla posteriormente, puede reintentarse la misma operación con la misma identidad lógica e idempotencia. La imposibilidad de demostrar ausencia no habilita el reintento.
5. **Recovery ante incertidumbre:** si no puede determinarse con certeza la presencia, ausencia o integridad de la declaración —incluyendo tail ambiguo, registro parcial, discontinuidad o inconsistencia—, no se confirma la evidencia ni se crea otra identidad. Se conserva el tratamiento bloqueante y se deriva a `RECOVERY_REQUIRED` conforme a la clasificación aplicable.
6. **Sin efectos sobre la ejecución:** este mecanismo resuelve exclusivamente la persistencia de la evidencia Member. No reintenta el handler, no autoriza una nueva ejecución y no habilita `COMMIT_GLOBAL` mientras la declaración no esté confirmada y el estado central no haya sido resuelto conforme al protocolo.
7. La secuencia A → B → C define el orden preferido: consultar por identidad, verificar contra el prefijo durable y recurrir a Recovery cuando la evidencia no permita una conclusión segura. No establece que la consulta por identidad, por sí sola, sea suficiente para demostrar presencia o ausencia.
8. Quedan pendientes para TASK-04 el contrato de identidad e idempotencia, la semántica de consulta, el formato y verificación del prefijo durable, los checkpoints, la detección de escrituras pendientes, las reglas de tail y los criterios exactos para demostrar ausencia. No se fijan aquí algoritmos ni detalles de implementación.

**Motivo:** una escritura ambigua no puede tratarse como éxito ni como fallo por defecto. La identidad estable evita duplicados; la verificación del prefijo durable permite confirmar únicamente evidencia íntegra y comprometida; y Recovery conserva el tratamiento fail-closed cuando la presencia o ausencia no puede demostrarse.



### VD-T06-87 — Representación de UNKNOWN en el snapshot mediante referencia al evento central

Se acepta la opción B para representar en el snapshot una declaración Member de resultado indeterminado que fue validada por el Coordinator y cuya transición central a `UNKNOWN` quedó confirmada durablemente:

1. El snapshot conserva el estado lógico de la Admission como `UNKNOWN` y el outcome correspondiente como `UNKNOWN`, manteniendo la distinción entre estado de coordinación y resultado de ejecución.
2. El snapshot conserva una referencia al evento central del Coordinator que confirmó la transición a `UNKNOWN`. El nombre de trabajo de ese evento continúa siendo `ADMISSION_OUTCOME_UNKNOWN`, sujeto al contrato definitivo del catálogo.
3. El evento central conserva la correlación con la declaración local Member y las referencias de evidencia necesarias conforme al contrato que se defina en TASK-04. El snapshot no copia el contenido completo de la declaración ni el ledger Member.
4. El Reducer valida la consistencia lógica entre el estado/outcome de Admission y la referencia al evento central conforme a la matriz y al contrato de eventos. No consulta ni interpreta directamente el ledger Member.
5. Replay conserva la responsabilidad de validar la existencia, secuencia, integridad y correlación histórica del evento central y de sus referencias de evidencia. La referencia del snapshot no sustituye esa validación ni demuestra por sí sola la existencia de la evidencia local.
6. Esta representación no incorpora una colección histórica genérica de EventId ni convierte el snapshot en un segundo journal. La referencia se conserva porque es necesaria para justificar el estado lógico `UNKNOWN`.
7. La forma exacta del campo/referencia, la identidad y payload definitivos del evento, y el conjunto preciso de referencias Member quedan pendientes de los contratos de TASK-03, Replay y TASK-04. Esta decisión no congela esquema Java ni CODEC.

**Motivo:** mantener el snapshot centrado en el estado durable conocido por Coordinator, conservar trazabilidad hacia la transición central y evitar acoplar el Reducer al formato interno del ledger Member o duplicar el historial durable.

## 3. Contratos y aspectos que permanecen abiertos

La aceptación de las decisiones anteriores no resuelve ni congela los siguientes puntos:

- Estructura exacta del snapshot: campos, cardinalidades globales y relaciones adicionales; VD-T06-49 fija un inventario inicial y VD-T06-50 identidades/referencias, sin establecer un esquema exhaustivo.
- Catálogo completo de eventos, payloads, precondiciones y efectos por transición.
- Matriz exhaustiva de transiciones y errores tipados/códigos definitivos.
- CODEC: tags, anchos, byte order, domain separators, nullability, límites, unknown fields y compatibilidad/versionado.
- Representación final de la confirmación de `HANDLER_START_INTENT` en evento/snapshot.
- Contrato detallado de la declaración local Member para `UNKNOWN` (tipo/nombre, payload, identidad, referencias, secuencia, integridad, garantías de persistencia y condiciones de emisión); idempotencia, consulta por identidad, prefijo durable, checkpoints, detección de escrituras pendientes y resolución de escrituras ambiguas; condiciones de reintento; payload y precondiciones completas del evento central y validación en Replay; campo/referencia exactos del snapshot. La dirección de responsabilidades queda decidida en VD-T06-83 y VD-T06-84; el tratamiento ante fallos de producción/persistencia, en VD-T06-85; la dirección para resolver escrituras ambiguas, en VD-T06-86; la representación lógica en snapshot, en VD-T06-87.
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

Este documento consolida las decisiones VD-T06-01 a VD-T06-135 aceptadas durante la conversación de validación de diseño de T03-06.

La numeración se conserva tal como fue aceptada; no se renumeran ni reinterpretan decisiones previas.

---

**Estado final:** addendum documental de decisiones aceptadas. No constituye autorización de implementación ni freeze integral del diseño.

### VD-T06-88 — Conservación de referencias al reconciliar UNKNOWN

Se acepta que la transición de una Admission desde `UNKNOWN` hacia `RECONCILED_UNKNOWN` conserve en el snapshot ambas referencias necesarias para mantener la trazabilidad de la reconciliación:

1. La referencia al evento central del Coordinator que estableció `UNKNOWN` (`unknownEventId` como nombre descriptivo provisional, no como nombre contractual de campo).
2. La referencia lógica al `RecoveryRecord` de reconciliación, conservando `originalEventId`, `recoveryEventId` y `recoveryAction`, conforme a VD-T06-63 y VD-T06-79.
3. El `originalEventId` del `RecoveryRecord` debe corresponder al mismo evento central que estableció `UNKNOWN` para esa Admission. Las referencias deben ser coherentes entre sí y con la Admission.
4. Tras la transición, el snapshot representa `Admission.state = RECONCILED_UNKNOWN` y mantiene `Admission.outcome = UNKNOWN`.
5. El Reducer valida la presencia y consistencia lógica de ambas referencias con las entidades del snapshot y la transición aplicada. No consulta el journal ni verifica la existencia histórica de los eventos.
6. Replay conserva la responsabilidad de verificar existencia durable, secuencia, integridad, unicidad, correlación histórica y aplicación de los eventos referenciados.
7. Esta decisión no convierte el outcome en `COMPLETE`, no habilita retry/re-ejecución del handler y no introduce una colección histórica genérica de EventId en el snapshot.
8. Los nombres/campos contractuales definitivos y la forma Java/CODEC permanecen abiertos. La referencia `unknownEventId` es una etiqueta descriptiva de esta decisión, no una congelación del esquema.

**Motivo:** preservar la trazabilidad desde el evento central que estableció UNKNOWN hasta la acción durable que justificó su reconciliación, sin duplicar el journal ni trasladar al Reducer responsabilidades de Replay.

### VD-T06-89 — Compatibilidad de recoveryAction con RECONCILED_UNKNOWN

Se acepta que el Reducer solo admita un `RecoveryRecord` como justificación de una transición a `RECONCILED_UNKNOWN` cuando su `recoveryAction` identifique una acción explícitamente habilitada para reconciliar `UNKNOWN`:

1. La acción debe estar incluida en el catálogo formal de acciones de recuperación compatibles con la reconciliación de `UNKNOWN`.
2. La acción debe declarar como estado esperado de origen `UNKNOWN` y como estado resultante `RECONCILED_UNKNOWN`.
3. El Reducer rechaza una acción no catalogada o incompatible, aunque las referencias del `RecoveryRecord` sean estructuralmente válidas.
4. No se aceptan para esta transición acciones destinadas a otros propósitos, incluida `REBIND_UNSTARTED_ADMISSION`.
5. `RECONCILE_UNKNOWN` continúa identificando la operación/evento central del Coordinator. Esta decisión no establece que ese nombre sea el valor literal de `recoveryAction`.
6. La enumeración definitiva de valores de `recoveryAction`, sus payloads y sus referencias queda pendiente del catálogo formal de acciones y de TASK-04.
7. La validación de compatibilidad de dominio corresponde al Reducer; Replay conserva la validación de identidad durable, unicidad, secuencia, integridad y correspondencia histórica del RecoveryRecord.

**Motivo:** impedir que cualquier RecoveryRecord válido pueda reutilizarse indistintamente para justificar una transición a RECONCILED_UNKNOWN.

### VD-T06-90 — Acciones de recuperación adicionales sobre RECONCILED_UNKNOWN

Se acepta que una Admission en estado `RECONCILED_UNKNOWN` pueda recibir acciones de recuperación adicionales, únicamente bajo un contrato explícito para cada acción:

1. La acción adicional debe estar catalogada expresamente como compatible con el estado `RECONCILED_UNKNOWN` y con la Admission involucrada.
2. Cada acción adicional debe conservar su identidad lógica propia `(originalEventId, recoveryAction)` y su correspondiente `recoveryEventId`, conforme al modelo de TASK-02 y VD-T06-63.
3. Todo `RecoveryRecord` confirmado es inmutable. Una acción posterior no puede modificarlo, reemplazarlo ni reutilizar su identidad para un payload diferente.
4. El Reducer valida que la acción adicional sea compatible con `RECONCILED_UNKNOWN`, con la Admission y con el evento original correspondiente.
5. Una acción adicional no puede, por inferencia:
   - Convertir el outcome `UNKNOWN` en `COMPLETE`.
   - Convertir la Admission en `NOT_STARTED` o `REJECTED_NOT_STARTED`.
   - Habilitar un nuevo intento o reejecución del handler.
   - Habilitar `REBIND_UNSTARTED_ADMISSION`.
6. La acción adicional solo puede conservar `RECONCILED_UNKNOWN` como estado resultante cuando su contrato declare expresamente esa transición sin cambio de estado. No se admiten transiciones sin cambio por defecto.
7. Replay conserva la validación de unicidad, secuencia, integridad, persistencia y correspondencia histórica de cada registro adicional.
8. Esta decisión no enumera ni habilita por sí misma acciones concretas; el catálogo formal debe declarar para cada acción sus estados de origen/resultantes, referencias, payload y compatibilidad.

**Motivo:** permitir trazabilidad durable de acciones posteriores legítimas sin alterar reconciliaciones confirmadas, reutilizar identidades ni reabrir ejecución a partir de un resultado que continúa siendo UNKNOWN.

### VD-T06-91 — Idempotencia de acciones de recuperación y nuevas acciones independientes

Se acepta la alternativa C para el tratamiento de una repetición de `RECONCILE_UNKNOWN` y de acciones posteriores sobre una misma Admission:

1. La repetición de una acción de recuperación con la misma `recoveryKey=(originalEventId,recoveryAction)` y el mismo contenido canónico es idempotente: conserva la identidad y el resultado durable originales; no crea un segundo `RecoveryRecord`.
2. La misma `recoveryKey` presentada con contenido canónico diferente constituye un conflicto. Debe rechazarse; no se sobrescribe ni reemplaza el registro confirmado. Replay conserva la validación durable de canonicalización, digest, unicidad y conflicto, aplicando el tratamiento de integridad correspondiente, incluido `FAILED_CLOSED` cuando corresponda según TASK-02/R3.1.
3. Una acción distinta puede producir un `RecoveryRecord` independiente únicamente si esa acción está expresamente catalogada como válida para el estado de origen y la Admission involucrada. Su clave lógica es distinta por el valor de `recoveryAction`.
4. Cada acción adicional conserva su propio `recoveryEventId`, derivado conforme al contrato de identidad/canonicalización aprobado. No se introduce un identificador de intento para eludir la clave lógica vigente.
5. Una repetición idempotente no provoca una nueva transición ni modifica el estado `RECONCILED_UNKNOWN`, el outcome `UNKNOWN` ni las referencias ya confirmadas.
6. Una acción distinta solo puede producir el efecto de estado expresamente declarado en su contrato y validado por el Reducer. No habilita por sí misma reejecución del handler ni convierte `UNKNOWN` en `COMPLETE`.
7. Esta decisión no amplía el conjunto de acciones permitidas ni autoriza recuperación automática adicional. El catálogo formal debe definir las acciones, estados de origen/resultantes, payloads, referencias y precondiciones.

**Motivo:** preservar la identidad `(originalEventId, recoveryAction)`, asegurar idempotencia frente a repeticiones y permitir únicamente acciones adicionales independientes que el dominio haya autorizado explícitamente.

### VD-T06-92 — Rechazo de RecoveryRecord con acción no catalogada o incompatible

Se acepta una política de rechazo estricto en el Reducer y clasificación del incidente en la capa superior:

1. Si un evento intenta aplicar un `RecoveryRecord` cuya `recoveryAction` no figura en el catálogo vigente, el Reducer rechaza la operación y no produce un snapshot nuevo ni modifica el snapshot anterior.
2. Si la acción figura en el catálogo, pero es incompatible con el estado de origen, la Admission, el evento original o el efecto declarado, el Reducer también rechaza la operación.
3. El rechazo del Reducer no determina por sí solo que exista corrupción ni asigna automáticamente un estado operativo global. La capa superior clasifica la situación conforme a la evidencia:
   - `RECOVERY_REQUIRED` cuando la situación requiere resolución explícita y no se ha demostrado una violación de integridad.
   - `FAILED_CLOSED` cuando se detecta corrupción, conflicto de identidad, digest inconsistente o contradicción de integridad, conforme a los criterios formales aplicables.
4. No se permite ignorar, eliminar, reemplazar ni reinterpretar el `RecoveryRecord` como otra acción para continuar el Replay.
5. Mientras la inconsistencia no esté resuelta mediante un procedimiento explícito y válido, no se habilita `COMMIT_GLOBAL` ni se deriva un estado nuevo a partir del registro rechazado.
6. El Reducer conserva la responsabilidad de validar compatibilidad de dominio y rechazar la transición. Replay conserva la validación de identidad, secuencia, canonicalización, digest, integridad y correspondencia durable. La capa superior aplica la clasificación y mantiene el comportamiento cerrado.
7. Los nombres de errores tipados, la clasificación exhaustiva y el procedimiento de resolución quedan pendientes del contrato formal; esta decisión no los inventa ni autoriza recuperación automática.

**Motivo:** evitar tanto la aceptación de estados cuya evidencia de recuperación el Reducer no comprende como la clasificación automática de toda incompatibilidad de catálogo como corrupción, manteniendo la separación de responsabilidades entre Reducer, Replay y Coordinator.

### VD-T06-93 — Matriz de transición T-01: creación de Admission mediante ADMIT

Se acepta la fila T-01 de la matriz formal de transiciones del Reducer, con los siguientes criterios y reservas explícitas:

1. **Origen:** no existe una Admission para la `TransitionKey` correspondiente.
2. **Evento:** `ADMIT`, emitido por Coordinator.
3. **Precondiciones:** `GLOBAL_ACTIVE`; ausencia de un `FENCE` que impida nuevas admisiones; Member, `processBootId`, membership, epoch, rollout, generación y hash válidos y correlacionados.
4. **Identidad:** `TransitionKey=(memberId, processBootId, requestId, ADMIT)`.
5. **Destino:** `Admission.state=ADMITTED`.
6. **Outcome:** esta transición no determina todavía un resultado de ejecución. El valor formal de `outcome` para `ADMITTED` permanece pendiente.
7. **Persistencia y autorización:** Coordinator registra ADMIT durablemente mediante el protocolo append/force/checkpoint aplicable antes de emitir Permit. Permit y ACK solo se emiten después de confirmar la persistencia requerida.
8. **Identidad de Admission:** se crea un `AdmissionId` único asociado al evento ADMIT. No se reutiliza aunque un ADMIT en tail deba abortarse conforme a la política aprobada.
9. **Idempotencia:** misma `TransitionKey` y mismo payload recuperan el resultado previo sin crear otra Admission; misma clave con payload diferente se rechaza como conflicto y no modifica el estado previo.
10. **FENCE:** un ADMIT posterior a un FENCE que cierre nuevas admisiones se rechaza y no emite Permit.
11. **ACK perdido:** se consulta STATUS con la misma clave; no se crea una admisión paralela.
12. **Fallo o ambigüedad de persistencia:** no se emite Permit ni se confirma `ADMITTED` sin la confirmación durable requerida.
13. **Tail sin checkpoint:** se aplica exclusivamente `ADMIT_ABORTED_UNCOMMITTED` bajo las condiciones aprobadas en R3.1 y el catálogo T03-01; no se emite Permit ni se crea otro `AdmissionId`.
14. **Correlación inválida:** Member, boot, epoch u otros datos de contexto inconsistentes provocan rechazo de la admisión.
15. **Replay:** valida identidad, secuencia, referencias, integridad y consistencia del evento con el snapshot.

**Reservas que permanecen abiertas y no quedan decididas por esta aceptación:**

- Valor exacto de `outcome` cuando `state=ADMITTED`.
- Si `ADMITTED` bloquea por sí mismo `COMMIT_GLOBAL` y su integración exacta al conjunto de admissions in-flight.
- Representación exacta de Admission en snapshot y referencias obligatorias del evento ADMIT.
- Contrato definitivo de errores y respuestas.

**Motivo:** aceptar la semántica de creación durable de Admission sin anticipar decisiones de outcome, bloqueo global, estructura del snapshot ni errores que requieren definición específica en el contrato formal.

### VD-T06-94 — Matriz de transición T-02: ADMITTED a PENDING mediante HANDLER_START_INTENT

Se acepta la fila T-02 de la matriz formal de transiciones del Reducer, con las siguientes reglas y reservas:

1. **Estado origen:** `ADMITTED`, que representa una autorización durable concedida por Coordinator.
2. **Operación/evento:** `HANDLER_START_INTENT`, solicitado por Member y confirmado como evento central durable por Coordinator.
3. **Identidad:** `TransitionKey=(admissionId, HANDLER_START_INTENT)`.
4. **Precondiciones:** ADMIT correspondiente confirmado; Admission identificada; Permit no consumido; receipt `PERMIT_RECEIVED` persistido durablemente en el ledger local del Member; identidad de Member y boot correlacionada; ausencia de rechazo previo o consumo anterior.
5. **Serialización:** validación de vigencia y persistencia de START_INTENT constituyen una sola operación lógica serializada con `ADMIT` y `FENCE`, conforme a D-01 y H-01.
6. **Vigencia:** el Permit debe estar vigente en el instante de confirmación durable de START_INTENT. Ese commit es el punto de consumo del Permit.
7. **Estado destino:** la confirmación durable de START_INTENT establece `Admission.state=PENDING`.
8. **Outcome:** el resultado de ejecución continúa sin resolverse. Esta transición no deriva `COMPLETE` ni `UNKNOWN`.
9. **Persistencia:** se requiere append completo y `force(true)` del journal, escritura/force/close del temporal de checkpoint y sustitución mediante `ATOMIC_MOVE`; la confirmación central requiere el checkpoint que referencia la secuencia/hash del evento nuevo, conforme al protocolo de persistencia aprobado. No se publica ACK antes de esa confirmación.
10. **Efecto:** consume el Permit y registra que existe una ejecución autorizada cuyo resultado todavía no está resuelto.
11. **Respuesta:** Coordinator envía ACK al Member solo después de la confirmación durable.
12. **COMMIT_GLOBAL:** `PENDING` bloquea `COMMIT_GLOBAL`, conforme a D-02.
13. **Evidencia local:** `PERMIT_RECEIVED` permanece evidencia del Member. Su persistencia local precede a continuar y solicitar START_INTENT; no se agrega una escritura central redundante. `HANDLER_ENTERED` y `HANDLER_RETURNED` también permanecen evidencias locales y no se convierten en eventos/estados centrales.
14. **Frontera de ejecución:** START_INTENT no acredita que el handler haya comenzado. La ausencia de `HANDLER_ENTERED` no permite inferir que el handler no comenzó.
15. **Expiración posterior:** si el Permit estaba vigente al commit durable de START_INTENT, una expiración posterior no cancela la ejecución ni habilita una nueva validación temporal en HANDLER_ENTERED o al invocar el body.
16. **Expiración previa:** si el Permit venció antes del commit, no se establece PENDING mediante START_INTENT. Coordinator persiste `ADMISSION_REJECTED_EXPIRED`; solo tras confirmar durablemente ese evento deriva `REJECTED_NOT_STARTED` y puede retirar la Admission del conjunto bloqueante. La entrega del ACK comunica el resultado ya persistido y no es condición adicional de la transición.
17. **Fallo/ambigüedad:** si la persistencia de START_INTENT falla o queda ambigua, no se confirma PENDING ni se habilita una rama alternativa; se conserva el tratamiento bloqueante y fail-closed. Para rechazo por expiración, fallo/ambigüedad de su persistencia también conserva el bloqueo.
18. **ACK perdido:** STATUS consulta la misma clave idempotente; no se crea otra transición ni se consume un Permit distinto.
19. **FENCE concurrente:** ADMIT, FENCE y START_INTENT respetan el orden lógico serializado H-01. Un FENCE no revoca una admisión cuyo START_INTENT ya fue confirmado durablemente.
20. **Replay:** valida identidad, secuencia, referencias, integridad y consistencia de la transición con la Admission. No ejecuta el handler ni fabrica evidencia Member.
21. **Tail:** un HANDLER_START_INTENT encontrado en tail sin checkpoint queda en `RECOVERY_REQUIRED`; no se confirma por analogía, no se envía ACK y no se reejecuta el handler automáticamente.

**Escenarios cubiertos:** Permit vigente con commit durable; expiración previa al commit; expiración posterior; ACK perdido y consulta STATUS; persistencia fallida/ambigua; ausencia de receipt local; ausencia de HANDLER_ENTERED; concurrencia con FENCE; HANDLER_START_INTENT en tail sin checkpoint.

**Reservas que permanecen abiertas y no quedan decididas por esta aceptación:**

- Representación exacta de la referencia a START_INTENT en el snapshot.
- Campos y payload contractual definitivo del evento.
- Valor formal de `outcome` mientras la Admission está `PENDING`, si corresponde a un campo explícito.
- Contrato definitivo de errores y clasificación de escrituras ambiguas.
- Evidencia y validación exactas del ledger Member, correspondientes a TASK-04.

**Motivo:** fijar la transición lógica a PENDING en el commit durable que consume el Permit, preservando la distinción entre autorización, intención durable y evidencia local de ejecución, sin anticipar el contrato del snapshot ni los detalles de TASK-04.

### VD-T06-95 — Matriz de transición T-03: PENDING a COMPLETE mediante confirmación central de escritura HTTP

Se acepta la fila T-03 de la matriz formal de transiciones del Reducer, adoptando la opción A: Coordinator confirma centralmente `COMPLETE` a partir de evidencia de que Reactor Netty completó la escritura HTTP en el servidor.

1. **Estado origen:** `PENDING`.
2. **Evento:** `COMPLETE`, confirmado durablemente por Coordinator.
3. **Origen de evidencia:** Reactor Netty/Member informa la finalización de la escritura HTTP en el servidor (`HTTP_RESPONSE_WRITE_COMPLETED`).
4. **Identidad:** `TransitionKey=(admissionId, COMPLETE)`.
5. **Precondiciones:** Admission existente en `PENDING`; `admissionId` y `processBootId` correlacionados; evidencia válida de `HTTP_RESPONSE_WRITE_COMPLETED`; ausencia de contradicciones.
6. **Estado destino:** `Admission.state=COMPLETE`.
7. **Semántica:** `COMPLETE` acredita exclusivamente la finalización de la escritura de la respuesta HTTP en el servidor. No acredita recepción por el cliente ni éxito del resultado de negocio.
8. **Persistencia:** Coordinator persiste el evento `COMPLETE` y confirma el checkpoint correspondiente antes de retirar la Admission del conjunto pendiente.
9. **Efecto:** cierra la Admission y deja de bloquear `COMMIT_GLOBAL`, sujeto a la confirmación durable.
10. **Respuesta:** ACK únicamente después de la confirmación durable. Si se pierde, STATUS consulta la misma clave; no se crea una transición paralela.
11. **Idempotencia:** una repetición con la misma clave y contenido devuelve el resultado anterior, sin crear otro COMPLETE. Contenido contradictorio para la misma clave se rechaza.
12. **Evidencia Member:** esta decisión adopta la opción A y no agrega como requisito una escritura durable adicional en el ledger Member para esta transición. La evidencia de escritura HTTP se informa desde Reactor Netty/Member y la transición central la confirma Coordinator.
13. **Distinción de fronteras:** `HANDLER_RETURNED` es evidencia local de retorno del cuerpo GET y no equivale a `HTTP_RESPONSE_WRITE_COMPLETED`. No se infiere COMPLETE a partir de HANDLER_RETURNED.
14. **Timeout/ausencia de respuesta:** no autorizan a inferir COMPLETE ni generan UNKNOWN automáticamente.
15. **Fallo/ambigüedad de persistencia central:** no se confirma COMPLETE ni se retira la Admission del conjunto bloqueante.
16. **Replay:** valida identidad, secuencia, integridad, referencias y consistencia del evento con la Admission.
17. **Tail:** un COMPLETE en tail sin checkpoint queda sujeto exclusivamente a la política de recuperación aprobada para COMPLETE; esta decisión no amplía esa política por analogía.

**Reserva que permanece abierta:** contrato exacto de validación y correlación que realiza Coordinator sobre la evidencia local recibida de Reactor Netty/Member, sin introducir una segunda escritura central redundante ni convertir `HANDLER_RETURNED` en prueba de write completion. También permanecen sujetos al contrato formal los campos/payload definitivos y los errores tipados.

**Motivo:** mantener COMPLETE como cierre durable central bajo autoridad del Coordinator, distinguiendo el retorno del handler de la finalización de escritura HTTP y evitando afirmar recepción del cliente o éxito de negocio.

### VD-T06-96 — Matriz de transición T-04: ADMITTED a REJECTED_NOT_STARTED por expiración previa al consumo del Permit

Se acepta la fila T-04 de la matriz formal de transiciones del Reducer, con origen `ADMITTED`. `PERMIT_RECEIVED` se considera evidencia local durable del Member y una precondición de la operación, no un estado central de Admission ni un estado de origen del Reducer.

1. **Estado origen:** `ADMITTED`, que representa la autorización durable concedida por Coordinator mediante ADMIT.
2. **Evento:** `ADMISSION_REJECTED_EXPIRED`, registrado y confirmado durablemente por Coordinator al rechazar `HANDLER_START_INTENT` porque el Permit venció antes de su commit durable.
3. **Identidad:** la transición se correlaciona con la Admission y el ADMIT original mediante una clave estable e idempotente. La forma contractual exacta de la clave y del payload permanece sujeta al contrato formal.
4. **Precondiciones:** ADMIT confirmado; Admission identificada; Member, `processBootId`, `requestId` y referencias correspondientes válidos y correlacionados; receipt `PERMIT_RECEIVED` durable en el ledger local del Member; Permit todavía no consumido; ausencia de `HANDLER_START_INTENT` confirmado; y expiración anterior al punto de confirmación durable de START_INTENT.
5. **Punto de decisión temporal:** Coordinator comprueba la vigencia en el instante en que habría de confirmar durablemente `HANDLER_START_INTENT`, dentro de la operación lógica serializada con ADMIT/FENCE conforme a D-01 y H-01. Si el Permit ya venció, no confirma START_INTENT ni establece PENDING.
6. **Evento y persistencia:** Coordinator registra `ADMISSION_REJECTED_EXPIRED` correlacionado con el ADMIT original y la Admission. La transición solo queda confirmada cuando se completa la persistencia durable y el checkpoint correspondiente, conforme al protocolo aprobado.
7. **Estado destino:** únicamente después de esa confirmación durable, `Admission.state=REJECTED_NOT_STARTED`.
8. **Efecto sobre bloqueo:** solo después de la confirmación durable del rechazo la Admission deja de integrar el conjunto bloqueante de WAIT_IN_FLIGHT/COMMIT_GLOBAL. La entrega del ACK al Member no es una condición adicional para derivar el estado ni liberar el bloqueo.
9. **Respuesta:** Coordinator comunica el rechazo únicamente después de la confirmación durable. Si el ACK se pierde, STATUS consulta la misma clave idempotente; no se crea una transición paralela.
10. **Idempotencia:** repetición con la misma clave y el mismo contenido devuelve el resultado durable previo. Misma clave con contenido contradictorio se rechaza; no se crea otro rechazo ni se sobrescribe el resultado.
11. **Fallo de persistencia:** si la persistencia falla, no se confirma `REJECTED_NOT_STARTED` ni se libera el bloqueo. La Admission conserva tratamiento bloqueante y fail-closed.
12. **Persistencia ambigua:** si no puede determinarse si el rechazo quedó confirmado, no se deriva el estado destino ni se libera el bloqueo. La resolución utiliza la misma clave y el procedimiento de STATUS/Recovery aplicable; no se infiere el rechazo por timeout.
13. **Tail sin checkpoint:** un `ADMISSION_REJECTED_EXPIRED` encontrado en tail sin checkpoint no confirma la transición. Conforme a D03-OPEN-02, no tiene recuperación automática; queda `RECOVERY_REQUIRED`, sin ACK, sin derivar `REJECTED_NOT_STARTED` y sin liberar el bloqueo.
14. **Frontera de ejecución:** tras el rechazo confirmado no se permite `HANDLER_ENTERED`, ejecución del cuerpo GET ni `COMPLETE`. No se reintenta automáticamente el handler.
15. **Distinción de evidencia:** `PERMIT_RECEIVED` permanece evidencia local Member, no estado central. Su presencia no consume el Permit ni establece PENDING; la expiración pre-consumo se decide en el commit durable de START_INTENT.
16. **Replay:** valida identidad, secuencia, integridad, referencias y consistencia de la transición con la Admission. No fabrica evidencia local Member ni convierte un tail en confirmación.
17. **Alcance temporal:** esta decisión no fija TTL, fuente de reloj, precisión, tolerancia ni renovación pre-consumo; permanecen pendientes conforme a D-VD03-1..4.

**Escenarios cubiertos:** Permit vencido antes del commit de START_INTENT; Permit vigente en el commit (aplica T-02 y estado PENDING); ACK perdido; repetición idempotente; payload contradictorio; persistencia fallida o ambigua; evento de rechazo en tail sin checkpoint; intento de ejecución posterior al rechazo.

**Motivo:** representar la expiración como rechazo de una Admission todavía ADMITTED, sin introducir PERMIT_RECEIVED como estado central ni permitir que una escritura no confirmada libere el bloqueo. Se preserva el punto de consumo del Permit definido por D-01 y la política de tail aprobada en D03-OPEN-02.

### VD-T06-97 — Frontera Reducer/Member para declaración local de resultado indeterminado

Se acepta el alcance de T-05.1 para separar la evidencia local del Member de la transición central de Admission:

1. El Reducer no procesa las declaraciones del ledger Member como eventos centrales ni las interpreta directamente.
2. La declaración local de resultado indeterminado constituye evidencia del Member. No cambia por sí misma el estado central de Admission ni establece `UNKNOWN`.
3. El Coordinator valida la integridad y correlación de la evidencia local antes de confirmar mediante un evento central explícito la transición a `UNKNOWN`, conforme a VD-T06-83 y VD-T06-84.
4. El Reducer recibe y aplica únicamente la transición central confirmada por Coordinator. Valida las referencias y la coherencia de dominio incluidas en el evento central conforme al contrato formal que se defina.
5. Replay verifica la existencia durable, secuencia, integridad y correlación histórica de las referencias. La referencia conservada en el snapshot no sustituye esa validación.
6. El Reducer no consulta, reconstruye ni interpreta el contenido del ledger Member. La estructura, identidad, secuencia, canonicalización, digest, persistencia e idempotencia de la declaración local corresponden a TASK-04.
7. Esta decisión no fija campos, payload, nombres contractuales, algoritmos de ledger ni referencias concretas todavía no definidas. Tampoco modifica las reglas de VD-T06-85 y VD-T06-86 para fallos o escrituras ambiguas.
8. La falta de declaración local confirmada no autoriza inferir `UNKNOWN` por timeout, ausencia de respuesta o ausencia de evidencia. Se mantiene el tratamiento bloqueante y fail-closed conforme a las decisiones ya aceptadas.

**Motivo:** preservar la autoridad del Coordinator sobre las transiciones centrales y la separación entre Reducer, Replay y ledger Member, sin anticipar el contrato de persistencia que corresponde a TASK-04.

### VD-T06-98 — Validación previa del Coordinator y punto central de establecimiento de UNKNOWN

Se acepta que la validación de la evidencia local por Coordinator sea una condición previa obligatoria a la persistencia del evento central que establece `UNKNOWN`:

1. El Coordinator debe validar la evidencia local Member y su correlación con la Admission antes de intentar confirmar la transición central.
2. La validación satisfactoria es una precondición necesaria, pero no establece por sí misma el estado central `UNKNOWN`.
3. El evento central explícito `ADMISSION_OUTCOME_UNKNOWN` —denominación de trabajo, pendiente de contrato definitivo— debe quedar confirmado durablemente por Coordinator.
4. La confirmación durable de ese evento es el único punto que permite al Reducer aplicar la transición de Admission a `UNKNOWN`.
5. Una declaración local Member, una validación aún no seguida de persistencia central, un timeout o una respuesta de transporte no establecen por sí mismos `UNKNOWN` en el snapshot.
6. Si la validación falla, es insuficiente o contradictoria, no se confirma el evento central ni se deriva `UNKNOWN`; se mantiene el tratamiento bloqueante y fail-closed conforme a las decisiones aceptadas.
7. Si la persistencia central falla o queda ambigua, no se confirma la transición ni se libera el bloqueo. La resolución debe seguir las reglas ya aceptadas para escrituras ambiguas y la misma identidad lógica; no se crea una transición paralela ni se infiere el resultado por timeout.
8. Esta decisión no define todavía el algoritmo exhaustivo de validación, el payload/campos del evento, sus referencias contractuales, su clave idempotente ni los errores tipados. Esos detalles permanecen sujetos al contrato formal y a las dependencias de TASK-04.

**Motivo:** preservar la autoridad exclusiva del Coordinator sobre los cambios centrales y hacer que el evento central durable, no la evidencia local ni la validación aislada, sea el punto de transición del Reducer.

### VD-T06-99 — Alcance conceptual de validación de evidencia UNKNOWN por Coordinator

Se acepta que la validación que realiza Coordinator sobre la declaración local Member, antes de confirmar el evento central que establece `UNKNOWN`, cubra conceptualmente los siguientes cuatro controles:

1. **Identidad:** verificar que la declaración se atribuye al `memberId`, `processBootId` y `admissionId` esperados para la Admission involucrada.
2. **Integridad:** verificar que la evidencia presentada es íntegra y que sus referencias pueden validarse conforme al contrato de evidencia aplicable.
3. **Correlación:** verificar que la declaración corresponde a la misma Admission y a la ejecución autorizada asociada, sin trasladarla a otra solicitud, boot o admisión.
4. **Consistencia:** verificar que la declaración no contradice eventos centrales ya confirmados ni el estado actual de la Admission.

Esta validación no debe demostrar que el proceso haya muerto ni que la ejecución haya dejado de ser posible. Esas condiciones pertenecen a la reconciliación posterior `UNKNOWN → RECONCILED_UNKNOWN`, no a la declaración ni al establecimiento inicial de `UNKNOWN`.

Los cuatro controles fijan el alcance conceptual, no sus mecanismos concretos. La estructura e identidad de la evidencia local, sus secuencias, referencias, canonicalización, digest y garantías de persistencia corresponden a TASK-04; los campos contractuales del evento central, las reglas formales de transición y los errores tipados permanecen sujetos al contrato formal de TASK-03. Esta decisión no inventa algoritmos ni criterios técnicos todavía no aprobados.

**Motivo:** asegurar que el Coordinator solo establezca `UNKNOWN` para evidencia íntegra y atribuible a la ejecución correcta, sin confundir incertidumbre de resultado con prueba de terminación del proceso.

### VD-T06-100 — Estado de origen exclusivo PENDING para transición a UNKNOWN

Se acepta que el evento central `ADMISSION_OUTCOME_UNKNOWN` —denominación de trabajo, pendiente de contrato definitivo— solo pueda establecer la transición `PENDING → UNKNOWN`:

1. El único estado de origen válido para esta transición es `PENDING`.
2. Una Admission en `ADMITTED` no puede pasar directamente a `UNKNOWN` mediante este evento. La autorización durable por sí sola no satisface el estado de origen.
3. Una Admission en `COMPLETE` o `REJECTED_NOT_STARTED` tampoco puede pasar a `UNKNOWN` mediante este evento.
4. La repetición de una operación ya confirmada debe resolverse conforme a la identidad idempotente y al contenido de la transición original, sin crear una segunda transición ni modificar el snapshot confirmado.
5. Si el estado actual de la Admission no es compatible con `PENDING → UNKNOWN`, el Reducer rechaza la transición y no produce un snapshot nuevo ni modifica el anterior.
6. Esta decisión no define la clave idempotente ni los campos contractuales del evento; permanecen pendientes del contrato formal.

**Motivo:** preservar la secuencia de estados aprobada, distinguiendo la autorización `ADMITTED` de la ejecución autorizada aún no resuelta `PENDING` y del resultado indeterminado `UNKNOWN`.

### VD-T06-101 — Clasificación de fallos en la validación de evidencia UNKNOWN por Coordinator

Se acepta distinguir los fallos de validación de la evidencia local Member según la naturaleza de la insuficiencia o contradicción detectada:

1. **Evidencia ausente, inaccesible, incompleta o insuficiente para decidir:** se clasifica como `RECOVERY_REQUIRED`. No se confirma el evento central `ADMISSION_OUTCOME_UNKNOWN`, no se deriva `UNKNOWN` y se conserva el tratamiento bloqueante.
2. **Evidencia corrupta, identidad contradictoria, digest inconsistente o violación de integridad demostrada:** se clasifica como `FAILED_CLOSED`. No se confirma el evento central `ADMISSION_OUTCOME_UNKNOWN`, no se deriva `UNKNOWN` y se conserva el tratamiento bloqueante.
3. En ambos casos, el Reducer no produce un snapshot nuevo a partir de la evidencia rechazada ni modifica el snapshot anterior.
4. Ninguno de estos resultados autoriza liberar la Admission del conjunto bloqueante, reintentar el handler, descartar o corregir automáticamente la evidencia, ni inferir un resultado de ejecución.
5. La resolución requiere el procedimiento explícito aplicable. La clasificación debe respetar las reglas ya aceptadas en VD-T06-85, VD-T06-86 y VD-T06-92, sin sustituirlas ni duplicar sus procedimientos.
6. Esta decisión establece la distinción conceptual de clasificación; los errores tipados, criterios exhaustivos y procedimientos operativos permanecen sujetos al contrato formal y a las tareas responsables.

**Motivo:** diferenciar la falta de evidencia suficiente para resolver de una violación de integridad demostrada, manteniendo en ambos casos el comportamiento fail-closed y la prohibición de establecer UNKNOWN sin validación satisfactoria y confirmación durable del evento central.

### VD-T06-102 — Idempotencia de ADMISSION_OUTCOME_UNKNOWN

Se acepta aplicar las siguientes reglas de idempotencia a la operación central `ADMISSION_OUTCOME_UNKNOWN`:

1. La repetición de la misma operación, con idéntica identidad lógica y contenido canónico, devuelve o reconoce el resultado durable previamente confirmado.
2. Una repetición idempotente no genera un segundo evento central, no produce una segunda transición del Reducer y no modifica el snapshot confirmado.
3. La reutilización de la misma identidad lógica con contenido canónico diferente constituye un conflicto y debe rechazarse.
4. El conflicto no sobrescribe, reemplaza ni reinterpreta el evento original, y no modifica el snapshot confirmado.
5. La identidad lógica debe estar vinculada a la Admission correspondiente y mantenerse estable entre reintentos de comunicación; no se crea una identidad nueva por cada intento.
6. La forma exacta de la identidad, los campos del contenido canónico, la canonicalización y la clasificación/error tipado del conflicto permanecen pendientes del contrato formal.

**Motivo:** asegurar que reintentos de comunicación no dupliquen la transición durable a UNKNOWN ni permitan alterar su contenido mediante reutilización de identidad.

### VD-T06-103 — Referencias de evidencia en ADMISSION_OUTCOME_UNKNOWN y snapshot

Se acepta separar las referencias conservadas por el evento central `ADMISSION_OUTCOME_UNKNOWN` de la referencia mínima mantenida en el snapshot:

1. El evento central debe conservar referencias suficientes para identificar y correlacionar:
   - la Admission afectada;
   - el Member y `processBootId` a los que se atribuye la declaración;
   - la declaración local Member que el Coordinator validó;
   - el propio evento central durable que establece `UNKNOWN`;
   - la ejecución autorizada correspondiente y las relaciones necesarias para comprobar esa correlación.
2. Estas referencias permiten a Replay verificar la existencia, integridad, secuencia y correlación histórica de la evidencia y del evento central, conforme a sus responsabilidades.
3. El snapshot conserva únicamente la referencia al evento central que estableció `UNKNOWN`, conforme a VD-T06-87. No duplica el contenido de la declaración local ni el ledger Member.
4. El Reducer valida la coherencia de dominio de la referencia al evento central presente en el snapshot, conforme al contrato formal; no consulta ni interpreta directamente el ledger Member.
5. Los nombres, tipos, cardinalidades y formato contractuales de los campos y referencias, así como las reglas exactas de validación de Replay, permanecen pendientes de definición formal.

**Motivo:** mantener trazabilidad suficiente para validación histórica sin duplicar evidencia local en el snapshot ni trasladar al Reducer responsabilidades propias de Coordinator o Replay.

### VD-T06-104 — Rechazo de transición UNKNOWN ante referencias inválidas

Se acepta que el Reducer rechace la transición `ADMISSION_OUTCOME_UNKNOWN` cuando las referencias requeridas por el contrato formal estén ausentes, sean inválidas o resulten contradictorias:

1. Si la referencia a la Admission no existe en el snapshot o no corresponde al evento, el Reducer rechaza la transición.
2. Si la identidad Member o `processBootId` contradice la identidad/correlación de la Admission, el Reducer rechaza la transición.
3. Si falta una referencia a la declaración local Member que el contrato exige para el evento, el Reducer rechaza la transición. El Reducer no consulta el ledger Member para suplirla.
4. Si las referencias presentes son inconsistentes entre sí o con el estado lógico del snapshot, el Reducer rechaza la transición.
5. El rechazo no produce un snapshot nuevo ni modifica el snapshot anterior.
6. La capa superior clasifica la situación como `RECOVERY_REQUIRED` cuando la información requerida falta, es inaccesible o no puede verificarse suficientemente; clasifica como `FAILED_CLOSED` cuando se demuestra una contradicción de identidad o integridad.
7. Replay conserva la responsabilidad de verificar existencia durable, secuencia, integridad y correlación histórica de las referencias. La validación del Reducer sobre el snapshot no sustituye esa verificación.
8. Los campos contractuales, las reglas exhaustivas de obligatoriedad y los errores tipados permanecen pendientes de definición formal.

**Motivo:** impedir que una transición a UNKNOWN se aplique sobre referencias incompletas o contradictorias, sin trasladar al Reducer la consulta del ledger Member ni la verificación histórica propia de Replay.

### VD-T06-105 — Efecto de UNKNOWN sobre COMMIT_GLOBAL

Se acepta formalizar el efecto bloqueante del estado `UNKNOWN` sobre la confirmación global:

1. Una Admission en estado `UNKNOWN` permanece dentro del conjunto de Admissions bloqueantes.
2. Mientras exista una Admission `UNKNOWN` bloqueante, Coordinator no puede confirmar `COMMIT_GLOBAL`.
3. La transición explícita y durable `UNKNOWN → RECONCILED_UNKNOWN` puede retirar esa Admission del conjunto bloqueante únicamente cuando satisface las condiciones de reconciliación ya aceptadas y el contrato formal aplicable.
4. La reconciliación no convierte el outcome: la Admission permanece con `outcome = UNKNOWN`; no se considera `COMPLETE` ni éxito.
5. No se puede retirar el bloqueo por timeout, STATUS, ausencia de mensajes, falta de respuesta ni inferencia del Coordinator.
6. Esta decisión no altera las reglas generales de D-02 ni la distinción de VD-T06-59; tampoco define por sí sola el conjunto completo de estados que bloquean `COMMIT_GLOBAL`.

**Motivo:** asegurar que un resultado indeterminado no se trate como terminado y que el desbloqueo dependa de una reconciliación explícita, durable y validada.

### VD-T06-106 — Precondiciones de la transición UNKNOWN a RECONCILED_UNKNOWN

Se acepta que el Reducer aplique la transición `UNKNOWN → RECONCILED_UNKNOWN` únicamente cuando se satisfagan todas las precondiciones siguientes:

1. La Admission existe en el snapshot y su estado actual es `UNKNOWN`.
2. El evento central `RECONCILE_UNKNOWN` fue confirmado durablemente por Coordinator.
3. El evento de reconciliación referencia el mismo evento central que estableció `UNKNOWN` para esa Admission.
4. Existe un `RecoveryRecord` válido, asociado a esa Admission y al evento que originó UNKNOWN.
5. La `recoveryAction` del registro está expresamente incluida en el catálogo de acciones habilitadas para reconciliar UNKNOWN y es compatible con el estado de origen, la Admission y el efecto declarado.
6. Las referencias del evento, de la Admission y del `RecoveryRecord` son coherentes con el snapshot y con la transición solicitada.
7. Si alguna precondición no se satisface, el Reducer rechaza la transición y no produce un snapshot nuevo ni modifica el anterior.

Esta decisión aplica VD-T06-61, VD-T06-64, VD-T06-88 y VD-T06-89. No agrega acciones de recuperación ni define por sí sola sus valores contractuales, payloads, algoritmos o verificaciones históricas de Replay.

**Motivo:** impedir que UNKNOWN se considere reconciliado sin una transición central durable, vinculada al mismo evento UNKNOWN y respaldada por un RecoveryRecord válido con acción expresamente habilitada.

### VD-T06-107 — Conservación del outcome y referencias después de reconciliar UNKNOWN

Se acepta consolidar las siguientes invariantes para el resultado de la transición `UNKNOWN → RECONCILED_UNKNOWN`:

1. El estado resultante de la Admission es `RECONCILED_UNKNOWN`.
2. El outcome de la Admission permanece `UNKNOWN`; la reconciliación no lo convierte en `COMPLETE` ni en éxito.
3. El snapshot conserva la referencia al evento central que estableció `UNKNOWN`.
4. El snapshot conserva la referencia lógica al `RecoveryRecord` que justifica la reconciliación, incluyendo la identidad y correlación requeridas por VD-T06-79 y VD-T06-88.
5. La Admission puede dejar de integrar el conjunto bloqueante para `COMMIT_GLOBAL` únicamente conforme a las condiciones aceptadas en VD-T06-105 y VD-T06-106.
6. Ninguna acción de reconciliación convierte implícitamente el outcome en `COMPLETE`, acredita finalización de ejecución ni habilita reejecución del handler.

Esta decisión consolida VD-T06-59, VD-T06-60, VD-T06-79, VD-T06-88 y VD-T06-105; no redefine las reglas de Replay ni amplía el catálogo de acciones de recuperación.

**Motivo:** preservar la distinción entre estado de coordinación reconciliado y resultado de ejecución todavía desconocido, manteniendo las referencias necesarias para trazabilidad.

### VD-T06-108 — Inmutabilidad del estado RECONCILED_UNKNOWN y sus referencias

Se acepta establecer las siguientes invariantes para una Admission que alcanzó `RECONCILED_UNKNOWN`:

1. La Admission no puede volver a `UNKNOWN`.
2. La Admission no puede pasar a `COMPLETE` mediante una acción posterior de recuperación.
3. La referencia al evento central que estableció `UNKNOWN` no puede modificarse ni reemplazarse.
4. Los `RecoveryRecord` ya confirmados permanecen inmutables; una acción posterior no puede modificar, reemplazar ni eliminar registros anteriores.
5. La Admission puede recibir acciones adicionales únicamente cuando estén expresamente catalogadas y sean compatibles con `RECONCILED_UNKNOWN`, conforme a VD-T06-90 y VD-T06-91.
6. Cada acción adicional debe conservar la inmutabilidad de los registros previos y no alterar el outcome, que permanece `UNKNOWN`.
7. Esta decisión no amplía el catálogo de acciones permitidas ni habilita reejecución del handler.

**Motivo:** impedir que la reconciliación sea tratada como un estado transitorio o que acciones posteriores reescriban la evidencia y la historia durable ya confirmadas.

### VD-T06-109 — Idempotencia de eventos y acciones de reconciliación UNKNOWN

Se acepta establecer las siguientes reglas para las operaciones de reconciliación de una Admission:

1. La repetición de `RECONCILE_UNKNOWN` con la misma identidad lógica y contenido canónico reconoce el resultado durable previamente confirmado.
2. Una repetición idempotente no genera una segunda transición ni crea un nuevo `RecoveryRecord`.
3. La reutilización de la misma identidad lógica con contenido canónico diferente constituye un conflicto y se rechaza; no modifica el evento, el `RecoveryRecord` ni el snapshot previamente confirmados.
4. Una acción de recuperación distinta puede producir otro `RecoveryRecord` únicamente si está expresamente catalogada y es compatible con el estado y la Admission, conforme a VD-T06-90 y VD-T06-91.
5. Una acción adicional válida no modifica ni reemplaza la transición original a `RECONCILED_UNKNOWN`, las referencias previamente confirmadas ni el outcome `UNKNOWN`.
6. La identidad lógica, canonicalización, contenido contractual y clasificación/error tipado de conflictos permanecen pendientes del contrato formal.

**Motivo:** distinguir una repetición idempotente de una acción de recuperación independiente, preservando la inmutabilidad de la reconciliación y de los registros ya confirmados.

### VD-T06-110 — Separación de responsabilidades entre Reducer, Replay, Coordinator, persistencia y capa superior

Se acepta consolidar la siguiente separación de responsabilidades:

1. **Reducer:** valida las precondiciones de dominio y la consistencia lógica del snapshot y del evento central recibido; aplica la transición válida y produce el snapshot resultante. No consulta el journal ni inspecciona directamente el ledger Member.
2. **Replay:** valida la existencia durable, secuencia, unicidad, integridad y correlación histórica de los eventos y referencias que reconstruye. No fabrica evidencia local Member ni sustituye validaciones de dominio del Reducer.
3. **Coordinator:** valida la evidencia local Member y su correlación antes de confirmar eventos centrales que dependan de ella, incluido `ADMISSION_OUTCOME_UNKNOWN`.
4. **Journal / Coordinator:** la persistencia de eventos y la confirmación de durabilidad preceden a la aplicación/confirmación de las transiciones centrales conforme al protocolo aprobado. El Reducer no realiza la persistencia.
5. **Capa superior:** clasifica los fallos y determina el tratamiento correspondiente, incluidos `RECOVERY_REQUIRED` y `FAILED_CLOSED`, conforme a los criterios aprobados. El Reducer rechaza entradas inválidas sin asumir el procedimiento de recuperación.

Esta separación no modifica los contratos ni las responsabilidades específicas ya aprobadas en D-01, D-02 y VD-T06-01 a VD-T06-109. Los detalles de interfaces, errores tipados y orden exacto de invocación permanecen sujetos a los contratos formales correspondientes.

**Motivo:** mantener al Reducer como función de transición de dominio sobre eventos centrales confirmados, preservando las fronteras entre validación de evidencia, integridad histórica, persistencia y clasificación operativa.

### VD-T06-111 — Pureza, determinismo y atomicidad del Reducer

Se acepta establecer el siguiente contrato para la operación del Reducer:

1. El Reducer recibe un snapshot de entrada y un evento central.
2. Valida todas las precondiciones de dominio y consistencia antes de producir un resultado.
3. Si la transición es válida, devuelve un snapshot nuevo, íntegro y consistente; no modifica el snapshot de entrada.
4. Si la transición es inválida, devuelve un error tipado y no modifica el snapshot de entrada.
5. El Reducer no expone snapshots parciales ni aplica mutaciones incrementales visibles. La transición se acepta o rechaza como una unidad lógica.
6. Para el mismo snapshot y el mismo evento, el Reducer produce el mismo resultado o la misma clase de rechazo, sin depender de estado externo, reloj, I/O ni efectos secundarios.
7. La pureza y el determinismo del Reducer no sustituyen las validaciones históricas de Replay ni las garantías de persistencia del Journal/Coordinator.

**Motivo:** obtener transiciones reproducibles y atómicas, facilitar Replay determinista y permitir pruebas unitarias aisladas, manteniendo la separación de responsabilidades aprobada en VD-T06-110.

### VD-T06-112 — Igualdad semántica de snapshots para determinismo del Reducer

Se acepta que la igualdad utilizada para verificar el determinismo del Reducer sea semántica y represente el estado lógico, no detalles accidentales de su representación:

1. Dos snapshots son equivalentes cuando representan el mismo estado lógico y conservan las mismas referencias significativas para ese estado.
2. El orden interno de una colección no afecta la igualdad cuando dicho orden no tiene significado de dominio.
3. Las referencias a eventos y a `RecoveryRecord` forman parte de la igualdad cuando son necesarias para representar o justificar el estado lógico.
4. No se ignoran diferencias de estado, outcome, identidad o correlación.
5. Esta decisión no define canonicalización binaria, serialización ni codec; esos aspectos permanecen sujetos a sus contratos específicos.

**Motivo:** evaluar el determinismo del Reducer sobre el significado del snapshot y no sobre variaciones irrelevantes de representación, sin descartar diferencias relevantes para el dominio.

### VD-T06-113 — Criterios semánticos para colecciones del snapshot

Se acepta establecer los siguientes criterios para las colecciones contenidas en el snapshot:

1. **Colecciones con orden semántico:** el Reducer conserva y valida el orden, porque forma parte del significado de dominio.
2. **Colecciones sin orden semántico:** su igualdad no depende del orden accidental de almacenamiento, conforme al criterio de igualdad semántica de VD-T06-112.
3. **Colecciones con identidad propia:** no admiten elementos con identidad duplicada ni elementos contradictorios para una misma identidad.
4. Estos criterios se aplican a las colecciones de Admissions y `RecoveryRecord`, respetando las identidades y cardinalidades ya aprobadas.
5. Esta decisión no selecciona estructuras Java (`List`, `Set`, `Map`) ni define representación, canonicalización o codec.

**Motivo:** preservar el significado de dominio de cada colección, evitando que el orden incidental afecte la igualdad y que identidades duplicadas o contradictorias queden representadas en el snapshot.

### VD-T06-114 — Validación de invariantes globales del snapshot después de cada transición

Se acepta que el Reducer valide las invariantes globales del snapshot resultante después de cada transición válida, además de las precondiciones y reglas específicas del evento:

1. Unicidad de las identidades de Admission.
2. Consistencia entre estado y outcome de cada Admission.
3. Integridad y coherencia de las referencias a eventos y `RecoveryRecord`.
4. Ausencia de referencias huérfanas o contradictorias.
5. Consistencia del conjunto de Admissions que bloquean `COMMIT_GLOBAL`.
6. Cumplimiento de las cardinalidades y relaciones de dominio aprobadas.

Si alguna invariante global queda violada, el Reducer rechaza el resultado completo, devuelve un error tipado y no produce ni expone un snapshot nuevo. El snapshot de entrada permanece sin modificaciones.

Esta decisión no sustituye las verificaciones históricas de Replay ni amplía las responsabilidades del Reducer definidas en VD-T06-110. La definición exhaustiva y formal de cada invariante y sus errores tipados permanece sujeta al contrato de dominio.

**Motivo:** evitar que una transición localmente válida produzca un snapshot globalmente inconsistente, preservando la atomicidad, pureza y determinismo establecidos en VD-T06-111.

### VD-T06-115 — Invariantes del snapshot genesis

Se acepta que el snapshot inicial (`genesis`) cumpla las invariantes globales establecidas en VD-T06-114, con las siguientes condiciones:

1. El snapshot genesis debe satisfacer las mismas invariantes globales de consistencia que cualquier snapshot resultante de una transición.
2. No puede contener Admissions ni `RecoveryRecord` sin eventos durables que justifiquen su existencia.
3. Sus colecciones deben estar inicializadas y respetar las reglas de identidad, cardinalidad y consistencia de dominio aprobadas.
4. Un genesis inválido se rechaza; no se corrige automáticamente ni se completa mediante inferencias.
5. La creación y persistencia del genesis corresponden al mecanismo de inicialización y no al Reducer.

Esta decisión no define el evento durable de avance de epoch ni modifica las decisiones específicas de inicialización y Replay.

**Motivo:** evitar que el estado inicial introduzca entidades o referencias sin justificación durable y asegurar que el primer snapshot cumpla las mismas garantías de consistencia que los posteriores.

### VD-T06-116 — Validación del snapshot de entrada antes de aplicar un evento

Se acepta que el Reducer valide el snapshot de entrada antes de procesar cada evento central:

1. Si el snapshot de entrada viola una invariante global, el Reducer rechaza la operación.
2. El Reducer no repara, normaliza ni completa por inferencia un snapshot de entrada inválido.
3. El Reducer no procesa el evento sobre un estado inconsistente.
4. El rechazo debe distinguir mediante errores tipados, como mínimo:
   - snapshot de entrada inválido;
   - evento inválido para el estado actual;
   - snapshot resultante inválido.
5. El Reducer no modifica el snapshot recibido.
6. Esta validación complementa la validación de precondiciones de transición y la validación global del snapshot resultante establecidas en VD-T06-111 y VD-T06-114.

La taxonomía definitiva, nombres y payloads de errores tipados permanecen pendientes del contrato formal.

**Motivo:** impedir que un estado inconsistente se propague o sea utilizado como base para una transición, completando la validación en los puntos de entrada, transición y resultado.

### VD-T06-117 — Categorías conceptuales de errores del Reducer

Se acepta distinguir, como mínimo, las siguientes categorías conceptuales de errores del Reducer:

1. **`INVALID_SNAPSHOT`:** el snapshot de entrada viola invariantes globales, por ejemplo, por identidades duplicadas, referencias huérfanas o inconsistencias entre estado y outcome.
2. **`INVALID_TRANSITION`:** el evento no es compatible con el estado actual o no satisface las precondiciones de la transición.
3. **`INVALID_RESULT`:** la aplicación de la transición produciría un snapshot que viola invariantes globales.

Se establecen además estas reglas:

4. Los errores deben ser deterministas y no depender de excepciones accidentales o detalles no controlados de ejecución.
5. Los errores no deben exponer información sensible innecesaria.
6. La detección y clasificación del error no implican reparación del snapshot, recuperación automática ni aplicación parcial de la transición.
7. La clasificación operativa posterior corresponde a la capa superior, conforme a las responsabilidades establecidas en VD-T06-110.
8. Los nombres definitivos, estructura, payloads y taxonomía exhaustiva de errores permanecen pendientes del contrato formal.

**Motivo:** distinguir fallos del estado recibido, de la transición solicitada y del resultado candidato, manteniendo el rechazo atómico y la separación entre Reducer y capa superior.

### VD-T06-118 — Política fail-fast determinista para errores del Reducer

Se acepta que el Reducer aplique una política fail-fast determinista, en lugar de acumular múltiples errores por operación:

1. Si el snapshot de entrada es inválido, devuelve `INVALID_SNAPSHOT` y no evalúa el evento.
2. Si la transición solicitada es inválida, devuelve `INVALID_TRANSITION` y no construye un snapshot resultado.
3. Si el snapshot candidato viola invariantes globales, devuelve `INVALID_RESULT` y no expone ese resultado.
4. El Reducer devuelve un único error principal por operación; no acumula una colección de errores.
5. Si existen varias violaciones dentro de una misma etapa, selecciona la primera conforme a un orden de validación estable y documentado.
6. El orden de validación y la selección del error deben ser deterministas para el mismo snapshot y evento.
7. Esta política no altera las responsabilidades de clasificación operativa de la capa superior ni autoriza reparación o recuperación automática.

El orden exhaustivo de validaciones y los detalles contractuales de los errores permanecen pendientes de definición formal.

**Motivo:** mantener resultados reproducibles, simplificar el contrato de error y evitar variaciones de diagnóstico que afecten Replay o las pruebas deterministas.

### VD-T06-119 — Orden determinista de validación del Reducer

Se acepta establecer el siguiente orden de validación para el Reducer:

1. **Validación estructural:** presencia de campos obligatorios, colecciones inicializadas y referencias con formato válido.
2. **Validación de integridad local:** unicidad de identidades, ausencia de duplicados y coherencia referencial interna.
3. **Validación de invariantes de dominio:** consistencia entre estado y outcome, cardinalidades, relaciones y conjunto de Admissions bloqueantes.
4. **Validación de transición:** compatibilidad del evento con el estado actual y cumplimiento de las precondiciones específicas.
5. **Validación global del resultado:** comprobación de todas las invariantes sobre el snapshot candidato.

La primera etapa que falle determina el error devuelto, conforme a la política fail-fast de VD-T06-118. El orden de validación dentro de cada etapa también deberá ser estable y documentado para mantener el determinismo.

Este orden no incorpora verificaciones históricas del journal ni consultas al ledger Member, que permanecen fuera de las responsabilidades del Reducer.

Los chequeos exhaustivos de cada etapa y su correspondencia con errores tipados permanecen pendientes del contrato formal.

**Motivo:** fijar una secuencia reproducible de validaciones y una prioridad inequívoca para el error principal, sin mezclar validación de dominio con integridad histórica o evidencia externa.

### VD-T06-120 — Política estricta para campos ausentes, inválidos o desconocidos

Se acepta que el Reducer aplique una política estricta a los campos del modelo lógico y de los eventos:

1. Un campo obligatorio ausente produce un rechazo tipado: `INVALID_SNAPSHOT` cuando corresponde al snapshot de entrada, o `INVALID_TRANSITION` cuando corresponde al evento o a la transición solicitada.
2. Un campo conocido cuyo valor no satisface las restricciones estructurales o de dominio aplicables se rechaza; no se normaliza ni se sustituye por inferencia.
3. Un campo desconocido en el modelo lógico recibido se rechaza; no se ignora silenciosamente.
4. El Reducer no asigna valores por defecto para completar información obligatoria faltante.
5. La compatibilidad entre versiones del modelo, la evolución del esquema y las reglas para interpretar versiones distintas corresponden al contrato de versionado, no al Reducer.

La clasificación exacta de cada error y los detalles de versionado permanecen sujetos al contrato formal.

**Motivo:** evitar interpretaciones divergentes del mismo estado o evento y mantener validación explícita, fail-fast y determinista.

### VD-T06-121 — Política explícita de nulabilidad y ausencia de campos

Se acepta distinguir explícitamente la nulabilidad y la ausencia de campos en el modelo lógico y los eventos:

1. Un campo obligatorio cuyo valor sea `null` se rechaza.
2. Un campo opcional con valor `null` solo se admite cuando el contrato de dominio declara expresamente que ese campo es nullable.
3. La ausencia de un campo opcional no equivale automáticamente a un valor `null`; su interpretación debe estar definida por el contrato.
4. El Reducer no convierte automáticamente un campo ausente en `null`, ni convierte `null` en un valor por defecto.
5. La nulabilidad de cada campo y la semántica de su ausencia deben quedar especificadas en el contrato formal del modelo.

**Motivo:** evitar que ausencia, nulidad y valores por defecto se interpreten como equivalentes sin una decisión explícita de dominio.

### VD-T06-122 — Validación estricta de rangos numéricos y valores temporales

Se acepta que el Reducer valide los rangos numéricos y valores temporales conforme a los tipos y restricciones definidos en el contrato:

1. Los identificadores numéricos deben respetar los rangos establecidos por su tipo contractual.
2. Los contadores y secuencias no pueden ser negativos cuando representan cantidades o posiciones.
3. Los valores temporales deben respetar la precisión y unidad definidas por el contrato.
4. No se permiten overflow, truncamiento ni conversiones implícitas.
5. El Reducer no define políticas de reloj, TTL, tolerancia ni expiración; valida únicamente los valores recibidos conforme al contrato aplicable.

Los rangos, unidades, precisiones y límites concretos permanecen pendientes de especificación formal.

**Motivo:** evitar que representaciones numéricas o temporales inválidas, ambiguas o fuera de rango alteren la interpretación determinista del estado y de los eventos.

### VD-T06-123 — Comparación y representación de identificadores

Se acepta establecer los siguientes criterios para los identificadores procesados por el Reducer:

1. Los identificadores se comparan por igualdad exacta, sin normalización implícita.
2. No se admiten identificadores vacíos cuando el contrato exige una identidad.
3. El Reducer no elimina espacios, no cambia capitalización ni transforma Unicode automáticamente.
4. Dos identificadores distintos no se consideran equivalentes por similitud textual.
5. La representación canónica y las restricciones de formato corresponden al contrato de identidad y al codec; el Reducer valida los valores recibidos conforme a ese contrato.

**Motivo:** impedir que el Reducer altere o interprete identidades de manera diferente a los componentes responsables de identidad, canonicalización y Replay.

### VD-T06-124 — Identidad lógica de entidades y referencias de objetos

Se acepta establecer los siguientes criterios para la identidad de entidades representadas en el snapshot:

1. Dos referencias que identifican el mismo identificador de dominio representan la misma entidad lógica, no dos entidades distintas.
2. El snapshot no puede contener dos objetos Admission con el mismo `AdmissionId`.
3. El snapshot no puede contener dos `RecoveryRecord` distintos con la misma clave `(originalEventId, recoveryAction)`.
4. El Reducer valida identidad lógica conforme a las claves de dominio; no utiliza identidad de referencia en memoria (`object identity`) como criterio de identidad de entidades.
5. Compartir internamente una referencia de objeto Java no modifica la semántica del dominio ni convierte una misma entidad lógica en entidades distintas.

Esta decisión no impone estructuras Java ni define políticas de copia, mutabilidad interna o representación del snapshot.

**Motivo:** evitar que la implementación confunda identidad de entidades del dominio con igualdad o identidad de instancias Java.

### VD-T06-125 — Inmutabilidad observable del snapshot y prevención de aliasing mutable

Se acepta establecer que el snapshot sea inmutable desde la perspectiva de sus consumidores:

1. El Reducer no modifica objetos ni colecciones pertenecientes al snapshot recibido.
2. El snapshot resultante no comparte estructuras mutables que permitan modificar indirectamente el snapshot anterior mediante aliasing.
3. Las colecciones expuestas por el snapshot no permiten modificaciones externas.
4. Las entidades y referencias conservan su identidad lógica aunque cambie su representación interna.
5. La estrategia técnica concreta para garantizar estas propiedades —copia defensiva, copia estructural, estructuras persistentes u otra alternativa— permanece pendiente del diseño técnico.

**Motivo:** evitar que una transición aparentemente pura produzca efectos secundarios sobre snapshots anteriores y reforzar las garantías de pureza, atomicidad y determinismo de VD-T06-111.

### VD-T06-126 — Concurrencia y ausencia de estado compartido en el Reducer

Se acepta establecer los siguientes criterios de concurrencia para el Reducer:

1. El Reducer no mantiene estado mutable compartido entre invocaciones.
2. Dos invocaciones concurrentes que reciben snapshots y eventos semánticamente equivalentes producen resultados semánticamente equivalentes, conforme a VD-T06-112.
3. El Reducer no requiere locks internos para proteger estado de dominio compartido, dado que no mantiene dicho estado mutable compartido.
4. La serialización de eventos y la exclusión mutua sobre el estado durable corresponden al Coordinator y al mecanismo de persistencia, no al Reducer.
5. La concurrencia interna del Reducer no altera el orden lógico de eventos establecido por el protocolo.

**Motivo:** separar la evaluación pura y concurrente de transiciones de la coordinación serializada y durable del protocolo.

### VD-T06-127 — Separación entre rechazos de dominio y fallos inesperados

Se acepta establecer las siguientes reglas para el tratamiento de fallos del Reducer:

1. Los rechazos esperados del dominio se representan mediante errores tipados, no mediante excepciones genéricas.
2. Las excepciones inesperadas no se convierten silenciosamente en `INVALID_TRANSITION` ni en otra categoría de rechazo de dominio.
3. El Reducer no captura fallos para continuar ni expone una transición parcial.
4. Ante un fallo inesperado, el Reducer no devuelve un snapshot candidato como resultado exitoso.
5. La captura en los límites apropiados, el registro seguro y la clasificación operativa de excepciones corresponden a la capa superior.

Los tipos definitivos de errores de dominio y los contratos de propagación de fallos inesperados permanecen pendientes del diseño formal.

**Motivo:** mantener diferenciados los rechazos deterministas del dominio de los defectos técnicos o fallos inesperados de ejecución, sin ocultar estos últimos ni comprometer la atomicidad.

### VD-T06-128 — Observabilidad externa al Reducer

Se acepta establecer la siguiente separación para la observabilidad:

1. El Reducer no emite logs, métricas ni trazas como efecto secundario de la evaluación.
2. Los errores del Reducer no incluyen datos sensibles innecesarios.
3. El resultado estructurado del Reducer proporciona información suficiente para que la capa superior registre y correlacione un rechazo, conforme al contrato de errores.
4. La correlación utiliza identificadores ya presentes en el contexto de la operación; el Reducer no genera identificadores nuevos para logging o trazabilidad.
5. Logging, métricas, trazas y alertas corresponden al Coordinator y a la infraestructura de observabilidad, fuera del Reducer.

Los campos concretos del resultado estructurado y las reglas de redacción segura permanecen pendientes del contrato formal de errores y observabilidad.

**Motivo:** preservar la pureza del Reducer y evitar acoplar la evaluación de dominio a infraestructura de logging, métricas o trazas.


### VD-T06-129 — Versionado del snapshot lógico

Se acepta establecer las siguientes reglas para el versionado del snapshot lógico:

1. El snapshot debe incluir una versión explícita de su esquema lógico.
2. El Reducer solo procesa snapshots cuyas versiones estén declaradas como compatibles por su contrato.
3. Una versión desconocida o incompatible se rechaza; el Reducer no intenta interpretarla por aproximación.
4. El Reducer no migra ni transforma snapshots entre versiones.
5. La migración y la compatibilidad entre versiones corresponden a un componente o procedimiento explícito de versionado, fuera del Reducer.
6. La versión del esquema lógico del snapshot se distingue de la versión del codec binario; no son intercambiables ni se presumen equivalentes.

La representación concreta del campo de versión, el catálogo de versiones soportadas y las reglas de compatibilidad/migración permanecen pendientes del contrato de versionado.

**Motivo:** impedir que el Reducer aplique transiciones sobre modelos lógicos cuya semántica no reconoce y mantener separadas la evolución del dominio y la representación binaria.


### VD-T06-130 — Política estricta de compatibilidad del snapshot lógico

Se acepta la opción A: el Reducer procesa exclusivamente snapshots cuya versión de esquema lógico coincide con la única versión vigente que declara soportar su contrato.

1. Un snapshot cuya versión lógica coincide con la versión vigente soportada puede continuar con las validaciones ordinarias del Reducer.
2. Un snapshot de una versión lógica anterior se rechaza; no se migra ni adapta automáticamente.
3. Un snapshot con una versión desconocida o no soportada se rechaza.
4. El Reducer no mantiene compatibilidad simultánea con versiones lógicas anteriores.
5. El Reducer no incorpora mecanismos de migración ni adaptación entre versiones.
6. Esta política se refiere exclusivamente a la versión del esquema lógico del snapshot y no determina la compatibilidad de versiones del codec binario.
7. La reconstrucción, recuperación o eventual tratamiento operativo de un snapshot rechazado queda fuera del Reducer y no se resuelve mediante esta decisión.

La identificación concreta de la versión vigente, su representación y los errores tipados asociados al rechazo permanecen pendientes del contrato formal de versionado y errores.

**Motivo:** mantener un único modelo lógico soportado por el Reducer, evitando compatibilidad implícita, migraciones internas y coexistencia de semánticas de versiones diferentes.


### VD-T06-131 — Diagnóstico estructurado para rechazo de snapshot inválido

Se acepta la opción C: cuando el Reducer rechaza un snapshot por incumplir el esquema lógico vigente o sus invariantes, el rechazo debe proporcionar un diagnóstico estructurado, sujeto a las siguientes reglas:

1. El rechazo se expresa mediante la categoría de error tipado correspondiente, conforme a VD-T06-117.
2. El diagnóstico identifica la etapa de validación en la que se detectó la primera violación, conforme al orden determinista de VD-T06-119.
3. El diagnóstico identifica la regla o invariante incumplida.
4. Cuando resulte aplicable y seguro, incluye una referencia al campo o entidad afectada y los identificadores de contexto ya disponibles.
5. Se conserva la política fail-fast: se informa un único error principal por operación, sin acumular múltiples errores.
6. El diagnóstico no incluye datos sensibles innecesarios ni valores completos del snapshot.
7. El Reducer no repara, normaliza, completa ni modifica el snapshot rechazado, y no expone un snapshot candidato.
8. La información estructurada permite que la capa superior registre y correlacione el rechazo, sin que el Reducer emita logs, métricas o trazas ni genere identificadores nuevos, conforme a VD-T06-128.

Los campos exactos del diagnóstico, su representación contractual, las reglas de redacción y el mapeo exhaustivo entre reglas e invariantes permanecen pendientes del contrato formal de errores y observabilidad.

**Motivo:** facilitar el diagnóstico operativo de snapshots inválidos sin debilitar el rechazo estricto, la política fail-fast, la pureza ni la protección de información sensible.


### VD-T06-132 — Validación global y específica de referencias con clasificación de responsabilidades

Se acepta combinar la opción B —validación global del snapshot, validación específica de la transición y validación global del resultado— con la clasificación de responsabilidades de la opción D.

1. **Snapshot de entrada:** antes de evaluar el evento, el Reducer valida la consistencia referencial global del snapshot conforme al orden determinista de VD-T06-119.
2. **Transición:** el Reducer valida específicamente las referencias requeridas por el tipo de evento y su correspondencia con el estado lógico y las precondiciones de la transición.
3. **Snapshot resultante:** después de aplicar conceptualmente la transición, el Reducer valida nuevamente las invariantes globales y la consistencia referencial del snapshot candidato. Si falla, rechaza el resultado completo y no expone un snapshot nuevo.
4. La validación referencial distingue las siguientes categorías:
   - **Estructurales:** presencia y formato válido de las referencias requeridas por el esquema lógico.
   - **De identidad:** correspondencia entre identificadores y entidades lógicas, sin duplicidades ni asociaciones contradictorias.
   - **De dominio:** validez de las relaciones y cardinalidades aprobadas entre entidades.
   - **Históricas:** el Reducer verifica únicamente la coherencia lógica y contextual de las referencias conservadas en el snapshot y requeridas por la transición. No verifica la existencia durable de eventos ni reconstruye su secuencia histórica.
5. La existencia durable de eventos y referencias, la secuencia, unicidad e integridad histórica del journal y la correlación histórica corresponden a Replay, conforme a las decisiones existentes.
6. La consulta o interpretación del ledger Member y la validación de evidencia externa no corresponden al Reducer.
7. Una referencia ausente, inválida, huérfana o contradictoria según el contrato aplicable provoca rechazo tipado, sin reparación, normalización ni mutación parcial.
8. Se mantiene la política fail-fast: se informa un único error principal conforme al orden determinista de validación. El diagnóstico estructurado respeta VD-T06-131.
9. Esta decisión no introduce recorridos incrementales como sustituto de la validación global ni altera las responsabilidades ya aprobadas de Reducer, Replay, Coordinator y TASK-04.

La matriz exhaustiva de referencias por entidad y tipo de evento, los campos obligatorios, formatos, cardinalidades y correspondencia con errores tipados permanecen pendientes del contrato formal.

**Motivo:** combinar garantías de integridad global antes y después de cada transición con validaciones específicas del evento, manteniendo explícita la frontera entre consistencia lógica del Reducer e integridad histórica de Replay.


### VD-T06-133 — Optimización de la validación exhaustiva mediante índices coherentes

Se acepta la opción B: mantener la validación exhaustiva del snapshot de entrada y del snapshot resultante en cada transición, permitiendo optimizar su ejecución mediante índices y estructuras auxiliares coherentes con las entidades canónicas.

1. La optimización no reduce ni sustituye las garantías de validación global establecidas en VD-T06-114, VD-T06-116, VD-T06-119 y VD-T06-132.
2. Antes de aplicar el evento, el Reducer valida exhaustivamente las invariantes estructurales, referenciales y de dominio del snapshot de entrada.
3. Después de construir conceptualmente el resultado de una transición válida, el Reducer valida exhaustivamente las invariantes globales del snapshot resultante.
4. Pueden utilizarse índices por identidades y estructuras auxiliares para evitar búsquedas lineales repetidas y optimizar verificaciones, siempre que sus resultados sean equivalentes a la validación exhaustiva contractual.
5. Las entidades canónicas del snapshot siguen siendo la única fuente de verdad. Los índices y datos derivados no constituyen fuentes independientes de verdad y deben ser coherentes con las entidades canónicas.
6. Toda inconsistencia entre una estructura auxiliar y las entidades canónicas provoca rechazo tipado; el Reducer no repara ni normaliza silenciosamente el snapshot.
7. La optimización debe preservar pureza, determinismo, inmutabilidad observable, ausencia de estado mutable compartido y atomicidad del Reducer.
8. No se adopta validación incremental como sustituto de la validación global ni se introducen certificados de integridad como mecanismo de confianza.
9. No se establece todavía una estructura Java, algoritmo, estrategia de indexación ni complejidad temporal concreta.

La selección de estructuras auxiliares, su construcción y mantenimiento, la estrategia de copia, los criterios de equivalencia y la eventual evaluación de rendimiento permanecen pendientes del diseño técnico. Cualquier benchmark o experimento requiere autorización explícita.

**Motivo:** preservar las garantías de integridad global ya aceptadas y permitir optimizaciones internas sin alterar la semántica del Reducer ni introducir fuentes de verdad adicionales.


### VD-T06-134 — Rechazo del Reducer y reconstrucción externa de estructuras auxiliares inconsistentes

Se acepta la opción C: ante una inconsistencia entre un índice o estructura auxiliar y las entidades canónicas del snapshot, el Reducer rechaza la operación y la eventual reconstrucción se realiza mediante un procedimiento externo.

1. El Reducer detecta la inconsistencia y devuelve un error tipado conforme al contrato de errores; no modifica el índice, las estructuras auxiliares ni las entidades canónicas.
2. El Reducer no reconstruye, repara, descarta ni normaliza estructuras auxiliares durante la aplicación de una transición.
3. El rechazo no produce ni expone un snapshot nuevo o parcialmente modificado.
4. La capa superior recibe el rechazo y determina la clasificación y el tratamiento operativo correspondiente, conforme a las políticas de recuperación aplicables.
5. Si se autoriza una reconstrucción, esta se realiza mediante un procedimiento externo al Reducer, a partir de las entidades canónicas. La reconstrucción no puede alterar ni reinterpretar esas entidades.
6. Una vez reconstruidas las estructuras auxiliares, el snapshot y las estructuras resultantes deben someterse nuevamente a las validaciones contractuales antes de reanudar la aplicación de eventos.
7. La reconstrucción no demuestra por sí misma la integridad histórica del journal ni sustituye las verificaciones de Replay.
8. Esta decisión no autoriza reconstrucción automática, retry de eventos, modificación de entidades canónicas ni continuación mientras la inconsistencia no haya sido resuelta y verificada.

El responsable concreto, las condiciones de autorización, la persistencia o regeneración de índices, los criterios de clasificación operativa y el procedimiento de reanudación permanecen pendientes del contrato de recuperación y del diseño técnico.

**Motivo:** mantener al Reducer puro y determinista, preservar las entidades canónicas como única fuente de verdad y permitir una recuperación controlada fuera del procesamiento de transiciones.


### VD-T06-135 — Ciclo de vida híbrido de índices derivados en memoria

Se acepta la opción D: los índices derivados son estructuras reconstruibles en memoria, asociadas al snapshot durante su procesamiento, sin persistencia durable independiente.

1. Las entidades canónicas del snapshot permanecen como única fuente de verdad.
2. Durante la carga inicial o recuperación, Replay/snapshot construye los índices derivados a partir de las entidades canónicas y verifica su coherencia conforme al contrato aplicable.
3. Los índices derivados pueden mantenerse en memoria durante el procesamiento y actualizarse como consecuencia de cada transición válida.
4. La actualización de entidades canónicas y estructuras derivadas debe ser lógica y atómicamente coherente en el snapshot resultante; no se expone un resultado parcial.
5. Los índices no constituyen una fuente durable independiente ni requieren persistencia separada en el journal o en el snapshot durable.
6. El Reducer puede utilizar y actualizar estas estructuras como parte de la evaluación pura de una transición, preservando determinismo, inmutabilidad observable, ausencia de estado mutable compartido y prevención de aliasing.
7. Si se detecta una inconsistencia entre índices y entidades canónicas, se aplica VD-T06-134: rechazo del Reducer y eventual reconstrucción mediante procedimiento externo autorizado. No se reconstruyen índices como reparación dentro de la invocación del Reducer.
8. Esta decisión no selecciona estructuras Java, algoritmos, estrategia de copia, duración de caché, límites de memoria ni política de descarte.

La interfaz entre snapshot y estructuras derivadas, el responsable concreto de su construcción durante carga/recuperación, las condiciones de invalidación y descarte, y las garantías técnicas de actualización atómica permanecen pendientes del diseño técnico y de los contratos de Replay/snapshot.

**Motivo:** evitar persistir información derivada como fuente adicional, reducir reconstrucciones durante transiciones y conservar las fronteras de responsabilidad entre Reducer, Replay y persistencia.

### VD-T06-136 — Propiedad e aislamiento de estructuras auxiliares entre invocaciones concurrentes

Se acepta la opción A: las estructuras auxiliares inmutables son propiedad lógica del snapshot y pueden compartirse entre invocaciones concurrentes, siempre que se preserven las garantías de aislamiento e inmutabilidad.

1. Los índices y estructuras auxiliares pertenecen lógicamente al snapshot cuyas entidades canónicas describen.
2. Dos o más invocaciones pueden consultar concurrentemente las mismas estructuras auxiliares del snapshot de entrada mientras estas permanezcan inmutables.
3. Ninguna invocación puede modificar estructuras auxiliares pertenecientes al snapshot de entrada ni estructuras compartidas con otras invocaciones.
4. Toda transición válida produce un nuevo snapshot lógico con estructuras auxiliares coherentes con sus entidades canónicas. Las actualizaciones se realizan mediante nuevas estructuras, sin mutar las compartidas por el snapshot anterior.
5. Se permite compartir internamente estructuras inmutables entre snapshots —incluido *structural sharing*— siempre que no exista una vía de modificación observable o indirecta sobre snapshots anteriores.
6. La construcción y actualización deben preservar la coherencia lógica y atómica entre entidades canónicas e índices; no se expone ni comparte un resultado parcialmente actualizado.
7. Esta decisión no selecciona colecciones Java, estructuras persistentes, *copy-on-write*, estrategia de copia ni algoritmo concreto. Esas decisiones permanecen pendientes del diseño técnico.

**Motivo:** garantizar aislamiento entre invocaciones concurrentes sin imponer copias completas ni introducir estado mutable compartido, manteniendo las propiedades de pureza, determinismo e inmutabilidad observable establecidas en VD-T06-125, VD-T06-126 y VD-T06-135.

### VD-T06-137 — Actualización incremental de índices con reconstrucción completa de respaldo

Se acepta la opción C: utilizar actualización incremental de índices derivados como mecanismo habitual durante una transición válida y permitir la reconstrucción completa de respaldo cuando la actualización incremental no pueda garantizar la coherencia del resultado.

1. La actualización incremental constituye el mecanismo habitual para mantener los índices derivados durante una transición válida.
2. La actualización se limita a las entradas y estructuras afectadas por los cambios en las entidades canónicas, pudiendo conservarse mediante *structural sharing* las partes inmutables no afectadas.
3. Ninguna actualización puede modificar estructuras compartidas con el snapshot de entrada ni con otros snapshots que continúen siendo utilizados por invocaciones concurrentes.
4. Cuando una transición válida requiera cambios que no puedan resolverse de manera segura mediante la actualización incremental, puede utilizarse una reconstrucción completa de los índices a partir de las entidades canónicas del resultado candidato.
5. La actualización incremental y la reconstrucción completa deben producir estructuras semánticamente equivalentes para las mismas entidades canónicas.
6. Tanto el mecanismo incremental como la reconstrucción completa deben preservar la coherencia lógica y atómica entre las entidades canónicas y los índices. No se expone un snapshot parcial ni una combinación temporalmente inconsistente.
7. La validación global del snapshot resultante continúa siendo obligatoria conforme a VD-T06-133; la estrategia incremental no sustituye ni reduce las garantías de validación exhaustiva.
8. Si la construcción o validación del resultado falla, la transición se rechaza sin exponer un nuevo snapshot ni modificar el snapshot de entrada.
9. La reconstrucción de respaldo se limita a construir los índices del resultado de una transición válida. No autoriza reparar ni reconstruir índices inconsistentes detectados en el snapshot de entrada, cuyo tratamiento continúa regido por VD-T06-134.

La selección de estructuras Java, algoritmos, criterios técnicos para elegir entre actualización incremental y reconstrucción completa, estrategia concreta de *structural sharing*, complejidades temporales y espaciales, y cualquier benchmark o experimento permanecen pendientes del diseño técnico y requieren autorización explícita cuando corresponda.

**Motivo:** equilibrar eficiencia y seguridad, evitando reconstrucciones completas innecesarias sin debilitar la coherencia de los datos derivados, la inmutabilidad observable, el aislamiento concurrente ni la validación global del Reducer.

### VD-T06-138 — Selección del mecanismo de actualización por garantía de coherencia

Se acepta la opción B: seleccionar entre actualización incremental y reconstrucción completa según la capacidad del mecanismo incremental para garantizar la coherencia de todas las estructuras derivadas afectadas por una transición válida.

1. La actualización incremental se utiliza cuando el mecanismo puede garantizar la actualización completa y coherente de todas las entradas y estructuras derivadas afectadas por los cambios en las entidades canónicas.
2. Si no puede garantizarse esa coherencia mediante actualización incremental, se utiliza la reconstrucción completa de los índices a partir de las entidades canónicas del resultado candidato, conforme a VD-T06-137.
3. La reconstrucción completa no utiliza índices ni estructuras derivadas anteriores como fuente de verdad; las entidades canónicas del resultado candidato son su única fuente.
4. La actualización incremental y la reconstrucción completa deben producir estructuras semánticamente equivalentes para las mismas entidades canónicas.
5. El snapshot resultante debe superar siempre la validación global establecida en VD-T06-133, independientemente del mecanismo utilizado para construir sus índices.
6. Si no puede construirse o validarse un resultado coherente, la transición se rechaza sin exponer un snapshot parcial ni modificar el snapshot de entrada.
7. La selección no depende de umbrales de rendimiento, estimaciones de costo ni mediciones durante la ejecución. Tampoco introduce una política específica por tipo de evento.
8. La imposibilidad de garantizar una actualización incremental no debe confundirse con una inconsistencia del snapshot de entrada. Si este último contiene índices inconsistentes con las entidades canónicas, corresponde aplicar VD-T06-134: rechazo y eventual reconstrucción externa autorizada, no reconstrucción como parte de la transición.

Los mecanismos técnicos para determinar la cobertura de las estructuras afectadas, identificar dependencias y demostrar la garantía de coherencia permanecen pendientes del diseño técnico. Esta decisión no selecciona algoritmos, estructuras Java ni criterios de rendimiento.

**Motivo:** priorizar la corrección y la coherencia verificable de los índices derivados, utilizando la reconstrucción completa como respaldo controlado sin introducir umbrales arbitrarios ni políticas redundantes por tipo de evento.

### VD-T06-139 — Tratamiento de fallos durante la construcción de índices derivados

Se acepta la opción D: distinguir los rechazos contractuales del dominio de los fallos técnicos inesperados durante la actualización incremental o la reconstrucción completa de índices, proporcionando diagnóstico estructurado sin convertir detalles de implementación en categorías de dominio.

1. Una inconsistencia entre los índices del snapshot de entrada y sus entidades canónicas provoca rechazo tipado, conforme a VD-T06-134. No se intenta reparar ni reconstruir dentro de la invocación del Reducer.
2. Si la actualización incremental no puede garantizar la coherencia de las estructuras afectadas, se utiliza la reconstrucción completa del resultado candidato conforme a VD-T06-137 y VD-T06-138.
3. Si la construcción del resultado candidato permite detectar una condición contractual de invalidez o una invariante de dominio incumplida, el Reducer rechaza la transición mediante el error tipado correspondiente.
4. Si la reconstrucción completa no puede producir un resultado coherente por una condición contractual verificable, la transición se rechaza sin exponer el snapshot candidato.
5. Los fallos técnicos inesperados durante la actualización incremental, reconstrucción o validación no se convierten silenciosamente en errores de dominio; se propagan conforme a VD-T06-127.
6. Los rechazos contractuales pueden proporcionar un diagnóstico estructurado que identifique la etapa de construcción o validación y la regla incumplida, respetando VD-T06-131 y la política fail-fast.
7. Los diagnósticos no incorporan detalles internos de implementación como nuevas categorías de dominio ni incluyen datos sensibles innecesarios.
8. Ante cualquier rechazo o fallo, el snapshot de entrada permanece inalterado y no se devuelve ni expone un snapshot parcial o candidato como resultado exitoso.
9. El Reducer no realiza reintentos automáticos, reparaciones silenciosas, logging, métricas ni trazas. La clasificación operativa y observabilidad de fallos técnicos corresponden a la capa superior, conforme a VD-T06-128.
10. Esta decisión no define códigos definitivos de error, clases Java de excepciones, catálogo de diagnósticos ni mecanismos técnicos de propagación.

**Motivo:** preservar la distinción entre rechazos deterministas del dominio y fallos técnicos inesperados, mantener el Reducer puro y atómico, y permitir diagnóstico suficiente sin acoplar el contrato de dominio a los mecanismos internos de indexación.

### VD-T06-140 — Equivalencia semántica verificable entre actualización incremental y reconstrucción completa

Se acepta la opción D: exigir equivalencia semántica completa entre los resultados construidos mediante actualización incremental y reconstrucción completa, con validación global independiente de la estrategia utilizada.

1. Para las mismas entidades canónicas resultantes de una transición válida, la actualización incremental y la reconstrucción completa deben producir snapshots semánticamente equivalentes.
2. La equivalencia comprende las entidades canónicas, sus relaciones, las asociaciones representadas por los índices derivados y los resultados observables de las consultas soportadas por el contrato.
3. La equivalencia no exige identidad de objetos Java, igualdad de referencias en memoria, igualdad de distribución interna ni identidad de representación física.
4. La validación global del snapshot resultante continúa siendo obligatoria conforme a VD-T06-133, independientemente del mecanismo de construcción utilizado.
5. La coherencia de los índices derivados debe poder contrastarse con las entidades canónicas, que permanecen como única fuente de verdad. La validación no puede confiar exclusivamente en que el mecanismo constructor haya producido un resultado correcto.
6. Toda diferencia semántica o inconsistencia detectada provoca rechazo tipado; no se expone el snapshot candidato ni se modifica el snapshot de entrada.
7. La estrategia concreta de comparación, los algoritmos de validación y las pruebas diferenciales para demostrar equivalencia permanecen pendientes del diseño técnico y del contrato de testing.

**Motivo:** establecer una garantía de corrección independiente de la estrategia interna de construcción de índices, preservando la fuente de verdad canónica, la validación global, la inmutabilidad observable y la libertad de implementación.

### VD-T06-141 — Estrategia de pruebas para verificar la equivalencia semántica de índices derivados

Se acepta una estrategia combinada de pruebas, con pruebas diferenciales como mecanismo central para verificar la equivalencia semántica entre la actualización incremental y la reconstrucción completa, complementadas por pruebas unitarias y pruebas basadas en propiedades e invariantes.

1. **Pruebas unitarias:** verifican de manera independiente el comportamiento de la actualización incremental y de la reconstrucción completa, incluyendo los casos válidos, inválidos y límites definidos por sus contratos.
2. **Pruebas diferenciales:** aplican ambos mecanismos sobre las mismas entidades canónicas resultantes y comparan sus resultados conforme al criterio de equivalencia semántica establecido en VD-T06-140.
3. **Pruebas basadas en propiedades e invariantes:** verifican que los índices derivados sean coherentes con las entidades canónicas, que se preserven las invariantes globales y que el resultado no dependa de la estrategia de construcción.
4. La cobertura debe incluir transiciones que no afecten índices, que afecten una o varias estructuras, altas, modificaciones y bajas de asociaciones cuando estén contempladas por el modelo, además de casos límite y rechazos contractuales aplicables.
5. Las pruebas deben verificar que las estructuras del snapshot de entrada permanecen inalteradas y que los resultados no presentan aliasing mutable observable, conforme a VD-T06-125 y VD-T06-136.
6. Las pruebas deben comprobar que los fallos de construcción o validación no exponen snapshots parciales y que se mantiene la distinción entre rechazos de dominio y fallos técnicos inesperados, conforme a VD-T06-139.
7. La estrategia será selectiva y proporcional al riesgo: se priorizan las invariantes, transiciones y mecanismos cuya falla pueda comprometer la coherencia del snapshot. No se impone una metodología de testing exhaustiva ni una duplicación innecesaria de casos.
8. La selección de frameworks, herramientas, organización de suites, generación de datos, cobertura cuantitativa y automatización concreta permanece pendiente del diseño de testing.

**Motivo:** verificar la equivalencia de ambos mecanismos desde perspectivas complementarias, detectar errores de actualización y preservar las garantías de coherencia, inmutabilidad, aislamiento concurrente y atomicidad sin introducir complejidad de testing innecesaria.

### VD-T06-142 — Construcción concurrente independiente con reutilización de estructuras inmutables

Se acepta la opción C: permitir que varias invocaciones construyan concurrentemente resultados independientes a partir del mismo snapshot de entrada, reutilizando estructuras auxiliares únicamente cuando sean inmutables.

1. Varias invocaciones del Reducer pueden procesar concurrentemente el mismo snapshot de entrada, sin serialización interna.
2. Cada invocación construye y devuelve su propio resultado lógico; ningún resultado se comparte como estado mutable con otra invocación.
3. Las invocaciones pueden reutilizar estructuras del snapshot de entrada o componentes de otros resultados únicamente cuando sean inmutables y su reutilización no permita modificar indirectamente ningún snapshot.
4. Toda estructura nueva o modificada pertenece lógicamente al resultado de la invocación que la construye; no se modifica el snapshot de entrada ni los resultados de otras invocaciones.
5. La construcción concurrente debe preservar la coherencia entre entidades canónicas e índices derivados, la validación global y la equivalencia semántica establecidas en las decisiones anteriores.
6. El Reducer no requiere locks, cachés mutables compartidas ni coordinación interna para construir los resultados.
7. Invocaciones con snapshots y eventos semánticamente equivalentes deben producir resultados semánticamente equivalentes, independientemente de su ejecución concurrente o del orden de intercalado.
8. La serialización de eventos y la exclusión mutua sobre el estado durable continúan siendo responsabilidad del Coordinator y de la persistencia; esta decisión no autoriza al Reducer a resolver conflictos entre resultados concurrentes ni a decidir cuál debe persistirse.

La implementación de estructuras inmutables, la estrategia concreta de *structural sharing*, la gestión de memoria, las optimizaciones de asignación y cualquier evaluación de rendimiento permanecen pendientes del diseño técnico. No se autoriza introducir cachés compartidas mediante esta decisión.

**Motivo:** permitir concurrencia segura y determinista en la construcción de snapshots derivados, reutilizando datos inmutables sin introducir estado mutable compartido, locks ni coordinación operativa dentro del Reducer.

### VD-T06-143 — Ciclo de vida y liberación de memoria de snapshots e índices derivados

Se acepta la recomendación: la permanencia en memoria de snapshots y estructuras derivadas se determina por su alcanzabilidad mediante referencias; la gestión de memoria corresponde al runtime y no al Reducer.

1. El Reducer no administra manualmente la liberación de memoria de snapshots, índices ni estructuras auxiliares.
2. Los snapshots y sus estructuras derivadas permanecen disponibles mientras existan referencias activas que los mantengan alcanzables; cuando dejan de ser alcanzables, su memoria queda sujeta a la gestión automática del runtime.
3. La reutilización (*structural sharing*) de estructuras inmutables es válida, pero una referencia compartida prolonga la vida útil de esa estructura mientras siga siendo alcanzable desde cualquier snapshot o componente que la retenga.
4. El Reducer no mantiene referencias globales, cachés mutables, pools ni registros de snapshots que prolonguen artificialmente su ciclo de vida.
5. La retención, publicación y descarte de referencias a snapshots vigentes o históricos corresponden a los componentes externos responsables del ciclo de vida del estado, como Coordinator y persistencia, según sus propias políticas.
6. Esta decisión no prescribe políticas de retención histórica, límites de snapshots en memoria, frecuencia de recolección, tuning del garbage collector ni mecanismos concretos de diagnóstico de memoria; esos aspectos quedan fuera del contrato del Reducer y pendientes del diseño de infraestructura.
7. La liberación de memoria no altera la inmutabilidad ni la semántica de snapshots que continúen siendo alcanzables y válidos para sus consumidores.

**Motivo:** mantener al Reducer libre de gestión de recursos y estado compartido, permitiendo que el runtime gestione la memoria según la alcanzabilidad de los objetos y dejando la política de retención de snapshots bajo responsabilidad de la capa que administra su ciclo de vida.

### VD-T06-144 — Límites de retención y presión de memoria fuera del contrato del Reducer

Se acepta mantener completamente fuera del contrato del Reducer la definición y aplicación de límites de retención de snapshots y las políticas de gestión de presión de memoria.

1. El Reducer no establece límites de cantidad, antigüedad o tamaño de snapshots retenidos.
2. El Reducer no decide qué snapshots pueden descartarse, cuándo liberar referencias ni cómo priorizar la retención de snapshots vigentes o históricos.
3. Las políticas de retención, descarte, backpressure, límites de memoria y respuesta ante presión de memoria corresponden a Coordinator, persistencia y/o infraestructura, según las responsabilidades de cada componente.
4. Ninguna política externa de retención puede modificar la semántica, inmutabilidad o coherencia de un snapshot que todavía deba estar disponible para sus consumidores.
5. El Reducer no incorpora mecanismos de monitorización de memoria, alertas, tuning del garbage collector ni estrategias de recuperación ante presión de memoria.
6. Esta decisión no prescribe una política concreta de retención ni un mecanismo de infraestructura; esos aspectos quedan pendientes de su diseño específico y fuera del alcance de TASK-03.

**Motivo:** preservar la responsabilidad acotada del Reducer como transformación pura y determinista, evitando introducir en su contrato políticas operativas de ciclo de vida, capacidad o gestión de recursos.

### VD-T06-145 — Separación entre identidad física y equivalencia semántica de snapshots

Se acepta separar explícitamente la identidad física de un snapshot de su identidad o equivalencia semántica. El mecanismo concreto de identificación y versionado queda pendiente del diseño técnico.

1. La identidad física refiere a la instancia u objeto concreto que representa un snapshot en memoria.
2. La equivalencia semántica refiere a que dos snapshots representan el mismo estado lógico, de acuerdo con las entidades canónicas, relaciones, índices derivados y comportamiento observable definido por el contrato.
3. Dos snapshots pueden ser semánticamente equivalentes sin ser la misma instancia física ni compartir las mismas referencias.
4. Dos snapshots físicamente distintos no deben considerarse estados lógicamente diferentes únicamente por tener identidades de objeto distintas.
5. El *structural sharing* no implica por sí mismo identidad semántica entre snapshots; solo describe una posible reutilización de estructuras inmutables.
6. La identidad física no se utiliza como sustituto de la validación de coherencia ni de la comparación semántica establecida en VD-T06-140.
7. Esta decisión no selecciona UUID, hashes de contenido, contadores de versión, secuencias, marcas temporales ni otro mecanismo concreto de identificación o versionado.
8. La asignación, persistencia, comparación y ciclo de vida de identificadores o versiones, si fueran necesarios, deberán definirse en el diseño de los componentes responsables del estado y su coordinación, sin introducir estado mutable compartido ni efectos secundarios en el Reducer.

**Motivo:** evitar que la identidad de objetos en memoria se confunda con la identidad lógica del estado, preservar la independencia respecto de la representación física y mantener pendiente la elección del mecanismo de versionado hasta contar con el diseño de arquitectura correspondiente.
