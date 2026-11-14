# ISO8583 Gateway Lab

**Versión:** 2.0  
**Estado documental:** aprobado por el usuario  
**Bootstrap:** BOOT-001 a BOOT-007 autorizadas  
**Implementación funcional:** pendiente de las siguientes autorizaciones  
**Repositorio:** [iso8583-gateway](https://github.com/marcelo1damian2prieto/iso8583-gateway)

## 1. Objetivo

Proyecto de laboratorio para diseñar e implementar un ISO8583 Gateway moderno, utilizando tecnologías actuales del ecosistema Java.

El laboratorio contempla autorización, procesamiento financiero y reversos, persistencia durable, idempotencia, recuperación ante fallos, seguridad criptográfica y observabilidad.

El objetivo es construir una solución técnicamente defendible, con arquitectura modular, contratos explícitos y pruebas verificables.

Este proyecto es un laboratorio: no representa una implementación certificada para redes Link o Prisma ni un sistema habilitado para procesar transacciones reales.

## 2. Documentación de referencia

Los documentos aprobados constituyen la fuente de verdad del proyecto:

- `AGENTS.md`: reglas e instrucciones obligatorias para agentes de desarrollo.
- `docs/SPEC.md`: requisitos funcionales, no funcionales y criterios de aceptación.
- `docs/PLAN.md`: arquitectura, componentes, contratos y estrategia de implementación.
- `docs/TASKS.md`: backlog, dependencias y criterios de salida.
- `docs/DESIGN-VALIDATION.md`: validación del diseño, escenarios, testing y riesgos.
- `docs/architecture-review-gateway-v2.0-integral.md`: arquitectura integral aprobada.
- `docs/contracts/`: contratos DOC-001 a DOC-009.

Los documentos deben interpretarse de manera consistente. Ante contradicciones, ambigüedades o decisiones que excedan el alcance aprobado, el agente debe detenerse y solicitar una decisión.

## 3. Arquitectura y tecnologías

Stack tecnológico previsto:

- Java 21.
- Spring Boot 3.x.
- Maven.
- Netty para transporte TCP/TLS.
- jPOS para mensajería ISO8583.
- Spring WebClient para integraciones REST.
- MongoDB como única base de datos del Gateway.
- Docker Compose.
- JUnit 5, Mockito y Testcontainers.

Servicios previstos:

1. **ISO8583 Gateway:** recepción TCP/TLS, framing, parsing, validación, correlación, idempotencia y orquestación.
2. **Backend/Core Simulator:** simulación de autorización, operación financiera, reversos y consulta de estado.
3. **Cryptographic Service:** cifrado documental, administración de lotes y recuperación.
4. **HSM Simulator:** simulación de custodia de KEK, generación y protección de DEK y operaciones PIN de prueba.
5. **MongoDB Replica Set:** persistencia operacional y documental del Gateway.

## 4. Decisiones arquitectónicas principales

- MongoDB es la única base de datos del Gateway. PostgreSQL queda excluido.
- Los MTI `0100/0110`, `0200/0210` y `0400/0410` forman parte del alcance inicial.
- `0420/0430` queda fuera del baseline.
- DE52/PIN block no se persiste en el baseline ejecutable.
- El HSM Simulator es un servicio independiente y no almacena transacciones.
- Se utiliza envelope encryption con AES-256-GCM.
- La DEK se genera por lote y permanece transitoriamente en memoria del Cryptographic Service.
- El lote se cierra al alcanzar 1.000 reservas o 15 minutos, lo que ocurra primero.
- Los servicios internos definidos utilizan mTLS.
- La revocación de certificados debe verificarse en el stack TLS real, con política fail-closed.

## 5. Estado de autorización y ejecución

### Documentación

Los siguientes entregables fueron aprobados documentalmente:

- SPEC v2.0.
- PLAN v2.0.
- TASKS v2.0.
- Arquitectura integral v2.0.
- DOC-001 a DOC-009.

### Bootstrap autorizado

El usuario autorizó explícitamente la ejecución de las tareas:

| ID | Descripción |
|---|---|
| BOOT-001 | Verificar y fijar versiones compatibles |
| BOOT-002 | Crear estructura de proyectos y servicios |
| BOOT-003 | Configurar Maven Wrapper y builds |
| BOOT-004 | Configurar Quality Gate |
| BOOT-005 | Crear Docker Compose y MongoDB Replica Set |
| BOOT-006 | Configurar secretos y certificados |
| BOOT-007 | Configurar health y readiness |

La autorización cubre exclusivamente estas tareas y sus verificaciones necesarias.

No habilita la implementación de las épicas funcionales posteriores.

### Gate obligatorio: CRL

Durante el Bootstrap, Codex debe verificar el soporte efectivo de revocación CRL en el stack TLS elegido.

La política debe ser fail-closed: si no puede comprobarse el estado de revocación, no se debe aceptar el certificado.

Si el stack no soporta el comportamiento requerido, Codex debe:

1. Detener la tarea afectada.
2. Documentar la limitación técnica y las alternativas.
3. Generar el informe de ejecución.
4. Solicitar autorización o decisión antes de continuar.

No debe sustituir CRL por OCSP, soft-fail u otra estrategia sin aprobación.

## 6. Metodología de trabajo

El proyecto utiliza Hybrid Spec-First SDD:

`SPEC → PLAN → TASKS → VALIDATE → APPROVE → IMPLEMENT → VERIFY`

Principios:

- DDD pragmático.
- TDD selectivo.
- Security First.
- Evitar overengineering.
- Mantener el dominio independiente de frameworks.
- Priorizar contratos explícitos y trazabilidad.
- No introducir cambios de arquitectura o alcance sin aprobación.

## 7. Flujo de trabajo con Codex

Codex es el agente responsable de ejecutar el desarrollo mediante prompts delimitados.

Para cada ejecución debe:

1. Leer `AGENTS.md` y los documentos pertinentes.
2. Verificar el alcance autorizado.
3. Revisar dependencias y criterios de aceptación.
4. Implementar únicamente las tareas indicadas en el prompt.
5. Ejecutar las pruebas y controles de calidad disponibles.
6. Registrar decisiones, limitaciones y desviaciones.
7. Detenerse al completar el alcance o encontrar un bloqueo.

### Informe obligatorio de ejecución

Al finalizar cada prompt, Codex debe generar un informe Markdown que incluya:

- Resumen de ejecución.
- Tareas abordadas y estado.
- Archivos creados, modificados o eliminados.
- Decisiones técnicas y justificación.
- Comandos ejecutados.
- Pruebas y resultados reales.
- Quality Gate y resultados.
- Verificaciones de seguridad.
- Desviaciones respecto de la documentación.
- Bloqueos y decisiones requeridas.
- Evidencias y referencias a commits.
- Recomendación del siguiente paso.

Los resultados no ejecutados deben identificarse como tales. No se permite declarar exitosas pruebas sin evidencia.

Los informes deben conservarse en el repositorio, dentro de `docs/reports/`, con nombres que identifiquen la tarea o ejecución.

## 8. Revisión y aprobación

El ciclo de revisión será:

1. ChatGPT analiza y prepara el prompt.
2. El usuario entrega el prompt a Codex.
3. Codex ejecuta el alcance autorizado y genera el informe.
4. El usuario comparte los resultados y actualiza GitHub.
5. ChatGPT revisa código, documentación, pruebas y evidencias.
6. Se identifican observaciones y correcciones.
7. Se aprueba el bloque antes de avanzar al siguiente.

La finalización declarada por Codex no equivale a aprobación técnica.

## 9. Seguridad y límites

- Utilizar exclusivamente datos y tarjetas sintéticos.
- No registrar PAN completo, PIN, PIN block, CVV/CVC, DEK, KEK ni secretos.
- No incluir credenciales, claves privadas o certificados privados en Git.
- No exponer payloads ISO completos en logs o trazas.
- No implementar fallback criptográfico a claves locales.
- No afirmar cumplimiento PCI DSS, PCI PIN, FIPS ni certificación HSM.
- No afirmar compatibilidad con redes Link o Prisma.
- No implementar clearing/settlement ni despliegue Azure/Kubernetes en el baseline inicial.

## 10. Estado actual

El repositorio contiene la documentación aprobada y las instrucciones de ejecución.

El siguiente trabajo autorizado es el Bootstrap BOOT-001 a BOOT-007 mediante Codex.

La implementación funcional comenzará en etapas posteriores, una vez revisados y aprobados los resultados del Bootstrap y autorizadas las tareas correspondientes.

---

**Regla fundamental:** ninguna tarea fuera del alcance explícitamente autorizado puede ejecutarse por inferencia. Ante cualquier bloqueo o decisión arquitectónica relevante, detenerse y solicitar aprobación.