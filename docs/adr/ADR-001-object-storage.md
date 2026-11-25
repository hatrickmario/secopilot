# ADR-002 — Almacenamiento de documentos originales

**Estado** aceptado para el MVP local · **Fecha de decisión** 2026-11-23

## Contexto

SECOPilot analiza los documentos de los procesos de contratación y los archivos que presentan las empresas. Para poder verificar un análisis, el sistema debe conservar los archivos tal como se recibieron y recuperar la versión exacta que sirvió de evidencia, incluso cuando se carguen versiones posteriores.

## Criterios para la decisión del MVP

| Aspecto | Cómo se tuvo en cuenta |
|---|---|
| **Código abierto** | Se buscaron ediciones autoalojables sin costo de licencia. |
| **Portabilidad entre proveedores** | Se valoró la compatibilidad con las operaciones S3 de carga, consulta y descarga que el backend puede reutilizar con `boto3` al cambiar de proveedor. |
| **Rendimiento y consumo de recursos** | Se compararon los tiempos de carga y descarga y el uso de RAM y CPU en el mismo equipo, considerando que el almacenamiento compartirá recursos con los demás componentes del MVP. |
| **Instalación reproducible** | Se valoró levantar el servicio con Docker Compose y se comprobó que los archivos persistieran al recrear el contenedor con el mismo volumen. |
| **Acceso privado** | Fue una condición básica. El piloto verificó que el servicio rechaza lecturas sin credenciales. |
| **Historial de documentos** | Se evaluó el versionado nativo como capacidad adicional del servicio. En Versity se recuperaron dos versiones de un mismo objeto. |
| **Respaldo y recuperación** | Se restauró una copia manual de los archivos de Versity con sus metadatos. |
| **Escalabilidad** | Se midieron varias transferencias simultáneas contra un solo nodo. Un despliegue distribuido requeriría una prueba específica si el proyecto necesita crecer a varios nodos. |

## Opciones candidatas

| Herramienta | Versión evaluada | Licencia | Configuración evaluada |
|---|---|---|---|
| Versity S3 Gateway | `v1.11.0` | Apache 2.0 | Backend POSIX sobre un volumen persistente. |
| SeaweedFS | `4.55` | Apache 2.0 | Servicio S3 y almacenamiento de objetos en un nodo. |
| RustFS | `1.0.6` | Apache 2.0 | Un nodo con un volumen de datos. |
| Garage | `v2.5.0` | AGPLv3 | Un nodo con SQLite y volúmenes separados para datos y metadatos. |
| JuiceFS Community | `v1.4.2` | Apache 2.0 | Gateway S3 con SQLite para metadatos y disco para los bloques de datos. |
| S3Proxy | `4.2.0` | Apache 2.0 | Backend `filesystem` sobre un volumen persistente. |

## Decisión

Usar **Versity S3 Gateway `v1.11.0` con backend POSIX** para guardar los documentos originales del MVP local.

- **Bajo consumo de recursos.** El almacenamiento debe dejar margen de memoria para los demás componentes del MVP que se ejecuten en el mismo equipo, en particular la extracción documental y la inferencia local. Con ocho transferencias simultáneas sobre 522 archivos, Versity registró el menor pico de RAM (**158,40 MiB**, frente a **224,00 MiB** de Garage, el siguiente candidato) y completó la carga, descarga y verificación del lote en **4,27 s**. En el ensayo con PDF escaneados volvió a registrar el menor pico de RAM, con **33,23 MiB**.
- **Código abierto.** Versity tiene licencia Apache 2.0 y se puede autoalojar sin costo de licencia.
- **Compatibilidad con S3.** Se usará `boto3` como cliente del bucket privado. Configurar el endpoint, el bucket, la región y las credenciales permite reutilizar las mismas operaciones al cambiar de proveedor compatible.
- **Almacenamiento en disco.** POSIX estandariza operaciones sobre archivos y directorios. Con este backend, las claves S3 corresponden a rutas de archivos dentro del volumen, lo que facilita inspeccionar el almacenamiento del MVP.
- **Instalación reproducible.** La imagen `ghcr.io/versity/versitygw:v1.11.0` se ejecuta con Docker Compose en un contenedor. El volumen persistente conserva los documentos al recrearlo.

Para migrar los documentos existentes a Amazon S3, se puede usar una herramienta como `rclone copy` para copiar los objetos entre buckets conservando sus claves. Antes de cambiar la configuración de SECOPilot, se verificarían la integridad, los metadatos requeridos y el acceso a los objetos en el destino.

## Resultados de la repetición controlada

Se probaron los seis servicios con los mismos **522 archivos de distintos formatos (229,80 MiB)**, claves generadas y volúmenes temporales por ronda. Se hicieron dos rondas por herramienta con uno, cuatro y ocho archivos simultáneos. Los tiempos son la mediana de subir, descargar y verificar todo el lote. Los picos de RAM, CPU y el recuento máximo de procesos e hilos del contenedor corresponden a las tandas con ocho simultáneas.

| Servicio | 1 a la vez (s) | 4 a la vez (s) | 8 a la vez (s) | RAM pico (MiB) | CPU pico (%) | Procesos e hilos |
|---|---:|---:|---:|---:|---:|---:|
| **Versity** | 16,73 | 5,01 | **4,27** | **158,40** | 177,82 | 20 |
| Garage | 11,21 | 5,22 | 4,61 | 224,00 | 494,33 | 36 |
| SeaweedFS | 11,14 | 5,03 | 4,82 | 893,20 | 402,75 | 21 |
| S3Proxy | 10,16 | 6,25 | 6,37 | 430,80 | 777,37 | 60 |
| RustFS | 24,47 | 14,04 | 16,96 | 375,20 | 134,78 | 61 |
| JuiceFS | 13,77 | 30,89 | 7,68 | 759,30 | 238,23 | 97 |

Los seis servicios recuperaron todos los archivos con el mismo SHA-256 que el original. JuiceFS registró ocho tiempos de espera agotados al borrar objetos en una tanda. El volumen temporal se eliminó al cerrar la ronda. Los resultados describen una prueba en un equipo con Docker Desktop y un nodo por servicio.

## Validación complementaria con PDF escaneados

El 24 de noviembre de 2026 se probaron tres PDF escaneados de 8,36 a 21,22 MiB. Todos los servicios recuperaron los archivos con el mismo SHA-256 y devolvieron el tipo de contenido y los metadatos enviados en la carga. La prueba también confirmó que Versity permite recuperar documentos y su historial en los siguientes casos.

- **Interrupción durante una carga.** Se detuvo el contenedor durante una subida por partes. Al reiniciar, el original previamente guardado seguía intacto y se pudo completar la carga en un nuevo intento.
- **Recuperación de versiones.** Con el versionado nativo activado para el ensayo, Versity conservó dos versiones de la misma clave y permitió recuperar cada una tras recrear el contenedor. Esta función experimental ofrece una forma adicional de recuperar sobrescrituras.
- **Restauración de un respaldo.** Se restauraron las dos versiones y sus metadatos en volúmenes nuevos. La copia incluyó el volumen de objetos y el de versiones, y conservó los atributos extendidos donde Versity guarda sus metadatos.

El cifrado en reposo puede aplicarse al volumen de Versity. Si SECOPilot requiere que el servicio gestione el cifrado por API mediante SSE-S3 o la expiración automática de versiones anteriores, **SeaweedFS y RustFS** son alternativas. Versity POSIX rechazó ambas operaciones en el ensayo. SeaweedFS y RustFS confirmaron el cifrado `AES256` por API y permitieron guardar y consultar la regla de expiración. RustFS requirió configurar una clave maestra de prueba. El acceso mediante S3 facilita el cambio porque el backend puede reutilizar las operaciones de `boto3` al configurar el nuevo proveedor y trasladar los documentos.

## Verificación del servicio de almacenamiento

Los seis candidatos superaron la carga y recuperación de PDF, DOCX y XLSX con `boto3`, el cliente previsto para SECOPilot. Los siguientes resultados corresponden a las comprobaciones del servicio elegido.

| Comprobación | Resultado observado en Versity |
|---|---|
| Cargar, consultar y recuperar PDF, DOCX y XLSX | `put_object`, `head_object` y `get_object` funcionaron con `boto3`. Las copias coincidieron con el tamaño y el SHA-256 de los originales. |
| Eliminar un objeto temporal | `delete_object` eliminó la copia de prueba. Una consulta posterior confirmó su ausencia. |
| Recrear el contenedor con el volumen existente | Los objetos se recuperaron con las mismas claves y huellas SHA-256. |
| Leer un objeto sin credenciales | Versity rechazó el acceso al contenido. |
