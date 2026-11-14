# DOC-002 — Perfil ISO8583 de laboratorio `LAB-ATM-1`

**Versión de perfil:** `LAB-ATM-1.0`  
**Estado:** perfil de laboratorio aprobado documentalmente por el usuario  
**Base:** ISO 8583:1987 como referencia conceptual; no es perfil oficial de Link, Prisma ni otra red.

## 1. Propósito y límites

Este perfil fija una representación reproducible para el laboratorio ATM Withdrawal. Todas las decisiones de codificación, framing, campos y códigos que siguen son decisiones privadas del laboratorio. No deben extrapolarse a redes productivas.

## 2. Transporte y framing

- Transporte TCP sobre TLS.
- Framing: prefijo binario de longitud de **2 bytes unsigned big-endian**, seguido de exactamente esa cantidad de bytes de mensaje ISO.
- El prefijo no se incluye en el hash del mensaje ISO ni en el payload ISO persistido; se conserva como metadata de transporte si se necesita.
- Longitud mínima/máxima del frame ISO: 12 a 4.096 bytes. El decoder rechaza cero, menos de 12 y más de 4.096.
- El decoder soporta fragmentación, concatenación y cierre con frame incompleto. Un frame inválido cierra la conexión sin intentar resincronización heurística.

## 3. Codificación ISO

- MTI: 4 caracteres ASCII.
- Bitmap primario/secundario: binario, 8 bytes cada uno; el bit 1 del primario indica bitmap secundario.
- Campos numéricos `n`: ASCII decimal, longitud fija salvo indicación `LLVAR`.
- Campos alfanuméricos `ans`: ASCII imprimible, longitud fija salvo indicación `LLVAR`.
- `LLVAR`: prefijo de longitud de 2 dígitos ASCII, seguido del valor.
- No se admiten bytes fuera de ASCII en los campos definidos como ASCII.
- No se incluye MAC ISO (DE64/DE128) en el perfil de laboratorio; la seguridad de transporte se demuestra mediante TLS. Esto no sustituye los controles de red productiva.
- No se incluyen EMV/ICC data (DE55), PIN block (DE52), CVV/CVC ni datos reales.

## 4. Diccionario de campos

| DE | Nombre de laboratorio | Formato | Longitud | Persistencia/uso |
|---:|---|---|---:|---|
| 2 | PAN sintético | `n LLVAR` | 1–19 | Solo sintético; en documentos parseados se omite o cifra por campo; nunca log |
| 3 | Processing Code | `n` | 6 | Código de operación del perfil |
| 4 | Amount, Transaction | `n` | 12 | Importe en unidad menor, cero-padding |
| 7 | Transmission Date & Time | `n` | 10 (`MMDDhhmmss`) | UTC en perfil de laboratorio (`MMDDhhmmss`); año se resuelve por ventana temporal de recepción y se acompaña de timestamp UTC interno |
| 11 | STAN | `n` | 6 | Referencia de mensaje, no clave idempotente aislada |
| 12 | Local Transaction Time | `n` | 6 (`hhmmss`) | Hora local de terminal simulada |
| 13 | Local Transaction Date | `n` | 4 (`MMDD`) | Fecha local de terminal simulada |
| 18 | Merchant Type | `n` | 4 | Valor de laboratorio ATM `6011` |
| 22 | POS Entry Mode | `n` | 3 | Valor permitido configurado en fixture; no se interpreta como regla de red |
| 32 | Acquiring Institution ID | `n LLVAR` | 1–11 | ID sintético del originador/adquirente |
| 33 | Forwarding Institution ID | `n LLVAR` | 1–11 | ID sintético; obligatorio en request financiero y reversible para DE90 |
| 37 | Retrieval Reference Number | `ans` | 12 | Generado por Backend Simulator; requerido para referencia/reversa |
| 39 | Response Code | `n` | 2 | Código privado de laboratorio definido en §7 |
| 41 | Terminal ID | `ans` | 8 | Identificador sintético de terminal |
| 42 | Card Acceptor ID | `ans` | 15 | Identificador sintético de comercio/adquirente |
| 48 | Private Correlation Token | `ans LLVAR` | 32 | Token aleatorio 128-bit representado como 32 hex ASCII; extensión privada |
| 49 | Currency Code, Transaction | `n` | 3 | Código numérico ISO 4217; laboratorio usa moneda configurada |
| 90 | Original Data Elements | `ans` | 42 | `originalMTI(4) + originalSTAN(6) + originalDE7(10) + originalDE32(11) + originalDE33(11)`; IDs numéricos zero-padded |

### Campos expresamente prohibidos en baseline

- DE52 PIN Data.
- DE55 Integrated Circuit Card System Related Data.
- DE CVV/CVC u otros datos de autenticación sensibles no incluidos en el perfil.
- DE64/DE128 MAC no se implementan en este perfil.
- Cualquier DE no listado para el MTI concreto se rechaza como no soportado; no se ignora silenciosamente.

## 5. MTI y campos obligatorios

Los campos enumerados como obligatorios deben estar presentes. Los campos opcionales solo se admiten si aparecen en la tabla de cada MTI. Los campos no enumerados están prohibidos.

| MTI | Descripción | Campos obligatorios | Opcionales |
|---|---|---|---|
| `0100` | Authorization Request | 2,3,4,7,11,18,22,32,33,41,42,49 | 12,13 |
| `0110` | Authorization Response | 3,4,7,11,32,33,37,39,41,49 | 12,13,48 |
| `0200` | Financial Request | 2,3,4,7,11,18,22,32,33,37,41,42,48,49 | 12,13 |
| `0210` | Financial Response | 3,4,7,11,32,33,37,39,41,49 | 12,13,48 |
| `0400` | Reversal Request | 3,4,7,11,32,33,37,41,42,49,90 | 12,13,48 |
| `0410` | Reversal Response | 3,4,7,11,32,33,37,39,41,49,90 | 12,13,48 |

El bitmap debe reflejar exactamente los campos presentes. El validador comprueba el bitmap, el diccionario del MTI, las longitudes y la coherencia de los campos. `0110` aprobado requiere DE48; `0110` rechazado no emite token. `0200` debe devolver exactamente el token DE48 asociado a la autorización. DE48 no es un estándar universal de correlación.

## 6. Processing Codes

Valores privados de laboratorio, DE3:

- `010000`: retiro ATM para autorización `0100` y financiero `0200`.
- `200000`: reservado/no soportado en `LAB-ATM-1`; no usar para `0400`.

`0200` debe conservar el mismo DE3 que la autorización correlacionada. `0400` debe conservar el DE3 del mensaje financiero original.

## 7. Response Codes — DE39

| Código | Significado de laboratorio | Terminalidad |
|---|---|---|
| `00` | Aprobado/completado | Éxito confirmado |
| `05` | Rechazado por backend | Rechazo confirmado |
| `12` | Request inválido/no soportado | Rechazo de validación |
| `14` | PAN sintético/formato de cuenta inválido | Rechazo de validación |
| `30` | Error de formato ISO | Rechazo de validación |
| `51` | Fondos insuficientes simulado | Rechazo de negocio |
| `54` | Autorización expirada/no elegible | Rechazo de negocio |
| `57` | Operación no permitida en estado actual | Rechazo de negocio |
| `91` | Dependencia no disponible antes de efecto confirmado | Fallo temporal conocido |
| `96` | Error interno conocido sin efecto confirmado | Fallo técnico conocido |

No existe DE39 para expresar de forma inequívoca un resultado financiero indeterminado. Si el backend pudo ejecutar la operación y la respuesta se perdió, el Gateway no debe mapearlo a `05`, `91` o `96` como si el resultado fuese conocido. Se mantiene estado interno `INDETERMINATE` y se aplica la política de DOC-006/DOC-007. El mapeo de respuesta al canal queda bloqueado hasta resolver el resultado o aplicar una respuesta de protocolo explícitamente acordada; baseline: no fabricar respuesta terminal.

## 8. Reglas de consistencia

- DE4 es 12 dígitos en unidad menor; no se admiten signo, decimal ni cero para retiro.
- DE7 se trata como referencia de mensaje y participa de la huella; el instante UTC de recepción se registra separadamente.
- DE11 no es único global ni suficiente para correlacionar/idempotencia.
- DE32/DE33/DE41/DE42 identifican el origen lógico en el perfil.
- DE37 se asigna al aceptar/procesar la operación según Backend Simulator; no se inventa antes de conocer el contrato.
- Para `0200`, DE37 debe coincidir con la referencia de autorización emitida en `0110` y guardada en operación, si el backend la devuelve.
- `0400` requiere DE90 que referencia un `0200` original y DE37 igual al RRN del original.
- Importe, moneda, DE3, origen DE32/33 y terminal DE41 deben coincidir con la operación original para admitir reverso.
- No se permite un segundo `0400` exitoso para la misma operación original.

## 9. Fixtures

Fixtures obligatorios, todos sintéticos:

1. `0100` válido y `0110` aprobado con DE48.
2. `0110` rechazado sin DE48.
3. `0200` válido con token de `0110` y RRN.
4. `0210` aprobado/rechazado.
5. `0400` con DE90 exacto y `0410` aprobado/rechazado.
6. Mensajes inválidos: bitmap inconsistente, DE requerido ausente, DE prohibido, LLVAR inválido, ASCII inválido, monto cero, token malformado y DE90 inconsistente.

## 10. Criterios de salida

- Perfil versionado en fixtures y configuración jPOS.
- Pack/unpack round-trip para los seis MTI.
- Bitmap y longitud validados negativamente.
- DE90 se construye y compara por sus componentes, no por substring heurístico.
- Ninguna documentación o log presenta `LAB-ATM-1` como perfil Link/Prisma.
