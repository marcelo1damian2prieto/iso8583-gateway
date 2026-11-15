# CRL-FEASIBILITY-009A.1 — Requisitos para provisión del workspace aislado

**Revisión:** WORKSPACE-PROVISIONING-REQUIREMENTS-DRAFT-4  
**Fecha:** 2026-11-15  
**Estado:** PROPOSAL — PENDING HUMAN REVIEW  
**Tipo:** planificación documental; no ejecuta operaciones de filesystem.  
**Autorización:** no autoriza copiar, crear, modificar o borrar archivos.

## 1. Propósito

Definir el gate previo para disponer el workspace de código aislado requerido por TASK-03, sin alterar el laboratorio histórico 009A.

La provisión es una operación separada de T03-00. T03-00 permanece exclusivamente read-only y solo puede comenzar cuando la ruta destino haya sido provisionada y su procedencia esté documentada.

## 2. Evidencia y rutas

### Origen candidato

El informe TASK-01 identifica el proyecto inspeccionado en:

C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a\project

Esta ruta es el origen candidato de la copia aislada, sujeto a comprobación de existencia, identidad y estado actual antes de cualquier operación. No se da por inspeccionado en esta revisión.

### Destino requerido

C:\Users\marce\AppData\Local\Temp\crl-feasibility-009a1\project

La captura del usuario del 2026-11-15 muestra que la raíz 009A.1 contiene ocho documentos Markdown y no muestra un subdirectorio project. El usuario confirmó que ese es todo su contenido.

### Restricción de preservación

El directorio 009A debe conservarse inmutable como evidencia histórica. La provisión futura debe ser una copia; nunca un movimiento, renombrado, limpieza o reconstrucción sobre el origen.

## 3. Decisiones previas obligatorias

Antes de copiar, una propuesta de provisión debe definir y recibir aprobación humana expresa para:

1. **Identidad del origen:** confirmar que el directorio fuente existe y que corresponde al proyecto lab.crl:crl-feasibility-009a:1.0.0 reportado, registrando estado Git y HEAD si existe.
2. **Alcance de copia:** enumerar por ruta los archivos que se copiarán. Incluir fuentes, tests, POM, configuración de proyecto, fixtures no sensibles y documentación necesaria; no dar por aprobada una copia indiscriminada de todo el directorio.
3. **.git:** decidir expresamente si se conserva metadata Git en la copia. Si existe, registrar branch, HEAD y estado previo; no alterar refs ni working tree del origen.
4. **Artefactos generados:** decidir el tratamiento de target/, caches, logs y reportes. No asumir que son fuente; preservar solo evidencia cuya retención sea necesaria y esté aprobada.
5. **Material sensible:** identificar por nombre/ruta la presencia de .p12, .pfx, .jks, .key, .pem, tokens, passwords u otros secretos sin leer su contenido. Definir por elemento si se excluye o si es fixture sintético local indispensable. No copiar secretos al repositorio GitHub ni a informes.
6. **Evidencia histórica:** decidir qué logs, ZIPs e informes de 009A son necesarios para el arnés aislado. Mantenerlos claramente etiquetados como evidencia 009A, no como resultado nuevo 009A.1.
7. **Destino:** confirmar que 009a1\project no existe o está vacío y que ninguna operación sobrescribirá archivos existentes.
8. **Reproducibilidad:** fijar manifiesto de origen/destino con ruta relativa, tamaño y SHA-256 de cada archivo no sensible copiado; registrar las exclusiones con motivo. Verificar igualdad de hashes origen/destino.
9. **Rollback:** definir cómo retirar únicamente la copia recién creada si falla la verificación, sin tocar el origen 009A ni los ocho documentos de la raíz 009A.1.

Si cualquiera de estas decisiones no puede resolverse con evidencia, detenerse y solicitar decisión; no completar huecos por inferencia.

## 4. Secuencia propuesta, todavía no autorizada

1. Revisar esta especificación y aprobar expresamente el alcance de copia.
2. Inspeccionar read-only el origen y generar inventario previo, incluyendo Git, archivos, material sensible por nombres y hashes de archivos no sensibles.
3. Confirmar que el destino no existe y que la raíz 009A.1 contiene únicamente los documentos previstos, sin sobrescrituras.
4. Ejecutar una copia controlada según el manifiesto aprobado, sin modificar el origen.
5. Generar inventario del destino y comparar SHA-256 de todos los archivos no sensibles copiados.
6. Registrar archivos excluidos y la justificación de cada exclusión.
7. Confirmar que 009A no cambió y que el destino contiene exactamente el alcance aprobado.
8. Entregar el informe de provisión para revisión humana.
9. Solo después de aceptar el resultado, ejecutar T03-00 en modo read-only sobre 009a1\project.

Esta secuencia no incluye Maven, Java, tests, builds ni experimentos.

## 5. Criterios de aceptación de provisión

- El proyecto origen 009A permanece sin cambios.
- El destino es exactamente %TEMP%\crl-feasibility-009a1\project.
- La copia coincide con el manifiesto aprobado.
- SHA-256 de cada archivo no sensible copiado coincide entre origen y destino.
- Archivos sensibles/excluidos quedan identificados por ruta y motivo, sin divulgar contenido o valores.
- Metadata Git, si existe, se trata según la decisión aprobada y se registra.
- No hay sobrescritura silenciosa ni mezcla con documentos de la raíz.
- El informe distingue evidencia histórica 009A de workspace aislado 009A.1.

## 6. Límites

- Esta propuesta no crea project ni copia archivos.
- No autoriza modificar 009A, los documentos principales del Gateway o el repositorio GitHub fuera de la publicación documental aprobada.
- No resuelve D03-OPEN-01..08.
- No modifica la aceptación humana de H-01.
- No autoriza TASK-03 implementación, Maven, builds, tests ni experimentos.
- No abre el gate de factibilidad CRL para producción.

**Estado resultante mientras no se apruebe y ejecute la provisión:** T03-00 = BLOCKED — WORKSPACE NOT PROVISIONED.
