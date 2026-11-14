# AGENTS — ISO8583 Gateway Lab v2.0

## Estado y autoridad

- Arquitectura integral v2.0 aprobada como dirección arquitectónica.
- DOC-001..DOC-009, SPEC v2.0, PLAN v2.0 y TASKS v2.0 fueron aprobados documentalmente por el usuario.
- La aprobación documental no equivale a autorización de ejecución. Bootstrap e implementación requieren autorización explícita por IDs/rango BOOT-*.
- La aprobación de DOC-001..DOC-009 no permite por sí sola crear código, scaffolding, builds, Docker Compose ni ejecutar BOOT-*.
- Ejecutar únicamente IDs/rangos expresamente autorizados por el usuario; respetar dependencias y gates.

## Metodología obligatoria

`SPEC → PLAN → TASKS → VALIDATE → APPROVE → IMPLEMENT → VERIFY`

Antes de cualquier implementación, revisar requisitos funcionales/no funcionales, reglas de negocio, escenarios positivos/negativos/borde, contratos, estrategia de testing, riesgos técnicos/operativos/seguridad, criterios verificables y dependencias. Si hay contradicción o gate bloqueante, detenerse y reportar.

## Principios

- Java 21, Spring Boot 3.x, Maven, Netty, jPOS, WebClient, MongoDB, Docker Compose.
- MongoDB única base del Gateway; no agregar PostgreSQL ni otra base transaccional.
- Arquitectura hexagonal pragmática; DDD pragmático; TDD selectivo; Security First.
- No bloquear EventLoops con llamadas remotas ni persistencia.
- No usar locks locales para coordinar instancias distribuidas.
- Preferir atomicidad de documento, índices únicos, updates condicionales y transacciones Mongo breves.
- No mantener transacciones Mongo durante llamadas remotas.
- No reintentar ciegamente operaciones financieras/reversos con resultado incierto.
- No persistir DE52/PIN block en baseline; PIN simulator solo con fixtures sintéticos y nunca devuelve PIN en claro.
- KEK protegida por HSM Simulator; DEK transitoria; nunca loguear claves.
- No afirmar compatibilidad Link/Prisma ni cumplimiento PCI/FIPS.
- `0420/0430`, clearing/settlement y Azure/Kubernetes quedan fuera del primer alcance.

## Seguridad

- Nunca registrar PAN completo, PIN/PIN block, CVV/CVC, DEK, KEK, secretos, certificados privados ni payload ISO completo.
- No usar datos de tarjetas reales; fixtures sintéticos.
- Secretos fuera del repositorio y de imágenes.
- TLS inbound y mTLS interno según DOC-005/DOC-009.
- Cifrado autenticado AES-256-GCM; nonce único por DEK; AAD contextual.
- No fallback criptográfico a claves locales.
- Backups de datos y claves separados; probar restauración conjunta.

## Calidad y verificación

- Pruebas unitarias para reglas/estados/idempotencia/correlación/reversos/lotes.
- Pruebas de integración con Mongo replica set y servicios reales del Compose.
- Pruebas multi-instancia, fault injection, recuperación y leakage de logs/traces.
- No afirmar éxito sin comandos y resultados observados.
- Registrar evidencia de VERIFY y detenerse al terminar el alcance autorizado.

## Gestión de cambios

Si se necesita cambiar alcance, contrato, decisión arquitectónica, dependencia, criterio de aceptación o tarea, detenerse y solicitar aprobación. No activar metodologías/skills adicionales salvo invocación explícita del usuario.
