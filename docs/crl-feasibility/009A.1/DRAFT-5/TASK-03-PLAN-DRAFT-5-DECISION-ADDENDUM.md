# TASK-03 — Enmienda DRAFT-5 al PLAN-DRAFT-4

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Revisión:** `TASK-03-PLAN-DRAFT-5-DECISION-ADDENDUM`  
**Fecha:** 2026-11-15  
**Estado:** `DRAFT — PENDING HUMAN REVIEW`  
**Documento base:** `TASK-03-PLAN-DRAFT-4.md`  
**Base Git:** `bef7aa7c615745e19f548b47854fb4e37d543b18`.

## 1. Efecto y precedencia

Esta enmienda es aditiva y se interpreta junto con PLAN-DRAFT-4. En las materias aquí tratadas prevalece sobre cualquier texto previo de PLAN-DRAFT-4. Los identificadores y dependencias T03-00..T03-12 no cambian. El conjunto se denomina `PLAN-DRAFT-5`; no autoriza implementación.

## 2. Actualización de T03-00 — baseline completada

Reemplazar el estado de §2 y del bloque T03-00 que indicaba `BLOCKED — WORKSPACE NOT PROVISIONED` por:

**T03-00: `COMPLETED — STATIC READ-ONLY BASELINE WITH ACCESS LIMITATION`.**

Workspace: `%TEMP%\<redacted>sibility-009a1\project`. Origen: `%TEMP%\<redacted>sibility-009a\project`. Manifiesto: `WORKSPACE-PROVISIONING-MANIFEST.json`. El inventario contiene 31 archivos (1 POM, 21 Java en `src/test/java/lab/crl`, 9 PEM públicos), 70 exclusiones y no incluye `.git`. Los 31 archivos del manifiesto coincidieron en tamaño y SHA-256 con las entradas del `project.zip` subido. No hay hashes históricos individuales en TASK-01 para una comparación hash-by-hash. El usuario inspeccionó y aceptó `T03-00-BASELINE-INSPECTION.md`.

La inspección fue estática y read-only. No se ejecutó Maven/build/tests/experimentos. El informe conserva la limitación de que no se recalcularon los 31 hashes leyendo directamente el disco C: desde esta sesión. El POM conserva `artifactId=crl-feasibility-009a`, versión 1.0.0, Java 21 y Spring Boot 3.5.16 declarado; no se verificó el árbol efectivo de dependencias. No hay `src/main/java` en el inventario. No se detectó Git.

## 3. Estado de T03-01 y decisiones humanas

T03-01 pasa a `DOCUMENTARY CATALOG ACCEPTED — DETAIL FREEZE PARTIALLY OPEN`:

- Aceptado: catálogo Coordinator/Member y separación de ownership, conforme al documento T03-01 y TASK-02.
- Cerrado: política de recuperación automática de tail limitada a ADMIT y COMPLETE (D03-OPEN-02).
- Cerrado: tratamiento conservador de temporales residuales, sin auto-promoción/reemplazo/borrado (D03-OPEN-05).
- Dirección aceptada / detalle abierto: codec binario (01), startup epoch (03), clasificación de directory force (04), estados de recuperación (06), precondiciones/boundary de REBIND (08).
- Gate aceptado, experimento no autorizado: Windows FileLock/ATOMIC_MOVE/directory force (07).

La aceptación humana no congela el detalle de tags/longitudes/byte order/domain separators, startupOperationId/semántica exacta de reinicio/overflow, clasificación por proveedor/excepción ni tabla exhaustiva de estados/locks. Estos detalles deben resolverse en SPEC antes de congelar los contratos afectados.

## 4. Dependencias actualizadas

El grafo de tareas conserva sus IDs y dependencias funcionales. Se añaden estas precondiciones de freeze:

1. **T03-02 Codec:** no congelar implementación/contrato binario hasta resolver los detalles pendientes de D03-OPEN-01 y registrar vectores golden esperados. Una dirección aceptada no sustituye el formato completo.
2. **T03-03 OS lock:** definir recurso, ownership y estados. No afirmar capacidad Windows real; la evidencia del filesystem requiere el gate experimental D03-OPEN-07.
3. **T03-04/T03-05 Journal y checkpoint:** aplicar D03-OPEN-04 y D03-OPEN-05: tres clases directory-force, no fallback atómico, temporales `CREATE_NEW`, detección residual y no auto-limpieza.
4. **T03-07 Replay/T03-08 Tail recovery:** permitir recuperación automática solo de ADMIT/COMPLETE. Los demás tipos tail, incluido epoch advance, quedan `RECOVERY_REQUIRED`.
5. **T03-09 Serialización:** preservar H-01 y D-01/D-VD03-5; no ampliar la política de tail aprobada. La frontera lógica intra-proceso no se confunde con OS lock.
6. **T03-11 Crash/verification:** las pruebas reales de Windows siguen sin autorizar. Cada experimento tendrá autorización separada, directorio aislado, fixtures sintéticos y rollback.
7. **T03-12 Cierre:** debe incluir todos los hallazgos/incompatibilidades analizados, su disposición, riesgos abiertos y pruebas no ejecutadas. Si existen colisiones relevantes, se prepara el plan de refactor consolidado antes de cualquier implementación de ese refactor.

## 5. Procedimiento de incompatibilidades y refactor

El plan incorpora el proceso aceptado:

- analizar cada incompatibilidad por separado con evidencia y clasificación (demostrada, hipotética, falta de información o diferencia estructural);
- mapear los invariantes/contratos/requisitos/estados/datos persistidos afectados;
- evaluar alternativas sin presuponer refactor;
- conservar el conflicto abierto hasta decisión humana;
- una vez analizadas las incompatibilidades relevantes, crear un plan de refactor consolidado para revisión humana;
- solicitar autorización independiente antes de implementar el refactor.

El plan de refactor deberá incluir inventario, evidencia, alcance, trazabilidad, alternativas, secuencia/dependencias, compatibilidad de datos y replay, migración/rollback, impacto en idempotencia/concurrencia/fencing/seguridad, testing/aceptación, riesgos residuales y approvals. No se asume que todas las incompatibilidades requieran cambios de código.

## 6. Secuencia operativa ajustada

1. T03-00 — completada y aceptada.
2. T03-01 — catálogo y decisiones a nivel de dirección aceptados; completar detalles de especificación pendientes y revisión cruzada antes del freeze.
3. T03-02 — especificación/implementación del codec, cuando sus detalles estén aprobados y exista autorización correspondiente.
4. T03-03 — lock OS: contrato separado; experimento Windows solo después de aprobación específica.
5. T03-04/T03-05 — journal y checkpoint conforme a las políticas aceptadas.
6. T03-06 — reducer determinista.
7. T03-07 — replay y validación del almacén.
8. T03-10 — índice de idempotencia.
9. T03-08 — tail recovery solo ADMIT/COMPLETE.
10. T03-09 — serialización integral H-01.
11. T03-11 — crash matrix y verificación solo con los permisos aplicables.
12. T03-12 — evidencia, colisiones, cierre y solicitud de revisión humana.

Se conserva el grafo de dependencias de PLAN-DRAFT-4; la secuencia no autoriza adelantar implementación respecto de esas precondiciones.

## 7. Gates no modificados

- La aprobación de estas decisiones no equivale a aprobación integral de TASK-03.
- T03-02..T03-12 no tienen autorización de implementación.
- D03-OPEN-07 mantiene `EXPERIMENT NOT AUTHORIZED`.
- TASK-04 y tareas posteriores siguen `NOT AUTHORIZED`.
- El gate de factibilidad CRL de producción permanece `CLOSED`.
- No ejecutar Maven, builds, tests ni experimentos durante esta revisión documental.

## 8. Estado

`PLAN-DRAFT-5`: **coherente con T03-00 aceptada y las decisiones D03-OPEN-01..08 aprobadas en los niveles indicados; pendientes los detalles de diseño identificados y sus revisiones humanas.**