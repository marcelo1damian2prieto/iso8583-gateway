# CRL-FEASIBILITY-001 — Informe de viabilidad técnica

**Proyecto:** ISO8583 Gateway v2.0  
**Tipo:** investigación aislada, fuera del repositorio  
**Autorización:** concedida por el usuario  
**Estado:** JSSE probado; integración Reactor Netty/Spring Boot pendiente de prueba ejecutable  
**Repositorio:** sin modificaciones; BOOT-001..BOOT-007 no iniciadas

## 1. Resumen ejecutivo

La investigación demuestra que Java 21 JSSE puede rechazar certificados revocados durante un handshake TLS real, tanto en validación de certificado de cliente (servidor mTLS) como en validación de certificado de servidor (cliente mTLS), cuando se configura un `TrustManagerFactory` PKIX con `PKIXRevocationChecker` en modo CRL estricto.

Se probaron CRL válida, certificado revocado, CRL Distribution Point inaccesible, CRL vencida y CRL malformada. Los escenarios de incertidumbre y revocación fallaron cerrados.

Esto acredita viabilidad técnica de la capa JSSE, pero **no cierra el gate del proyecto**: no se pudo ejecutar un probe con Reactor Netty/Spring Boot en este entorno porque no hay Maven instalado, no hay dependencias en caché y el entorno no resuelve Maven Central. La integración efectiva con el listener y los clientes elegidos debe probarse antes de declarar BOOT-006 completada.

## 2. Entorno de prueba

- OpenJDK Debian `21.0.12.1`.
- OpenSSL `3.5.5`.
- `javac` disponible.
- Maven no instalado.
- Docker no disponible en este entorno de investigación.
- Acceso DNS/HTTP a Maven Central no disponible desde el contenedor.
- Probe ejecutado con `SSLServerSocket` y `SSLSocket`, usando TLS/mTLS y un `TrustManagerFactory` PKIX personalizado.
- CA, certificados, claves y CRL generados exclusivamente para esta prueba; no se utilizaron certificados reales.

## 3. Configuración PKIX probada

El trust manager se construyó mediante:

1. Trust anchor de la CA de laboratorio.
2. `PKIXBuilderParameters`.
3. `PKIXRevocationChecker` obtenido de `CertPathValidator.getInstance("PKIX")`.
4. Opciones `PREFER_CRLS` y `NO_FALLBACK`.
5. Sin opción `SOFT_FAIL`.
6. `CertPathTrustManagerParameters` pasado a `TrustManagerFactory.getInstance("PKIX")`.
7. Trust managers instalados en `SSLContext` y usados durante el handshake.
8. `com.sun.security.enableCRLDP=true` para recuperar CRL desde el CRL Distribution Point del certificado.

La documentación Java indica que `PREFER_CRLS` prioriza CRL y `NO_FALLBACK` deshabilita el fallback. La opción `SOFT_FAIL` permite ignorar ciertos errores de red y no fue configurada.

## 4. Resultados observados

| Caso | Lado verificado | Resultado observado | Esperado | Estado |
|---|---|---|---|---|
| CRL vigente, certificado no revocado | Handshake mTLS | Aceptado | Aceptar | PASS |
| Certificado de cliente revocado | Servidor valida cliente | Rechazado con `certificate_revoked` | Rechazar | PASS |
| Certificado de servidor revocado | Cliente valida servidor | Rechazado con `certificate_revoked` | Rechazar | PASS |
| CRL Distribution Point inaccesible (HTTP 503) | Ambos lados | Rechazado; `Unable to determine revocation status due to network error` | Rechazar | PASS |
| CRL vencida (`nextUpdate` pasado) | Ambos lados | Rechazado; `Could not determine revocation status` | Rechazar | PASS |
| CRL malformada | Ambos lados | Rechazado; error de parseo CRL | Rechazar | PASS |
| CRL no disponible en el almacén y DP inaccesible | Ambos lados | Rechazado | Rechazar | PASS |

Los escenarios cliente y servidor revocados se aislaron mediante CRL distintas: una revocaba solamente el certificado cliente y la otra solamente el certificado servidor. Así se verificaron separadamente las dos direcciones de validación mTLS.

## 5. Evaluación Reactor Netty / Spring Boot

La documentación pública permite identificar un camino de integración plausible:

- Reactor Netty puede utilizar el proveedor JDK o OpenSSL; la documentación indica que OpenSSL se selecciona por defecto si está disponible y que el proveedor puede seleccionarse explícitamente.
- Netty `SslContextBuilder` expone `trustManager(TrustManagerFactory)`, por lo que puede recibir el `TrustManagerFactory` PKIX configurado para CRL.
- Para evitar que una elección implícita de proveedor cambie el comportamiento, la prueba de integración siguiente debe seleccionar explícitamente `SslProvider.JDK` y conectar el `TrustManagerFactory` personalizado tanto al servidor como al cliente.
- Spring Boot SSL Bundles documenta la configuración de trust material JKS/PEM. La documentación consultada no demuestra por sí sola que un bundle estándar active CRL estricto; por lo tanto, no debe considerarse evidencia suficiente sin integrar el trust manager PKIX personalizado.

### Conclusión técnica provisional

**JSSE: viabilidad demostrada.**  
**Netty API: existe un punto de extensión para inyectar el trust manager.**  
**Reactor Netty/Spring Boot real: pendiente de prueba ejecutable.**

No se debe declarar satisfecho el gate CRL hasta ejecutar una prueba de integración contra las versiones exactas elegidas para el proyecto, cubriendo el listener server-side y todos los clientes internos mTLS relevantes.

## 6. Prueba de integración requerida para cerrar el gate

En un entorno con Maven y resolución de dependencias:

1. Crear un proyecto temporal fuera del repositorio con versiones fijadas de Java 21, Spring Boot 3.x, Reactor Netty y Netty.
2. Configurar un `TrustManagerFactory` PKIX con `PKIXRevocationChecker(PREFER_CRLS, NO_FALLBACK)` y sin `SOFT_FAIL`.
3. Construir `SslContext` de Netty seleccionando explícitamente `SslProvider.JDK` e inyectando el trust manager personalizado.
4. Ejecutar un listener TCP de Reactor Netty con `clientAuth(REQUIRE)`.
5. Ejecutar un cliente Reactor Netty con identidad mTLS y trust manager propio.
6. Probar certificado válido, cliente revocado, servidor revocado, CRL inaccesible, CRL vencida, CRL malformada y ausencia de estado de revocación.
7. Confirmar el rechazo durante el handshake, no después de aceptar la conexión a nivel de aplicación.
8. Repetir con las configuraciones efectivas que Spring Boot aplicará al servidor y a los clientes internos.
9. Registrar versiones exactas, proveedor TLS seleccionado, configuración, comandos, logs y resultados.

Si no se puede demostrar que todos los canales usan el trust manager configurado, el gate sigue bloqueado.

## 7. Limitaciones y riesgos pendientes

- No se ejecutó Reactor Netty ni Spring Boot; Maven y dependencias no estaban disponibles en el entorno aislado.
- No se probó proveedor nativo OpenSSL de Netty. La recomendación de prueba inicial es fijar `SslProvider.JDK` para usar el trust manager PKIX de JSSE sin ambigüedad.
- No se probaron rotación de CRL, múltiples puntos de distribución, CRL indirectas, delta CRL ni certificados intermedios con políticas complejas.
- El test usa una CA simple y CRL HTTP de laboratorio; no representa una PKI productiva.
- Se debe definir una política operacional de actualización/caché de CRL sin permitir que una CRL vencida o una descarga fallida resulte en aceptación.
- La validación de hostname/SAN sigue siendo un control separado y también debe estar habilitada en clientes TLS.

## 8. Impacto en el proyecto

- Ningún archivo del repositorio fue modificado.
- No se inició BOOT-001..BOOT-007.
- No se actualizó `docs/TASKS.md`.
- No se generó commit ni push.
- No se implementó alternativa a CRL.
- E2 permanece sin iniciar.

## 9. Decisión recomendada

Continuar con una segunda prueba aislada, ahora con Maven y dependencias de versiones candidatas disponibles, para validar Reactor Netty/Spring Boot con `SslProvider.JDK` y el trust manager PKIX personalizado. Mantener el gate bloqueante hasta contar con esa evidencia.

## 10. Fuentes técnicas

- Oracle, Java SE 21 JSSE Reference Guide: https://docs.oracle.com/en/java/javase/21/security/java-secure-socket-extension-jsse-reference-guide.html
- Oracle, Java SE 21 `PKIXRevocationChecker`: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/security/cert/PKIXRevocationChecker.html
- Oracle, Java SE 21 `PKIXRevocationChecker.Option`: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/security/cert/PKIXRevocationChecker.Option.html
- Netty, `SslContextBuilder`: https://netty.io/4.2/api/io/netty/handler/ssl/SslContextBuilder.html
- Reactor Netty Reference Guide, SSL/TLS: https://docs.spring.io/projectreactor/reactor-netty/docs/current/reference/html/
- Spring Boot Reference, SSL Bundles: https://docs.spring.io/spring-boot/reference/features/ssl.html
