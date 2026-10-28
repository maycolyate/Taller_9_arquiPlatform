# Reflexión arquitectónica

## Reto 3: Reflexión Arquitectónica

### 1. Sin Dapr: ¿dónde vive el código de TTL y consumer group?

Sin Dapr, el TTL dependería del broker o de la configuración utilizada por el SDK. En Redis, el producer o la plataforma podría establecer metadatos o expiración sobre las entradas del stream, por ejemplo de forma externa con scripts o políticas de infraestructura. En Kafka, el TTL se implementaría principalmente mediante *retention* de particiones y el tiempo de retención configurado en el tópico o broker.

El consumer group se configuraría explícitamente en el SDK o en el cliente del broker. En Redis habría que establecer el nombre del grupo al crear o recuperar el stream. En Kafka se configuraría la propiedad `group.id` en el productor o consumidor. Además, ambos casos requerirían código específico para manejar autenticación, reintentos, reconexión y fallos.

La responsabilidad operativa sería compartida, pero recaería principalmente en el equipo de plataforma o broker para configurar tópicos, retención y cuotas; el equipo de aplicación seguiría siendo responsable de usar correctamente las opciones del SDK y de mantener el nombre de cada consumer group. Dapr desplaza gran parte de esa integración al componente YAML y al lado del sidecar, pero no elimina la necesidad de establecer políticas de plataforma y acuerdos con producto.

### 2. El patrón sidecar como frontera de responsabilidad

Producto debe ser propietario del SLA de negocio: cuándo una orden deja de ser válida, cuánto tiempo puede esperar su procesamiento y cuál es el impacto de descartarla. Ese requisito puede expresarse, por ejemplo, como `ttlInSeconds: 120`.

Plataforma debe transformar ese requisito en una configuración segura y mantenible del componente Pub/Sub. También debe seleccionar el broker, administrar componentes, secretos, namespaces, disponibilidad, retención, observabilidad y límites de recursos. El código de negocio no debería conocer Redis, Kafka ni los detalles de implementación.

Una buena división sería:

- Producto: define el SLA, casos límite y el comportamiento de mensajes vencidos.
- Plataforma: establece plantillas de componentes, namespaces, permisos y mecanismos de despliegue.
- Equipo de aplicación: declara el tópico y los contratos CloudEvents, procesa los mensajes y garantiza idempotencia.

El valor TTL puede estar en un componente común o sobrescribirse por entorno. Lo importante es que un requisito de producto no sea codificado de forma invisible en Python, sino documentado y gobernado como configuración de infraestructura que pueda auditarse y desplegarse por ambiente.

### 3. Limitaciones de la abstracción: ¿migración 100% transparente?

Dapr reduce el acoplamiento mediante una API uniforme de Pub/Sub, pero no hace que todas las migraciones sean completamente transparentes. Los componentes no tienen semánticas idénticas.

Un ejemplo concreto es TTL: Redis puede aplicar una entrada `ttlInSeconds` al publicar mensajes, pero Kafka no ofrece el mismo concepto por mensaje mediante el componente y normalmente usa políticas de retención por tópico o partición. Una migración podría modificar el tiempo de conservación observable y requerir soporte de la plataforma antes de que el comportamiento coincidiera.

Los consumer groups también presentan diferencias. Redis Streams usa un grupo mediante `consumerID`, mientras que Kafka usa `group.id` y puede rebalancear particiones, mantener offsets y cambiar con políticas distintas. Dapr puede ofrecer placeholders como `{namespace}`, pero el criterio de particionado y distribución no es necesariamente equivalente.

Otras diferencias pueden incluir entrega y reintentos, orden, tamaño de mensajes, compatibilidad de características, autenticación, observabilidad y tratamiento de errores. Por ello, la API de aplicación puede mantenerse estable, pero el componente y los metadatos cambian. Además, deben verificarse las pruebas de carga, pérdida, entrega duplicada, orden y retención antes de migrar.
