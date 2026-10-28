# Análisis de trazabilidad con CloudEvents

## TODO 4: Análisis de Trazabilidad (CloudEvents)

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

Dapr genera el UUID del atributo `id` al envolver el mensaje publicado en un sobre CloudEvents. El código de la aplicación no crea ni modifica este identificador.

El `id` identifica de manera única el evento dentro de su `source`. Es útil para la idempotencia porque el consumidor puede almacenar los identificadores ya procesados y descartar duplicados producidos por reintentos o entrega al menos una vez. Para que funcione, el servicio debe verificar el `id` antes de realizar efectos secundarios y mantener ese registro durante el periodo de auditoría o retención definido.

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

En este laboratorio el atributo `source` debería ser el app ID del publisher:

```text
source: order-publisher
```

Dapr obtiene el valor de la anotación `dapr.io/app-id` de la aplicación que publica. Como el Job está configurado con `dapr.io/app-id: "order-publisher"`, ese valor identifica el origen lógico del evento. El código Python no establece el atributo `source`.

### 3. El atributo `traceparent` y su relación con OpenTelemetry

`traceparent` implementa el estándar W3C Trace Context. Lleva el identificador de la trazabilidad distribuida y el estado de muestreo del contexto actual; por ejemplo, versión, trace ID, span ID y flags.

Dapr propaga ese contexto entre aplicaciones y componentes. OpenTelemetry usa el mismo estándar para representar y correlacionar trazas. Si el publisher inicia o recibe una trazabilidad OpenTelemetry y Dapr la propaga, el subscriber puede continuar la misma traza y registrar su operación como un nuevo span. Así se obtiene una visión completa desde la publicación hasta el procesamiento.

Dapr o la instrumentación de OpenTelemetry deben estar configurados correctamente para que el atributo sea generado o propagado. CloudEvents transporta el contexto, pero no reemplaza el recolector y el backend de trazas de OpenTelemetry.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

Los atributos más útiles para una auditoría de las últimas 24 horas serían:

- `id`: demuestra la identidad única del evento y permite detectar duplicados o reentregas.
- `source`: identifica qué aplicación produjo el evento.
- `type`: permite clasificar el tipo de ocurrencia de negocio, por ejemplo `com.dapr.event.sent` o un tipo más específico definido por el equipo.
- `time`: registra cuándo ocurrió el evento y permite ordenar o consultar por periodos.
- `subject`: puede identificar el recurso de negocio afectado si se usa de forma consistente, como el identificador de la orden.
- `traceparent`: permite correlacionar el evento con trazas y localizar fallos en servicios aguas abajo.
- `datacontenttype`: identifica cómo interpretar el payload.
- `data`: contiene el detalle de negocio necesario para reconstruir el hecho y sus cambios.

Para este laboratorio conviene que los servicios registren los atributos estructurales junto con una referencia al payload y el resultado del procesamiento. La combinación de `id`, `source`, `time`, `type`, `subject`, `traceparent` y `data` permite probar cuándo ocurrió un evento, quién lo produjo, qué entidad afectó, qué aplicación lo procesó y cómo se relacionó con el resto de la operación.
