# analisis-cloudevents.md
# Completa este archivo con tus respuestas al TODO 4 de la Fase 3.

## TODO 4: Análisis de Trazabilidad (CloudEvents)

### 1. El atributo `id`: ¿quién lo genera y para qué sirve?

Si el publisher envía datos a la API Pub/Sub de Dapr sin proporcionar un CloudEvent completo, el runtime de Dapr (`daprd`) construye el sobre CloudEvents y genera el identificador `id`. Ahora, si el publicador envía un CloudEvent ya formado, el `id` puede provenir del propio publicador.

El `id` identifica un evento, no necesariamente una orden de negocio.

Es importante para la idempotencia sporque los sistemas de mensajería pueden volver a entregar un evento. El subscriber puede registrar los identificadores ya procesados y, si recibe otra vez el mismo evento, evitar repetir acciones como descontar inventario o generar un cobro.

### 2. El atributo `source`: ¿qué valor tiene y de dónde lo toma Dapr?

En el log observado:

```text
source : order-publisher
```

En el flujo habitual de publicación de datos mediante la API Pub/Sub, Dapr utiliza el app ID de la aplicación publicadora para establecer el atributo `source`. Ese identificador se configura en Kubernetes con la anotación:

```yaml
dapr.io/app-id: "order-publisher"
```

Por eso el valor observado es `order-publisher`. 

### 3. El atributo `traceparent` y su relación con OpenTelemetry

`traceparent` corresponde al estándar **W3C Trace Context**, que define cómo propagar identificadores de trazas entre servicios. Contiene información como el identificador de la traza (trace ID), el identificador del tramo (span ID) y las opciones de muestreo.

OpenTelemetry es un conjunto de APIs, SDK y herramientas para generar, recopilar y exportar telemetría, incluidas las trazas distribuidas.

Por tanto, `traceparent` es un formato estándar de propagación, mientras que OpenTelemetry ayuda a instrumentar y observar el recorrido de una solicitud o evento.

### 4. Valor de negocio para auditoría: atributos más útiles del sobre CloudEvents

| Atributo | Utilidad para auditoría |
|---|---|
| `id` | Identificar cada evento y detectar entregas duplicadas. |
| `source` | Saber qué aplicación produjo el evento. |
| `type` | Distinguir la clase de evento y facilitar búsquedas o agrupaciones. |
| `time` (si se incluye) | Ubicar cuándo ocurrió el evento y filtrar la ventana de 24 horas. |
| `subject` (si se incluye) | Identificar el recurso de negocio al que se refiere el evento. |
| `traceparent` (si está presente) | Correlacionar el evento con otras operaciones de una traza distribuida. |
| `data` | Consultar el contenido de negocio, por ejemplo `orderId` y `status`. |
