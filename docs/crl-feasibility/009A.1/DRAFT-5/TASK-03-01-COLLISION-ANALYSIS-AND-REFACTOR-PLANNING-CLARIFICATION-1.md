# TASK-03-01 — Aclaración del procedimiento para incompatibilidades y refactor

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Fecha:** 2026-11-15  
**Documento relacionado:** `TASK-03-01-HUMAN-ACCEPTANCE-ADDENDUM-1.md`  
**Estado:** `ACCEPTED — HUMAN PROCESS CLARIFICATION`  
**Alcance:** proceso de análisis y planificación. No autoriza implementación, modificación de código/configuración, Maven, builds, tests ni experimentos.

---

## 1. Aclaración de la aceptación humana

Se mantiene la aceptación humana de las ocho propuestas conservadoras D03-OPEN-01..08, con los estados y reservas consignados en el addendum relacionado.

La intención precisa respecto de posibles incompatibilidades es:

**Primero se analizará cada incompatibilidad detectada, individualmente y con evidencia. Una vez analizadas las incompatibilidades relevantes, se elaborará un plan de refactor consolidado.**

No se presume que necesariamente habrá colisiones. Tampoco se decide de antemano que la solución será siempre un refactor mínimo, ni se selecciona una solución de implementación antes de completar el análisis.

Esta aclaración reemplaza cualquier lectura del §1 del addendum relacionado que pudiera interpretarse como obligación de proponer inmediatamente un refactor mínimo por cada colisión detectada.

## 2. Procedimiento para cada incompatibilidad

Cuando aparezca una posible incompatibilidad:

1. **Registrar el hallazgo:** describir el comportamiento o contrato observado e identificar la fuente de evidencia (archivo, símbolo, sección, estado persistido, log o resultado de una actividad expresamente autorizada).
2. **Confirmar la colisión:** distinguir incompatibilidad demostrada de riesgo hipotético, falta de información o diferencia meramente estructural.
3. **Analizar impacto:** mapear requisitos, decisiones aprobadas, invariantes, contratos, estados, datos existentes, journal/checkpoint, replay, claves idempotentes, seguridad y escenarios afectados.
4. **Evaluar opciones:** considerar conservar el diseño y adaptar el punto de integración, introducir una capa de compatibilidad, modificar el diseño, realizar un refactor u otra alternativa justificada. Registrar ventajas, riesgos, alcance y evidencia necesaria.
5. **Registrar disposición:** dejar la incompatibilidad abierta hasta que exista una decisión humana o una resolución aprobada. No alterar silenciosamente decisiones aceptadas ni relajar invariantes.

El análisis puede concluir que no hace falta refactor. Esa conclusión también debe quedar justificada.

## 3. Plan de refactor consolidado

Después de analizar las incompatibilidades relevantes, se elaborará un plan de refactor que incluya, como mínimo:

- inventario consolidado de incompatibilidades confirmadas y su evidencia;
- objetivo y alcance del refactor, con componentes incluidos y excluidos;
- trazabilidad hacia requisitos, decisiones e invariantes afectadas;
- alternativas consideradas y justificación de la opción elegida;
- secuencia de cambios y dependencias;
- estrategia de compatibilidad con datos/formatos existentes, replay, migración y rollback, cuando corresponda;
- impacto en idempotencia, concurrencia, fencing, seguridad y recuperación;
- estrategia de testing y criterios de aceptación verificables;
- riesgos residuales, límites y evidencia que deberá obtenerse;
- puntos de revisión y aprobaciones humanas requeridas.

El plan se someterá a revisión y aprobación humana antes de implementar cualquier refactor. La autorización de implementación será un gate separado.

## 4. Reglas de gobierno preservadas

- Las ocho decisiones aceptadas continúan como baseline de diseño, salvo que un análisis de incompatibilidad demuestre una colisión que requiera reconsideración explícita.
- La detección de una colisión no revoca automáticamente una decisión aceptada.
- No se modificará código/configuración ni se ejecutarán builds, tests, Maven o experimentos como parte del análisis documental no autorizado.
- Los experimentos específicos del filesystem Windows mantienen su gate independiente.
- La aprobación de este procedimiento no aprueba el diseño integral TASK-03, no autoriza implementación TASK-03/TASK-04 y no abre el gate CRL de producción.

## 5. Estado

- D03-OPEN-01..08: propuestas aceptadas en los niveles y con las reservas del addendum relacionado.
- Análisis de incompatibilidades: se realizará cuando se identifiquen, de forma individual y basada en evidencia.
- Plan de refactor: se elaborará después de analizar las incompatibilidades relevantes; aún no existe ni se presume necesario.
- Implementación: `NOT AUTHORIZED`.

**Resultado:** queda aclarado que el trabajo seguirá la secuencia análisis de incompatibilidades → plan de refactor consolidado → revisión/aprobación humana → autorización separada de implementación.