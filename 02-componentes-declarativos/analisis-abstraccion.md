# analisis-abstraccion.md
# Completa este archivo con tus respuestas a los TODOs de la Fase 2.

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

El archivo principal que habría que modificar es 02-componentes-declarativos/pubsub-redis.yaml, o sustituirlo por otro manifiesto para Kafka. En ese `Component` se mantiene `metadata.name: order-pubsub` y se cambia `spec.type: pubsub.redis` por `spec.type: pubsub.kafka`. También se reemplaza la configuración de conexión: en lugar de `redisHost` y `redisPassword`, se configuran los brokers de Kafka y los parámetros de autenticación que correspondan.

Además, habría que actualizar los manifiestos o scripts del repositorio que instalan y configuran Redis, para desplegar o conectarse a Kafka, y ajustar los secretos y configuraciones de infraestructura correspondientes. No se puede afirmar qué otros archivos concretos cambian sin ver el repositorio completo.

**¿Cambiaría el código Python del publisher o del subscriber?** Inicialmente, no, siempre que ambos se comuniquen con Dapr mediante la API Pub/Sub, mantengan el nombre lógico `order-pubsub` y los mismos tópicos y contratos de mensajes. Dapr oculta al código la tecnología del broker. Sí podrían requerirse cambios si el código usa directamente una librería de Redis, depende de características específicas del broker o si cambian los tópicos, formatos o garantías de entrega.


### 2. Inventario de 5 brokers soportados por Dapr

| Broker | Valor de `spec.type` |
|---|---|
| Redis Streams | `pubsub.redis` |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Apache Pulsar | `pubsub.pulsar` |
| Azure Service Bus Topics | `pubsub.azure.servicebus.topics` |

El campo que identifica la implementación es **`spec.type`**. También cambian los valores de **`spec.metadata`**, porque cada broker necesita parámetros de conexión y autenticación distintos. Es posible conservar `metadata.name: order-pubsub` para que el publisher y el subscriber sigan usando el mismo nombre lógico.


### 3. Manejo seguro de secretos en producción

Dapr permite utilizar Secret Stores y referenciar secretos desde un `Component` mediante **`secretKeyRef`**, sin escribir la contraseña en texto plano en el manifiesto. En Kubernetes puede utilizarse el almacén de secretos integrado (`auth.secretStore: kubernetes`, que también es el valor predeterminado), o integraciones con gestores externos de secretos, como Azure Key Vault o HashiCorp Vault.

Ejemplo para el mismo componente del laboratorio, suponiendo que existe un Secret de Kubernetes llamado `redis-credentials` en el namespace `default` con una clave `password`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-pubsub
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: "redis-master.redis.svc.cluster.local:6379"
    - name: redisPassword
      secretKeyRef:
        name: redis-credentials
        key: password
auth:
  secretStore: kubernetes
```