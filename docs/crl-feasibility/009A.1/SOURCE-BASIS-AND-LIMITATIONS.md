# TASK-03 — Base documental y limitaciones de inspección

**Estado:** nota de trazabilidad para revisión humana.  
**Fecha:** 2026-11-15.

## Fuentes consultadas

- `CRL-FEASIBILITY-009A.1-DESIGN(9).md`: revisión R3.1, 2026-11-14; secciones 3.2, 3.11, 3.14, 3.15, 3.17, 3.18, 3.20 y requisitos/aceptación.
- Siete archivos TASK-02-DRAFT-2 publicados en `docs/crl-feasibility/009A.1/` de GitHub main y aprobados por el usuario.
- `Markdown pegado.md`: informe de inspección read-only TASK-01, dictamen VIABLE CON CONDICIONES, con inventario del source set y VD-01..VD-09.
- `CRL-FEASIBILITY-009A-evidence.zip`: evidencia histórica del baseline 009A; utilizado solo para contrastar el esqueleto previo, no como prueba de source exacto 009A.1.
- GitHub `main`, commit `2df2b8e683fface11cd25efd38e95abec3cb205d`, árbol de 36 entradas. Es documental y no contiene `pom.xml`, `src/` ni Java del arnés.

## Estado de evidencia

| Tema | Evidencia disponible | Clasificación |
|---|---|---|
| Diseño 009A.1 | R3.1 en archivo fuente, §§3.11/3.15/3.17/3.18 | Fuente normativa |
| Decisiones TASK-02 | Paquete publicado, encabezados APPROVED y addendum al final | Aprobado por usuario |
| TASK-01 inspección | Workspace `%TEMP%\\crl-feasibility-009a1\\project`, fuentes bajo `src/test/java/lab/crl`; baseline y VD parciales | Inspección estática anterior; debe revalidarse antes de código |
| Código GitHub | No existe en repo público actual | No disponible allí |
| Código ZIP 009A | `RotationCoordinator009A.java`, `DistributedTlsInstance009A.java` y pruebas del baseline 009A | Evidencia histórica, no identidad del source 009A.1 |
| Filesystem Windows | ATOMIC_MOVE/directory force no probados en el destino | No validado; requiere experimento autorizado |
| Builds/tests TASK-03 | Ninguno ejecutado en esta preparación | No ejecutados |

## Advertencias de trazabilidad

- No atribuir al repo `iso8583-gateway` el source code del laboratorio: el repositorio es el lugar de documentación aislada, no el workspace Java 009A.1.
- No usar dependency tree rotulado artifact `crl-feasibility-008` como resolución exacta de 009A.1.
- No asumir que el ZIP histórico 009A coincide byte por byte con la copia 009A.1; hashes del workspace deben capturarse en la siguiente inspección read-only.
- No afirmar soporte real de atomic move, directory force o durabilidad de energía por leer el código/API.
- No reabrir D-01/02/03 ni D-VD03-5; no fijar parámetros TTL/reloj/renewal pendientes.
