# reflexion-arquitectonica.md
# Completa este archivo con tus respuestas al Reto 3 de la Fase 4.

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

**Sin Dapr (SDK de Kafka):** el productor o consumidor implementaría en su aplicación la política de expiración, por ejemplo incluyendo una fecha de vencimiento en el mensaje y verificándola antes de procesarlo. Alternativamente, el equipo de plataforma configuraría la retención del tópico (`retention.ms`), que **no equivale** a un TTL individual de entrega. El `group.id` del consumidor se configuraría en el cliente Kafka, normalmente mediante variables de entorno o configuración de la aplicación.

**Con Dapr:** la aplicación utiliza la API Pub/Sub sin depender directamente del SDK de Kafka o Redis. La configuración de conexión y de consumidores se declara en el `Component` de Dapr. El TTL individual se indica como metadata **al publicar** (`ttlInSeconds`), y Dapr evita entregar mensajes vencidos a sus suscriptores.

**Responsabilidades:** el equipo de producto define las reglas de expiración y el aislamiento funcional de consumidores; el equipo de plataforma administra el broker, la configuración de componentes, los accesos y la observabilidad. Ambos deben acordar cómo aplicar y probar esas reglas.

### 2. El patrón sidecar como frontera de responsabilidad

La regla "una orden deja de ser válida después de 120 segundos" pertenece al equipo de producto, porque expresa una condición de negocio. Este equipo define el SLA, los casos de expiración y las pruebas de aceptación.

El equipo de plataforma selecciona y opera Redis/Kafka, mantiene los componentes YAML, configura los grupos por namespace y garantiza que la infraestructura soporte la política acordada.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

No es una migración sin cambios. El código que invoca la API Pub/Sub de Dapr puede permanecer igual si se conserva el nombre lógico del componente y los tópicos, pero deben ajustarse y validarse la configuración del broker, las credenciales, el aprovisionamiento de tópicos, la retención, los permisos y la operación.

| Aspecto | Redis Streams | Apache Kafka |
|---|---|---|
| Configuración de conexión | `redisHost` y credenciales Redis | `brokers`, autenticación y seguridad Kafka |
| Grupos de consumidores | `consumerID`, admite `{namespace}` | `consumerGroup` y/o `consumerID`; admite `{namespace}` |
| Retención | Políticas propias de Redis Streams | Retención del tópico, por ejemplo `retention.ms` |
| TTL de entrega de Dapr | Metadata por mensaje; Dapr comprueba la expiración | Metadata por mensaje; Dapr comprueba la expiración |

