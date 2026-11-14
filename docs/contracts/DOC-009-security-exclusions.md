# DOC-009 — Exclusiones, datos sensibles, PCI y límites de seguridad

**Versión:** 2.0.1  
**Estado:** baseline de seguridad documental aprobado por el usuario.

## 1. Exclusiones funcionales

Fuera del baseline v2.0:

- `0420/0430` aviso de reverso; extensión separada que requiere nueva decisión y perfil.
- `0800/0810` administración de red.
- Clearing, settlement, liquidación, conciliación contable y archivos propietarios.
- Conexión real a Link/Prisma, certificación de red, HSM físico o core bancario real.
- Azure/Kubernetes en primera entrega.
- Persistencia de DE52/PIN block y cualquier CVV/CVC posterior a autorización.
- Uso de tarjetas, PAN, PIN, claves, certificados o datos reales.

## 2. Clasificación y tratamiento de datos

| Dato | Clasificación del laboratorio | Persistencia | Logs/traces/metrics |
|---|---|---|---|
| PAN sintético DE2 | CHD sintético | Omitir en parseo o cifrar campo; nunca claro | Nunca completo; preferir no registrar |
| PIN block DE52 | SAD | Excluido; descartar antes de archivar | Prohibido |
| CVV/CVC | SAD | No persistir después de autorización; baseline no lo usa | Prohibido |
| DE48 correlation token | Identificador opaco sensible | Guardar hash para lookup; valor solo transitorio/protegido si imprescindible | Prohibido en claro |
| STAN/RRN/DE7 | Referencias transaccionales | Solo minimizadas y con retención | No labels métricos; en logs usar IDs internos o truncar/enmascarar |
| DEK | Secreto criptográfico | Nunca en claro | Prohibido |
| KEK/root secret | Secreto de custodia | Solo key store simulado protegido fuera de Mongo transaccional | Prohibido |
| Wrapped DEK | Material de clave protegido | Sí, como metadata cifrada/opaque | No registrar valor |
| ISO payload | Datos transaccionales | Solo bytes saneados cifrados | Prohibido |
| Certificado privado/key material | Secreto | Fuera de Git e imágenes | Prohibido |

## 3. Saneamiento de mensaje

- Parser puede leer campos necesarios en memoria para validar reglas.
- DE52 y datos SAD se eliminan antes de crear representación persistible.
- La trama retenida se reconstruye desde mensaje parseado saneado y perfil versionado; no se conserva byte-for-byte la trama original si contiene DE52.
- No calcular digest sobre bytes originales con DE52. `digestScope=SANITIZED_ISO_BYTES`.
- El bitmap persistido debe corresponder al mensaje saneado y reflejar la ausencia de DE52. Bitmap original solo se podría retener como metadato si se justifica, pero baseline no lo necesita.
- PAN sintético se minimiza; si un test necesita comprobar igualdad, utilizar fixture ID o digest keyed, no PAN en logs.

## 4. Excepción PCI DSS 3.3.3 — no habilitada

La arquitectura conserva el tema como investigación regulatoria separada. La excepción de retención de SAD para emisores/entidades que soportan servicios de emisión exige necesidad legítima y documentada, limitación a lo necesario, controles de protección y validación con QSA/marcas. Ser banco o procesador no concede automáticamente la excepción.

**Decisión ejecutable:** DE52/PIN block no se persiste. Cifrar la trama completa no cambia esta exclusión ni elimina el alcance PCI. No implementar flag para habilitarlo sin aprobación formal externa y revisión de alcance/controles.

## 5. mTLS — ciclo de vida de laboratorio

- CA raíz privada de laboratorio creada fuera del repositorio; la clave privada de CA permanece offline.
- Certificados leaf separados por servicio/identidad; SAN DNS obligatorio y EKU `serverAuth`/`clientAuth` según rol.
- Vigencia inicial máxima: 30 días. No usar wildcard ni compartir certificado/clave entre servicios.
- Rotación: emitir leaf nuevo antes de vencimiento; aceptar temporalmente ambos certificados mediante bundle de trust durante ventana de 7 días; retirar leaf viejo al terminar.
- Revocación: CRL de la CA distribuida a trust managers; pruebas de certificado revocado deben fallar handshake. Si el stack elegido no soporta CRL fiable en el listener/cliente usado, gate bloqueante: no sustituirlo silenciosamente por expiración solamente; documentar y aprobar alternativa antes de Bootstrap.
- Compromiso de CA: rotar CA completa, emitir nuevos leafs y retirar CA antigua de trust bundle.
- Certificados y claves no se almacenan en Git ni Docker image; Compose los monta desde directorio local ignorado por Git con permisos restrictivos.
- No usar certificados self-signed leaf sin CA y trust explícito.

## 6. Autorización por identidad

| Identidad | Permisos |
|---|---|
| `gateway` | CS: archive/status; no decrypt general. Backend: authorize/process/reverse/status |
| `crypto-service` | HSM: generate/unwrap DEK; Mongo: colecciones messages/batches/reservations |
| `hsm-simulator` | Key store propio; no Mongo Gateway ni Backend |
| `ops-readonly` | Metadata saneada/audit de solo lectura; no decrypt |

Las identidades se verifican por certificado y autorización de aplicación; mTLS autentica canal, no reemplaza autorización.

## 7. Logging, métricas y trazas

- Logs JSON con `traceId`, `operationId`, `messageId`, service, event, outcome, duration y códigos no sensibles.
- Prohibido loguear request/response body, ISO bytes, PAN, DE52, DE48 token, claves, certificados privados, Authorization headers o excepciones que incluyan payload.
- Métricas con labels de cardinalidad acotada: servicio, MTI, resultado normalizado, dependencia. No PAN, STAN, RRN, token, operationId, messageId ni terminal ID como labels.
- Trazas sin payload; atributos de negocio reducidos a MTI/estado/código normalizado.
- Excepciones deben mapear a códigos saneados; no serializar objetos ISO/DTO completos.

## 8. Límites de afirmaciones

El proyecto es un laboratorio educativo y demostrativo. No afirmar:

- Compatibilidad/certificación Link o Prisma.
- Cumplimiento PCI DSS, PCI PIN, FIPS 140-3 o certificación HSM.
- Seguridad equivalente a HSM físico o vault gestionado.
- Alta disponibilidad con replica set Mongo de un nodo.
- Throughput/latencia de producción sin benchmark y entorno representativo.

## 9. Criterios de aceptación

- Fixtures sintéticos únicamente.
- Tests automáticos que buscan patrones de PAN, DE52, PIN, DE48, claves y payload en logs/traces/exceptions.
- Certificados válidos/inválidos/expirados/revocados y rotación overlap probados.
- Verificación de permisos por identidad para cada endpoint.
- Inspección de documentos Mongo confirma DE52 ausente, PAN no claro y ciphertext authenticated.
- README y mensajes de salida contienen límites de compliance/compatibilidad.
