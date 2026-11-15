# Estado del laboratorio CRL-FEASIBILITY-009A.1

**Fecha de corte:** 2026-11-15  
**TASK-02:** `CLOSED — HUMAN APPROVED`  
**TASK-03:** `AUTHORIZED — DESIGN PREPARATION AND VALIDATION ONLY`  
**Gate CRL producción:** `CLOSED`

## Estado

TASK-02-DRAFT-2 fue aprobada y cerrada por decisión humana explícita. Sus siete documentos se conservan en este directorio como paquete de referencia.

TASK-03 está autorizada exclusivamente para:
- inspección read-only del repositorio y del estado real del código pertinente;
- preparación de SPEC;
- preparación de PLAN ejecutable;
- desglose TASKS;
- validación del diseño, incluidos escenarios, pruebas, riesgos, seguridad y criterios de aceptación.

La implementación permanece bloqueada hasta completar la validación del diseño y recibir aprobación humana explícita. No se autorizan TASK-04 ni tareas posteriores.

## Restricciones heredadas

- Coordinator único de laboratorio; sin consenso, HA, multirregión ni dependencias externas nuevas.
- GET sintético; no extrapolar decisiones a operaciones financieras.
- `PENDING` y `UNKNOWN` no reconciliado bloquean `COMMIT_GLOBAL`.
- `RECONCILED_UNKNOWN` conserva resultado desconocido y nunca equivale a `COMPLETE`.
- `PERMIT_RECEIVED` se fuerza en ledger local del Member antes de enviar ACK al Coordinator.
- El rechazo por expiración se confirma con persistencia durable de `ADMISSION_REJECTED_EXPIRED` en Coordinator; el ACK solo comunica el resultado.
- Fail-closed ante corrupción, truncamiento, contradicción o persistencia ambigua.
- No afirmar durabilidad frente a pérdida de energía/hardware sin evidencia.
- TTL, reloj, tolerancia de desfase y renovación pre-consumo siguen pendientes; no inventar parámetros.
- El gate de factibilidad CRL para producción permanece cerrado.

## Separación del proyecto ISO8583 Gateway

Este directorio contiene documentación de la investigación de factibilidad CRL 009A.1. No modifica la arquitectura integral ISO8583 Gateway v2.0, sus contratos DOC-001..DOC-009, ni las autorizaciones de Bootstrap BOOT-001..BOOT-007.

La documentación principal del Gateway permanece en `README.md`, `AGENTS.md` y `docs/`. Esta subcarpeta mantiene aislado el registro CRL.

## Próximo paso

Preparar y revisar SPEC → PLAN → TASKS → VALIDATE para TASK-03, sin modificar código ni ejecutar implementación.
