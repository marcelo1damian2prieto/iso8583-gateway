# Contratos y decisiones DOC-001..DOC-009

Estos documentos fueron elaborados bajo autorización del usuario y posteriormente aprobados documentalmente junto con SPEC/PLAN/TASKS v2.0. Definen el perfil y contratos privados del laboratorio; no son APIs implementadas ni evidencia de pruebas.

| ID | Documento | Entregable |
|---|---|---|
| DOC-001 | `DOC-001-traceability.md` | Matriz RF/NFR → componente → tarea → evidencia |
| DOC-002 | `DOC-002-iso8583-lab-profile.md` | Perfil `LAB-ATM-1.0`, framing, campos, MTI y DE39 |
| DOC-003 | `DOC-003-correlation-reversal.md` | Correlación, idempotencia y referencia inequívoca de reverso |
| DOC-004 | `DOC-004-backend-api.md` | Contrato REST `backend-api-v1` |
| DOC-005 | `DOC-005-crypto-hsm-contract.md` | Contratos `crypto-api-v1` y `hsm-api-v1` |
| DOC-006 | `DOC-006-durability-policy.md` | Durabilidad y orden transaccional |
| DOC-007 | `DOC-007-timeouts-retention.md` | Timeouts, idempotency window, retención y expiración |
| DOC-008 | `DOC-008-mongodb-consistency.md` | MongoDB concerns, colecciones, índices, CAS y recovery |
| DOC-009 | `DOC-009-security-exclusions.md` | Clasificación, exclusiones, PCI y ciclo mTLS |

## Decisiones privadas del laboratorio

- `LAB-ATM-1.0` usa framing length-prefix de 2 bytes big-endian y perfil ASCII con bitmap binario. No atribuirlo a Link/Prisma.
- Correlación 0100→0200 usa token DE48 privado; reverso 0400 identifica original mediante DE90 + DE37 + origen + datos de negocio.
- El Gateway archiva request inbound cifrado/saneado antes de invocar Backend y archiva respuesta outbound antes de enviarla.
- Defaults temporales: 24 h de detección de duplicados, retención idempotencia 7 días tras terminalidad, mensajes 30 días, operaciones 90 días, audit 180 días; todos configurables.
- MongoDB usa readConcern majority, writeConcern majority+journal, primary para lecturas críticas; transacciones breves.
- mTLS usa CA privada offline, leafs por servicio, 30 días de vigencia, rotación con overlap de 7 días y CRL. El soporte CRL del stack elegido es gate técnico antes de considerar mTLS listo.
- `0420/0430`, DE52 persistido, clearing/settlement, redes reales y PCI compliance quedan fuera del baseline.

## Pendientes técnicos, no decisiones de alcance

- Fijar versiones de dependencias y validar compatibilidad en BOOT-001.
- Elegir herramientas/umbrales del Quality Gate en BOOT-004.
- Probar soporte CRL en el stack TLS y detenerse si no cumple DOC-009.
- Generar OpenAPI 3.1 y contract tests a partir de DOC-004/005 durante implementación autorizada.
- Validar todos los criterios con tests reales; esta etapa solo cerró definición documental.
