# ISO8583 Gateway Lab — Documentación v2.0

**Estado:** arquitectura, SPEC v2.0, PLAN v2.0, TASKS v2.0 y contratos DOC-001..DOC-009 aprobados documentalmente por el usuario.  
**Arquitectura:** v2.0 integral aprobada por el usuario.  
**Código/Bootstrap:** pendientes de autorización explícita por IDs de tarea BOOT-*; la aprobación documental no habilita por sí sola la ejecución.

## Contenido

- `docs/SPEC.md`: requisitos funcionales/no funcionales, reglas de negocio y criterios de aceptación.
- `docs/PLAN.md`: arquitectura, contratos, persistencia, criptografía, fases y gates.
- `docs/TASKS.md`: backlog trazable con dependencias y criterios de salida.
- `docs/DESIGN-VALIDATION.md`: revisión de requisitos, escenarios, testing, riesgos y gates pendientes.
- `docs/architecture-review-gateway-v2.0-integral.md`: copia de la arquitectura aprobada como referencia de origen.
- `docs/contracts/`: nueve entregables DOC-001..DOC-009 de trazabilidad, perfil ISO, correlación/reverso, contratos REST, durabilidad, timeouts/retención, MongoDB y seguridad/exclusiones.
- `AGENTS.md`: instrucciones para herramientas/agentes que trabajen sobre el proyecto.

## Decisiones clave

- MongoDB es la única base de datos del Gateway; PostgreSQL queda excluido.
- `0400/0410` está incluido; `0420/0430` queda fuera del baseline inicial.
- DE52/PIN block no se persiste en el baseline ejecutable.
- HSM Simulator independiente; Crypto Service multi-instancia.
- DEK por lote; lote híbrido: 1.000 reservas o 15 minutos, lo que ocurra primero.
- mTLS para los canales internos definidos.

## Límites

El perfil ISO es propio del laboratorio. No se afirma compatibilidad con Link/Prisma, certificación HSM, PCI DSS ni PCI PIN. Azure/Kubernetes, clearing/settlement y operación productiva quedan fuera de la primera entrega.

## Gate de ejecución

La autorización actual cubre solo DOC-001..DOC-009. No iniciar Bootstrap ni implementar código hasta validar los contratos, actualizar/aprobar SPEC/PLAN/TASKS y recibir autorización explícita para IDs BOOT-* concretos.
