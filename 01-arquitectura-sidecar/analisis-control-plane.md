# analisis-control-plane.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 1.

## TODO 1: Análisis del Plano de Control

### 1. ¿Qué tipo de objeto de Kubernetes es el `dapr-sidecar-injector`?

Partiendo de la salida que nos da los comandos :

kubectl get pods -n dapr-system -o wide

kubectl describe pod -n dapr-system -l app=dapr-sidecar-injector

se observa el **Pod** `dapr-sidecar-injector-76d4cb6cf4-5vpt2`, con un contenedor llamado `dapr-sidecar-injector`, que ejecuta la imagen `ghcr.io/dapr/injector:1.19.0`

El **inyector** y la **`MutatingWebhookConfiguration`** no son el mismo objeto. El **inyector** es el servicio que implementa la lógica de inyección; **`MutatingWebhookConfiguration`** registra ante el API Server de Kubernetes un *admission webhook* que debe invocarse cuando se crean Pods que cumplen sus reglas. Así, el API Server puede enviar la definición del Pod al inyector antes de almacenarla y admitir un parche que agregue el contenedor sidecar de Dapr (`daprd`) y su configuración.

Esta integración es necesaria para la **inyección automática**: sin el webhook, Kubernetes no sabría que debe consultar al inyector.

### 2. Réplicas del `dapr-operator` y comparación con Istio

En el listado aparece **un Pod de `dapr-operator`**: `dapr-operator-7c8bb589b7-fwvt5`, con `READY 1/1`, `STATUS Running` y cero reinicios. Por tanto, hay **una réplica** visible corriendo en el momento de la consulta.

El plano de control de Dapr suele ser más liviano que el de Istio porque Dapr se enfoca en proporcionar funcionalidades específicas para aplicaciones distribuidas, como comunicación entre microservicios, gestión de estados y mensajería.

En cambio, Istio está orientado a administrar la comunicación de red entre servicios, incluyendo el control del tráfico, la seguridad y las políticas de comunicación.


### 3. Flujo de inyección del sidecar

1. **Declaro la aplicación.** Creo un Deployment para mi microservicio (por ejemplo, `pagos-service`) y agrego al **template del Pod**, en `spec.template.metadata.annotations`, `dapr.io/enabled: "true"`. Habitualmente también defino `dapr.io/app-id` y `dapr.io/app-port`, según la aplicación.
2. **Envío el Deployment.** Al ejecutar `kubectl apply -f deployment.yaml`, el API Server recibe y almacena el Deployment. El controlador de Deployment crea o actualiza un ReplicaSet, que solicita crear los Pods correspondientes.
3. **Kubernetes intercepta la creación del Pod.** Durante la fase de admisión de cada Pod, el API Server aplica las reglas de la `MutatingWebhookConfiguration` de Dapr y llama al servicio `dapr-sidecar-injector` si corresponde.
4. **Dapr modifica la especificación.** El inyector detecta `dapr.io/enabled: "true"` y devuelve un patch que agrega el contenedor `daprd` y la configuración necesaria. No modifica el código Java ni la imagen de la aplicación.
5. **Kubernetes crea el Pod modificado.** El scheduler lo asigna a un nodo; el kubelet y el runtime de contenedores descargan las imágenes y arrancan los contenedores de la aplicación y de Dapr.
6. **El sidecar queda operativo.** `daprd` se inicia, obtiene la configuración y la identidad que correspondan y puede ofrecer sus APIs a la aplicación dentro del mismo Pod. Ambos contenedores comparten la red del Pod, por lo que pueden comunicarse mediante `localhost` y los puertos configurados.


## TODO 2: Exploración de los CRDs de Dapr

### 4. CRDs instalados por Dapr

El comando `kubectl get crds | findstr "dapr.io"` nos muestra **siete CRDs**. 

| CRD | Función principal |
|---|---|
| `components.dapr.io` | Permite declarar componentes de integración de Dapr, por ejemplo un almacén de estado Redis o un broker de mensajes. |
| `configurations.dapr.io` | Permite definir ajustes del runtime de Dapr, como trazabilidad y controles de acceso. |
| `httpendpoints.dapr.io` | Permite definir endpoints HTTP externos que Dapr puede utilizar para invocaciones. |
| `mcpservers.dapr.io` | Permite definir recursos de servidor MCP relacionados con la exposición de herramientas mediante Dapr. |
| `resiliencies.dapr.io` | Permite configurar políticas de resiliencia, como reintentos, tiempos de espera y circuit breakers. |
| `subscriptions.dapr.io` | Permite declarar suscripciones a eventos de publicación/suscripción (*Pub/Sub*). |
| `workflowaccesspolicies.dapr.io` | Permite definir políticas de acceso relacionadas con operaciones de Dapr Workflow. |


### 5. Relación entre `components.dapr.io` y el concepto de Building Block

Un Building Block es una capacidad de Dapr que la aplicación consume mediante una API común, como **Pub/Sub**. La aplicación usa esa API sin tener que programar directamente contra cada proveedor.

La CRD `components.dapr.io` permite declarar recursos `Component` que indican qué tecnología concreta utilizará Dapr y cómo conectarse a ella. Por ejemplo, un `Component` de tipo `state.redis` puede configurar Redis como proveedor para el Building Block de State Management.

La abstracción es el Building Block y su API; el Component es la configuración de una implementación concreta.

