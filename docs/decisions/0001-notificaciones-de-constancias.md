---
estado: "Propuesta"
fecha: 2026-11-03
---

# Procesamiento asíncrono mediante notificaciones persistentes para el aviso de constancias

## Contexto y problema

ConstanciaUV debe generar y almacenar constancias digitales (R2), mantener su historial (R3) y notificar al estudiante cuando estén disponibles (R4). El envío de correos electrónicos lo realiza un proveedor externo (RT1).

Si dicho proveedor no está disponible, la generación y el almacenamiento de constancias deben continuar (RT2), las notificaciones pendientes deben conservarse para enviarse posteriormente (RT3) y el sistema debe prevenir solicitudes de notificación duplicadas para una misma constancia, sin suponer que el proveedor garantiza una entrega única (RT4).

La decisión consiste en determinar cómo organizar la comunicación entre la generación de constancias y el servicio externo de notificaciones: ¿cuándo se intenta realizar el envío y qué componente es responsable de llevarlo a cabo?

## Factores de decisión

* **D1. Continuidad de la generación ante fallos del proveedor (RT2, R2):** una caída o lentitud del proveedor no debe impedir la generación ni el almacenamiento de las constancias.
* **D2. Conservación de notificaciones pendientes y prevención de duplicados (RT3, RT4, R4):** ninguna notificación debe perderse y una misma constancia no debe generar más de una solicitud de notificación.

## Alternativas consideradas

* **Alternativa A:** comunicación directa con recuperación ante fallos.
* **Alternativa B:** procesamiento asíncrono mediante eventos persistidos.

## Resultado de la decisión

**Opción elegida: Alternativa B: procesamiento asíncrono mediante eventos persistidos**, porque satisface mejor los dos factores de decisión:

* **RT2:** el proceso de generación únicamente realiza operaciones en su propia base de datos y nunca invoca directamente al proveedor externo. Por lo tanto, la disponibilidad de este último no afecta la generación ni el almacenamiento de las constancias.
* **RT3:** cada notificación se registra como pendiente al almacenar la constancia y únicamente cambia de estado después de un envío exitoso. Las notificaciones pendientes forman parte del funcionamiento normal del sistema y no representan una ruta de excepción.
* **RT4:** el registro de notificaciones contará con una restricción de unicidad por constancia y tipo de notificación, además de un identificador de solicitud estable que se enviará al proveedor. De esta manera, el control de duplicados se concentra en un solo punto.

La constancia y su notificación pendiente se registrarán dentro de una misma transacción, de modo que no pueda existir una constancia sin su correspondiente aviso pendiente.

### Consecuencias

* **Positivo:** la generación de constancias queda aislada de los fallos y de los tiempos de respuesta del proveedor externo.
* **Positivo:** las notificaciones pendientes se conservan de manera permanente y pueden reintentarse después de reinicios o caídas del sistema.
* **Positivo:** los cambios en la interfaz del proveedor quedan limitados al componente encargado de enviar las notificaciones.
* **Negativo:** se incorpora un componente independiente que debe desplegarse, supervisarse y mantenerse.
* **Negativo:** la notificación deja de ser inmediata, por lo que el estudiante podría recibirla con cierto retraso.
* **Riesgo:** si el componente encargado del envío falla o se detiene, las notificaciones pendientes pueden acumularse sin que los estudiantes sean avisados. Por ello, se requiere supervisar la cola de notificaciones.
* **Compromiso:** se acepta una mayor complejidad operativa y una menor inmediatez a cambio de una mayor disponibilidad del proceso de generación y una mejor garantía de conservación de las notificaciones.

### Confirmación

En una implementación futura, el cumplimiento de esta decisión podría comprobarse mediante las siguientes pruebas (aún no se han ejecutado):

* **Continuidad ante fallos del proveedor (RT2):** utilizar un proveedor simulado que se encuentre fuera de servicio o presente tiempos de espera agotados. Generar varias constancias y verificar que se almacenen correctamente, aparezcan en el historial y tengan sus notificaciones en estado pendiente.
* **Conservación de notificaciones pendientes (RT3):** reiniciar el componente de envío cuando existan notificaciones pendientes y verificar que estas se procesen nuevamente y se envíen cuando el proveedor restablezca su servicio.
* **Prevención de duplicados (RT4):** intentar registrar dos veces la notificación de una misma constancia y verificar que el segundo registro sea rechazado por la restricción de unicidad. Ejecutar dos instancias simultáneas del componente de envío y comprobar que solo se emita una solicitud por constancia.
* **Revisión del diseño:** verificar en el código fuente que el proceso de generación no contenga invocaciones directas al proveedor externo.

## Ventajas y desventajas de las alternativas

### Alternativa A: comunicación directa con recuperación ante fallos

Después de almacenar la constancia, la aplicación solicita directamente el envío de la notificación. Si el proveedor no responde, registra el envío como pendiente para intentar realizarlo nuevamente más adelante.

* **Positivo:** es una alternativa más sencilla, ya que requiere menos componentes para desarrollar y operar.
* **Positivo:** la notificación puede enviarse casi inmediatamente cuando el proveedor está disponible.
* **Negativo:** el proceso de generación queda vinculado a los tiempos de respuesta y a los fallos del proveedor, lo que pone en riesgo el cumplimiento de RT2.
* **Negativo:** el registro de notificaciones pendientes se convierte en un mecanismo de excepción que podría utilizarse con poca frecuencia. Además, existe el riesgo de perder una notificación pendiente si la aplicación falla entre el intento de envío y su registro (RT3).
* **Negativo:** un tiempo de espera agotado puede generar incertidumbre sobre si el envío se realizó correctamente. Esto podría provocar que se intente enviar nuevamente una notificación que ya fue recibida, por lo que serían necesarios controles adicionales para evitar duplicados (RT4).

### Alternativa B: procesamiento asíncrono mediante eventos persistidos

Después de almacenar la constancia, la aplicación registra una notificación pendiente. Posteriormente, un componente independiente se encarga de procesarla y comunicarse con el proveedor externo.

* **Positivo:** separa el proceso de generación de constancias de los fallos del proveedor externo (RT2).
* **Positivo:** las notificaciones pendientes forman parte del funcionamiento normal del sistema y se conservan de manera duradera (RT3).
* **Positivo:** la prevención de notificaciones duplicadas se controla desde un punto centralizado (RT4).
* **Negativo:** incorpora un componente independiente que debe desarrollarse, operarse y supervisarse.
* **Negativo:** las notificaciones pueden entregarse con cierto retraso respecto al momento en que se genera la constancia.

## Información adicional

Este registro de decisión arquitectónica (ADR) es una propuesta académica y mantiene el estado `proposed`.

Se documenta únicamente la decisión sobre la comunicación con el servicio externo de notificaciones, no la arquitectura completa de ConstanciaUV.

### Revisión 1. Cambio en el proveedor de notificaciones

**Cambio recibido:** el proveedor externo modifica su interfaz de comunicación y cambia la manera en que informa los errores de envío. Este ADR no supone un nuevo formato específico, ya que todavía no se dispone de su documentación técnica.

**Impacto sobre la alternativa seleccionada:** en la Alternativa B, la comunicación con el proveedor está concentrada en el componente independiente encargado del envío de notificaciones.

El cambio afecta principalmente la parte de dicho componente que interpreta las respuestas de error y determina si un envío debe reintentarse o marcarse como fallido.

No afecta la generación ni el almacenamiento de las constancias, el historial ni el registro de notificaciones pendientes, por lo que el cambio puede localizarse en un componente específico.

**Estado de la decisión:** la decisión se mantiene como `proposed` y su justificación original no cambia.

Los argumentos basados en RT2, RT3 y RT4 no dependían del formato utilizado por el proveedor para informar los errores. Sin embargo, la información disponible cambia: aumenta el riesgo de interpretar incorrectamente los errores del proveedor y será necesario evaluar el nuevo contrato de comunicación cuando esté disponible.

**Consecuencias y comprobaciones que deben actualizarse:**

* **Consecuencia a revisar:** una clasificación incorrecta de los errores podría provocar que se descarten notificaciones que deberían reintentarse (afectando RT3), generar reintentos indefinidos o repetir solicitudes cuyo resultado es ambiguo (afectando RT4).

* **Confirmación: comprobaciones que deben agregarse en una implementación futura** (aún no ejecutadas):
  * Realizar pruebas de contrato del adaptador utilizando respuestas de error simuladas con el nuevo formato, verificando que cada error se clasifique correctamente como reintentable o definitivo.
  * Verificar que los errores no reconocidos se consideren reintentables hasta alcanzar un límite de intentos y que posteriormente queden marcados para revisión, en lugar de descartarse.
  * Verificar que los reintentos reutilicen el mismo identificador de solicitud, de manera que un resultado ambiguo no provoque el envío de una segunda notificación.
  * Repetir las pruebas de continuidad de la generación utilizando un proveedor simulado fuera de servicio, para confirmar que el cambio no afecte el cumplimiento de RT2.
