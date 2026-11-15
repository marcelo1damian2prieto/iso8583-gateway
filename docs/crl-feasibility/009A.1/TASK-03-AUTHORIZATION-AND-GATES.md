# TASK-03 — Autorización y gates

**Proyecto:** CRL-FEASIBILITY-009A.1  
**Fecha:** 2026-11-15  
**Estado:** `AUTHORIZED — DESIGN PREPARATION AND VALIDATION ONLY`

## 1. Autorización recibida

El usuario aprobó el cierre documental de TASK-02 y autorizó expresamente TASK-03.

La autorización permite preparar especificación, plan ejecutable, desglose de tareas y validación del diseño. No equivale a aprobación de una implementación concreta.

## 2. Alcance previsto

TASK-03 se centra en la persistencia durable del Coordinator, conforme al diseño fuente R3.1 y al paquete TASK-02 aprobado:

- journal append-only, secuencia e integridad/hash-chain;
- checkpoint temporal, force y reemplazo atómico;
- replay y reconstrucción;
- recuperación tipada del tail posterior al checkpoint;
- idempotencia y recuperación de ACK perdido;
- serialización de la operación lógica ADMIT/FENCE;
- fallo cerrado ante corrupción, truncamiento, contradicción y persistencia ambigua.

Los componentes y rutas de código concretos se determinarán únicamente después de inspeccionar el repositorio.

## 3. Límites

No se autoriza:
- modificar código o configuración funcional;
- iniciar pruebas de implementación, builds o experimentos;
- implementar Member ledger (TASK-04), wire protocol (TASK-05), HTTP gate (TASK-06) u otras tareas posteriores;
- agregar consenso, HA, multirregión, bases de datos, Kafka, Redis u otras dependencias;
- afirmar durabilidad frente a pérdida de energía/hardware sin evidencia;
- fijar TTL, fuente de reloj, skew o renovación pendientes;
- extrapolar semántica del GET sintético a operaciones financieras.

El gate de factibilidad CRL para producción permanece cerrado.

## 4. Secuencia obligatoria

`INSPECT READ-ONLY → SPEC → PLAN → TASKS → VALIDATE → HUMAN APPROVAL → IMPLEMENT → VERIFY`

La validación deberá cubrir:
- requisitos funcionales y no funcionales;
- invariantes, reglas y transiciones;
- escenarios positivos, negativos y casos borde;
- estrategia de pruebas y crash matrix;
- riesgos técnicos, filesystem/Windows, seguridad y recuperación;
- criterios de aceptación verificables y evidencia requerida;
- archivos/componentes dentro del alcance y exclusiones.

## 5. Gate de implementación

No modificar código hasta que la especificación y el plan estén completos, la validación del diseño esté documentada y el usuario otorgue aprobación explícita para implementar TASK-03.
