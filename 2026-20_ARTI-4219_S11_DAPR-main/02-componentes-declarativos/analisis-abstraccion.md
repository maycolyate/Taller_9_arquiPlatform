# Análisis de la abstracción de Pub/Sub

## TODO 2: Análisis de la Abstracción

### 1. Migración de broker: ¿qué archivos cambiarían?

La migración cambiaría principalmente el componente Pub/Sub y sus metadatos. En este proyecto habría que modificar `02-componentes-declarativos/pubsub-redis.yaml` o sustituirlo por un nuevo componente con `spec.type: pubsub.kafka`, además de configurar brokers, autenticación y las propiedades propias de Kafka.

El publisher y el subscriber continuarían usando la misma API HTTP de Dapr: `/v1.0/publish/order-pubsub/orders` y la suscripción declarativa de la aplicación. Por tanto, no sería necesario cambiar `03-aplicaciones/publisher/app.py` ni `03-aplicaciones/subscriber/app.py` para que funcionen con Kafka, siempre que se conserve el nombre lógico `order-pubsub` y el tópico `orders`.

La infraestructura externa sí cambiaría: habría que desplegar o aprovisionar Kafka y actualizar el pipeline, los certificados o secretos necesarios. Tampoco sería suficiente cambiar el `spec.type` si las propiedades declaradas siguen siendo específicas de Redis; los metadatos deben corresponder al componente Kafka seleccionado.

### 2. Inventario de 5 brokers soportados por Dapr

| Broker | `spec.type` en el YAML |
|---|---|
| Redis | `pubsub.redis` |
| Apache Kafka | `pubsub.kafka` |
| RabbitMQ | `pubsub.rabbitmq` |
| Azure Service Bus | `pubsub.azure.servicebus.topics` |
| AWS SNS/SQS | `pubsub.aws.snssqs` |
| MQTT 3 | `pubsub.mqtt3` |

Para cada implementación cambian `spec.type` y el contenido de `spec.metadata`. Por ejemplo, Redis usa `redisHost`; Kafka usa `brokers`, `consumerGroup` y, según el mecanismo de autenticación, `authType` o certificados; RabbitMQ usa `host`; Azure Service Bus usa cadenas de conexión o identidad; AWS SNS/SQS usa región, credenciales o roles; MQTT usa `url` y posibles credenciales.

El nombre lógico del componente, por ejemplo `order-pubsub`, puede permanecer igual. Así el código de la aplicación continúa sin conocer el broker concreto.

### 3. Manejo seguro de secretos en producción

Dapr permite leer valores sensibles mediante un componente Secret Store. Primero se declara el componente, por ejemplo `secretstores.kubernetes` para secretos de Kubernetes, y después se referencia un secreto usando `secretKeyRef` en lugar de escribir su contraseña directamente:

```yaml
- name: redisPassword
  secretKeyRef:
    name: redis-credentials
    key: password
```

Para habilitar el acceso, también puede indicarse el nombre del Secret Store mediante `secretStoreComponent`:

```yaml
- name: secretStoreComponent
  value: kubernetes-secrets
```

El objeto real se crea sin exponer la contraseña en el repositorio:

```bash
kubectl create secret generic redis-credentials \
  --from-literal=password='<contraseña-segura>'
```

En producción conviene usar gobernanza RBAC, scopes de componentes y Secret Stores administrados por la plataforma, como Azure Key Vault o HashiCorp Vault. El mecanismo de referencia `secretKeyRef` evita que la contraseña viaje en texto plano dentro del repositorio o del YAML aplicado.
