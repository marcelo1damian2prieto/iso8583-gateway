# Correcciones puntuales — revisión arquitectónica integral v2.0

**Archivo de destino:** `docs/architecture-review-gateway-v2.0-integral.md`

Este documento contiene reemplazos puntuales para conservar intacto el cuerpo técnico de la revisión arquitectónica. No reemplazar el archivo completo: aplicar únicamente los siguientes cambios.

## 1. Reemplazar el bloque de metadatos inicial

Reemplazar las líneas desde `**Estado:**` hasta `**Propósito:**` por:

```markdown
**Estado:** arquitectura integral v2.0 aprobada documentalmente por el usuario
**Versión:** 2.0-integral
**Base:** SPEC/PLAN/TASKS v1.0 aprobados, `architecture-review-v2.0.md`, decisiones posteriores del usuario y descripción del puesto Java Senior ISO8583 de Coderio.
**Implementación:** autorizada exclusivamente para BOOT-001..BOOT-007. Las épicas funcionales posteriores requieren autorización independiente.
**Propósito:** definir la arquitectura objetivo de un prototipo bancario moderno, operable y verificable, sin afirmar certificación ni compatibilidad con redes productivas.
```

## 2. Actualizar el principio arquitectónico 13 de la sección 3

Reemplazar:

```markdown
13. Ninguna implementación comienza sin aprobación explícita del SPEC/PLAN/TASKS y de las tareas.
```

Por:

```markdown
13. La documentación SPEC/PLAN/TASKS y la arquitectura integral v2.0 fueron aprobadas documentalmente. BOOT-001..BOOT-007 están autorizadas explícitamente; ninguna tarea funcional posterior puede ejecutarse sin autorización independiente.
```

## 3. Reemplazar la sección 15 completa

```markdown
## 15. Impacto sobre v1.0 y estado de aprobación

Los documentos v1.0 aprobados permanecen intactos y describen una línea base anterior. La arquitectura integral v2.0 no los modifica ni los invalida retroactivamente.

La arquitectura integral v2.0, SPEC v2.0, PLAN v2.0, TASKS v2.0 y los contratos DOC-001..DOC-009 fueron aprobados documentalmente por el usuario.

La aprobación documental no equivale a evidencia de implementación ni de pruebas ejecutadas.

El usuario autorizó explícitamente BOOT-001..BOOT-007. La autorización cubre únicamente esas tareas, sus dependencias declaradas y las verificaciones necesarias para satisfacer sus criterios de salida. Las épicas funcionales posteriores requieren autorización independiente.
```

## 4. Reemplazar la sección 16 completa

```markdown
## 16. Estado de validación y próximos pasos

La validación documental de la arquitectura v2.0 fue completada. Los requisitos funcionales y no funcionales, reglas de negocio, escenarios positivos/negativos/de borde, estrategia de testing, riesgos, criterios de aceptación y contratos se encuentran desarrollados en SPEC v2.0, PLAN v2.0, TASKS v2.0, DESIGN-VALIDATION.md y DOC-001..DOC-009.

Las decisiones de alcance vigentes incluyen:

1. MongoDB como única base de datos del Gateway; PostgreSQL queda excluido.
2. `0420/0430` fuera del baseline funcional.
3. DE52/PIN block no se persiste en el baseline ejecutable; operaciones PIN simuladas solo con fixtures sintéticos.
4. Reserva y recuperación de lotes coordinadas mediante MongoDB, con soporte multi-instancia.
5. Reconciliación operacional de operaciones indeterminadas, sin clearing/settlement completo.

El siguiente paso autorizado es ejecutar BOOT-001..BOOT-007 respetando dependencias, criterios de salida y gates.

**Gate CRL obligatorio:** verificar el soporte efectivo de revocación CRL en el stack TLS elegido y aplicar política fail-closed. Si el soporte no existe o no puede demostrarse, detener la tarea afectada, documentar evidencia y alternativas, y solicitar decisión. No sustituir CRL por OCSP, soft-fail ni otra política sin aprobación.

La ejecución debe detenerse al completar BOOT-001..BOOT-007 o ante un bloqueo/decisión fuera del alcance autorizado. No iniciar épicas funcionales posteriores sin autorización independiente.

**Estado final de este documento:** arquitectura integral v2.0 aprobada documentalmente. Bootstrap BOOT-001..BOOT-007 autorizado. Implementación funcional posterior no autorizada. No se afirma que exista implementación ni evidencia de tests ejecutados.
```

## 5. Verificación posterior

Después de aplicar los reemplazos:

- Buscar `borrador para validación humana`, `no aprobado`, `Implementación: no autorizada` y `propuesta integral para revisión humana`; no deben quedar como estado vigente.
- Mantener sin cambios el contenido técnico de las secciones 1 a 14, salvo el principio 13 indicado arriba.
- Confirmar que la sección 16 indique BOOT-001..BOOT-007 autorizadas y el gate CRL fail-closed.
