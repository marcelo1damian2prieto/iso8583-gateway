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

Este documento consolida las decisiones VD-T06-01 a VD-T06-99 aceptadas durante la conversación de validación de diseño de T03-06.

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
